A demonstration ext2 filesystem (for Linux) and disk image (for Mac OS X) have been created that you can mount read-only and view using s3backer.

**No Amazon S3 account is required!**

This mechanism can be used to provide humongous world-readable (read-only) filesystems that can be easily mounted by users the world over. Might be a useful way to publicize all kinds of file-based information.

To see the demo, first build and install **s3backer**. See the BuildAndInstall wiki page for details. Don't forget to
```
$ sudo sh -c 'echo user_allow_other >> /etc/fuse.conf'
```

### Linux Users ###

Run these commands:

```
$ mkdir s3b.mnt demo.mnt
$ s3backer --readOnly s3backer-demo s3b.mnt
s3backer: auto-detecting block size and total file size...
s3backer: auto-detected block size=4k and total size=20m
$ sudo mount -o ro,loop s3b.mnt/file demo.mnt
$ cat demo.mnt/README
```

You will then be able to see the files in the demo filesystem.

When you're done, you can unmount the filesystem via:

```
$ sudo umount demo.mnt && sudo umount s3b.mnt
```

### Mac OS Users ###

Run these commands:

```
$ mkdir s3b.mnt
$ s3backer --filename=demo.dmg --prefix=macos --readOnly s3backer-demo s3b.mnt
s3backer: auto-detecting block size and total file size...
s3backer: auto-detected block size=4k and total size=640k
```

You will then see `demo.dmg` inside the `s3b.mnt` directory, which is a Mac OS disk image you can then mount normally.

Depending on the speed of your Internet connection, it may take several seconds to mount the disk image.

When finished, unmount the disk image by dragging the disk image icon to the trash on your Mac OS X desktop. Then unmount the **s3backer** filesystem via

```
$ umount s3b.mnt
```