---
title:  "How to Make an Axel Win32 Build"
date:   2020-10-02 17:06:07 +0800
tags:
  - C
  - axel
categories:
  - 折腾
---

> This post is for those who want to use Axel on windows. I  was on progress with the win32 build of Axel some months ago. But now it is stalled, as I am busy. Anyway, this article ought to be useful for those who want to make a latest build even in the future.

### Preparation

#### Download and compile MinGW

For example, on Ubuntu:
```shell
sudo apt install mingw-w64
```
#### Download and compile OpenSSL against MinGW

[Reference](https://marc.xn--wckerlin-0za.ch/computer/cross-compile-openssl-for-windows-on-linux)

First, download the 1.1.1 release on Github (or the latest, it depends on you).
Before making, 3 lines should be added to the file `include/openssl/x509v3.h`, just after the line of `#define HEADER_X509V3_H`:

```c
#ifdef X509_NAME
#undef X509_NAME
#endif
```

After that, run compile commands:

```shell
cd /path/to/openssl
./Configure mingw shared --cross-compile-prefix=x86_64-w64-mingw32- --prefix=/path/to/installation
make -j4 && make install
```

By default, OpenSSL will be installed to `/usr/local`, and usually there is a necessity to configure the installation path with `--prefix` like `/home/jason/OpenSSL-mingw64`.

### Build Axel

Autotools are required if you are building Axel from the master branch
For more information, please check the [Axel installation guide](https://github.com/axel-download-accelerator/axel#building-from-source).

If you are building from a release tarball, the command could be like this:

```shell
./configure --host=x86_64-w64-mingw32 --with-ssl=/path/to/OpenSSL
make && sudo make install
```

And now, Axel is built and installed to `/usr/local/bin`.
