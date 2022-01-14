# Building ProteoWizard components in MacOS (on Catalina 10.15.7)


# Trying to get TagRecon to build
## Probably necessary
```bash
brew install gcc@7
export CC=$(which gcc-7)
export CXX=$(which g++-7)
export CPP=$(which cpp-7)
export LD=$(which gcc-7)
```
## Probably not necessary
```bash
export LDFLAGS="-L/usr/local/opt/expat/lib"
export CPPFLAGS="-I/usr/local/opt/expat/include"
export PKG_CONFIG_PATH="/usr/local/opt/expat/lib/pkgconfig"
```

### More necessary stuff
- Using `asdf`, set `asdf global python 2.7.18` so it uses 2.7.x
  - Old: ~~Using pyenv, set `python global system` so it uses 2.7.x~~

# Files To Edit
*(paths relative to bumbershoot root directory)*
## ✏️ **`pwiz_tools/Bumbershoot/freicore/stdafx.h`**
1. ## **Lines ~30 and ~36:**
   Try commenting out the `#define WIN32` and `#define WIN64`
<br><br>

2. ## **Lines ~82-84:**
   Change this:
    ```cpp
    #ifndef __CYGWIN__
    #include <sys/sysinfo.h>
    #endif
    ```
    to this:
    ```cpp
    #if defined(WIN32) && ! defined (__CYGWIN__)
    #include <sys/sysinfo.h>
    #endif
    ```
<br>

## ✏️ **`libraries/expat-2.0.1/lib/xmlparse.c`**
    
1. Around line 77, define `HAVE_MEMMOVE`:
   ```c
   /* Handle the case where memmove() doesn't exist. */
   #define HAVE_MEMMOVE // INSERT THIS LINE
   #ifndef HAVE_MEMMOVE
   ```
<br>

## ✏️ **`libraries/expat-2.0.1/lib/xmltok.c`**
1. Around line 21, define `BYTORDER` as 1234:
   ```c
   #endif /* ndef COMPILED_FROM_DSP */
   
   #define BYTEORDER 1234 // INSERT THIS LINE
   
   #include "expat_external.h"
   ```
- You may see a warning about `BYTEORDER` being redefined. Here's what I have to say about that: **GOOD!** The existing definition is basically `BYTEORDER=null` and is thus worthless.

# BUILDING
## Build with:
```bash
./quickbuild.sh --without-binary-msdata toolset=gcc runtime-link=shared pwiz_tools/Bumbershoot/tagrecon
```
<br>

## Rebuilding
- You can add `-a` to rebuild without doing a `clean` first, which is handy because cleaning would erase all your modified files.
  ```bash
  ./quickbuild.sh --without-binary-msdata -a toolset=gcc runtime-link=shared pwiz_tools/Bumbershoot/tagrecon
  ```
- If you get an error because it's using `clang` by default on a Mac, and not respecting the `toolset=gcc` parameter, try the following.
<br><br>


## Editing files
## ✏️ **`libraries/boost-build/src/engine/build.sh`**
- Edit this file to force the toolset to be `gcc`.
- In the latest version, I just commented out **line 157**, inside the **`check_toolset ()`** function:
  ```cpp
  # Prefer Clang (clang) on macOS..
  # if test_toolset clang && test_uname Darwin && test_compiler clang++ -x c++ -std=c++11 ; then B2_TOOLSET=clang ; return ${TRUE} ; fi
  ```
- Might be able to just change `B2_TOOLSET=clang` to `B2_TOOLSET=gcc`, not sure.

<br>
<br>
<br>

# Building idpQonvert
## Environment
- `brew install gcc@8`
- Using **gcc 8.5.0**, with `setgcc 8`
  - This creates symlinks in `/usr/local/bin`:
    - `gcc -> gcc-8`
    - `g++ -> g++-8`
  - And also sets these variables:
    ```
    export CC=$(which gcc-10)
    export CXX=$(which g++-10)
    export CPP=$(which cpp-10)
    export LD=$(which gcc-10)
    ```

## Build command
- `./quickbuild.sh --without-binary-msdata address-model=64 toolset=gcc runtime-link=shared pwiz_tools/Bumbershoot/idpicker//install_cli`
  
## Editing files
## ✏️ ADD THIS FILE: **`pwiz_tools/Bumbershoot/idpicker/Qonverter/waffles/fenv.h`**
```cpp
#pragma once

#include_next <fenv.h>

#if defined(__APPLE__) && defined(__MACH__)

// Public domain polyfill for feenableexcept on OS X
// http://www-personal.umich.edu/~williams/archive/computation/fe-handling-example.c

inline int feenableexcept(unsigned int excepts)
{
    static fenv_t fenv;
    unsigned int new_excepts = excepts & FE_ALL_EXCEPT;
    // previous masks
    unsigned int old_excepts;

    if (fegetenv(&fenv)) {
        return -1;
    }
    old_excepts = fenv.__control & FE_ALL_EXCEPT;

    // unmask
    fenv.__control &= ~new_excepts;
    fenv.__mxcsr   &= ~(new_excepts << 7);

    return fesetenv(&fenv) ? -1 : old_excepts;
}

inline int fedisableexcept(unsigned int excepts)
{
    static fenv_t fenv;
    unsigned int new_excepts = excepts & FE_ALL_EXCEPT;
    // all previous masks
    unsigned int old_excepts;

    if (fegetenv(&fenv)) {
        return -1;
    }
    old_excepts = fenv.__control & FE_ALL_EXCEPT;

    // mask
    fenv.__control |= new_excepts;
    fenv.__mxcsr   |= new_excepts << 7;

    return fesetenv(&fenv) ? -1 : old_excepts;
}

#endif
```

## ✏️ **`pwiz_tools/Bumbershoot/idpicker/Qonverter/crawdad/msmat/crawutils.h`**
    
1. ## Around line 43, define `DARWIN` and include the newly-created `fenv.h` file:
   Change this:
   ```cpp
   using namespace GClasses;
   ```

   To this:
   ```cpp
   #define DARWIN 1
   #ifdef DARWIN
   # include "fenv.h"
   #endif
   
   using namespace GClasses;
   ```
<br>

2. ## Around line 593, change the `if` block so the section for `DARWIN` is the same as the default:
   Change this:
   ```cpp
    #	ifdef DARWIN
      // todo: Anyone know how to do this on Darwin?
    #	else
      feenableexcept(FE_INVALID | FE_DIVBYZERO | FE_OVERFLOW);
    #	endif
   ```

   To this:
   ```cpp
   # ifdef DARWIN
   	// todo: Anyone know how to do this on Darwin?
     feenableexcept(FE_INVALID | FE_DIVBYZERO | FE_OVERFLOW);
   # else
   	feenableexcept(FE_INVALID | FE_DIVBYZERO | FE_OVERFLOW);
   # endif
   ```
<br>

## ✏️ **`pwiz_tools/Bumbershoot/idpicker/Qonverter/crawdad/msmat/crawutils.h`**
    
1. ## Around line 80, just define `_LARGEFILE_SOURCE` and `_FILE_OFFSET_BITS=64`:
   Change this:
   ```cpp
   #ifndef _LARGEFILE_SOURCE
   #error "need to define _LARGEFILE_SOURCE!!"
   #endif    /* end _LARGEFILE_SOURCE */
   ```

   To this:
   ```cpp
   #define _LARGEFILE_SOURCE
   #define _FILE_OFFSET_BITS=64
   #ifndef _LARGEFILE_SOURCE
   #error "need to define _LARGEFILE_SOURCE!!"
   #endif    /* end _LARGEFILE_SOURCE */
   ```
<br>

## ✏️ **`pwiz_tools/Bumbershoot/idpicker/Qonverter/waffles/GThread.h`**
    
1. Around line 23, we're going to force-define `DARWIN` before the `#ifdef WINDOWS`, so that `THREAD_HANDLE` will be defined properly:

   Change this:
   ```cpp
   #ifndef DARWIN
   #define DARWIN 1
   #endif
   
   #ifdef WINDOWS
   #	define BAD_HANDLE (void*)1
   ```

   To this:
   ```cpp
   #ifndef DARWIN
   #define DARWIN 1
   #endif
   
   #ifdef WINDOWS
   #	define BAD_HANDLE (void*)1
   ```
<br>

<!-- FOR FUTURE USE:

## ✏️ **`${relative_path_to_file}`**
    
1. ## Around line ${line_number}, ${description_of_actions_to_take}:

   Change this:
   ```cpp
   ${initial_code_indented}
   ```

   To this:
   ```cpp
   ${final_code_indented}
   ```
<br> -->


<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
# Probably not working/useful

```bash
export CFLAGS+="-isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk"
export CCFLAGS+="-isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk"
export CXXFLAGS+="-isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk"
export CPPFLAGS+="-isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk"
export CPLUS_INCLUDE_PATH+="/usr/local/Cellar/gcc/10.2.0/include/c++/10.2.0/:/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/"
export CPLUS_INCLUDE_PATH="/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/"
```

```bash
echo ${CFLAGS}
echo ${CCFLAGS}
echo ${CXXFLAGS}
echo ${CPPFLAGS}
echo ${CPLUS_INCLUDE_PATH}
```

```bash
unset CFLAGS
unset CCFLAGS
unset CXXFLAGS
unset CPPFLAGS
unset CPLUS_INCLUDE_PATH
```

```bash
export CC=$(which gcc-10)
export CXX=$(which g++-10)
export CPP=$(which cpp-10)
export LD=$(which gcc-10)
```

```bash
unset CC
unset CXX
unset CPP
unset LD
```



<!-- BUILDING PROTEOWIZARD

Run "quickbuild.sh" to build ProteoWizard. (Use "quickbuild.bat" for Windows.) 

Use the "--help" switch for some useful hints on building ProteoWizard.  

Open doc/index.html in a browser for more detailed info on ProteoWizard, known issues, etc. -->

