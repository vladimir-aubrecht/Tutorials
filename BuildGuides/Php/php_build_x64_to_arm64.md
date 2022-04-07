# Build Php (x64 -> arm64, VS2022/VC17)

This guide is based on effort of [Dixyes](https://github.com/dixyes). Original thread is [here](https://github.com/php/php-sdk-binary-tools/pull/1).

This guide does not cover compilation of all typically used extensions neither integration with Apache Httpd.

# Software requirements
- [Visual Studio 2022](https://visualstudio.microsoft.com/vs/community/)
- Python 2.7 (I'll assume location C:\Python27)
- [Vcpkg](https://github.com/vladimir-aubrecht/vcpkg) - It's necessary to use this repo for now as not all necessary ports are merged yet.
- Folder structure

```cmd
C:\Development
   └ php
   └ dependencies
   └ vcpkg
   └ libxml2
```

# Expected result of this guide

After build you should get binaries with following configuration:

```text
Enabled extensions:
-----------------------
| Extension  | Mode   |
-----------------------
| bcmath     | static |
| bz2        | static |
| calendar   | static |
| com_dotnet | static |
| ctype      | static |
| curl       | static |
| date       | static |
| dom        | static |
| filter     | static |
| hash       | static |
| iconv      | static |
| json       | static |
| libxml     | static |
| mysqli     | static |
| mysqlnd    | static |
| openssl    | shared |
| pcre       | static |
| pgsql      | static |
| phar       | static |
| reflection | static |
| session    | static |
| simplexml  | static |
| soap       | static |
| spl        | static |
| standard   | static |
| tokenizer  | static |
| xml        | static |
| xmlreader  | static |
| xmlwriter  | static |
| zip        | static |
| zlib       | static |
-----------------------


Enabled SAPI:
--------------------
| Sapi Name        |
--------------------
| apache2_4handler |
| cgi              |
| cli              |
--------------------


-----------------------------------------
|                     |                 |
-----------------------------------------
| Build type          | Release         |
| Thread Safety       | Yes             |
| Compiler            | Visual C++ 2022 |
| Target Architecture | arm64           |
| Host Architecture   | x64             |
| Optimization        | PGO disabled    |
| Native intrinsics   | NEON            |
| Static analyzer     | disabled        |
-----------------------------------------
```

# Preparing dependencies

```cmd
cd /D C:\Development\dependencies

git clone https://github.com/vladimir-aubrecht/vcpkg

cd /D C:\Development\dependencies\vcpkg

git checkout a76eb002a71b6cf7bad343f5e3376dfe6bb83c5c

./bootstrap-vcpkg

./vcpkg install openssl:arm64-windows-static
./vcpkg install curl[openssl,http2,ssh]:arm64-windows-static
./vcpkg install zlib:arm64-windows-static
./vcpkg install libpq:arm64-windows
./vcpkg install libiconv:arm64-windows-static

./vcpkg install libzip:arm64-windows-static
./vcpkg install bzip2:arm64-windows-static
./vcpkg install liblzma:arm64-windows-static
./vcpkg install libxslt:arm64-windows-static
./vcpkg install tidy-html5:arm64-windows
./vcpkg install sqlite3[zlib]:arm64-windows-static
./vcpkg install libsodium:arm64-windows
./vcpkg install gettext:arm64-windows-static
./vcpkg install icu:arm64-windows
./vcpkg install libffi:arm64-windows-static
./vcpkg install libjpeg-turbo:arm64-windows-static
./vcpkg install freetype:arm64-windows
./vcpkg install libpng:arm64-windows-static
./vcpkg install libwebp:arm64-windows-static
./vcpkg install libxpm:arm64-windows-static

xcopy C:\Development\vcpkg\installed\arm64-windows C:\Development\dependencies /E /H /C /I && xcopy C:\Development\vcpkg\installed\arm64-windows-static C:\Development\dependencies /E /H /C /I /Y

cd /D C:\Development\dependencies\lib

copy zlib.lib zlib_a.lib && move zip.lib libzip_a.lib && move bz2.lib libbz2_a.lib && move lzma.lib liblzma_a.lib && move jpeg.lib libjpeg_a.lib && move libpng16.lib libpng_a.lib && move tidys.lib tidy.lib && move sqlite3.lib libsqlite3.lib && move intl.lib libintl.lib && move libxpm.lib libxpm_a.lib && move webp.lib libwebp.lib

```

**WARNING**: Note libpq is installed as NON-static version. If you use static version, you'll be missing exports.

**NOTE**: Zlib.lib will be needed for building of libxml, but zlib_a.lib for building php itself.

# Building
For all builds use terminal opened by comamnd below (it has necessary environment variables set).
```cmd
%comspec% /k "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvarsall.bat x64_arm64"
```

## Libxml 2.9.9

Set environment variables:
- Python2_EXECUTABLE to C:\Python27\python.exe
- Python2_LIBRARIES to C:\Python27\libs

```cmd
cd /D C:\Development

git clone https://github.com/GNOME/libxml2

cd /D C:\Development\libxml2

git checkout v2.9.9

cd /D C:\Development\libxml2\win32

cscript configure.js compiler=msvc prefix=C:\Development\dependencies include=C:\Development\dependencies\include lib=C:\Development\dependencies \lib zlib=yes

nmake

nmake install
```

Result of this step should be 3 files prefixed by libxml2 string in ``C:\Development\dependencies\lib`` folder.

## PHP
Start by building Apache Httpd by checking [this guide](./php_build_x64_to_arm64.md).

Following steps are using same paths as guide on the link and assumes you prepared build of Apache Httpd.


```cmd

cd /D C:\Development\php

git clone --single-branch --branch arm64 https://github.com/dixyes/php-sdk-binary-tools

git clone --single-branch --branch win_arm64 https://github.com/dixyes/php-src

call php-sdk-binary-tools/phpsdk-starter.bat -c vc17 -a arm64

cd /D C:\Development\php\php-src

buildconf

configure --enable-cli --enable-soap --enable-apache2-4handler --disable-opcache --with-openssl --with-extra-includes=C:\Apache24\include; --with-extra-libs=C:\Apache24\lib --with-curl --with-php-build=C:\Work\dependencies --with-mysqli --with-pgsql --with-bz2 --with-xsl --with-tidy --enable-ftp --with-sodium --with-sqlite3 --with-gettext --with-dba --enable-exif --enable-mbstring --enable-fileinfo --enable-odbc --enable-sockets --enable-sysvshm --enable-zend-test --enable-pdo --with-ffi --enable-shmop --enable-intl --with-pdo-odbc 
--with-pdo-pgsql --with-pdo-sqlite
```

Now when ``C:\Development\php\php-src\configure.js`` is generated, open it and locate line:

``AC_DEFINE('HAVE_GD_FREETYPE', 1, "Freetype support");``

after that add this line (should be around line number: 5338):

``AC_DEFINE('FOR_MSW', 1, "LibXPM is for MSW");``

Then edit file ``C:\Development\php\php-src\ext\gd\libgd\gd_interpoation.c``
And drop block:
```cpp
#ifdef _MSC_VER
# pragma optimize("t", on)
# include <emmintrin.h>
#endif
```

Now PHP should be buildable, continue with these commands:
```cmd
nmake

nmake install
```