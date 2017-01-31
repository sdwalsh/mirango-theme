+++
title = "ADB and Fastboot Errors"
date = "2016-02-04 08:29:03"
description = "(bootloader) Data size exceeds download buffer"
draft = false
+++

I ran into a couple interesting problems when I flashed a new ROM on a Nexus 6.  One involving adb sideload and another with fastboot (`flash-all.sh` provided by google to quickly flash the stock image)

>Note: Using Nexus 6 with Cyanogen Mod 12 (Lollipop) and Ubuntu 15.10

## Android Debug Bridge Protocol Fault ####

I installed`android-tools-adb` and `android-tools-fastboot` from the official Ubuntu repositories in order to connect to the phone.  I ran `adb devices` to identify the phone and then attempted to sideload an image.

I was met with `error: protocol fault (no status)`.  Uh oh.

```bash
$ adb devices
List of devices attached
<id of phone> sideload
$ adb sideload android.zip
error: protocol fault (no status)
```

Adb version `1.0.32` is required to communicate with Android 5.0 (or greater).  The offical Ubuntu repositories contain version `1.0.31`.

[Nicolas Bernaerts][adb-update] published a guide to updating adb on Ubuntu.  I ran the following commands to update adb.

```bash
$ adb version
Android Debug Bridge version 1.0.31
$ wget -O - https://skia.googlesource.com/skia/+archive/cd048d18e0b81338c1a04b9749a00444597df394/platform_tools/android/bin/linux.tar.gz | tar -zxvf - adb
$ sudo mv adb /usr/bin/adb
$ sudo chmod +x /usr/bin/adb
$ adb version
Android Debug Bridge version 1.0.32
```

## Fastboot Remote Failure (data size exceeds download buffer) ####

Instead of flashing a new Cyanogen Mod image, I decided to flash [Google's factory image][nexus-factory] for the Nexus 6.  Google provides a handy tarball that contains all the files necessary to flash your phone and a handy shell script `flash-all.sh` which should flash all the image files.

![Tarball Contents](/assets/nexus-tarball.png)

When I ran the shell script I was met with a `(bootloader) Data size exceeds download buffer` error.  Uh oh.

```bash
$ ./flash-all.sh
...
sending 'system' (2048781 KB)...
(bootloader) Data size exceeds download buffer
FAILED (remote failure)
finished. total time: 0.940s
```

I extracted the contents of `image-shamu-mmb29q.zip` reveling `boot.img` `cache.img` `recovery.img` `system.img` and `userdata.img`.  I manually flashed each file in the following order while inside the bootloader:

>Running `fastboot flash userdata userdata.img` will wipe your phone

```bash
fastboot flash bootloader bootloader.img
fastboot reboot-bootloader
fastboot flash radio radio.img
fastboot reboot-bootloader
fastboot flash system system.img
fastboot flash userdata userdata.img
fastboot flash boot boot.img
fastboot flash recovery recovery.img
fastboot erase cache
fastboot flash cache cache.img
```

After I flashed each of the image files I booted into recovery and did a `factory data reset` and rebooted.

[nexus-factory]: https://developers.google.com/android/nexus/images
[adb-update]: http://bernaerts.dyndns.org/linux/74-ubuntu/328-ubuntu-trusty-android-adb-fastboot-qtadb
