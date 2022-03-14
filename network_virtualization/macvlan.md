# What is MACVLAN

MACVlan is a very lightweight driver, because rather than using any Linux bridging or port mapping, it connects interfaces directly to host interfaces.

MACVLAN allows you to define virtual interfaces on the physical interface (we call this parent interface), and each virtual interface can have their owen MAC address. And then you can configure IP address on macvlan interfaces to communicate between them.

![MACVLAN Description](../pic/macvlan%20-%20description.png)

The VMs or containers that are connected by the macvlan interfaces that under the same parent interface are under one L2 network, and they share the same broadcasting zone.

Comparing to the Linux `Bridge`, it does not need bridge in stack, so the configurations is much easier and the network efficient is much better than the bridge.

## Three modules of MACVLAN

### VEPA（Virtual Ethernet Port Aggregator）

In the VEPA mode, all the traffic coming from the MACVLAN interfaces are all sent out from the parent interface to the outside switch, no matter what's the traffic destinations, even it's destination is to the their parent interface. The MACVLAN interfaces can not communicate with each other even they're bounded to the same parent interface.

![VEPA](../pic/macvlan%20-%20vepa.png)

NOTE: In this mode, the outside switch needs the `hairpin` support, which makes traffic that the source and destination mac addresses are from the same port and send the traffic back to this port.

### Bridge

In the Bridge mode, the virtual MACVLAN interfaces that bounded to the same parent interface can directly communicate with each other. It does not need the traffic sent outside the parent interface. But the communication between the virtual interface that bounded on different parent interface still need to go outside of the parent interfaces.

![Bridge](../pic/macvlan%20-%20bridge.png)

NOTE: In the bridge mode, if the parent interface is down, then all the children virtual interfaces are also down.

### Private

In the Private mode, all the traffic between the virtual interfaces that are bounded on the same parent interfaces are blocked, even the outside switch supports the `hairping`.

![Private](../pic/macvlan%20-%20private.png)

### Passthrough

In the Passthrough mode, one parent interface can only created one sub virtual interface. The sub virtual interface is bounded to its parent interface, and has the same MAC address as the parent interface.

![Passthrough](../pic/macvlan%20-%20passthrough.png)

## Practice

TBD