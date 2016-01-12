On Mac OS 10, there is no "loopback" option to the **mount(8)** command. However, Apple provides equivalent functionality using **disk images**, which are simply normal files containing the raw contents of a formatted hard disk.

Note that with Linux, the thing that s3backer is backing when you do a loopback mount is a _filesystem_, whereas with Apple it's a _disk image_. In other words, on Linux you treat the file that s3backer gives you like a partition  on a hard disk, whereas on Apple you treat it like an entire hard disk (which of course will contain within it at least one partition formatted as a filesystem).

However, you can do the "whole disk" thing on Linux as well if you want to. You would use **losetup(8)** to associate the s3backer file with a loopback device. For more information see the **losetup(8)**, **sfdisk(8)**, and **mkfs(8)** man pages.

### Creating Mac OS Virtual Disks ###

On MacOS, you can use the **hditutil(1)** command to attach disk images. So first you start s3backer to create your virtual disk, then run **hdiutil** to attach it:

```
$ mkdir s3b.mnt
$ s3backer --prefix=dmgtest --compress --blockSize=1m --size=100g --filename=dmgtest.dmg --listBlocks mybucket s3b.mnt
s3backer: auto-detecting block size and total file size...
s3backer: auto-detection failed; using configured block size 1m and file size 100g
$ hdiutil attach -nomount s3b.mnt/dmgtest.dmg
```

At this point the disk is attached but no partition(s) are mounted. Initially no partitions will exist.
Next, open the Mac OS Disk Utility to create one or more partition(s) on the disk:

```
$ open '/Applications/Utilities/Disk Utility.app'
```

To create a single partition:

  * Select **dmgest.dmg** disk on the left
  * Select the **Partition** tab
  * Select **1 Partition** for **Volume Scheme**
  * Give a name to your new partition (e.g., **My s3backer Disk**)
  * Select format **Mac OS Extended (Journaled)** (the default)
  * Click **Apply**

Your new partition will be mounted automatically, and you can then start putting files in there.

### Using s3backer with Time Machine ###

Time Machine will not normally allow you to select an s3backer disk image for the backup. Running the following command will cause Time Machine to show volumes normally hidden:

```
$ defaults write com.apple.systempreferences TMShowUnsupportedNetworkVolumes 1
```

However, even though the s3backer volume was listed, when I select it for backup Time Machine just shows the list again.

If someone knows a way around this, please add a comment.

See RunningTheDemo for how to mount the demonstration disk image.