---
title: Installation
permalink: /docs/getting/
---

# Pre-requisites
## All
OpenSSE's schemes implementation need a compiler supporting C++14. It has been successfully built and tested on Ubuntu 14 LTS using both clang 3.6 and gcc 4.9.3 and on Mac OS X.10 using clang 7.0.0.

First, you will need essential dependencies in order to properly get and build the source.

## Linux (Ubuntu)

On Ubuntu, you will to install the following packages:
```sh
 $ [sudo] apt install build-essential autoconf libtool yasm libssl-dev scons
```
If you are using a different distribution, you might need to adapt this line to your favorite package manager.

### Installing Boost
One small part of the code uses Boost's [Endian](http://www.boost.org/doc/libs/release/libs/endian/) library. Unfortunately, this library has only been present in Boost starting from release 1.58, and thus unavailable as a regular package.
Hence, you will need to [install Boost manually](http://www.boost.org/doc/libs/1_65_1/more/getting_started/unix-variants.html).
Hopefully, Endian is a header-only library, so you can just download the headers and put them in your compiler's header search path.

### Installing gRPC
OpenSSE uses Google's [gRPC](http://grpc.io) as its RPC machinery.
Follow the instructions to install gRPC's C++ binding (see [here](https://github.com/grpc/grpc/tree/master/src/cpp)).

### Installing RocksDB
OpenSSE uses Facebook's [RocksDB](http://rocksdb.org) as its storage engine. OpenSSE has been tested with the 5.7 release. See the [installation guide](https://github.com/facebook/rocksdb/blob/master/INSTALL.md).


## Mac OS X
First, make sure you can the developpers' command line tools installed:
```sh
 $ [sudo] xcode-select --install
```

Then, if you still haven't, you should get [Homebrew](http://brew.sh/). 
You will actually need it to install dependencies.
Run the following line to get all of them (this might take some time). 

```sh
 $ brew install automake cmake autoconf yasm openssl scons grpc rocksdb boost
```

## Relic

The last dependency you will need is not packaged and you will have to install it manually.
Some features (puncturable encryption) are based on cryptographic pairings. These are implemented using the [RELIC](https://github.com/relic-toolkit/relic) toolkit. RELIC has many compilation options. To install RELIC, you can do the following:
```sh
$ git clone https://github.com/relic-toolkit/relic.git
$ cd relic
$ mkdir build; cd build
$ cmake -G "Unix Makefiles" -DMULTI=PTHREAD -DCOMP="-O3 -funroll-loops -fomit-frame-pointer -finline-small-functions -march=native -mtune=native" -DARCH="X64"  -DRAND="UDEV" -DWITH="BN;DV;FP;FPX;EP;EPX;PP;PC;MD" -DCHECK=off -DVERBS=off -DDEBUG=off -DBENCH=0 -DTESTS=1 -DARITH=gmp -DFP_PRIME=254 -DFP_QNRES=off -DFP_METHD="INTEG;INTEG;INTEG;MONTY;LOWER;SLIDE" -DFPX_METHD="INTEG;INTEG;LAZYR" -DPP_METHD="LAZYR;OATEP" -DBN_PRECI=256 -DFP_QNRES=on ../.
$ make 
$ make install
```
You can also replace the `-DARITH=gmp` option by `-DARITH=x64-asm-254` (for better performance) or `-DARITH=easy` (to get rid of the gmp dependency). Note that the first two depend on [gmp](https://gmplib.org) which can itself be installed as a package with
```sh
 $ [sudo] apt install libgmp-dev
```
on Ubuntu and
```sh
 $ brew install gmp
```
using Homebrew.
