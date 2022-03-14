# What is MACVLAN

MACVlan is a very lightweight driver, because rather than using any Linux bridging or port mapping, it connects interfaces directly to host interfaces.

MACVLAN allows you to define virtual interfaces on the physical interface (we call this parent interface), and each virtual interface can have their owen MAC address. And then you can configure IP address on macvlan interfaces to communicate between them.

MACVLAN Description

The VMs or containers that are connected by the macvlan interfaces that under the same parent interface are under one L2 network, and they share the same broadcasting zone.

Comparing to the Linux `Bridge`, it does not need bridge in stack, so the configurations is much easier and the network efficient is much better than the bridge.

## Three modules of MACVLAN

