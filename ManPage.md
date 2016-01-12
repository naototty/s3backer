
```
S3BACKER(1)                          BSD General Commands Manual                         S3BACKER(1)

NAME
     s3backer -- FUSE-based single file backing store via Amazon S3

SYNOPSIS
     s3backer [options] bucket /mount/point

     s3backer --test [options] dir /mount/point

     s3backer --erase [options] bucket

     s3backer --reset-mounted-flag [options] bucket

DESCRIPTION
     s3backer is a filesystem that contains a single file backed by the Amazon Simple Storage Ser-
     vice (Amazon S3).  As a filesystem, it is very simple: it provides a single normal file having
     a fixed size.  Underneath, the file is divided up into blocks, and the content of each block is
     stored in a unique Amazon S3 object.  In other words, what s3backer provides is really more
     like an S3-backed virtual hard disk device, rather than a filesystem.

     In typical usage, a `normal' filesystem is mounted on top of the file exported by the s3backer
     filesystem using a loopback mount (or disk image mount on Mac OS X).

     This arrangement has several benefits compared to more complete S3 filesystem implementations:

     o   By not attempting to implement a complete filesystem, which is a complex undertaking and
         difficult to get right, s3backer can stay very lightweight and simple. Only three HTTP
         operations are used: GET, PUT, and DELETE.  All of the experience and knowledge about how
         to properly implement filesystems that already exists can be reused.

     o   By utilizing existing filesystems, you get full UNIX filesystem semantics.  Subtle bugs or
         missing functionality relating to hard links, extended attributes, POSIX locking, etc. are
         avoided.

     o   The gap between normal filesystem semantics and Amazon S3 ``eventual consistency'' is more
         easily and simply solved when one can interpret S3 objects as simple device blocks rather
         than filesystem objects (see below).

     o   When storing your data on Amazon S3 servers, which are not under your control, the ability
         to encrypt and authenticate data becomes a critical issue.  s3backer supports secure
         encryption and authentication.  Alternately, the encryption capability built into the Linux
         loopback device can be used.

     o   Since S3 data is accessed over the network, local caching is also very important for per-
         formance reasons.  Since s3backer presents the equivalent of a virtual hard disk to the
         kernel, most of the filesystem caching can be done where it should be: in the kernel, via
         the kernel's page cache.  However s3backer also includes its own internal block cache for
         increased performance, using asynchronous worker threads to take advantage of the parallel-
         ism inherent in the network.

   Consistency Guarantees
     Amazon S3 makes relatively weak guarantees relating to the timing and consistency of reads vs.
     writes (collectively known as ``eventual consistency'').  s3backer includes logic and configu-
     ration parameters to work around these limitations, allowing the user to guarantee consistency
     to whatever level desired, up to and including 100% detection and avoidance of incorrect data.
     These are:

     1.  s3backer enforces a minimum delay between consecutive PUT or DELETE operations on the same
         block.  This ensures that Amazon S3 doesn't receive these operations out of order.

     2.  s3backer maintains an internal block MD5 checksum cache, which enables automatic detection
         and rejection of `stale' blocks returned by GET operations.

     This logic is configured by the following command line options: --md5CacheSize, --md5CacheTime,
     and --minWriteDelay.

   Zeroed Block Optimization
     As a simple optimization, s3backer does not store blocks containing all zeroes; instead, they
     are simply deleted.  Conversely, reads of non-existent blocks will contain all zeroes.  In
     other words, the backed file is always maximally sparse.

     As a result, blocks do not need to be created before being used and no special initialization
     is necessary when creating a new filesystem.

     When the --listBlocks flag is given, s3backer will list all existing blocks at startup so it
     knows ahead of time exactly which blocks are empty.

   File and Block Size Auto-Detection
     As a convenience, whenever the first block of the backed file is written, s3backer includes as
     meta-data (in the ``x-amz-meta-s3backer-filesize'' header) the total size of the file.  Along
     with the size of the block itself, this value can be checked and/or auto-detected later when
     the filesystem is remounted, eliminating the need for the --blockSize or --size flags to be
     explicitly provided and avoiding accidental mis-interpretation of an existing filesystem.

   Block Cache
     s3backer includes support for an internal block cache to increase performance.  The block cache
     cache is completely separate from the MD5 cache which only stores MD5 checksums transiently and
     whose sole purpose is to mitigate ``eventual consistency''.  The block cache is a traditional
     cache containing cached data blocks.  When full, clean blocks are evicted as necessary in LRU
     order.

     Reads of cached blocks will return immediately with no network traffic.  Writes to the cache
     also return immediately and trigger an asynchronous write operation to the network via a sepa-
     rate worker thread.  Because the kernel typically writes blocks through FUSE filesystems one at
     a time, performing writes asynchronously allows s3backer to take advantage of the parallelism
     inherent in the network, vastly improving write performance.

     The block cache can be configured to store the cached data in a local file instead of in mem-
     ory.  This permits larger cache sizes and allows s3backer to reload cached data after a
     restart.  Reloaded data is verified via MD5 checksum with Amazon S3 before reuse.

     The block cache is configured by the following command line options: --blockCacheFile,
     --blockCacheMaxDirty, --blockCacheNoVerify, --blockCacheSize, --blockCacheSync,
     --blockCacheThreads, --blockCacheTimeout, and --blockCacheWriteDelay.

   Read Ahead
     s3backer implements a simple read-ahead algorithm in the block cache.  When a configurable num-
     ber of blocks are read in order, block cache worker threads are awoken to begin reading subse-
     quent blocks into the block cache.  Read ahead continues as long as the kernel continues read-
     ing blocks sequentially.  The kernel typically requests blocks one at a time, so having multi-
     ple worker threads already reading the next few blocks improves read performance by taking
     advantage of the parallelism inherent in the network.

     Note that the kernel implements a read ahead algorithm as well; its behavior should be taken
     into consideration.  By default, s3backer passes the -o max_readahead=0 option to FUSE.

     Read ahead is configured by the --readAhead and --readAheadTrigger command line options.

   Encryption and Authentication
     s3backer supports encryption via the --encrypt, --password, and --passwordFile flags.  When
     encryption is enabled, SHA1 HMAC authentication is also automatically enabled, and s3backer
     rejects any blocks that are not properly encrypted and signed.

     Encrypting at the s3backer layer is preferable to encrypting at an upper layer (e.g., at the
     loopback device layer), because if the data s3backer sees is already encrypted it can't opti-
     mize away zeroed blocks or do meaningful compression.

   Compression
     s3backer supports block-level compression, which minimizes transfer time and storage costs.

     Compression is configured via the --compress flag.  Compression is automatically enabled when
     encryption is enabled.

   Read-Only Access
     An Amazon S3 account is not required in order to use s3backer.  The filesystem must already
     exist and have S3 objects with ACL's configured for public read access (see --accessType
     below); users should perform the looback mount with the read-only flag (see mount(8)) and pro-
     vide the --readOnly flag to s3backer.  This mode of operation facilitates the creation of pub-
     lic, read-only filesystems.

   Simultaneous Mounts
     Although it functions over the network, the s3backer filesystem is not a distributed filesystem
     and does not support simultaneous read/write mounts.  (This is not something you would normally
     do with a hard-disk partition either.)  As a safety measure, s3backer attempts to detect this
     situation using an 'already mounted' flag in the data store, and will fail to start if it does.

     This detection may produce a false positive if a former s3backer process was not shutdown
     cleanly; if so, the --reset-mounted-flag flag can be used to reset the 'already mounted' flag.
     But see also BUGS below.

   Statistics File
     s3backer populates the filesystem with a human-readable statistics file.  See --statsFilename
     below.

   Logging
     In normal operation s3backer will log via syslog(3).  When run with the -d or -f flags,
     s3backer will log to standard error.

OPTIONS
     Each command line flag has two forms, for example --accessFile=FILE and -o accessFile=FILE.
     Only the first form is shown below.  Either form many be used; both are equivalent.  The second
     form allows mount options to be specified directly in /etc/fstab and passed seamlessly to
     s3backer by FUSE.

     --accessFile=FILE
             Specify a file containing `accessID:accessKey' pairs, one per-line.  Blank lines and
             lines beginning with a `#' are ignored.  If no --accessKey is specified, this file will
             be searched for the entry matching the access ID specified via --accessId; if neither
             --accessKey nor --accessId is specified, the first entry in this file will be used.
             Default value is $HOME/.s3backer_passwd.

     --accessId=ID
             Specify Amazon S3 access ID.  Specify an empty string to force no access ID.  If no
             access ID is specified (and none is found in the access file) then s3backer will still
             function, but only reads of publicly available filesystems will work.

     --accessKey=KEY
             Specify Amazon S3 access key. To avoid publicizing this secret via the command line,
             use --accessFile instead of this flag.

     --accessType=TYPE
             Specify the Amazon S3 access privilege ACL type for newly written blocks.  The value
             must be one of `private', `public-read', `public-read-write', or `authenticated-read'.
             Default is `private'.

     --baseURL=URL
             Specify the base URL, which must end in a forward slash. Default is `http://s3.amazon-
             aws.com/'.

     --blockCacheFile=FILE
             Specify a file in which to store cached data blocks.  Without this flag, the block
             cache lives entirely in process memory and the cached data disappears when s3backer is
             stopped.  The file will be created if it doesn't exist.

             Cache files that have been created by previous invocations of s3backer are reusable as
             long as they were created with the same configured block size (if not, startup will
             fail).  This is true even if s3backer was stopped abruptly, e.g., due to a system
             crash; however, this guarantee rests on the assumption that the filesystem containing
             the cache file will not reorder writes across calls to fsync(2).

             If an existing cache is used but was created with a different size, s3backer will auto-
             matically expand or shrink the file at startup.  When shrinking, blocks that don't fit
             in the new, smaller cache are discarded.  This process also compacts the cache file to
             the extent possible.

             In any case, only clean cache blocks are recoverable after a restart.  This means a
             system crash will cause dirty blocks in the cache to be lost (of course, that is the
             case with an in-memory cache as well).  Use --blockCacheWriteDelay to limit this win-
             dow.

             By default, when having reloaded the cache from a cache file, s3backer will verify the
             MD5 checksum of each reloaded block with Amazon S3 before its first use.  This verify
             operation does not require actually reading the block's data, and therefore is rela-
             tively quick.  This guards against the cached data having unknowingly gotten out of
             sync since the cache file was last used, a situation that is otherwise impossible for
             s3backer to detect.

     --blockCacheMaxDirty=NUM
             Specify a limit on the number of dirty blocks in the block cache.  When this limit is
             reached, subsequent write attempts will block until an existing dirty block is success-
             fully written (and therefore becomes no longer dirty).  This flag limits the amount of
             inconsistency there can be with respect to the underlying S3 data store.

             The default value is zero, which means no limit.

     --blockCacheNoVerify
             Disable the MD5 verification of blocks loaded from a cache file specified via
             --blockCacheFile.  Using this flag is dangerous; use only when you are sure the cached
             file is uncorrupted and the data it contains is up to date.

     --blockCacheSize=SIZE
             Specify the block cache size (in number of blocks).  Each entry in the cache will con-
             sume approximately block size plus 20 bytes.  A value of zero disables the block cache.
             Default value is 1000.

     --blockCacheThreads=NUM
             Set the size of the thread pool associated with the block cache (if enabled).  This
             bounds the number of simultaneous writes that can occur to the network.  Default value
             is 20.

     --blockCacheTimeout=MILLIS
             Specify the maximum time a clean entry can remain in the block cache before it will be
             forcibly evicted and its associated memory freed.  A value of zero means there is no
             timeout; in this case, the number of entries in the block cache will never decrease,
             eventually reaching the maximum size configured by --blockCacheSize and staying there.
             Configure a non-zero value if the memory usage of the block cache is a concern.
             Default value is zero (no timeout).

     --blockCacheWriteDelay=MILLIS
             Specify the maximum time a dirty block can remain in the block cache before it must be
             written out to the network.  Blocks may be written sooner when there is cache pressure.
             A value of zero configures a ``write-through'' policy; greater values configure a
             ``write-back'' policy.  Larger values increase performance when a small number of
             blocks are accessed repeatedly, at the cost of greater inconsistency with the underly-
             ing S3 data store.  Default value is 250 milliseconds.

     --blockCacheSync
             Forces synchronous writes in the block cache layer.  Instead of returning immediately
             and scheduling the actual write to operation happen later, write requests will not
             return until the write has completed.  This flag is a stricter requirement than
             --blockCacheWriteDelay=0, which merely causes the writes to be initiated as soon as
             possible (but still after the write request returns).

             This flag requires --blockCacheWriteDelay to be zero.  Using this flag is likely to
             drastically reduce write performance.

     --blockSize=SIZE
             Specify the block size.  This must be a power of two and should be a multiple of the
             kernel's native page size.  The size may have an optional suffix 'K' for kilobytes, 'M'
             for megabytes, etc.

             s3backer supports partial block operations, though this forces a read before each
             write; use of the block cache and proper alignment of the s3backer block size with the
             intended use (e.g., the block size of the `upper' filesystem) will help minimize the
             extra reads.  Note that even when filesystems are configured for large block sizes, the
             kernel will often still write page-sized blocks.

             s3backer will attempt to auto-detect the block size by reading block number zero at
             startup.  If this option is not specified, the auto-detected value will be used.  If
             this option is specified but disagrees with the auto-detected value, s3backer will exit
             with an error unless --force is also given.  If auto-detection fails because block num-
             ber zero does not exist, and this option is not specified, then the default value of 4K
             (4096) is used.

     --cacert=FILE
             Specify SSL certificate file to be used when verifying the remote server's identity
             when operating over SSL connections.  Equivalent to the --cacert flag documented in
             curl(1).

     --compress[=LEVEL]
             Compress blocks before sending them over the network.  This should result in less net-
             work traffic (in both directions) and lower storage costs.

             The compression level is optional; if given, it must be between 1 (fast compression)
             and 9 (most compression), inclusive.  If omitted, the default compression level is
             used.

             This flag only enables compression of newly written blocks; decompression is always
             enabled and applied when appropriate.  Therefore, it is safe to switch this flag on or
             off between different invocations of s3backer on the same filesystem.

             This flag is automatically enabled when --encrypt is used, though you may also specify
             --compress=LEVEL to set a non-default compression level.

             When using an encrypted upper layer filesystem, this flag adds no value because the
             data will not be compressible.

     --directIO
             Disable kernel caching of the backed file.  This will force the kernel to always pass
             reads and writes directly to s3backer.  This reduces performance but also eliminates
             one source of inconsistency.

     --debug
             Enable logging of debug messages.  Note that this flag is different from -d, which is a
             flag to FUSE; however, the -d FUSE flag implies this flag.

     --debug-http
             Enable printing of HTTP headers to standard output.

     --encrypt[=CIPHER]
             Enable encryption and authentication of block data.  See your OpenSSL documentation for
             a list of supported ciphers; the default if no cipher is specified is AES-128 CBC.

             The encryption password may be supplied via one of --password or --passwordFile.  If
             neither flag is given, s3backer will ask for the password at startup.

             Note: the actual key used is derived by hashing the password, the bucket name, the pre-
             fix name (if any), and the block number.  Therefore, encrypted data cannot be ported to
             different buckets or prefixes.

             This flag implies --compress.

     --erase
             Completely erase the file system by deleting all non-zero blocks, clear the 'already
             mounted' flag, and then exit.  User confirmation is required unless the --force flag is
             also given.  Note, no simultaneous mount detection is performed in this case.

             This option implies --listBlocks.

     --filename=NAME
             Specify the name of the backed file that appears in the s3backer filesystem.  Default
             is `file'.

     --fileMode=MODE
             Specify the UNIX permission bits for the backed file that appears in the s3backer
             filesystem.  Default is 0600, unless --readOnly is specified, in which case the default
             is 0400.

     --force
             Proceed even if the value specified by --blockSize or --size disagrees with the auto-
             detected value, or s3backer detects that another s3backer instance is still mounted on
             top of the same S3 bucket (and prefix).  In any of these cases, proceeding will lead to
             corrupted data, so the --force flag should be avoided for normal use.

             The simultaneous mount detection can produce a false positive when a previous s3backer
             instance was not shut down cleanly.  In this case, don't use --force but rather run
             s3backer once with the --reset-mounted-flag flag.

             If --erase is given, --force causes s3backer to proceed without user confirmation.

     -h --help
             Print a help message and exit.

     --initialRetryPause=MILLIS
             Specify the initial pause time in milliseconds before the first retry attempt after
             failed HTTP operations.  Failures include network failures and timeouts, HTTP errors,
             and reads of stale data (i.e., MD5 mismatch); s3backer will make multiple retry
             attempts using an exponential backoff algorithm, starting with this initial retry pause
             time.  Default value is 200ms.  See also --maxRetryPause.

     --insecure
             Do not verify the remote server's identity when operating over SSL connections.  Equiv-
             alent to the --insecure flag documented in curl(1).

     --keyLength
             Override the length of the generated block encryption key.

             Versions of s3backer prior to 1.3.6 contained a bug where the length of the generated
             encryption key was fixed but system-dependent, causing it to be possibly incompatible
             on different systems for some ciphers.  In version 1.3.6, this bug was corrected; how-
             ever, in some cases this changed the generated key length, making the encryption no
             longer compatible with previously written data.  This flag can be used to force the
             older, fixed key length.  The value you want to use is whatever is defined for
             EVP_MAX_KEY_LENGTH on your system, typically 64.

             It is an error to specify a value smaller than the cipher's natural key length; how-
             ever, a value of zero is allowed and is equivalent to not specifying anything.

     --listBlocks
             Perform a query at startup to determine which blocks already exist.  This enables opti-
             mizations whereby, for each block that does not yet exist, reads return zeroes and
             zeroed writes are omitted, thereby eliminating any network access.  This flag is useful
             when creating a new backed file, or any time it is expected that a large number of
             zeroed blocks will be read or written, such as when initializing a new filesystem.

             This flag will slow down startup in direct proportion to the number of blocks that
             already exist.

     --maxUploadSpeed=BITSPERSEC

     --maxDownloadSpeed=BITSPERSEC
             These flags set a limit on the bandwidth utilized for individual block uploads and
             downloads (i.e., the setting applies on a per-thread basis).  The limits only apply to
             HTTP payload data and do not include any additional overhead from HTTP or TCP headers,
             etc.

             The value is measured in bits per second, and abbreviations like `256k', `1m', etc. may
             be used.  By default, there is no fixed limit.

             Use of these flags may also require setting the --timeout flag to a higher value.

     --maxRetryPause=MILLIS
             Specify the total amount of time in milliseconds s3backer should pause when retrying
             failed HTTP operations before giving up.  Failures include network failures and time-
             outs, HTTP errors, and reads of stale data (i.e., MD5 mismatch); s3backer will make
             multiple retry attempts using an exponential backoff algorithm, up to this maximum
             total retry pause time.  This value does not include the time it takes to perform the
             HTTP operations themselves (use --timeout for that).  Default value is 30000 (30 sec-
             onds).  See also --initialRetryPause.

     --minWriteDelay=MILLIS
             Specify a minimum time in milliseconds between the successful completion of a write and
             the initiation of another write to the same block. This delay ensures that S3 doesn't
             receive the writes out of order.  This value must be set to zero when --md5CacheSize is
             set to zero (MD5 cache disabled).  Default value is 500ms.

     --md5CacheSize=SIZE
             Specify the size of the MD5 checksum cache (in number of blocks).  If the cache is full
             when a new block is written, the write will block until there is room.  Therefore, it
             is important to configure --md5CacheTime and --md5CacheSize according to the frequency
             of writes to the filesystem overall and to the same block repeatedly.  Alternately, a
             value equal to the number of blocks in the filesystem eliminates this problem but con-
             sumes the most memory when full (each entry in the cache is approximately 40 bytes).  A
             value of zero disables the MD5 cache.  Default value is 1000.

     --md5CacheTime=MILLIS
             Specify in milliseconds the time after a block has been successfully written for which
             the MD5 checksum of the block's contents should be cached, for the purpose of detecting
             stale data during subsequent reads.  A value of zero means `infinite' and provides a
             guarantee against reading stale data; however, you should only do this when
             --md5CacheSize is configured to be equal to the number of blocks; otherwise deadlock
             will (eventually) occur.  This value must be at least as big as --minWriteDelay. This
             value must be set to zero when --md5CacheSize is set to zero (MD5 cache disabled).
             Default value is 10 seconds.

             The MD5 checksum cache is not persisted across restarts.  Therefore, to ensure the same
             eventual consistency protection while s3backer is not running, you must delay at least
             --md5CacheTime milliseconds between stopping and restarting s3backer.

     --noAutoDetect
             Disable block and file size auto-detection at startup.  If this flag is given, then the
             block size defaults to 4096 and the --size flag is required.

     --password=PASSWORD
             Supply the password for encryption and authentication as a command-line parameter.

     --passwordFile=FILE
             Read the password for encryption and authentication from (the first line of) the speci-
             fied file.

     --prefix=STRING
             Specify a prefix to prepend to the resource names within bucket that identify each
             block.  By using different prefixes, multiple independent s3backer disks can live in
             the same S3 bucket.

             The default prefix is the empty string.

     --quiet
             Suppress progress output during initial startup.

     --readAhead=NUM
             Configure the number of blocks of read ahead.  This determines how many blocks will be
             read into the block cache ahead of the last block read by the kernel when read ahead is
             active.  This option has no effect if the block cache is disabled.  Default value is 4.

     --readAheadTrigger=NUM
             Configure the number of blocks that must be read consecutively before the read ahead
             algorithm is triggered.  Once triggered, read ahead will continue as long as the kernel
             continues reading blocks sequentially.  This option has no effect if the block cache is
             disabled.  Default value is 2.

     --readOnly
             Assume the filesystem is going to be mounted read-only, and return EROFS in response to
             any attempt to write.  This flag also changes the default mode of the backed file from
             0600 to 0400 and disables the MD5 checksum cache.

     --reset-mounted-flag
             Reset the 'already mounted' flag on the underlying S3 data store.

             s3backer detects simultaneous mounts by checking a special flag.  If a previous invoca-
             tion of s3backer was not shut down cleanly, the flag may not have been cleared.  Run-
             ning s3backer --erase will clear it manually.  But see also BUGS below.

     --rrs   When writing blocks, specify Reduced Redundancy Storage.

     --size=SIZE
             Specify the size (in bytes) of the backed file to be exported by the filesystem.  The
             size may have an optional suffix 'K' for kilobytes, 'M' for megabytes, 'G' for giga-
             bytes, 'T' for terabytes, 'E' for exabytes, 'Z' for zettabytes, or 'Y' for yottabytes.
             s3backer will attempt to auto-detect the block size by reading block number zero.  If
             this option is not specified, the auto-detected value will be used.  If this option is
             specified but disagrees with the auto-detected value, s3backer will exit with an error
             unless --force is also given.

     --ssl   Equivalent to --baseURL https://s3.amazonaws.com/

     --statsFilename=NAME
             Specify the name of the human-readable statistics file that appears in the s3backer
             filesystem.  A value of empty string disables the appearance of this file.  Default is
             `stats'.

     --test  Operate in local test mode.  Filesystem blocks are stored as regular files in the
             directory dir.  No network traffic occurs.

             Note if dir is a relative pathname (and -f is not given) it will be resolved relative
             to the root directory.

     --timeout=SECONDS
             Specify a time limit in seconds for one HTTP operation attempt.  This limits the entire
             operation including connection time (if not already connected) and data transfer time.
             The default is 30 seconds; this value may need to be adjusted upwards to avoid prema-
             ture timeouts on slower links and/or when using a large number of block cache worker
             threads.

             See also --maxRetryPause.

     --version
             Output version and exit.

     --vhost
             Use virtual hosted style requests.  For example, this will cause s3backer to use the
             URL http://mybucket.s3.amazonaws.com/path/uri instead of
             http://s3.amazonaws.com/mybucket/path/uri.

             This flag is required when S3 buckets have been created with location constraints (for
             example `EU buckets').  Put another way, this flag is required for buckets defined out-
             side of the US region.

     In addition, s3backer accepts all of the generic FUSE options as well.  Here is a partial list:

     -o uid=UID
             Override the user ID of the backed file, which defaults to the current user ID.

     -o gid=GID
             Override the group ID of the backed file, which defaults to the current group ID.

     -o sync_read
             Do synchronous reads.

     -o max_readahead=NUM
             Set maximum read-ahead (in bytes).

     -f      Run in the foreground (do not fork).  Causes logging to be sent to standard error.

     -d      Enable FUSE debug mode.  Implies -f.

     -s      Run in single-threaded mode.

     In addition, s3backer passes the following flags which are optimized for s3backer to FUSE
     (unless overridden by the user on the command line):

     -o kernel_cache
     -o fsname=<baseURL><bucket>/<prefix>
     -o subtype=s3backer
     -o use_ino
     -o entry_timeout=31536000
     -o negative_timeout=31536000
     -o max_readahead=0
     -o attr_timeout=0
     -o default_permissions
     -o allow_other
     -o nodev
     -o nosuid

FILES
     $HOME/.s3backer_passwd
             Contains Amazon S3 `accessID:accessKey' pairs.

SEE ALSO
     curl(1), losetup(8), mount(8), umount(8), fusermount(8).

     s3backer: FUSE-based single file backing store via Amazon S3, http://s3backer.googlecode.com/.

     Amazon Simple Storage Service (Amazon S3), http://aws.amazon.com/s3.

     FUSE: Filesystem in Userspace, http://fuse.sourceforge.net/.

     MacFUSE: A User-Space File System Implementation Mechanism for Mac OS X,
     http://code.google.com/p/macfuse/.

     Google Search for `linux page cache', http://www.google.com/search?q=linux+page+cache.

BUGS
     Due to a design flaw in FUSE, an unmount of the s3backer filesystem will complete successfully
     before s3backer has finished writing back all dirty blocks.  Therefore, when using the block
     cache, attempts to remount the same bucket and prefix may fail with an 'already mounted' error
     while the former s3backer process finishes flusing its cache.  Before assuming a false positive
     and using --reset-mounted-flag, ensure that any previous s3backer process attached to the same
     bucket and prefix has exited.  See issue #40 for details.

     For cache space efficiency, s3backer uses 32 bit values to index individual blocks.  Therefore,
     the block size must be increased beyond the default 4K when very large filesystems (greater
     than 16 terabytes) are created.

     s3backer should really be implemented as a device rather than a filesystem.  However, this
     would require writing a kernel module instead of a simple user-space daemon, because Linux does
     not provide a user-space API for devices like it does for filesystems with FUSE.  Implementing
     s3backer as a filesystem and then using the loopback mount is a simple workaround.

     On Mac OS X, the kernel imposes its own timeout (600 seconds) on FUSE operations, and automati-
     cally unmounts the filesystem when this limit is reached.  This can happen when a combination
     of --maxRetryPause and/or --timeout settings allow HTTP retries to take longer than this value.
     A warning is emitted on startup in this case.

     Filesystem size is limited by the maximum allowable size of a single file.

     The default block size of 4k is non-optimal from a compression and cost perspective.  Typi-
     cally, users will want a larger value to maximize compression and minimize transaction costs,
     e.g., 1m.

AUTHOR
     Archie L. Cobbs <archie@dellroad.org>

BSD                                       September 7, 2009                                      BSD
```