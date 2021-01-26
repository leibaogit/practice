As we know, the network namespace is an isolated space that independently run the network resources with the root name space, such as IP address, routing table, port, iptables, /proc/net directories and so on. So after you create a namespace, then :
 - How to connect to the root namespace (local machine)?
 - How to connect to the external network?
 - How to connect to other network namespace in the same machine?
 - How to connect to other network namespace in other machine?

This document will show you how to achieve.

# Network namespace
Before start, let's learn something about the network namespace.

I have a CentOS 8.0 virtual machine, and let's create &list a network namespace with these commands:
```
# ip netns add netns1
# ip netns
netns1
```

Actually, after the network namespace, the system mounted it under the `/var/run/netns/`, then it can be easily managed and make sure it can be running even there is no process in it.
```
# ls /var/run/netns/
netns1
```

Then let see what's the network configuration in the net namespace:

```
# ip netns exec netns1 ip link list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

In the namespace, there is only one `lo` interface, it's status is `DOWN`, so this `lo` interface is not reachable:
```
# ip netns exec netns1 ping 127.0.0.1
connect: Network is unreachable
```

Then let's configure this `lo` interface, and make it `UP`:
```
# ip netns exec netns1 ip link set dev lo up
# ip netns exec netns1 ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.072 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.072 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.070 ms
64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.072 ms
64 bytes from 127.0.0.1: icmp_seq=5 ttl=64 time=0.085 ms
^C
--- 127.0.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 121ms
rtt min/avg/max/mdev = 0.070/0.074/0.085/0.007 ms
```
OK, now, this network namespace is UP, and then let's make to reach to others.

# Connect to the local machine
To make the network namespace connect to the local machine, we need create a `Virtual Ethernet`, AKA `veth pair`, which can always make the packet entering one peer to other peer.

## Create `veth pair`:
```
# ip link add veth0 type veth peer name veth1
# ip addr
...
247: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether c2:e1:74:47:0b:0e brd ff:ff:ff:ff:ff:ff
248: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 4a:43:94:97:00:82 brd ff:ff:ff:ff:ff:ff
```
We can see, by default, the veth is added in the root network namespace, and they're both DOWN.

## Assign one peer in the network namespace
```
# ip link set veth1 netns netns1
# ip addr
...
248: veth0@if247: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 4a:43:94:97:00:82 brd ff:ff:ff:ff:ff:ff link-netns netns1

# ip netns exec netns1 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
247: veth1@if248: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether c2:e1:74:47:0b:0e brd ff:ff:ff:ff:ff:ff link-netnsid 0
```
We can see the veth1 is moved into the netns1 space.

## Bring up the veth
```
# ip netns exec netns1 ifconfig veth1 10.1.1.1/24 up
# ifconfig veth0 10.1.1.2/24 up
```

Ping the netns veth interface from the local host machine:
```
# ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.132 ms
64 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=0.062 ms
^C
--- 10.1.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 48ms
rtt min/avg/max/mdev = 0.062/0.097/0.132/0.035 ms
```

Ping the local host machine from the netns1 namespace
```
# ip netns exec netns1 ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=0.083 ms
64 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=0.111 ms
64 bytes from 10.1.1.2: icmp_seq=3 ttl=64 time=0.101 ms
^C
--- 10.1.1.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 83ms
rtt min/avg/max/mdev = 0.083/0.098/0.111/0.014 ms
```
Now both sides are reachable.