

# Gollvm

Gollvm is an LLVM-based Go compiler. It incorporates "gofrontend" (a Go language front end written in C++ and shared with GCCGO), a bridge component (which translates from gofrontend IR to LLVM IR), and a driver that sends the resulting IR through the LLVM back end.

Gollvm is set up to be a subproject within the LLVM tools directory, similar to
how things work for "clang" or "compiler-rt": you check out a copy of the LLVM
source tree, then within the LLVM tree you check out additional git repos.


# Table of contents <a name="toc"></a>

 * [Building gollvm](#building)
 * [Work area setup](#workarea)
 * [Invoking cmake and ninja](#cmakeninja)
 * [Installing gollvm](#installing)
 * [Using an installed copy of gollvm](#using)
 * [Crosscompiling gollvm](#crosscompiling)
 * [Information for gollvm developers](#developers)

[FAQ](#FAQ)

  * [Where should I post questions about gollvm?](#wheretopostquestions)
  * [Where should I file gollvm bugs?](#wheretofile)
  * [How can I go about contributing to gollvm?](#contributing)
  * [Is gollvm a replacement for the main Go compiler (gc)?](#replacegc)
  * [Which architectures and operating systems are supported for gollvm?](#supported)
  * [How does the gollvm runtime differ from the main Go runtime?](#runtimediffs)
  * [Shared linkage is the default for gollvm. How do I build non-shared?](#buildstatic)
  * [What command line options are supported for gollvm?](#cmdlineopts)
  * [How do I see the LLVM IR generated by gollvm?](#seetheir)
  * [What is the relationship between gollvm and gccgo?](#gollvmandgccgo)
  * [Can I use FDO or Thin LTO with gollvm?](#thinltofdo)
  * [Can I use the race detector?](#racedetector)
  * [I am seeing "undefined symbol: `__get_cpuid_count`" from my gollvm install](#getcpuidcount_undefined)
  
# Building gollvm <a name="building"></a>

Gollvm is currently in development -- releases are not yet available for download.  Instructions for building gollvm follow.

## Setting up a gollvm work area <a name="workarea"></a>

To set up a work area for Gollvm, check out a copy of LLVM, the overlay the gollvm repo (and other associated dependencies) within the LLVM tools subdir, as follows:

```
// Here 'workarea' will contain a copy of the LLVM source tree and one or more build areas
% mkdir workarea
% cd workarea

// Sources
% git clone https://github.com/llvm/llvm-project.git
...
% cd llvm-project/llvm/tools
% git clone https://go.googlesource.com/gollvm
...
% cd gollvm
% git clone https://go.googlesource.com/gofrontend
...
% cd libgo
% git clone https://github.com/libffi/libffi.git
...
% git clone https://github.com/ianlancetaylor/libbacktrace.git
...
%
```

## Building gollvm with cmake and ninja <a name="cmakeninja"></a>

You'll need to have an up-to-date copy of cmake on your system (3.6 or later vintage) to build Gollvm, as well as a C/C++ compiler (V10.0 or later for Clang, or V6.0 or later of GCC), and a working copy of 'm4'.

Create a build directory (separate from the source tree) and run 'cmake' within the build area to set up for the build. Assuming that 'workarea' is the directory created as above:

```
% cd workarea
% mkdir build-debug
% cd build-debug
% cmake -DCMAKE_BUILD_TYPE=Debug -DLLVM_USE_LINKER=gold -G Ninja ../llvm-project/llvm
...
% ninja gollvm
...
%
```

This will build the various tools and libraries needed for Gollvm. To select a specific C/C++ compiler for the build, you can use the "-DCMAKE_C_COMPILER" and "-DCMAKE_CXX_COMPILER" options to select your desired C/C++ compiler when invoking cmake (details [here](https://gitlab.kitware.com/cmake/community/wikis/FAQ#how-do-i-use-a-different-compiler)). Use the "-DLLVM_USE_LINKER=<variant>" cmake variable to control which linker is selected to link the Gollvm compiler and tools (where variant is one of "bfd", "gold", "lld", etc).

The Gollvm compiler driver defaults to using the gold linker when linking Go programs.  If some other linker is desired, this can be accomplished by passing "-DGOLLVM_DEFAULT_LINKER=<variant>" when running cmake. Note that this default can still be overridden on the command line using the "-fuse-ld" option.

Gollvm's cmake rules expect a valid value for the SHELL environment variable; if not set, a default shell of /bin/bash will be used.

## Installing gollvm <a name="installing"></a>

A gollvm installation will contain 'llvm-goc' (the compiler driver), the libgo standard Go libraries, and the standard Go tools ("go", "vet", "cgo", etc).

The installation directory for gollvm needs to be specified when invoking cmake prior to the build:

```
% mkdir build.rel
% cd build.rel
% cmake -DCMAKE_INSTALL_PREFIX=/my/install/dir -DCMAKE_BUILD_TYPE=Release -DLLVM_USE_LINKER=gold -G Ninja ../llvm-project/llvm

// Build all of gollvm
% ninja gollvm
...

// Install gollvm to "/my/install/dir"
% ninja install-gollvm

```

## Using an installed copy of gollvm  <a name="using"></a>

Programs build with the Gollvm Go compiler default to shared linkage, meaning that they need to pick up the Go runtime library via LD_LIBRARY_PATH:

```
// Root of Gollvm install is /tmp/gollvm-install
% export LD_LIBRARY_PATH=/tmp/gollvm-install/lib64
% export PATH=/tmp/gollvm-install/bin:$PATH
% go run himom.go
hi mom!
%
```

## Crosscompiling gollvm  <a name="crosscompiling"></a>
You need a working version of gollvm on host system to cross compile. The following script will build and install gollvm on a cross compile system.

```
#!/bin/bash
set -e
mkdir -p build
cd build

RISCV=$HOME/toolchain
SOURCE=$HOME/llvm-project/llvm
TRIPLE=riscv64-unknown-linux-gnu
INSTALL=/tmp/gollvm-install

# host
cmake -G Ninja -S $SOURCE -B build-x86 \
    -DCMAKE_INSTALL_PREFIX=install-x86 \
    -DCMAKE_BUILD_TYPE=Debug \
    -DLLVM_USE_LINKER=bfd \
    -DGOLLVM_DEFAULT_LINKER=bfd \
    -DLLVM_TARGET_ARCH="X86-64,RISCV64" \
    -DLLVM_TARGETS_TO_BUILD="X86;RISCV"

# crosscompile
cmake -G Ninja -S $SOURCE -B build-riscv \
    -DCMAKE_INSTALL_PREFIX=$INSTALL \
    -DCMAKE_BUILD_TYPE=Debug \
    -DLLVM_USE_LINKER=bfd \
    -DGOLLVM_DEFAULT_LINKER=bfd \
    -DCMAKE_CROSSCOMPILING=True \
    -DLLVM_TARGET_ARCH=RISCV64 \
    -DLLVM_DEFAULT_TARGET_TRIPLE=$TRIPLE \
    -DLLVM_TARGETS_TO_BUILD=RISCV \
    -DCMAKE_C_COMPILER=$RISCV/bin/$TRIPLE-gcc \
    -DCMAKE_CXX_COMPILER=$RISCV/bin/$TRIPLE-g++ \
    -DLLVM_TABLEGEN=$PWD/build-x86/bin/llvm-tblgen \
    -DGOLLVM_DRIVER_DIR=$PWD/build-x86/bin \
    -DGOLLVM_EXTRA_GOCFLAGS="--target=$TRIPLE \
                             --gcc-toolchain=$RISCV/ \
                             --sysroot=$RISCV/sysroot" \
    -DGOLLVM_USE_SPLIT_STACK=OFF \
    -DCMAKE_C_FLAGS=-latomic \
    -DCMAKE_CXX_FLAGS=-latomic


# build gollvm crosscompiler
ninja -C build-x86 llvm-goc llvm-goc-token llvm-godumpspec

# cross compile gollvm, go tools and install
ninja -C build-riscv install-gollvm
```

# Information for gollvm developers <a name="developers"></a>

## Source code structure

Within \<workarea\>/llvm/tools/gollvm, the following directories are of interest:

.../llvm/tools/gollvm:

 * contains rules to build third party libraries needed for gollvm,
   along with common definitions for subdirs.

.../llvm/tools/gollvm/driver,
.../llvm/tools/gollvm/driver-main:

 * contains build rules and source code for llvm-goc

.../llvm/tools/gollvm/gofrontend:

 * source code for gofrontend and libgo (note: no cmake files here)

.../llvm/tools/gollvm/bridge:

 * contains build rules for the libLLVMCppGoFrontEnd.a, a library that contains both the gofrontend code and the LLVM-specific middle layer (for example, the definition of the class Llvm_backend, which inherits from Backend).

.../llvm/tools/gollvm/libgo:

 * build rules and supporting infrastructure to build Gollvm's copy of the Go runtime and standard packages.

.../llvm/tools/gollvm/unittests:

 * source code for the unit tests

## The llvm-goc program

The executable llvm-goc is the main compiler driver for gollvm; it functions as a compiler (consuming source for a Go package and producing an object file), an assembler, and/or a linker.  While it is possible to build and run llvm-goc directly from the command line, in practice there is little point in doing this (better to build using "go build", which will invoke llvm-goc on your behalf.

```
// From within <workarea>/build.opt:

% ninja llvm-goc
...
% cat micro.go
package foo
func Bar() int {
	return 1
}
% ./bin/llvm-goc -fgo-pkgpath=foo -O3 -S -o micro.s micro.go
%
```


## Building and running the unit tests

Here are instructions on building and running the unit tests for the middle layer:

```
// From within <workarea>/build.opt:

// Build unit test
% ninja GoBackendCoreTests

// Run a unit test
% ./tools/gollvm/unittests/BackendCore/GoBackendCoreTests
[==========] Running 10 tests from 2 test cases.
[----------] Global test environment set-up.
[----------] 9 tests from BackendCoreTests
[ RUN      ] BackendCoreTests.MakeBackend
[       OK ] BackendCoreTests.MakeBackend (1 ms)
[ RUN      ] BackendCoreTests.ScalarTypes
[       OK ] BackendCoreTests.ScalarTypes (0 ms)
[ RUN      ] BackendCoreTests.StructTypes
[       OK ] BackendCoreTests.StructTypes (1 ms)
[ RUN      ] BackendCoreTests.ComplexTypes
[       OK ] BackendCoreTests.ComplexTypes (0 ms)
[ RUN      ] BackendCoreTests.FunctionTypes
[       OK ] BackendCoreTests.FunctionTypes (0 ms)
[ RUN      ] BackendCoreTests.PlaceholderTypes
[       OK ] BackendCoreTests.PlaceholderTypes (0 ms)
[ RUN      ] BackendCoreTests.ArrayTypes
[       OK ] BackendCoreTests.ArrayTypes (0 ms)
[ RUN      ] BackendCoreTests.NamedTypes
[       OK ] BackendCoreTests.NamedTypes (0 ms)
[ RUN      ] BackendCoreTests.TypeUtils

...

[  PASSED  ] 10 tests.
```

The unit tests currently work by instantiating an LLVM Backend instance and making backend method calls (to mimic what the frontend would do), then inspects the results to make sure they are as expected. Here is an example:

```
TEST(BackendCoreTests, ComplexTypes) {
  LLVMContext C;

  Type *ft = Type::getFloatTy(C);
  Type *dt = Type::getDoubleTy(C);

  std::unique_ptr<Backend> be(go_get_backend(C, llvm::CallingConv::X86_64_SysV));
  Btype *c32 = be->complex_type(64);
  ASSERT_TRUE(c32 != NULL);
  ASSERT_EQ(c32->type(), mkTwoFieldLLvmStruct(C, ft, ft));
  Btype *c64 = be->complex_type(128);
  ASSERT_TRUE(c64 != NULL);
  ASSERT_EQ(c64->type(), mkTwoFieldLLvmStruct(C, dt, dt));
}
```

The test above makes sure that the LLVM type we get as a result of calling Backend::complex_type() is kosher and matches up to expectations.

## Building libgo (Go runtime and standard libraries)

To build the Go runtime and standard libraries, use the following:

```
// From within <workarea>/build.opt:

// Build Go runtime and standard libraries
% ninja libgo_all

```

This will compile static (\*.a) and dynamic (\*.so) versions of the library.

# FAQ <a name="FAQ"></a>

## Where should I post questions about gollvm? <a name="wheretopostquestions"></a>

Please send questions about gollvm to the [golang-nuts](https://groups.google.com/d/forum/golang-nuts) mailing list. Posting questions to the issue tracker is generally not the right way to start discussions or get information.

## Where should I file gollvm bugs? <a name="wheretofile"></a>

Please file an issue on the golang [issue tracker](https://github.com/golang/go/issues); please be sure to use "gollvm" somewhere in the headline.

## How can I go about contributing to gollvm? <a name="contributing"></a>

Please see the Go project guidelines at [https://golang.org/doc/contribute.html](https://golang.org/doc/contribute.html). Changes to [https://go.googlesource.com/gollvm](https://go.googlesource.com/gollvm) can be made by any Go contributor; for changes to gofrontend see [the gccgo guidelines](https://golang.org/doc/gccgo_contribute.html).

## Is gollvm a replacement for the main Go compiler? (gc) <a name="replacegc"></a>

Gollvm is not intended as a replacement for the main Go compiler -- the
expectation is that the bulk of users will want to continue to use the main Go
compiler due to its superior compilation speed, ease of use, broader
functionality, and higher-performance runtime. Gollvm is intended to provide a
Go compiler with a more powerful back end, enabling such benefits as better
inlining, vectorization, register allocation, etc.

## Which architectures and operating systems are supported for gollvm? <a name="supported"></a>

Gollvm is currently supported only for x86_64, aarch64 and riscv64 Linux.

## How does the gollvm runtime differ from the main Go runtime?  <a name="runtimediffs"></a>

The main Go runtime supports generation of accurate stack maps, which allows the
garbage collector to do precise stack scanning; gollvm does not yet support
stack map generation (note that we're actively working on fixing this), hence
for gollvm the garbage collector has to scan stacks conservatively (which can
lead to longer scan times and increased memory usage). The main Go runtime
compiles to a different calling convention, whereas Gollvm uses the standard
C/C++ calling convention. There are many other smaller differences as well.

## Shared linkage is the default for gollvm. How do I build non-shared?  <a name="buildstatic"></a>

Linking with "-static-libgo" will yield a binary that incorporates a full copy of the Go runtime. Example:

```
 % go build -gccgoflags -static-libgo myprogram.go
```

Note that this will increase binary size.

## What command line options are supported for gollvm?  <a name="cmdlineopts"></a>

You can run 'llvm-goc -help' to see a full set of supported options. These can be passed to the compiler via '-gccgoflags' option. Example:

```
% go build -gccgoflags -fno-inline mumble.go
```

## How do I see the LLVM IR generated by gollvm?  <a name="seetheir"></a>

The 'llvm-goc' command supports the -emit-llvm flag, however passing this option
to a "go build" command is not practical, since the "go build" won't be
expecting the compiler to emit LLVM bitcode or assembly.

A better recipe is to run "go build" with "-x -work" to capture the commands
being executed, then rerun the llvm-goc command shown adding "-S -emit-llvm".
The resulting output will be an LLVM IR dump. Example:

```
% go build -work -x mypackage.go 1> transcript.txt 2>&1
% egrep '(WORK=|llvm-goc -c)' transcript.txt
WORK=/tmp/go-build887931787
/t/bin/llvm-goc -c -g -m64 -fdebug-prefix-map=$WORK=/tmp/go-build \
  -gno-record-gcc-switches -fgo-pkgpath=command-line-arguments \
  -fgo-relative-import-path=/mygopath/src/tmp -o $WORK/b001/_go_.o \
  -I $WORK/b001/_importcfgroot_ ./mypackage.go
% /t/bin/llvm-goc -c -g -m64 -fdebug-prefix-map=$WORK=/tmp/go-build \
  -gno-record-gcc-switches -fgo-pkgpath=command-line-arguments \
  -fgo-relative-import-path=/mygopath/src/tmp \
  -I $WORK/b001/_importcfgroot_ -o mypackage.ll -S -emit-llvm \
  ./mypackage.go
% ls -l mypackage.ll
...
%
```


## What is the relationship between gollvm and gccgo?  <a name="gollvmandgccgo"></a>

Gollvm and gccgo share a common front end (gofrontend) and associated runtime
(libgo), however each uses a separate back end. When using "go build", the Go
command currently treats gollvm as an instance of gccgo (hence the need to pass
compile flags via "-gccgoflags"). This is expected to be temporary.

## Can I use FDO or ThinLTO with gollvm?  <a name="thinltofdo"></a>

There are plans to support FDO, AutoFDO, and ThinLTO for gollvm, however these features have not yet been implemented.

## Can I use the race detector?  <a name="racedetector"></a>

Gollvm does not support the Go race detector; please use the main Go compiler for this purpose.

## I am seeing "undefined symbol: `__get_cpuid_count`" from my gollvm install <a name="getcpuidcount_undefined">

The Gollvm build procedure requires an up-to-date C/C++ compiler; there is code in the gollvm runtime (libgo) that refers to functions defined in `<cpuid.h>`, however some older versions of clang (prior to 5.0) don't provide definitions for all the needed functions. If you encounter this problem, rerun `cmake` to configure your build to use a more recent version of Clang (or use GCC), as described above.
