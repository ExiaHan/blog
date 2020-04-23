---
title: "Build SQLITE For Android"
date: 2018-08-01T22:18:20+08:00
categories: ['DevOps']
tags: ['Linux', 'ArchLinux', 'Android', 'Build', 'SQLITE']
---

## Describe

Recently I need to use sqlite3 to quickly modify values in a db file located in an android device.  Usually the easy way is to directly copy an sqlite3 ELF file from an exist AOSP rom's side products of target architecture, but sometimes we may not have any aosp rom built on our disk or even no aosp source files on disk. So it may be not convenience to archive a executable sqlite3 ELF file from a built rom's side products. So it will be  fine to build sqlite3 from sqlite3 source file directly. This article will tell you how to build a usable sqlite3 from sqlite's source file via android-ndk.

**I Assume you are using Linux.**

### Prepare

- ANDROID NDK: Because we are going to build ELF file for android, so you must have [Android-NDK](https://developer.android.com/ndk/) installed. If you don't have it, install it via distribution's package manager\(If you are using Linux\) or directly download from official site.
- The Source File: Then we should download the source file of sqlite. The source file can be downloaded easily from its [Official Site](https://www.sqlite.org/download.html).
- Essential Building Components: Also make sure your system has essential building components installed. If it alerts any missing when building, just install them.

### Build

Now everything is ok. We can start our building action. To avoid any not errors when building. Here I recommend you use the [Standalone ToolChain](https://developer.android.com/ndk/guides/standalone_toolchain).

A standalone toolchain can be easily setup via do what the tutorial says.

```bash
# Create an arm64 API 26 libc++ toolchain.
export NDK={path to ndk}
$NDK/build/tools/make_standalone_toolchain.py \
  --arch arm64 \
  --api 26 \
  --stl=libc++ \
  --install-dir=/{path you want}/{tool chain name}

# Add the standalone toolchain to the search path.
export PATH=$PATH:/{path you want}/{tool chain name}/bin

# Tell configure what tools to use.
# If you want armeabi, just change target_host to
# target_host=arm-linux-androideabi
target_host=aarch64-linux-android
export AR=$target_host-ar
export AS=$target_host-clang
export CC=$target_host-clang
export CXX=$target_host-clang++
export LD=$target_host-ld
export STRIP=$target_host-strip

# Tell configure what flags Android requires.
export CFLAGS="-fPIE -fPIC"
export LDFLAGS="-pie"
```

Now everything is ok. Extract the source file to some place you want, then use configure and make like below shows, the sqlite ELF file will be generated successfully.

```bash
$tar -xzvf sqlite-snapshot-201807272333.tar.gz
$./configure --host=arm-linux-androideabi
$make
...
$file sqlite3
sqlite3: ELF 32-bit LSB pie executable ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /system/bin/linker, with debug_info, not stripped
```

All done.
