Hint: This article is a work in progress. I am still working on it. But if you have any questions, feel free to contact me.

# Embedded CPP - Make std::chrono clock work with Newlib Nano on Cortex M

# TL;DR
- In order to make `std::chrono` work with Newlib Nano, you have to implement the system call `_gettimeofday`.
- The implementation of `_gettimeofday` has to return the current time since the epoch (in this case start of the system) in the struct `timeval` and return 0.
- To obtain the current time, you can set up a timer interrupt that increments a counter variable, which then is used to fill the struct `timeval` in the `_gettimeofday` implementation.

# Is this article helpful?
If you like this article and want to support me, you can do so by buying me a coffee, pizza or other developer essentials by clicking the this link:
[Support me on PayPal](https://www.paypal.com/donate/?hosted_button_id=TGDGATFR63N3G)

# Environment 
The code you see in this article is compiled with the arm-none-eabi-g++ cross compiler for the Cortex M platform, using the Newlib Nano C standard library implementation.
The exact flags and command are listed in the sections [Compile and link commands used](#compile-and-link-commands-used) and [Environment](#environment).

# What is the problem with using std::chrono with Newlib Nano?

Given this trivial example code:

```cpp
#include <chrono>

int main()
{
    auto now_steady = std::chrono::steady_clock::now();

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

This means that the C standard library implementation is trying to make a call to `_gettimeofday`, but this function is neither implemented in the library itself nor in our user code (`main.cpp`).

## What is the dependency between std::chrono::steady_clock and gettimeofday?
Looking at the disassembly of the code, we can extract the following information:
```
std::chrono::_V2::steady_clock::now() calls std::chrono::_V2::system_clock::now()
std::chrono::_V2::system_clock::now() calls gettimeofday
gettimeofday calls _gettimeofday_r
_gettimeofday_r calls _gettimeofday
```


## What is gettimeofday, _gettimeofday and _gettimeofday_r, and why isn't it available?
As the error originates from the Newlib Nano library, we can find a hint to the function `gettimeofday` in 
[this section of its documentation](https://sourceware.org/newlib/libc.html#time):

> Supporting OS subroutine required: Some implementations require gettimeofday.

This means that `gettimeofday` is an *OS subroutine*, which is another name for *system call*.
So `gettimeofday` is a function that is supposed to interact with the operating system.

Many other platforms provide this particular system call to interact with the OS, e.g. the [Linux kernel]((https://man7.org/linux/man-pages/man2/syscalls.2.html)).
The function is also part of the [POSIX operating system standard](https://pubs.opengroup.org/onlinepubs/9699919799.2018edition/functions/gettimeofday.html).

However, on the Cortex M platform, we do not have an operating system kernel that is able to provide the current time.
Therefore, this function is nowhere to be found in the platform specific version of the newlib C standard library (`libg_nano.a` in this case).

If you are wondering, why the linker did not complain about the function `gettimeofday`, but `_gettimeofday`, you can find the answer in the section [Why did the linker not complain about gettimeofday, but _gettimeofday?](#why-did-the-linker-not-complain-about-gettimeofday-but-_gettimeofday).

# Implementing _gettimeofday to make std::chrono work
> For the example implementation, I am using the Nucleo-STM32G474RE board from STMicroelectronics. If you just want to look at the result, you can find the code in my [embedded_template repository](https://github.com/devzeb/embedded_template) at the branch named `gettimeofday-stm32g4`. Look at the files `embedded_template/gettimeofday.cpp` and `embedded_template/stm32g4xx_hal_timebase_tim.cpp`.

Now that we know that `_gettimeofday` is missing to make std::chrono::steady_clock::now() work, we can implement it ourselves.

From the documentation of `gettimeofday` in the [POSIX standard](https://pubs.opengroup.org/onlinepubs/9699919799.2018edition/functions/gettimeofday.html), we can find out what the function is supposed to do:

> DESCRIPTION
>
> The gettimeofday() function shall obtain the current time, expressed as seconds and microseconds since the Epoch, and store it in the timeval structure pointed to by tp. The resolution of the system clock is unspecified.
>
>RETURN VALUE
>
>The gettimeofday() function shall return 0 and no value shall be reserved to indicate an error.

This means we have to do two things in our implementation of `_gettimeofday`:
- modify the pointer struct timeval to contain the current time since the epoch
- return 0 to indicate success

The timezone struct is defined in the include file `sys/_timeval.h` (in the newlib repository: `newlib/libc/include/sys/_timeval.h`):

```c
/*
 * Structure returned by gettimeofday(2) system call, and used in other calls.
 */
struct timeval {
	time_t		tv_sec;		/* seconds */
	suseconds_t	tv_usec;	/* and microseconds */
};
```

So the `struct timeval` contains both seconds and microseconds since the epoch. 
The epoch is usually defined as January 1st, 1970, 00:00:00 UTC. 
In our case though, we have no way of knowing the current date, so we have to define the epoch as the start of the system.
Therefore, we have to obtain the number of seconds and microseconds since the start of the system.

## Obtaining the number of seconds and microseconds since the start of the system
Two possible ways of counting the time since the start of the system are:
- use a timer peripheral together with an interrupt to increment a "time counter" variable
- use a real time clock (RTC) peripheral

As most Cortex M microcontrollers do include a timer peripheral, I chose to use the interrupt based approach.

## Implementation
To increment the "time counter" variable on our bare metal system, we have to:
- Implement a "time counter" function that increments the "time counter" variable
- Initialize the timer to issue an interrupt every with a fixed interval (e.g. 1ms)
- Implement an interrupt handler to call the function.

When your system uses a RTOS, you can even use the already existing RTOS tick function to call the "time counter" function.

## Implementing the "time counter" function
```cpp
namespace 
{
  // use the type of timeval::tv_usec to make sure that the type is compatible with the struct timeval
  using TMicroseconds = decltype(std::declval<timeval>().tv_usec);

  // not using TSeconds = decltype(std::declval<timeval>().tv_sec) because it's internally a long long (int64_t), which is not compatible with std::atomic on the cortex m platform
  using TSeconds = std::uint32_t; //  4294967295 seconds = 136,1 years -> max uptime of 136 years is enough

  // updated from the timer interrupt, so the variables need to be atomic
  std::atomic<TSeconds> seconds_passed{0};
  std::atomic<TMicroseconds> microseconds_passed{0};

  // make sure that the atomics can be used without support for locks
  static_assert(decltype(seconds_passed)::is_always_lock_free, "seconds_passed is not lock free");
  static_assert(decltype(microseconds_passed)::is_always_lock_free, "microseconds_passed is not lock free");
}

void on_microseconds_passed(const TMicroseconds microseconds)
{
  static constexpr auto microseconds_per_second = 1000 * 1000;

  // use a non atomic variable to store the result, instead of re-reading the atomic, which is more expensive
  const auto result = microseconds_passed += microseconds;

  // if a second has passed, increment the seconds counter
  if (result >= microseconds_per_second)
  {
    seconds_passed += 1;
    // adapt the microseconds counter to make sure that seconds_passed + microseconds_passed is always the current time
    microseconds_passed -= microseconds_per_second;
  }
}
```
I chose to use two counter variables for seconds and microseconds, because it is required by the struct timeval.
The variables are updated from the timer interrupt, so they need to be atomic.
I also made sure that the atomics can be used without support for locks, because the Cortex M platform does not support locks out of the box. In order to achieve this, the type of TSeconds must not be larger than 32 bits, because the Cortex M platform does not support atomic operations on 64 bit variables.
Therefore, that max number of seconds that can be counted is 4294967295 seconds, which is equal to 136,1 years. This is enough for my particular use case.

### Implementing the `gettimeofday` function
With this code in place, we can now implement the `_gettimeofday` function:

```cpp
extern "C" {
  int _gettimeofday(timeval* ptimeval,
                    [[maybe_unused]] void* ptimezone)
  {
    ptimeval->tv_sec = seconds_passed;
    ptimeval->tv_usec = microseconds_passed;
    return 0;
  }
}
```
This implementation is trivial. We just fill the required variables of the struct timeval with our counter variables and return 0 to indicate success.

### Implementing the timer interrupt
In order to implement the timer interrupt, I use the [stm32g4xx_hal library](https://github.com/STMicroelectronics/stm32g4xx_hal_driver) from STMicroelectronics. The hal provides an [example implementation](https://github.com/STMicroelectronics/stm32g4xx_hal_driver/blob/master/Src/stm32g4xx_hal_timebase_tim_template.c) of a timer interrupt, that is executed every 1ms.

We just have to call the `on_microseconds_passed` function from the interrupt handler:

```cpp
// note: even tough the original implementation is in C, I am using C++
extern "C" {
  void TIM6_DAC_IRQHandler()
  {
    HAL_TIM_IRQHandler(&TimHandle);

    static constexpr auto microseconds_per_millisecond = 1000;
    
    // call the function that increments the "time counter" variable
    on_microseconds_passed(microseconds_per_millisecond);
  }
}
```

I am using the "on_microseconds_passed" function, even though the timer interrupt is called every 1ms. This is so you can adapt this example to your needs, e.g. if you have a sub-millisecond resolution for your particular use case.

## Full source code example
As I mentioned before, you can find the full source code example in my [embedded_template repository](https://github.com/devzeb/embedded_template) at the branch named `gettimeofday-stm32g4`. Look at the files `embedded_template/gettimeofday.cpp` and `embedded_template/stm32g4xx_hal_timebase_tim.cpp` from the repository root.

# Conclusion
With everything implemented, the adaption is complete. I can now use `std::chrono::steady_clock::now()` on my Cortex M microcontroller:
  
```cpp
#include "stm32g4xx_hal_timebase_tim.h"
#include <chrono>

int main() {
  // initialize the timer to update every 1ms and then call on_microseconds_passed(1000) in the interrupt handler
  if (embedded_template_HAL_InitTick(0) != HAL_OK) {
    return 1;
  }
  
  while (true) {
    auto now_steady = std::chrono::steady_clock::now();
    auto seconds = std::chrono::duration_cast<std::chrono::seconds>(now_steady.time_since_epoch()).count();
    if (seconds >= 5) {
      // exit the program after 5 seconds
      break;
    }
  }

  return 0;
}
```

# Additional information
## What is libg_nano.a?
According to this site: https://www.sourceware.org/legacy-ml/newlib/2011/msg00647.html
- `libg` is a debugging-enabled libc.
- `libc` is the C standard library.

Therefore, `libg_nano.a` is a debugging-enabled libc from the Newlib Nano project.

Newlib can be found here: https://sourceware.org/newlib/

> Newlib is a C library intended for use on embedded systems.

The Newlib Nano project is a subset of the newlib project, providing the C standard library for embedded systems.


## Why did the linker not complain about `gettimeofday`, but `_gettimeofday`? 
This has to do with the fact, that newlib provides a reentrant version of the C standard library functions.
From [this section of the newlib documentation](https://sourceware.org/newlib/libc.html#Reentrancy).
> Reentrancy is a characteristic of library functions which allows multiple processes to use the same address space with assurance that the values stored in those spaces will remain constant between calls. The Red Hat newlib implementation of the library functions ensures that whenever possible, these library functions are reentrant. However, there are some functions that can not be trivially made reentrant. Hooks have been provided to allow you to use these functions in a fully reentrant fashion.
> 
> These hooks use the structure _reent defined in reent.h. A variable defined as ‘struct _reent’ is called a reentrancy structure. All functions which must manipulate global information are available in two versions. The first version has the usual name, and uses a single global instance of the reentrancy structure. The second has a different name, normally formed by prepending ‘_’ and appending ‘_r’, and takes a pointer to the particular reentrancy structure to use.

This is the reason why the system call `gettimeofday` is actually implemented as a trampoline function to the reentrant version `_gettimeofday_r`, which then calls the actual system call function `_gettimeofday`.

To sum it up: Newlib renamed the actual system call function from `gettimeofday` to `_gettimeofday` to enable the function to be reentrant.
Both `gettimeofday` and `_gettimeofday` have the same signature and the documentation of `gettimeofday` is also valid for `_gettimeofday`.

If you want more information about this, look at the section [Proof that the _gettimeofday is the actual system call function](#proof-that-the-_gettimeofday-is-the-actual-system-call-function).



## Proof that the _gettimeofday is the actual system call function

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
