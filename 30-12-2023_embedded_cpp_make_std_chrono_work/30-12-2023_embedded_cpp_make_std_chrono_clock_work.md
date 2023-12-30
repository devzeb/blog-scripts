Hint: This article is a work in progress. I am still working on it. But if you have any questions, feel free to contact me.

# Embedded Cpp - Make std::chrono clock work

# What is the problem with using std::chrono on cortex m?

Given this trivial example code in the given environment (described at the end of this article):

```cpp
#include <chrono>

int main()
{
    auto now_steady = std::chrono::steady_clock::now();

    volatile auto counts = now_steady.time_since_epoch().count(); // dont optimize counts away

    return 0;
}
```

Trying to compile this code yields the following linker error:

```
C:/Users/seb/opt/arm-gnu-toolchain-13.2.rel1-mingw-w64-i686-arm-none-eabi/bin/../lib/gcc/arm-none-eabi/13.2.1/../../../../arm-none-eabi/bin/ld.exe: C:/Users/seb/opt/arm-gnu-toolchain-13.2.rel1-mingw-w64-i686-arm-none-eabi/bin/../lib/gcc/arm-none-eabi/13.2.1/thumb/v7e-m+dp/hard\libg_nano.a(libc_a-gettimeofdayr.o): in function `_gettimeofday_r':

gettimeofdayr.c:(.text._gettimeofday_r+0xe): undefined reference to `_gettimeofday'

collect2.exe: error: ld returned 1 exit status
```

The linker is missing the function `_gettimeofday`, therefore the linking process fails.
The error message hints, that the call is made from the function `_gettimeofday_r` of the library `libg_nano.a` (see [What is libg_nano.a?](#what-is-libg_nanoa)). 

This means that the C standard library  in use is trying to make a call to `_gettimeofday`, but this function is neither implemented in the library itself nor in our user code (`main.cpp`).

## How is the dependency between std::chrono::steady_clock and gettimeofday?
Looking at the disassembly of the code, we can extract the following information:
```
std::chrono::_V2::steady_clock::now() calls std::chrono::_V2::system_clock::now()
std::chrono::_V2::system_clock::now() calls gettimeofday
gettimeofday calls _gettimeofday_r
_gettimeofday_r calls _gettimeofday
```


## What is gettimeofday, _gettimeofday and _gettimeofday_r?
As the error originates from the newlib implementation, we will find further information there.

We can find a hint to the function `gettimeofday` in 
[this section of the newlib documentation](https://sourceware.org/newlib/libc.html#time):

> Supporting OS subroutine required: Some implementations require gettimeofday.

This hints that `gettimeofday` is a `OS subroutine`.

As `OS subroutine` suggests, it is a function that is provided by the operating system.
The function is even part of the POSIX operating system standard: https://pubs.opengroup.org/onlinepubs/9699919799.2018edition/functions/gettimeofday.html

But on the Cortex M platform, we do not have an operating system.

Therefore this function is no where to be found in the platform specific version of the newlib C standard library (`libg_nano.a` in this case).

### Wait, but why did the linker not complain about `gettimeofday`, but `_gettimeofday`? 
This has to do with the fact, that newlib provides a reentrant version of the C standard library.
From [this section of the newlib documentation](https://sourceware.org/newlib/libc.html#Reentrancy).
> Reentrancy is a characteristic of library functions which allows multiple processes to use the same address space with assurance that the values stored in those spaces will remain constant between calls. The Red Hat newlib implementation of the library functions ensures that whenever possible, these library functions are reentrant. However, there are some functions that can not be trivially made reentrant. Hooks have been provided to allow you to use these functions in a fully reentrant fashion.
> 
> These hooks use the structure _reent defined in reent.h. A variable defined as ‘struct _reent’ is called a reentrancy structure. All functions which must manipulate global information are available in two versions. The first version has the usual name, and uses a single global instance of the reentrancy structure. The second has a different name, normally formed by prepending ‘_’ and appending ‘_r’, and takes a pointer to the particular reentrancy structure to use.

This is the reason why the system call `gettimeofday` is actually implemented as a trampoline function to the reentrant version `_gettimeofday_r`, which then calls the actual system call `_gettimeofday`.

To sum it up: Newlib renamed the actual system call from `gettimeofday` to `_gettimeofday` to enable reentrancy.

If you want more information about this, look at the section [Proof that the _gettimeofday is the actual system call](#proof-that-the-_gettimeofday-is-the-actual-system-call).

# Implementing _gettimeofday to make std::chrono work

According to the documentation of gettimeofday in the [POSIX standard](https://pubs.opengroup.org/onlinepubs/9699919799.2018edition/functions/gettimeofday.html):

> DESCRIPTION
>
> The gettimeofday() function shall obtain the current time, expressed as seconds and microseconds since the Epoch, and store it in the timeval structure pointed to by tp. The resolution of the system clock is unspecified.
>
>RETURN VALUE
>
>The gettimeofday() function shall return 0 and no value shall be reserved to indicate an error.

This means we have to do two things in our implementation of `_gettimeofday`:
- modify the pointer struct timeval to contain the current time since the epoch
- return 0

The timezone struct is defined in the include file `sys/_timeval.h` (`newlib/libc/include/sys/_timeval.h` in the newlib repository):

```c
/*
 * Structure returned by gettimeofday(2) system call, and used in other calls.
 */
struct timeval {
	time_t		tv_sec;		/* seconds */
	suseconds_t	tv_usec;	/* and microseconds */
};
```






### What is libg_nano.a?
According to this site: https://www.sourceware.org/legacy-ml/newlib/2011/msg00647.html
- `libg.a` is a debugging-enabled libc.
- `libc` is the C standard library.

Therefore, `libg_nano.a` is a debugging-enabled libc from the Newlib Nano project.

Newlib can be found here: https://sourceware.org/newlib/

> Newlib is a C library intended for use on embedded systems.

The Newlib Nano project is a subset of the newlib project, providing the C standard library for embedded systems.



## Compile and link commands used

- Cortex M7 (STM32H7) 
- arm-none-eabi-g++ version 13.2

```bash
arm-none-eabi-g++.exe   -g -std=gnu++20 -fdiagnostics-color=always --specs=nano.specs -mthumb -ffunction-sections -fdata-sections -fno-rtti -fno-exceptions -fno-common -fno-non-call-exceptions -fno-use-cxa-atexit -MD -MT embedded_template/CMakeFiles/embedded_template.dir/gettimeofday.cpp.obj -MF embedded_template\CMakeFiles\embedded_template.dir\gettimeofday.cpp.obj.d -o embedded_template/CMakeFiles/embedded_template.dir/gettimeofday.cpp.obj -c C:/Users/seb/code/embedded_template/embedded_template/gettimeofday.cpp

arm-none-eabi-g++.exe   -g -std=gnu++20 -fdiagnostics-color=always --specs=nano.specs -mthumb -ffunction-sections -fdata-sections -fno-rtti -fno-exceptions -fno-common -fno-non-call-exceptions -fno-use-cxa-atexit -MD -MT embedded_template/CMakeFiles/embedded_template.dir/main.cpp.obj -MF embedded_template\CMakeFiles\embedded_template.dir\main.cpp.obj.d -o embedded_template/CMakeFiles/embedded_template.dir/main.cpp.obj -c C:/Users/seb/code/embedded_template/embedded_template/main.cpp

arm-none-eabi-g++.exe -g --specs=nano.specs -mthumb -Wl,--gc-sections -Wl,--print-memory-usage embedded_template/CMakeFiles/embedded_template.dir/main.cpp.obj embedded_template/CMakeFiles/embedded_template.dir/gettimeofday.cpp.obj -o embedded_template\embedded_template

```


## Environment
- ARM GNU Toolchain (arm-none-eabi) (I am using version 13.2)
- Git repository: https://github.com/devzeb/embedded_template
- Git revision: 1ce47b58f1bf52749c70e59b2ec7258aa7eb6d9b

Compiler:
- `-std=gnu++20`
- `--specs=nano.specs`
- `-mthumb`
- `-mcpu=cortex-m7`
- `-mfloat-abi=hard`
- `-ffunction-sections`
- `-fdata-sections`
- `-fno-rtti`
- `-fno-exceptions`
- `-fno-common`
- `-fno-non-call-exceptions`
- `-fno-use-cxa-atexit`

Linker:
- `--specs=nano.specs`
- `-mthumb`
- `-mcpu=cortex-m7`
- `-mfloat-abi=hard`
- `-Wl,--gc-sections`
- `-Wl,--print-memory-usage`
- `-Wl,-Tembedded_template/device/stm32h723/STM32H723ZGTX_FLASH.ld`


## Proof that the _gettimeofday is the actual system call

The following comments are based on the following revision of newlib:
- website: https://sourceware.org/newlib/
- git repo: https://sourceware.org/git/newlib-cygwin.git
- git revision: `1a177610d8e181d09206a5a8ce2d873822751657`

We can extract the following hints:
- in `newlib/libc/include/sys/time.h` we find the declaration of `gettimeofday` and `_gettimeofday`:

```c
int gettimeofday (struct timeval *__restrict __p,
			  void *__restrict __tz);
...
#ifdef _LIBC
int _gettimeofday (struct timeval *__p, void *__tz);
#endif
```



We can also find a stub implementation of `_gettimeofday` in `libgloss/libnosys/gettod.c`:
When you look closely at the file path, you can see that this implementation is not part of newlibs libc, but part of libgloss. Libgloss is a library that provides the low level functions for a specific target. In this case, the target is `nosys`, which means that the target does not provide any system calls. Therefore, the implementation of `_gettimeofday` is a stub that returns an error.

```c
int
_gettimeofday (struct timeval  *ptimeval,
        void *ptimezone)
{
  errno = ENOSYS;
  return -1;
}
```


- in `newlib/libc/syscalls/sysgettod.c` we can find the implementation of `gettimeofday`:

```c
int
gettimeofday (struct timeval *ptimeval,
     void *ptimezone)
{
  return _gettimeofday_r (_REENT, ptimeval, ptimezone);
}
```

- in `newlib/libc/reent/gettimeofdayr.c` we can find the implementation of `_gettimeofday_r`:

```c
int
_gettimeofday_r (struct _reent *ptr,
     struct timeval *ptimeval,
     void *ptimezone)
{
  int ret;

  errno = 0;
  if ((ret = _gettimeofday (ptimeval, ptimezone)) == -1 && errno != 0)
    _REENT_ERRNO(ptr) = errno;
  return ret;
}
```

And there is the call to `_gettimeofday`!