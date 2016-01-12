## What's the Problem? ##

A common question that comes up with **s3backer** is: suppose you have a normal "upper" filesystem on top of **s3backer** via the usual loopback mount, and you have created some huge file in that upper layer filesystem. Now suppose sometime later you delete that huge file. Why is Amazon still charging for storing all that data?

When filesystems "delete" a file, they don't actually zero out the block data; instead, they simply mark those blocks as unused in their meta-data structure on disk. So although the filesystem will know that these blocks as unused and available, **s3backer**, not knowing anything at all about what the upper layer filesystem is doing, will think they are still in use. So you will continue paying Amazon for them whether they are holding actual file content or not.

Because it is "upper layer agnostic", the only time **s3backer** knows a block is not used is when you write all zeroes to it. So one way to clean up your filesystem is to use a utility like `zerofree`. However, this requires unmounting the upper filesystem and manually running the utility. It can also take a long time.

## What's the Solution? ##

The ideal solution would be for the upper layer filesystem to somehow communicate to **s3backer** which blocks it no longer cares about. However, for this to work, the communication has to go through several layers:
  1. From the upper layer filesystem to it's block device (which in this case is really a loopback mount)
  1. From the loopback "block device" to the underlying filesystem file on which it is mounted (in this case **s3backer**'s `file`).
  1. From the underlying filesystem file to the filesystem containing it (in this case FUSE)
  1. From the FUSE filesystem to the **s3backer** process

Fortunately, **this is now possible in Linux** as all the links in the chain exist. However, you have to have new enough versions of everything.

## Requirements ##

Here is what is required to get this working. The requirements below correspond to the links in the chain listed above:
  1. The "upper" filesystem must support the `TRIM` block device operation. Examples include Ext4, Btrfs (Linux >= 3.7), and XFS. For Ext4, you must also mount with the `discard` mount option.
  1. Loopback device support for `TRIM` requires Linux >= 3.2.
  1. FUSE filesystem support for `FALLOC_FL_PUNCH_HOLE` requires Linux >= 3.5
  1. FUSE API support for `fallocate()` requires FUSE >= 2.9.2 and **s3backer** >= 1.3.4.

So the bottom line is:
  * **s3backer** >= 1.3.4
  * FUSE >= 2.9.2
  * Linux kernel >= 3.5
  * A supporting "upper layer" filesystem (mounted with `-o discard` if ext4).

## What if it still doesn't work? ##

Check the following things:
  * Running `uname -r` should show kernel version 3.5.0 or higher
  * When you `./configure` **s3backer** you should see this message: `checking for fallocate() support in fuse... yes`
  * Running `fallocate -np -l 1024 testfile` inside a directory in your "upper layer" filesystem should not generate an error