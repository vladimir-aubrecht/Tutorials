# Build Apache httpd 2.4.52 (x64 -> arm64, openssl 1.1.1m, VS2022/VC17)
Page describes all necessary steps to build Apache httpd. Original source was [this page](https://www.apachelounge.com/viewtopic.php?t=6462), which is already out of date.

# Software requirements
- [Visual Studio 2022](https://visualstudio.microsoft.com/vs/community/)
- [ActivePerl for Windows](https://www.activestate.com/products/perl/)
- [CMake for Windows](https://cmake.org/download/)
- [GNU Awk for Windows](http://gnuwin32.sourceforge.net/packages/gawk.htm)
- [Netwide Assembler (NASM)](https://www.nasm.us/)
- Folder structure

```bash
C:\Development
   └ Apache24
      ├ src
      │   ├ apr
      │   ├ apr-util
      │   ├ httpd
      │   ├ libexpat
      │   ├ openssl
      │   └ pcre-8.36
      │
      └ build
         ├ apr
         ├ apr-util
         ├ httpd
         ├ libexpat
         └ pcre
```

# Building dependencies
Clone all repositories into **src** folder.

For all builds use terminal opened by comamnd below (it has necessary environment variables set).
```bash
%comspec% /k "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvarsall.bat x64_arm64"
```

x64_arm64

## PCRE
```bash
cd /D C:\Development\Apache24\src

git clone https://github.com/asmlib/pcre-8.36

cd /D C:\Development\Apache24\build\pcre

cmake -G "NMake Makefiles" -DCMAKE_INSTALL_PREFIX=C:\Apache24 -DCMAKE_BUILD_TYPE=RelWithDebInfo -DBUILD_SHARED_LIBS=ON -DPCRE_BUILD_TESTS=OFF -DPCRE_BUILD_PCRECPP=OFF -DPCRE_BUILD_PCREGREP=OFF -DPCRE_SUPPORT_PCREGREP_JIT=OFF -DPCRE_SUPPORT_UTF=ON -DPCRE_SUPPORT_UNICODE_PROPERTIES=ON -DPCRE_NEWLINE=CRLF -DINSTALL_MSVC_PDB=OFF ..\..\src\pcre-8.36

nmake

nmake install
```

## OpenSSL
```bash
cd /D C:\Development\Apache24\src

git clone https://github.com/openssl/openssl

git reset --hard OpenSSL_1_1_1m

cd /D C:\Development\Apache24\src\openssl

perl Configure VC-WIN64-ARM --prefix=C:\Apache24 --openssldir=C:\Apache24\conf enable-camellia no-idea no-mdc2 no-ssl2 no-ssl3 no-zlib no-asm

nmake
nmake install
```

Comments:
- no-asm - for some reason perl couldn't find nasm which I had in my PATH. Impact of no-asm is 4x lower performance in certain scenarios.

## Libexpat
```bash
cd /D C:\Development\Apache24\src

git clone https://github.com/libexpat/libexpat

git reset --hard R_2_4_6

cd /D C:\Development\Apache24\build\libexpat

echo set(CMAKE_SYSTEM_NAME Windows) > win64-arm-toolchain.cmake

echo set(CMAKE_SYSTEM_PROCESSOR aarch64) >> win64-arm-toolchain.cmake

cmake -G "NMake Makefiles" -DCMAKE_TOOLCHAIN_FILE=win64-arm-toolchain.cmake -DCMAKE_INSTALL_PREFIX=C:\Apache24 -DCMAKE_BUILD_TYPE=RelWithDebInfo ..\..\src\libexpat\expat

nmake

nmake install

```

## Apr
Apr build pipeline is generating executable (gen_test_char.exe) in format of target platform and this executable is used later in build process to generate a header file.
This cannot work for cross compiling scenario x64 -> arm64 as generated binary will not run on original system. Due to this, workaround is needed.

First, build APR library for [x64](./apache_build_x64_to_x64.md) (steps up to nmake, no need for nmake install).

```bash

cd /D C:\Development\Apache24\build\apr

move gen_test_char.exe gen_test_char_host.exe

powershell /c "ls | where { $_.Name -ne 'gen_test_char_host.exe' } | Remove-Item -Force -Recurse"

%comspec% /k "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvars64.bat" # Switch to right cross compiling environment variables

cmake -G "NMake Makefiles" -DCMAKE_INSTALL_PREFIX=C:\Apache24 -DCMAKE_BUILD_TYPE=RelWithDebInfo -DMIN_WINDOWS_VER=0x0600 -DAPR_HAVE_IPV6=ON -DAPR_INSTALL_PRIVATE_H=ON -DAPR_BUILD_TESTAPR=OFF -DINSTALL_PDB=OFF ..\..\src\apr

powershell /c "(Get-Content MakeFile) -replace 'gen_test_char: cmake_check_build_system', 'gen_test_char_host: cmake_check_build_system' | Out-File -encoding ASCII Makefile" # Line 145, target must be renamed

nmake # expected to fail on missing header file, continue with next step

gen_test_char_host.exe > apr_escape_test_char.h

nmake

nmake install
```

## Apr-Util
```bash
cd /D C:\Development\Apache24\src

git clone https://github.com/apache/apr-util

git reset --hard 1.6.1

cd /D C:\Development\Apache24\build\apr-util

cmake -G "NMake Makefiles" -DCMAKE_INSTALL_PREFIX=C:\Apache24 -DOPENSSL_ROOT_DIR=C:\Apache24 -DCMAKE_BUILD_TYPE=RelWithDebInfo -DAPU_HAVE_CRYPTO=ON -DAPR_BUILD_TESTAPR=OFF -DINSTALL_PDB=OFF ..\..\src\apr-util

nmake

nmake install

```

# Building Apache httpd
Httpd build pipeline is generating executable (gen_test_char.exe) in format of target platform and this executable is used later in build process to generate a header file.
This cannot work for cross compiling scenario x64 -> arm64 as generated binary will not run on original system. Due to this, workaround is needed.

First, build Httpd for [x64](./apache_build_x64_to_x64.md) (steps up to nmake, no need for nmake install).

```bash
cd /D C:\Development\Apache24\build\httpd

move gen_test_char.exe gen_test_char_host.exe

powershell /c "ls | where { $_.Name -ne 'gen_test_char_host.exe' } | Remove-Item -Force -Recurse"

%comspec% /k "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvars64.bat" # Switch to right cross compiling environment variables

echo set(CMAKE_SYSTEM_NAME Windows) > win64-arm-toolchain.cmake

echo set(CMAKE_SYSTEM_PROCESSOR aarch64) >> win64-arm-toolchain.cmake

echo set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /RunBelow4GB") >> win64-arm-toolchain.cmake

echo set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /RunBelow4GB") >> win64-arm-toolchain.cmake

echo set(CMAKE_STATIC_LINKER_FLAGS "${CMAKE_STATIC_LINKER_FLAGS} /RunBelow4GB") >> win64-arm-toolchain.cmake

cmake -G "NMake Makefiles" -DCMAKE_TOOLCHAIN_FILE=win64-arm-toolchain.cmake -DCMAKE_INSTALL_PREFIX=C:\Apache24 -DCMAKE_BUILD_TYPE=RelWithDebInfo -DENABLE_MODULES=i -DINSTALL_PDB=OFF ..\..\src\httpd

powershell /c "(Get-Content MakeFile) -replace 'gen_test_char: cmake_check_build_system', 'gen_test_char_host: cmake_check_build_system' | Out-File -encoding ASCII Makefile" # Line 145, target must be renamed

echo ; > BaseAddr.ref # Lets remove hardcoded optimization on base addresses and let Windows decide. Original addresses doesn't work on ARM64 and general guidance is, that it's safer to let Windows decide.

nmake # will fail on cannot execute gen_test_char.exe

gen_test_char_host.exe > test_char.h

copy ..\..\src\openssl\ms\applink.c C:\Apache24\include\openssl\applink.c

nmake # should pass

nmake install

```
