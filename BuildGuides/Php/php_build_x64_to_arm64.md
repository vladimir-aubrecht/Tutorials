# Build Php (x64 -> arm64, VS2022/VC17)

This guide is based on effort of [Dixyes](https://github.com/dixyes). Original thread is [here](https://github.com/php/php-sdk-binary-tools/pull/1).

This guide does not cover compilation of all typically used extensions neither integration with Apache Httpd.

# Software requirements
- [Visual Studio 2022](https://visualstudio.microsoft.com/vs/community/)
- Folder structure

```bash
C:\Development
   └ php
```
# Building
For all builds use terminal opened by comamnd below (it has necessary environment variables set).
```bash
%comspec% /k "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvarsall.bat x64_arm64"
```

```bash
cd /D C:\Development\php

git clone --single-branch --branch arm64 https://github.com/dixyes/php-sdk-binary-tools

git clone --single-branch --branch win_arm64 https://github.com/dixyes/php-src

call php-sdk-binary-tools/phpsdk-starter.bat -c vc17 -a arm64

cd /D C:\Development\php\php-src

buildconf

configure --disable-all --enable-cli

nmake
```

Binaries are found in generated ``C:\Development\php\php-src\arm64`` folder.