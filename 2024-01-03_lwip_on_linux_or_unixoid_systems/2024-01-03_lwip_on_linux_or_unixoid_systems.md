# lwIP on Linux or Unix systems

# Disclaimer
I have not tested this port on another system than Linux, therefore I cannot guarantee that it will work on any Unix based system.
The name of the port is *unix*, so I expect the steps described here to work on any Unix based system.



# Motivation
Usually, lwIP is running on an embedded device. 
If you have ever worked with embedded systems, you are probably familiar with the fact that it is not always easy to debug and test your code on the target system.

Thankfully, lwIP can also be used on Linux or other Unix based systems.
The *STABLE-2_2_0_RELEASE* tag of the [lwIP GitHub repository](https://github.com/lwip-tcpip/lwip/tree/STABLE-2_2_0_RELEASE) contains a Unix port of lwIP, located in the *contrib/ports/unix* directory.

# Environment
- Linux

# How to use lwIP on Linux or other Unix based systems
The following steps are required to make it work:
- Make lwIP compile by supplying the configuration file `lwipopts.h`
- Configure a TAP network interface and register it as the default network interface in lwIP
- Set the interface up to start operation

# Implementation
## Make lwIP compile by supplying the configuration file `lwipopts.h`
### Creating / choosing a `lwipopts.h` file
[This directory](https://github.com/lwip-tcpip/lwip/tree/STABLE-2_2_0_RELEASE/contrib/ports/unix/lib) of the Unix port contains a README file that states:
> This directory contains an example of how to compile lwIP as a shared library
on Linux.

The directory also contains a `lwipopts.h` file, which is used to configure the lwIP shared library.
For this reason, it is a good starting point for our own `lwipopts.h` file.

I made the following modifications to the file:

#### Remove the includes (as they are probably added by mistake)
Of this file, I first removed the includes, as they seem to be only there by mistake.

The reason why I think it is a copy-paste mistake from the developers is: 

Normally in lwIP, the global include file `lwip/opt.h` includes the user-specific configuration `lwipopts.h`.
In `lwip/opt.h`, you can even find the following comment and includes:
```cpp
/*
 * Include user defined options first. Anything not defined in these files
 * will be set to standard values. Override anything you don't like!
 */
#include "lwipopts.h"
#include "lwip/debug.h"
```

But in the `lwipopts.h` file of the Unix port, you can find the exact same comment and includes. This hints that the file probably originally copied from `lwip/opt.h` and then modified to fit the needs of the Unix port. Most likely, the includes were forgotten to be removed.
I even had trouble compiling the project with the includes in place, which is another hint that they are not supposed to be there. Therefore, I removed the lines from the code example above.

#### Add C++ guards
To be compatible with C++ code, I added the default C++ guards:
```cpp
... // include guard

#ifdef __cplusplus
extern "C" {
#endif

... // file content

#ifdef __cplusplus
}
#endif

```

#### Enabled the system errno header instead of the lwIP supplied one
In the past I used lwIP with Newlib Nano, where it was mandatory to use the *errno header* from the standard library instead of the lwIP supplied one.

As my host system (Linux) supplies a standard library, I also enabled the system errno header by defining
```cpp
#define LWIP_ERRNO_STDINCLUDE 1
```

#### Enable debugging
For demonstration purposes (and to get everything working in the first place), I enabled all the debugging options. 
The only exception is the `DHCP_DEBUG` option, which I kept disabled from experiences in another project, where the ARP announcement failed when the option was enabled.
Your mileage may vary, so feel free to enable it if you want to try it.
```cpp
#define LWIP_DEBUG 1
#define LWIP_DBG_MIN_LEVEL LWIP_DBG_LEVEL_ALL
#define PPP_DEBUG LWIP_DBG_ON
#define MEM_DEBUG LWIP_DBG_ON
...
// #define DHCP_DEBUG LWIP_DBG_ON //  WARNING ! do not enable DHCP_DEBUG, this makes the arp announcement fail
... // all the other debug options found in 
#define TCP_RST_DEBUG LWIP_DBG_ON

#define LWIP_DBG_TYPES_ON ( LWIP_DBG_ON | LWIP_DBG_TRACE | LWIP_DBG_STATE | LWIP_DBG_FRESH | LWIP_DBG_HALT )
```




# A little bit of background information about TAP / TUN interfaces

From [Wikipedia](https://en.wikipedia.org/wiki/TUN/TAP):

> Packets sent by an operating system via a TUN/TAP device are delivered to a user space program which attaches itself to the device. A user space program may also pass packets into a TUN/TAP device. In this case the TUN/TAP device delivers (or "injects") these packets to the operating-system network stack thus emulating their reception from an external source.




# Additional information
## Other, apparently unused files that might be useful
When browsing through the repository, I found other files that apparently contain alternative lwIP interfaces to the `tapif.c` file.

- [pcapif.c](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/port/netif/pcapif.c)
- [vdeif.c](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/port/netif/vdeif.c)

### pcapif
[This README](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/README) contains the following description of the `pcapif`:
> pcapif: Network interface that replays packages from a PCAP dump file, and
    discards packages sent out from it

This sounds very useful for testing purposes, but it seems to be disabled for Linux. 
At least [the pcapif.c source file](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/port/netif/pcapif.c) contains the line 
```
#ifndef linux  /* Apparently, this doesn't work under Linux. */
``` 

### vdeif
This network interface relates to *VirtualSquare*, a project of the [University of Bologna (Italy)](https://www.unibo.it/en).

According to [this readme](https://github.com/virtualsquare/view-os/blob/master/lwipv6/LWIPv6_Programming_Guide).

> LWIPv6 stacks communicate using three different types of interfaces:
> * ...
> * vde: it gets connected to a Virtual Distributed Ethernet switch. 

This gives us more information about the terms:
- `vde` means `Virtual Distributed Ethernet` 
- `vdeif` means `Virtual Distributed Ethernet interface`.

But what is `Virtual Distributed Ethernet`?

According to [this pdf document](http://www.cs.unibo.it/~renzo/virtualsquare/V2.201101.pdf):

> Virtual Distributed Ethernet is the V2 Virtual Networking project. VDE is a
Virtual Ethernet, whose nodes can be distributed across the real Internet. The
idea of VDE sums up VPN, tunnel, Virtual Machines interconnection, overlay
networking, as all these different entities can be implemented by VDE.

And also:

> VDE is an Ethernet-compliant, virtual network, able to interconnect virtual
and real machines in a distributed fashion, even if they are hosted on different physical hosts.

Further references:
- [Virtual Square Wiki](http://wiki.virtualsquare.org/#/?id=virtual-distributed-ethernet)

- [Debian packages managed by Virtual Square](https://qa.debian.org/developer.php?login=virtualsquare@cs.unibo.it)
