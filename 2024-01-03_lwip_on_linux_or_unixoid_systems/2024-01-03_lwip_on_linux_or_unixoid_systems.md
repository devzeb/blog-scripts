# lwIP on Linux or Unix systems

# Disclaimer
I have not tested this port on another system than Linux, therefore I cannot guarantee that it will work on any Unix based system.
The name of the port is *unix*, so I expect the steps described here to work on any Unix based system.

# Motivation
Usually, lwIP is running on an embedded device. 
If you have ever worked with embedded systems, you are probably familiar with the fact that it is not always easy to debug and test your code on the target system.

Thankfully, lwIP can also be used on Linux or other Unix based systems.
The *STABLE-2_2_0_RELEASE* tag of the [lwIP GitHub repository](https://github.com/lwip-tcpip/lwip/tree/STABLE-2_2_0_RELEASE) contains a Unix port of lwIP, located in the *contrib/ports/unix* directory.


# How to use lwIP on Linux or other Unix based systems
The following steps are required to make it work:
- make 

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
