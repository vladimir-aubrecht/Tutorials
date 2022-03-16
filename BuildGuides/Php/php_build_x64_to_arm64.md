# Build Php (x64 -> arm64, VS2022/VC17)

This guide is based on effort of [Dixyes](https://github.com/dixyes). Original thread is [here](https://github.com/php/php-sdk-binary-tools/pull/1).

This guide does not cover compilation of all typically used extensions neither integration with Apache Httpd.

# Software requirements
- [Visual Studio 2022](https://visualstudio.microsoft.com/vs/community/)
- Folder structure

```cmd
C:\Development
   └ php
```
# Building
For all builds use terminal opened by comamnd below (it has necessary environment variables set).
```cmd
%comspec% /k "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvarsall.bat x64_arm64"
```

# Without OpenSSL, Phar and OpCache
```cmd
cd /D C:\Development\php

git clone --single-branch --branch arm64 https://github.com/dixyes/php-sdk-binary-tools

git clone --single-branch --branch win_arm64 https://github.com/dixyes/php-src

call php-sdk-binary-tools/phpsdk-starter.bat -c vc17 -a arm64

cd /D C:\Development\php\php-src

buildconf

configure --enable-cli --disable-phar --disable-opcache

nmake
```

Binaries are found in generated ``C:\Development\php\php-src\arm64`` folder.

# With OpenSSL and Phar

Phar requires OpenSSL extension which requires OpenSSL itself.

Start by building OpenSSL by checking [this guide](./php_build_x64_to_arm64.md).
Following steps are using same paths as guide on the link and assumes you prepared build and source codes of OpenSSL.

```cmd
cd /D C:\Development\php

git clone --single-branch --branch arm64 https://github.com/dixyes/php-sdk-binary-tools

git clone --single-branch --branch win_arm64 https://github.com/dixyes/php-src

call php-sdk-binary-tools/phpsdk-starter.bat -c vc17 -a arm64

cd /D C:\Development\php\php-src

buildconf

mkdir openssl

copy C:\Development\Apache24\src\openssl\ms\applink.c C:\Development\php\php-src\openssl\applink.c

configure --enable-cli --disable-opcache --with-openssl=C:\Development\Apache24\src\openssl --with-extra-includes=C:\Development\Apache24\src\openssl\include

copy C:\Development\Apache24\src\openssl\libcrypto-1_1-arm64.dll C:\Development\php\php-src\arm64\Release_TS\libcrypto-1_1-arm64.dll

copy C:\Development\Apache24\src\openssl\libssl-1_1-arm64.dll C:\Development\php\php-src\arm64\Release_TS\libssl-1_1-arm64.dll

nmake
```

Binaries are found in generated ``C:\Development\php\php-src\arm64`` folder.