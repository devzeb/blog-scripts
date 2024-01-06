# lwIP on Linux or Unix systems

# TL;DR
> If you are just interested in looking at the working code, see the [full source code example repository](https://github.com/devzeb/example_lwip_linux_unix) on my GitHub.

To make lwIP work on Linux or other Unix based systems, the following steps are required:
- compile lwIP with a configuration file that is suitable for Unix systems
- initialize lwIP
- use the lwIP supplied code to create a TAP network interface on the host system and register it as the default network interface in lwIP


# Was this article helpful?
If you like this article and want to support me, you can do so by buying me a coffee, pizza or other developer essentials by clicking this link:
[Support me with PayPal](https://www.paypal.com/donate/?hosted_button_id=TGDGATFR63N3G)

# Disclaimer
I have not tested this port on another system than Linux, therefore I cannot guarantee that it will work on any Unix based system.
The name of the port folder in the official repository contains the word *unix*, so I expect the steps described here to work on any Unix based system.


# Motivation
[Lightweight IP (lwIP)](https://savannah.nongnu.org/projects/lwip/) is a network stack that is usually running on embedded devices. 
If you have ever worked with embedded systems, you are probably familiar with the fact that it is not always easy to debug and test your code on the target system.

Thankfully, lwIP can also be used on Linux or other Unix based systems.
This is very useful, as you can test and debug using all the tools that are available on your host system (e.g. sanitizers, better logging, etc.).

In this article, I will show you how to make lwIP work on Linux.

# Environment
- Linux
- Clang 16.0.6 (any other compiler should work as well)
- lwIP version 2.2.0

# How to use lwIP on Linux or other Unix based systems
The *STABLE-2_2_0_RELEASE* tag of the [lwIP GitHub repository](https://github.com/lwip-tcpip/lwip/tree/STABLE-2_2_0_RELEASE) contains a Unix port of lwIP, located in the *contrib/ports/unix* directory.

The following steps are required to make it work:
- Configure lwIP by supplying the configuration file `lwipopts.h`
- Initialize lwIP
- Configure a TAP network interface and register it as the default network interface in lwIP
- Set the interface up to start operation

# Implementation


## Configure lwIP by supplying the configuration file `lwipopts.h`
I created a `lwipopts.h` file based on the example code from the Unix port.
This file contains all the configuration options that are required to make lwIP work on Linux.
For more information about the example configuration from lwIP and my modifications, see [this section below](#my-lwipoptsh-options-file-for-compiling-lwip).

## Initialize lwIP
To initialize lwIP, we have to call `tcpip_init` in the `main` function:
```cpp
int main()
{
    // initialize lwIP
    // use lcpip_init instead of lwip_init because we are not using lwIP's NO_SYS mode
    tcpip_init(nullptr, nullptr);
    ...
}
```

As we are using lwIP with an operating system, we have to call `tcpip_init` instead of `lwip_init` according to [this comment](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/src/core/init.c):

```c
/**
 * @ingroup lwip_nosys
 * Initialize all modules.
 * Use this in NO_SYS mode. Use tcpip_init() otherwise.
 */
void lwip_init(void){
    ...
}
```

## Configure a TAP network interface and register it as the lwIP default network interface

lwIP needs a (default) network interface to in order to send and receive network packets.
On Linux / Unix, the lwIP port uses a so-called *TAP network interface* to communicate with the host system.

From [Wikipedia](https://en.wikipedia.org/wiki/TUN/TAP):

> Packets sent by an operating system via a TUN/TAP device are delivered to a user space program which attaches itself to the device. A user space program may also pass packets into a TUN/TAP device. In this case the TUN/TAP device delivers (or "injects") these packets to the operating-system network stack thus emulating their reception from an external source.

I will further explain the effects of adding a TAP interface on the host system in [this section below](#what-happens-on-the-host-system-when-you-create-a-tap-network-interface). But first, let's continue with the implementation.


You can register multiple interfaces in lwIP, but only one can be the default interface, which is used for any outgoing packets that do not match a specific (better suited) route.
Therefore, we need to add a network interface to lwIP and make it the default interface.

In order to register the default network interface in lwIP, two steps are required:
- initialize the interface by calling `netif_add`
- set the initialized interface as the default network interface by calling `netif_set_default`



### Implement the network interface initialization function
I used [this example code from the Unix port](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/example_app/default_netif.c) and adapted it to my needs.

```cpp
void init_default_netif(const ip4_addr_t* ipaddr, 
                        const ip4_addr_t* netmask, 
                        const ip4_addr_t* gw)
{
    netif_add(
        &default_network_interface,
        ipaddr, 
        netmask, 
        gw, 
        nullptr, 
        tapif_init, // initialize this interface as a TAP interface
        tcpip_input // NO_SYS = 0, -> this function is the correct callback
        );

    netif_set_default(&default_network_interface);
}
```

As you can see, we are passing `tapif_init` as the network initialization function parameter (`netif_init_fn`).
This function is invoked by lwIP as soon as the network interface needs to be initialized.

When `tapif_init` is called, multiple things happen:
- A new TAP interface is created on the host system. \
(This operation requires the application to run with root privileges, which is a security concern. There is also the option of creating the TAP device (with root privileges) before starting the application. See [this section](#create-the-tap-device-before-starting-the-application-make-the-application-not-require-root-privileges) for more information.) \
On Linux, the default device-file for the interface is `/dev/net/tun`, and it has the interface name `tap0`. The default settings can be changed by defining the macros `DEVTAP_DEFAULT_IF` and `DEVTAP` in either `lwipopts.h` or using compiler definitions (through the build system).
- A new thread is started that continuously reads from the TAP device (= polling) and passes the received data to lwIP by calling `tapif_input`.


The implementation of the mechanisms described above can be seen in [this file of the port](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/port/netif/tapif.c).


#### Make the file c++ compatible
Unfortunately, the example code of the latest stable branch 2.2.0 of lwIP is not C++ compatible. I submitted a [pull request](https://github.com/lwip-tcpip/lwip/pull/26) in order to properly fix this issue, but until then, we have to fix it manually in the user code by adding a `extern "C"` block around the include header:
```cpp
extern "C" {
    #include "netif/tapif.h"
}
```

If we don't do this, and we include the header in a C++ file, we get the following error:
```
lwip_linux_unix_tap_network_interface.cpp:(.text+0x2f): undefined reference to `tapif_init(netif*)'
```
The reason for this is, that the compiler thinks `tapif_init` is a function with C++ linkage, because its header file with the function declaration does not contain a `extern "C"` block.
The function definition is in a C file, so it actually has C linkage. This makes the linker think that the function is not defined at all, because it is looking for a function with C++ linkage.

#### Remove the other functions from the example code
The other functions `default_netif_poll` and `default_netif_shutdown` from the [example code from the Unix port](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/example_app/default_netif.c) are actually not used by lwIP. It seems that they are dead code, so I just removed them for this example.

### Invoke the network interface initialization function
Now that we have implemented the network interface initialization function, we need to invoke it.
We have to provide the IP address, default gateway and netmask of the network interface, which is explained in the next section.
I did this in the `main` function of my application:
```cpp
int main()
{
    // ip address of this application using a TAP interface
    // use this ip address to talk to this application from the host
    // (e.g. ping this address to check if lwip is alive)
    static constexpr std::array<uint8_t, 4> ip_address_data = {192, 168, 115, 2};

    // gateway address of the TAP interface, which is created by calling init_default_netif()
    static constexpr std::array<uint8_t, 4> default_gateway_data = {192, 168, 115, 1};

    // netmask of the TAP interface, which is created by calling init_default_netif()
    static constexpr std::array<uint8_t, 4> netmask_data = {255, 255, 255, 0};


    ip4_addr_t ipaddr;
    ip4_addr_set_u32(
        &ipaddr,
        ip_address_data[0] << 0u | 
        ip_address_data[1] << 8u | 
        ip_address_data[2] << 16u | 
        ip_address_data[3] << 24u);

    ip4_addr_t gw;
    ip4_addr_set_u32(
        &gw, 
        default_gateway_data[0] << 0u | 
        default_gateway_data[1] << 8u | 
        default_gateway_data[2] << 16u |
        default_gateway_data[3] << 24u);

    ip4_addr_t netmask;
    ip4_addr_set_u32(
        &netmask, 
        netmask_data[0] << 0u | 
        netmask_data[1] << 8u | 
        netmask_data[2] << 16u | 
        netmask_data[3] << 24u);


    LOCK_TCPIP_CORE();
    // create the TAP interface on the unix host
    init_default_netif(&ipaddr, &netmask, &gw);
    ...
    UNLOCK_TCPIP_CORE();
    ...
}
```

I am using a std::array to display the IP address, default gateway and netmask in a human-readable format.

The function `ip4_addr_set_u32` is used to fill the actual address from the respective array.

## Set the interface up to start operation
Finally, we enable the interface in lwIP by calling `netif_set_up`:
```cpp
int main()
{
    ...
    LOCK_TCPIP_CORE();
    ...

    // set the interface up to start operation
    netif_set_up(&get_default_netif());
    UNLOCK_TCPIP_CORE();

    ...
}
```

This makes lwIP start sending and receiving packets through the TAP interface.

# What happens on the host system when you create a TAP network interface?
You might be wondering why we have to specify an IP address, default gateway and netmask, even though our host already (probably) has a network configuration?

This is because we are creating the TAP interface, which serves as a completely separate network interface on the host system.

If we execute the code shown above and look at the network interfaces, we can see the following information:
```
$ ip addr show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    ...
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    ... // this is the ethernet interface of my host system
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    ...
11: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 1000
    link/ether 1a:6e:e3:96:fb:5a brd ff:ff:ff:ff:ff:ff
    inet 192.168.115.1/24 brd 192.168.115.255 scope global tap0
       valid_lft forever preferred_lft forever
    inet6 fe80::186e:e3ff:fe96:fb5a/64 scope link 
       valid_lft forever preferred_lft forever
```

As you can see, the TAP interface named `tap0` has been created and is using the gateway address *192.168.115.1*, which we set in the `main` function of our application.
The broadcast address *255.255.255.0* of the TAP interface also matches our configured netmask.

You can also see that the actual IP address of our application is not shown in the output of `ip addr show`. This is expected, because the command only shows the network interface, but not any hosts on these networks.

To sum it up: When the application creates the TAP interface, it is added to the host system. 
This makes our system know a route to the network of the TAP interface, which is *192.168.115.0/24* in our case.
The host will send any IP-packets that are addressed to this network through the TAP interface. 
Therefore, when we send a packet to the IP address of our application, which is part of the new network, the host will send it through the TAP interface, which is then received by our application.

Keep in mind that the gateway address (and netmask) for this interface must not be used by any other network interface on the host system, otherwise your network configuration will be broken.

# Check if it works

After performing all initialization steps, our application is ready to send and receive network packets through its IP address in our new TAP network (192.168.115.2).
We can test this by sending ping packets to our application:
```
$ ping 192.168.115.2

PING 192.168.115.2 (192.168.115.2) 56(84) bytes of data.
64 bytes from 192.168.115.2: icmp_seq=1 ttl=255 time=0.452 ms
64 bytes from 192.168.115.2: icmp_seq=2 ttl=255 time=0.280 ms
```

As we enabled debugging in our lwIP configuration, we can also observe that the lwIP stack in our application responds to the ping packets.

```
ip4_input:
IP header:
+-------------------------------+
| 4 | 5 |  0x00 |        84     | (v, hl, tos, len)
+-------------------------------+
|    48836      |010|       0   | (id, flags, offset)
+-------------------------------+
|   64  |    1  |    0x1490     | (ttl, proto, chksum)
+-------------------------------+
|  192  |  168  |  115  |    1  | (src)
+-------------------------------+
|  192  |  168  |  115  |    2  | (dest)
+-------------------------------+ 
...
icmp_input: ping
...
ip4_output_if: tp0
IP header:
+-------------------------------+
| 4 | 5 |  0x00 |        84     | (v, hl, tos, len)
+-------------------------------+
|    48836      |010|       0   | (id, flags, offset)
+-------------------------------+
|  255  |    1  |    0x558f     | (ttl, proto, chksum)
+-------------------------------+
|  192  |  168  |  115  |    2  | (src)
+-------------------------------+
|  192  |  168  |  115  |    1  | (dest)
+-------------------------------+
ip4_output_if: call netif->output()
...
ethernet_output: sending packet 0x55f29ed1a698
```

And there you have it, the lwIP network stack is running on Linux.

# Full source code example
You can find the [full source code example repository](https://github.com/devzeb/example_lwip_linux_unix) on my GitHub.

# Additional information

## Create the TAP device before starting the application (make the application not require root privileges)

As mentioned before, the application needs to be run with root privileges in order to create the TAP device.
This is a security concern, as the application can now do anything on the host system.

To avoid this, we can create the TAP device before starting the application.

The file [tapif.c](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/port/netif/tapif.c) states how to perform this:
```c
/*
 * Creating a tap interface requires special privileges. If the interfaces
 * is created in advance with `tunctl -u <user>` it can be opened as a regular
 * user. The network must already be configured. If DEVTAP_IF is defined it
 * will be opened instead of creating a new tap device.
 *
 * You can also use PRECONFIGURED_TAPIF environment variable to do so.
 */
```

So if we execute the following command, the TAP interface is created:
```shell
$ sudo tunctl -u $USER
Set 'tap0' persistent and owned by uid 1000
```

We can observe this by looking at the network interfaces:
```
$ ip addr show
...
13: tap0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 1a:6e:e3:96:fb:5a brd ff:ff:ff:ff:ff:ff
```

Now we can invoke the application without root privileges by specifying the environment variable `PRECONFIGURED_TAPIF` to contain the TAP interface name shown above:
```shell
$ PRECONFIGURED_TAPIF=tap0 ./example_lwip_linux_unix
```

Note that we still need root privileges to create the TAP interface, but we can now start the application without root privileges, which is far less dangerous.


## My `lwipopts.h` options file for compiling lwIP 
> You can find the complete configuration file [in the repository of this example.](https://github.com/devzeb/example_lwip_linux_unix/blob/main/port_configuration/lwip/lwipopts.h)


[This directory](https://github.com/lwip-tcpip/lwip/tree/STABLE-2_2_0_RELEASE/contrib/ports/unix/lib) of the Unix port contains a README file that states:
> This directory contains an example of how to compile lwIP as a shared library
on Linux.

The directory also contains a `lwipopts.h` file, which is used to configure the lwIP for running on Linux / Unix.
For this reason, it is a good starting point for our own `lwipopts.h` file.

I made the following modifications to the file:

### Remove the includes (as they are probably added by mistake)
In this file, I first removed the includes, as they seem to be only there by mistake.
I believe this is a copy-paste error from the developers, because of this reason: 

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

But in the `lwipopts.h` file of the Unix port, you can find the exact same comment and includes. This hints that the file was probably originally copied from `lwip/opt.h` and then modified to fit the needs of the Unix port. Most likely, the includes were forgotten to be removed.
Also, the other `lwipopts.h` files in the lwIP repository do not contain these includes.

I even had trouble compiling the project with the includes in place, which is another hint that they are not supposed to be there. Therefore, I removed the lines from the code example above.

### Add C++ guards
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

### Enable the system errno header instead of the lwIP supplied one
In the past I used lwIP with the Newlib Nano C standard library. In that use case, it was mandatory to use the *errno header* from Newlib instead of the lwIP supplied one. 
This is because Newlib Nano provided a special implementation of the *errno* variable, which made it thread safe. This mechanism is not present in the lwIP supplied *errno header*.

So as a rule of thumb, I always use the system supplied *errno header* instead of the lwIP supplied one, when it is available.

As my host system (Linux) supplies this header file, I also enabled the system errno header by defining
```cpp
#define LWIP_ERRNO_STDINCLUDE 1
```

### Enable debugging
For demonstration purposes (and to get everything working in the first place), I enabled all the debugging options. 

The only exception is the `DHCP_DEBUG` option, which I kept disabled from experiences in another project, where the ARP announcement failed when the option was enabled.
I might was mistaken then, or the problem could have been fixed in the meantime, so feel free to enable it if you want to try it.
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

## Other files in the lwIP Unix port directory that might be useful to you
When browsing through the repository, I found other files that apparently contain alternative lwIP interfaces to the `tapif.c` file.

- [pcapif.c](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/port/netif/pcapif.c)
- [vdeif.c](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/port/netif/vdeif.c)

### pcapif
[This README](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/README) contains the following description of the `pcapif`:
> pcapif: Network interface that replays packages from a PCAP dump file, and
    discards packages sent out from it

This sounds very useful for testing purposes, but it seems to be disabled for Linux. 
At least [the pcapif.c source file](https://github.com/lwip-tcpip/lwip/blob/STABLE-2_2_0_RELEASE/contrib/ports/unix/port/netif/pcapif.c) contains the following comment:
```
#ifndef linux  /* Apparently, this doesn't work under Linux. */
``` 

### vdeif
This network interface relates to *VirtualSquare*, a project of the [University of Bologna (Italy)](https://www.unibo.it/en).

According to [this README](https://github.com/virtualsquare/view-os/blob/master/lwipv6/LWIPv6_Programming_Guide).

> LWIPv6 stacks communicate using three different types of interfaces:
> * ...
> * vde: it gets connected to a Virtual Distributed Ethernet switch. 

This gives us more information about the terms:
- `vde` means `Virtual Distributed Ethernet` 
- `vdeif` means `Virtual Distributed Ethernet interface`.

But what is `Virtual Distributed Ethernet`?

According to [this PDF document](http://www.cs.unibo.it/~renzo/virtualsquare/V2.201101.pdf):

> Virtual Distributed Ethernet is the V2 Virtual Networking project. VDE is a
Virtual Ethernet, whose nodes can be distributed across the real Internet. The
idea of VDE sums up VPN, tunnel, Virtual Machines interconnection, overlay
networking, as all these different entities can be implemented by VDE.

And also:

> VDE is an Ethernet-compliant, virtual network, able to interconnect virtual
and real machines in a distributed fashion, even if they are hosted on different physical hosts.

Additional references:
- [Virtual Square Wiki](http://wiki.virtualsquare.org/#/?id=virtual-distributed-ethernet)

- [Debian packages managed by Virtual Square](https://qa.debian.org/developer.php?login=virtualsquare@cs.unibo.it)
