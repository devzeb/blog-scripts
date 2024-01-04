# lwIP on Linux or Unix systems

# TL;DR
> If you are just interested in looking at the working code, see the [full source code example repository](https://github.com/devzeb/example_lwip_linux_unix) on my GitHub.

**todo** add steps to make it work

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
> You can find the complete configuration file [in the repository of this example.](https://github.com/devzeb/example_lwip_linux_unix/blob/main/port_configuration/lwip/lwipopts.h)


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

## Configure a TAP network interface and register it as the default network interface in lwIP

lwIP needs a (default) network interface to in order to send and receive network packets.
You can register multiple interfaces in lwIP, but only one can be the default interface, which is used for any outgoing packets that do not match a specific (better suited) route.
Therefore, we need to register a network interface in lwIP and make it default.

In order to register the default network interface in lwIP, two steps are required:
- initialize the interface by calling `netif_add`
- set the initialized interface as the default network interface by calling `netif_set_default`



### Initialize the network interface
I once again used the [example code from the Unix port](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/example_app/default_netif.c) and adapted it to my needs:

#### Implement the network interface initialization function
```cpp
void init_default_netif(const ip4_addr_t* ipaddr, const ip4_addr_t* netmask, const ip4_addr_t* gw)
{
    netif_add(&default_network_interface, ipaddr, netmask, gw, nullptr, tapif_init, tcpip_input);

    netif_set_default(&default_network_interface);
}
```

As you can see, we are passing `tapif_init` as the network initialization function parameter (`netif_init_fn`).

This function is then invoked by lwIP as soon as the network interface needs to be initialized.

When this function is called, multiple things happen:
- A new TAP device (file) is created on the host system. On Linux, the default device is `/dev/net/tun`, and it has the interface name `tap0`. The default settings can be changed by redefining the macros `DEVTAP_DEFAULT_IF` and `DEVTAP`. This operation requires the application to run with root privileges, but there is an alternative way. See [this section](#create-the-tap-device-before-starting-the-application-make-the-application-not-require-root-privileges) for more information.
- A new thread is started that continuously reads from the TAP device (= polling) and passes the received data to lwIP by calling `tapif_input`.

See [this file](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/port/netif/tapif.c) for the implementation of the mechanism described above.

#### Make the file c++ compatible
Unfortunately, the example code in the latest stable branch 2.2.0 of lwIP is not C++ compatible. I submitted a [pull request](https://github.com/lwip-tcpip/lwip/pull/26) in order to properly fix this issue, but until then, we have to fix it manually in the user code by adding a `extern "C"` block around the include header:
```cpp
extern "C" {
    #include "netif/tapif.h"
}
```

if we don't do this, and we include the header in a C++ file, we get the following error:
```
lwip_linux_unix_tap_network_interface.cpp:(.text+0x2f): undefined reference to `tapif_init(netif*)'
lwip_linux_unix_tap_network_interface.cpp:(.text+0x7c): undefined reference to `tapif_poll(netif*)'
```

In addition to that, we also need to add the `extern "C"` block around the `default_netif_poll` and `default_netif_shutdown` functions, as they are called from within the lwip stack.

```cpp
extern "C" {
void default_netif_poll(void)
{
    ...
}

void default_netif_shutdown(void)
{
    ...
}
}
```

#### 





## Set the interface up to start operation





# Additional information
## Create the TAP device before starting the application (make the application not require root privileges)

## A little bit of background information about TAP / TUN interfaces

From [Wikipedia](https://en.wikipedia.org/wiki/TUN/TAP):

> Packets sent by an operating system via a TUN/TAP device are delivered to a user space program which attaches itself to the device. A user space program may also pass packets into a TUN/TAP device. In this case the TUN/TAP device delivers (or "injects") these packets to the operating-system network stack thus emulating their reception from an external source.



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
