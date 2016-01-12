To use **s3backer**, you need a filesystem to play with. This page describes how to create one.

For an example of an existing filesystem, see RunningTheDemo.

To create an encrypted filesystem, read this page first and then see CreatingAnEncryptedFilesystem.

Creating your own new filesystem is very easy. Just follow these simple steps:

  * Download, build, and install **s3backer**.
  * Put your Amazon S3 access ID and access key (as a colon-separated pair) in `~/.s3backer_passwd` like this (the information below is an example only):

```
  # my S3 account accessID and accessKey
  0KAODKRXJM39543K343:+MkIE9MA/dkwEaldRoaPP83dfa03=
```

  * Decide how big you want your filesystem to be. Let's say we want a modest 1 terabyte of storage.
  * Decide what you want your **s3backer** block size to be. See ChoosingBlockSize for more info. You may want to use a larger block size than the default of 4k. Let's use 128k for this example.
  * Decide what bucket you want to store the filesystem blocks in. For this example, we'll use `mybucket`. This is just an example; you will have to create and use your own bucket of course.
  * Create a directory where the **s3backer** filesystem can be mounted: `mkdir ~/mnt-s3b`
  * Fire up s3backer: `s3backer --blockSize=128k --size=1t --listBlocks mybucket ~/mnt-s3b`.
  * Now create your filesystem, pretending that `~/mnt-s3b/file` was a block device such as a disk partition: `mkreiserfs -f -b 4096 -s 513 ~/mnt-s3b/file`. Of course, you don't have to use the Reiser filesystem, you can use ext2, XFS, minix, NTFS, or whatever.
  * Finally, mount your newly-created filesystem using a loopback mount: `sudo mount -o loop ~/mnt-s3b/file /mnt`
  * You now have 1 terabyte of storage mounted under `/mnt` to play with!

**Note**: the number of blocks that have to be written to initialize a filesystem (and therefore the amount of time it takes) depends entirely on the filesystem. Use `--listBlocks` to ensure that if any of those blocks are all zeroes, no network traffic has to occur.

Experience has shown that the ext2 filesystem writes a lots and lots of zero blocks when initializing large filesystems. You might consider reiserfs or xfs instead.

When initializing Reiserfs, you can look for the "Number of blocks consumed by mkreiserfs formatting process: NNNN" message.

Examples of filesystem initialization commands:

  * **Reiser**: `mkreiserfs -f -b 4096 -s 513 ~/mnt-s3b/file`
  * **ext4**: `mke2fs -t ext4 -E nodiscard -F ~/mnt-s3b/file`
  * **XFS**: `mkfs.xfs ~/mnt-s3b/file`

To unmount your filesystem:

  * Unmount the "upper" filesystem: `sudo umount /mnt`
  * Unmount the **s3backer** filesystem: `sudo umount ~/mnt-s3b`