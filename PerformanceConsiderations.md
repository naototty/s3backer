### Overview ###

To understand **s3backer** performance, it helps to understand its overall design as well as how the kernel and FUSE interact with it.

FUSE is a user-land library that talks to the kernel on one end and dispatches the kernel's read and write requests to **s3backer** on the other end, using one or more FUSE worker threads. AFAICT, the kernel makes no attempt to increase performance by doing reads and writes in parallel. In other words, if you have one application running performing reads and writes via a single thread, then **s3backer** will see those reads and writes in a single thread as well. This kindof makes sense when you remember that disk drives (the technology that underlies traditional filesystems) can more or less only do one thing at a time anyway. The kernel will however try to read ahead when it predicts that the application is reading data sequentially.

**s3backer**'s job is to chop up these file read and write requests into block read and write requests and send them to Amazon S3. **s3backer** consists of four "layers" stacked on top of each other in this order (top to bottom):

  1. **FUSE interface:** This layer talks to FUSE to create the filesytem, and passes any read or write requests to the next layer down.
  1. **Block Cache:** The block cache keeps a local cache of data blocks to increase performance. It also has an associated pool of worker threads that perform reads and writes asynchronously.
  1. **MD5 Cache:** This layer's job is to protect against reads of stale data from Amazon S3 due to "eventual consistency." Whenever a block is written, its MD5 checksum is cached for a short time so we can verify the right data is returned by any subsequent reads. In addition, this layer prevents two writes of the same block from occurring too closely together, which could cause them to be processed in the wrong order by Amazon S3. This layer also handles the zero block optimization, whereby blocks containing all zeroes are deleted instead of being written.
  1. **HTTP I/O:** This layer speaks the HTTP protocol to Amazon S3 using Amazon's REST API.

Note that when the `--test` flag is used, **s3backer** replaces the bottom HTTP I/O layer with an equivalent **Test I/O** layer that simply reads and writes blocks from a local directory.

When **s3backer** receives a read request from FUSE, it first checks the block cache. If the block is found, we return it immediately. Otherwise, the FUSE worker thread making the request first checks whether this read operation should trigger additional read ahead. If so, a block cache worker thread is woken up to perform the read ahead in parallel. Then the original thread performs the read of the block by invoking the next layer down, adds the result to the block cache, and returns it.

When **s3backer** receives a write request from FUSE, the block is added to the block cache as a dirty block (if not already there), a worker thread is awoken to handle the request, and the original thread returns immediately. If the block cache is full (meaning there are no clean blocks which could be evicted), the original thread will block until space is available.

In the MD5 cache layer, reads pass right through, except that they are annotated with the expected MD5 checksum (if the block is in the MD5 cache). As a result, reads of stale data from Amazon S3 will be rejected and retried in the HTTP I/O layer. Writes cause an entry to be added to the MD5 cache for the corresponding block; if the block is in the MD5 cache already and `--minWriteDelay` milliseconds have not passed since the previous write, the thread will block until they have before passing the write request on to the HTTP I/O layer.

The HTTP I/O layer is straightforward, converting read and write requests into GET, PUT, and DELETE HTTP requests. This is also the layer where compression of blocks occurs, if enabled (decompression always occurs when necessary).

### Caching ###

When you consider the whole picture including both the kernel, FUSE, and **s3backer**, there are many levels of caching involved.

_Note:_ I am an expert on neither FUSE nor the Linux kernel; some of this information may be incorrect. If so please let me know!

This discussion assumes we're talking about Linux.

**Kernel Caches**

Linux has several caches, including:

  * The buffer cache, which caches raw data blocks from block devices (such as hard disks)
  * The page cache, which caches blocks of data contained within logical UNIX files
  * The dentry/inode cache, which caches file and directory meta-data associated with filesystems
  * The swap cache, which caches pages that have been written out to swap, in hopes that the next time such a page needs to be swapped out there won't have been any modifications to it, thereby eliminating the need to write it to disk again

A good tutorial I've found is [here](http://www.linux-tutorial.info/modules.php?name=MContent&pageid=260).

The dentry/inode cache is not important to **s3backer**, because there are only two files in the whole filesystem.

The page cache however is: by virtue of the fact that the loopback device has the underlying file open, the contents of that file will be cached in the page cache just like any other file. `losetup(8)` does not open the underlying file in direct I/O mode (i.e., with the `O_DIRECT` flag), so loopbacked files' contents can be cached (here's [a patch](http://article.gmane.org/gmane.linux.utilities.util-linux-ng/1731) to add that ability). But even if it did, FUSE does not currenly support opening a FUSE filesystem file with `O_DIRECT`. Instead, use the `direct_io` flag, which does work and has the same effect, if you want to eliminate caching in the page cache.

As far as I can tell, since the loopback device is a block device, it does use the buffer cache _NEED VERIFICATION_. This means some data is cached twice, aka the "double caching" problem.

Regarding the swap cache, Linux only writes out pages to swap if those pages cannot be stored or retrieved in another way. In particular, pages that represent file data blocks need not be written to swap, because they can instead be written back to the file they belong to (if dirty) or simply discarded and read again later (if clean).

So in summary, the page and buffer caches are the only kernel caches that affect **s3backer** in normal usage.

_Question:_ If you run `fsync(2)` on a file in a filesystem mounted on top of the **s3backer** backed file via loopback, does the data really get synced all the way to Amazon?

_Answer:_ No. The filesystem will flush the file data to the loopback device. This will cause the loopbacked file's contents to be updated, but they can still just sit in the page cache. An additional `sync(2)` call may be necessary to force the loopback file's pages to be flushed to the FUSE filesystem. This will send them to **s3backer**, where there will be some additional delay before they reach Amazon S3 (depends on configuration).

_Question:_ Should the "upper" filesystem be mounted with the `sync` flag when using **s3backer**?

_Answer:_ Probably. This will minimize the caching of data in the kernel prior to the page cache.

**s3backer Block Cache**

The **s3backer** block cache has a huge impact on the performance of **s3backer**. There are a couple of reasons for this:

  * HTTP operations are an order of magnitude slower (or more) than the analogous hard disk operations, so caching is that much more important.
  * The kernel issues I/O requests more or less sequentially, but you can send as many HTTP requests in parallel as you want, up to the point where your Internet connection and/or Amazon S3 is saturated.

**s3backer**'s cache is a traditional cache of blocks, where entries in the cache are either clean or dirty. Actually there are several states:

  * `CLEAN`: The block is clean and will be evicted if the cache is full and space is needed, or if it times out (due to a non-zero `--blockCacheTimeout` setting).
  * `READING`: The block is currently being read; additional attempts to read the block will wait for the first one to complete. This state helps avoid the silliness of two threads reading the same block at the same time.
  * `DIRTY`: The block is modified and will immediately (if `--blockCacheWriteDelay` is zero) or eventually (if `--blockCacheWriteDelay` is non-zero) be written back.
  * `WRITING`: The block is currently being written back. When the write completes, it becomes `CLEAN`.
  * `WRITING2`: The block is currently being written back, but was also modified yet again after the write back operation started. When the write completes, it stays `DIRTY`.

The writing back of dirty blocks is performed asynchronously by a pool of worker threads. This allows the kernel to issue several write requests in rapid succession and have these requests be handled in parallel.

The worker threads also perform read ahead, again in parallel with the kernel's normal one-at-a-time reading of blocks.

The block cache lives in **s3backer**'s normal heap memory, and so is eligible for being swapped out to disk. Obviously, it can't be greater than 4GB on 32 bit machines, but on 64 bit machines it can potentially be huge (as long as you have enough memory and/or swap to back it).

Note: in version 1.3.0, **s3backer** adds support for storing the block cache in a local disk file. This allows **s3backer** to retain its cache across restarts. See the ManPage for details.

**s3backer MD5 Cache**

The MD5 cache is not really a data cache. It only caches the MD5 checksums of data blocks that have recently been written to Amazon S3, and only for long enough to be sure that subsequent reads of the same block will not return stale contents.

The only performance impact this cache can have is negative, due to one of the following situations:

  * _An attempt is made to write a block not already in the MD5 cache, and the cache is full_. In this case the write will wait until some entry in the MD5 cache times out. However, you won't notice this directly because writes in **s3backer** occur asychronously. You could notice it indirectly because it means one more worker thread is going to be temporarily unavailable.
  * _An attempt is made to write the same block twice in a row in rapid succession_. In this case we need to make sure Amazon S3 doesn't process these writes out of order, and so we enforce the second write to wait until a minimum separation time has passed (configured via `--minWriteDelay`).

Blocks only live in the MD5 cache as long as necessary (`--md5CacheTime` milliseconds). So whether the first situation can occur, and how likely, is affected by the `--md5CacheSize` and `--md5CacheTime` settings.

### Space Considerations ###

Only non-zero blocks that have been written to Amazon S3 will actually exist (and cost you money). So even if you create a huge filesystem, you will only pay in proportion to the amount of real data contained within it.

Well, that's true to a first approximation. In reality, a couple of factors can cause you to pay for storage when you don't need to. First, when you delete files in a filesystem, the filesystem does not zero out the no-longer used blocks. Instead, it just unreferences them and leaves them as they are.

But wait! Thanks to the inevitable march of progress in the Linux kernel, this problem can now be solved neatly with the help of `fallocate()` and **s3backer** >= 1.3.4. For more information including important filesystem and version requirements, see the UnsedBlockDeletion wiki page.

If you can't take advantage of `fallocate`, there are tools for various filesystems that allow you to zero out these unused blocks. For example, the [zerofree](http://intgat.tigress.co.uk/rmy/uml/index.html) utility works with ext2/ext3 filesystems. For more information about ext2 block allocation, see [this link](http://www.linux-security.cn/ebooks/ulk3-html/0596005652/understandlk-CHP-18-SECT-6.html).

Another problem when not using `fallocate` is that if you have an encrypted filesystem, data that is zeroes when unencrypted will not be zeroes when encrypted. In fact you will have no zero blocks at all. This will only matter for files that contain zero blocks but are not stored sparsely on the disk (which is really a separate problem). You can correct these files easily by copying them via `cp --sparse=always` and then replacing the original with the copy.

Note: in version 1.3.0, **s3backer** adds support for native encryption and authentication. This avoids the aforementioned problem.

**Choosing the Right Block Size**

In practice you should choose a block size at least as big as the kernel's native page size (4096 bytes for most systems), because this is the size of the chunks the kernel will be reading and writing. A value of 4096 will however result in a huge number of individual Amazon S3 objects for large filesystems, plus it has about a 9% overhead at the HTTP level.

Larger block sizes are fine, although because of the mismatch between the kernel and filesystem block sizes they are less efficient. This inefficiency comes from the fact that for **s3backer** to handle a read or a write that is less than an entire block, it must first read the whole block, change a portion of it, and then write it back. So there is not only an extra read in there, but the read and the write may be transferring more data than is actually changing. Also, larger block sizes are more cost efficient; a reasonable choice might be 1M.

Because of this, when using large block sizes proper use and configuration of the block cache is important. A generous block cache will allow many operations to happen in memory, avoiding network I/O altogether. In addition, it is important to configure a non-zero value for `--blockCacheWriteDelay`, so that if a block in the cache is modified several times in rapid succession (as would be the case when the smaller kernel-sized blocks are written out sequentially), these changes are coalesced into a single HTTP PUT request.

Note that it's not necessarily the case that you will have more S3 objects than files in your filesystem. This ratio depends on how the "upper" filesystem stores its data. For example, the Reiser filesystem can store several small files into a single block. So your filesystem could actually result in fewer Amazon S3 objects than files it contains.

Another thing to remember: larger block sizes take longer to transfer. Depending on the speed of your Internet connection, you may need to specify a longer `--timeout` than the default, which is 30 seconds. This is especially true if all of the block cache threads reading or writing at the same time would oversaturate your connection, which will cause them all to transfer more slowly.

See also ChoosingBlockSize.

**Choosing the Right Number Of Block Cache Threads**

Ideally, the correct number of block cache threads should be determined empirically. At some point I hope to write a utility program that performs tests for the best setting.

But generally speaking, the faster your Internet connection, the more threads you'll want; the larger your block size, the fewer threads you'll want. The default value of 20 is just a wild guess.

**Compression**

Unless you have a good reason not to, you probably want to enable compression, as it will reduce network transfer time and Amazon S3 charges. One good reason not to, however, is if you are using an encrypted file system on top of the backed file. In this case, every block written will appear to contain random data and therefore won't compress at all. In versions 1.3.0 and later, you can use the built-in encryption (which automatically enables compression) to avoid this problem.

### Tips & Tricks ###

**Disconnected Operation**

Because the network connection is HTTP-based and stateless, **s3backer** does not need a continuous connection to Amazon S3. For example, you might have a laptop on which is mounted an **s3backer** filesystem. Travelling back and forth between home and work should pose no problem; each time you reconnect to the network you magically "reconnect" to Amazon S3.

If the kernel tries to read a block while disconnected, and the block is not in the block cache, the read will return an error. On the other hand, kernel writes always succeed even when disconnected, though a worker thread may be stuck repeatedly trying to send the block over the network (it will retry indefinitely until successful). If your block cache size is too small, however, it will fill up with dirty blocks and attempts to write additional blocks will, well, block.

For disconnected operation support without the possibility of the kernel seeing any errors, you must configure the block cache to be large enough to contain all the blocks in the filesystem and then load them all into the cache (e.g., using a command like `dd if=s3b.mnt/file of=/dev/null bs=4096`) . If this is not feasible, you can still get by as long as (a) the files you need to access are pre-cached, and (b) the block cache is big enough to contain all the blocks that are written while disconnected.

In addition, using the on-disk block cache support added in versions 1.3.0 and later allows you to shutdown and restart without losing your cached blocks.

**Changing Block Size**

Once you create a new **s3backer**-backed file configured to use a specific block size, you have committed yourself to that block size. If you remount the backed file with a different block size (doing so requires you to use the `--force` flag), all hell will break loose.

However, there is a way to change block size of an existing backed file without changing its contents: simply run **s3backer** twice, with the first instance pointing at the backed file with the old block size, and the second instance pointing at a new (empty) backed file having a different bucket name and/or prefix, and then copy the contents of the first file into the second. If you want to use the original bucket name and prefix, you'll have to copy it back again the same way.