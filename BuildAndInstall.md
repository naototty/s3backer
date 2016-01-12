### Requirements ###

**s3backer** requires the following additional packages:

| **Name** | **Description** | **Home Page** |
|:---------|:----------------|:--------------|
| libcurl-devel | HTTP library for C | http://curl.haxx.se/ |
| fuse-devel | User-space filesystems | http://fuse.sourceforge.net/ |
| libopenssl-devel | Encryption library | http://www.openssl.org/ |
| zlib-devel | Compression library | http://www.zlib.net/ |
| libexpat-devel | XML parsing library | http://expat.sourceforge.net/ |
| pkg-config | Software library manager | http://pkgconfig.freedesktop.org/ |

On various different systems, these may be already installed and/or have different package names.

#### Requirements On Ubuntu ####

This should be all that's required for Ubuntu:

```
$ sudo apt-get install libcurl4-openssl-dev libfuse-dev libexpat1-dev
```

#### Requirements On Mac OS X ####

Versions of **s3backer** prior to 1.3.1 rely on the [MacPorts](http://www.macports.org/) package infrastructure. First, install MacPorts itself, then use MacPorts to install the other missing requirements:

```
$ sudo port install pkgconfig fuse
```

**s3backer** versions 1.3.1 and later do not require MacPorts. Instead, install MacFUSE using the `.dmg` installer. You can install `pkg-config` any way you like (including using MacPorts) as long as the `./configure` script can find it on your `$PATH`.

Then follow the instructions below for **Building And Installing From Source** except you'll need to run the `./configure` script like this:

```
$ PKG_CONFIG_PATH=/usr/local/lib/pkgconfig ./configure
```

**NOTE:** MacFUSE 2.0.3 has a bug. You should enable "Show beta versions" in the MacFUSE system preferences widget and update to version 2.1.5 or later.

**NOTE:** If you still get errors, edit `/usr/local/lib/pkgconfig/fuse.pc` and change the `-lfuse` to `-lfuse_ino64`, then reconfigure and rebuild:

```
$ PKG_CONFIG_PATH=/usr/local/lib/pkgconfig ./configure
$ perl -p -i -e 's/-lfuse/-lfuse_ino64/g' Makefile
$ make clean && make
$ sudo make install
```

See also [Issue #19](http://code.google.com/p/s3backer/issues/detail?id=19).

### Installing Using Pre-Built RPMs ###

If you are running [openSUSE](http://www.opensuse.org/) or a few other Linux variants, you can find pre-built RPMs on the openSUSE build server [here](http://download.opensuse.org/repositories/home:/archie172/).

### Building And Installing From Source ###

Like lots of other software packages, **s3backer** uses [GNU Autoconf](http://www.gnu.org/software/autoconf/) for its build process so once you have installed the other required packages, building and installing **s3backer** is usually as easy as:

```
$ ./configure
$ make
$ sudo make install
```

### FUSE Configuration ###

If you want to allow normal users to mount **s3backer** filesystems, you need to add the `user_allow_other` option to `/etc/fuse.conf`. It must be on a line by itself.

For example:

```
$ sudo sh -c 'echo user_allow_other >> /etc/fuse.conf'
```