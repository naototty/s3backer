This page describes how to create a simple encrypted filesystem using the Linux support for encrypted loopback mounts. See also how to [create a new filesystem](CreatingANewFilesystem.md).

Note: in versions 1.3.0 and later, **s3backer** supports native encryption and authentication. This is much simpler and also more efficient, because it allows **s3backer** to compress the data before encrypting it and also to optimize away zeroed blocks.

You may want to also read the `losetup(8)` man page for details, especially how to create an encrypted filesystem that uses multiple keys instead of a single key.

To create an encrypted filesystem, edit the configuration settings at the top of the following script and then run it; you will be asked for a password (type carefully, you only get one chance):

```
#!/bin/sh

#
# This script creates an encrypted filesystem backed by Amazon S3 using s3backer.
# Note: this will destroy any existing filesystem using the same BUCKET and PREFIX!
#

# Bail on errors
set -e

# Configure me here
BUCKET='myfavoritebucket'
PREFIX=''
SIZE="20g"
S3B_BLOCK_SIZE="4k"
ENCRYPTION="serpent128"
# Choose "ext2", "reiser", or "xfs"
FSTYPE="xfs"
S3B_MOUNT_DIR="s3b.mnt"

# Install kernel modules
if [ -n "${ENCRYPTION}" ]; then
    sudo modprobe cryptoloop >/dev/null 2>&1 || true
    sudo modprobe `echo "${ENCRYPTION}" | sed -r 's/[0-9]//g'` >/dev/null 2>&1 || true
fi

# Mount s3backer filesystem
mkdir -p "${S3B_MOUNT_DIR}"
echo 'starting s3backer...' 1>&2
s3backer \
  --blockSize="${S3B_BLOCK_SIZE}" --size="${SIZE}" \
  --listBlocks --prefix="${PREFIX}" \
  "${BUCKET}" "${S3B_MOUNT_DIR}"

# Find a free loopback device
LOOP_DEVICE=`sudo losetup -f`
if [ "${LOOP_DEVICE}" = "" ]; then
    echo "error: can't find a free loopback device" 1>&2
    exit 1
fi

# Configure loopback for encryption
echo "configuring loopback device ${LOOP_DEVICE}..." 1>&2
sudo losetup -e "${ENCRYPTION}" "${LOOP_DEVICE}" "${S3B_MOUNT_DIR}"/file

# Create filesystem
echo "initializing "${FSTYPE}" filesystem..." 1>&2
case "${FSTYPE}" in
    ext2)
        sudo mke2fs -b 4096 "${LOOP_DEVICE}"
        ;;
    reiser)
        sudo mkreiserfs -f -b 4096 -s 513 -l 's3disk' "${LOOP_DEVICE}"
        ;;
    xfs)
        sudo mkfs.xfs -L 's3disk' "${LOOP_DEVICE}"
        ;;
    *)
        echo "ERROR: unknown filesystem type: ${FSTYPE}" 1>&2
        exit 1
esac


# Done
sync
echo 'stopping s3backer...' 1>&2
sudo losetup -d "${LOOP_DEVICE}"
sudo umount "${S3B_MOUNT_DIR}"
echo "Done! To mount your encrypted filesystem, run these commands:"
if [ -n "${PREFIX}" ]; then
    echo "  \$ s3backer" --prefix="${PREFIX}" "${BUCKET}" "${S3B_MOUNT_DIR}"
else
    echo "  \$ s3backer" "${BUCKET}" "${S3B_MOUNT_DIR}"
fi
if [ -n "${ENCRYPTION}" ]; then
    echo "  \$ sudo mount -o loop,encryption=${ENCRYPTION} ${S3B_MOUNT_DIR}/file /mnt"
else
    echo "  \$ sudo mount -o loop ${S3B_MOUNT_DIR}/file /mnt"
fi

```

Notes:

  * There will be no zeroed blocks with an encrypted filesystem (because when they get encrypted they will be different). So S3 usage and filesystem initialization time (especially with ext2) may be higher.
  * If you worry about the privacy of your data flying over the Internet but you trust Amazon, you might consider the alternative of _not_ encrypting your filesystem but instead encrypting your communication with Amazon. To do this, simply use the `--ssl` flag.