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

# Connect to the external network
Let's try to ping external network from netns1;
```
# ip netns exec netns1 route
[root@bali-cdn-1 ~]# ip netns exec netns1 ping 8.8.8.8
connect: Network is unreachable
```
It's unreachable, and let's check the route:
```
# ip netns exec netns1 route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.1.1.0        0.0.0.0         255.255.255.0   U     0      0        0 veth1
```
We can see there is no default GW, so let's add the default GW to the veth0 in the local host machine:
```
# ip netns exec netns1 route add default gw 10.1.1.2
# ip netns exec netns1 route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.1.1.2        0.0.0.0         UG    0      0        0 veth1
10.1.1.0        0.0.0.0         255.255.255.0   U     0      0        0 veth1
```

But when we ping outside, it's not reachable:
```
# ip netns exec netns1 ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 38ms
```

And let's have a quick debug, why ping not work:
First check whether the veth0 get the packets:
```
# tcpdump -i veth0 -n
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
08:05:10.029478 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 7122, seq 169, length 64
08:05:11.053404 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 7122, seq 170, length 64
08:05:12.077443 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 7122, seq 171, length 64
08:05:13.101462 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 7122, seq 172, length 64
08:05:14.125370 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 7122, seq 173, length 64
08:05:15.149409 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 7122, seq 174, length 64
08:05:16.173384 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 7122, seq 175, length 64
08:05:17.197434 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 7122, seq 176, length 64
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
```
With the tcpdump, we can see the ICMP packets were sent to the veth0.

Then check the route table on the host machine:
```
# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         163.68.67.225   0.0.0.0         UG    101    0        0 eth1
10.1.1.0        0.0.0.0         255.255.255.0   U     0      0        0 veth0
...
```
From the routing table, we can see the packet should be sent out from the eth1, so let's dump the eth1 packets with the `host 10.1.1.1` to only capture the packets with source IP address is `10.1.1.1`:
```
# tcpdump -i eth1 -n host 10.1.1.1
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
```
There is no packets from `10.1.1.1`. So the packets were lost before routing. 

By default, the Linux kernel has a variable named `ip_forward` that keeps this value. It is accessible using the file `/proc/sys/net/ipv4/ip_forward`. The default value is 0 which means no IP Forwarding, because a regular user who runs a single computer without further components is not in need of that, usually.

So let's check whether the `ip_forward` enabled:
```
# sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0
```
0 means the `ip_forward` is disabled, so let's enable it:
```
# sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
```

And in `IPtable filter table FORWARD chain`, the default behavior is `DROP`, so we need to allow the traffic forwarding between the `veth0` and `eth1`:
```
# iptables -t filter -A FORWARD -i eth1 -o veth0 -j ACCEPT
# iptables -t filter -A FORWARD -o eth1 -i veth0 -j ACCEPT
```

Then let's capture the packets of `eth1`:
```
# tcpdump -i eth1 -n host 10.1.1.1
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
08:58:43.917472 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 65312, seq 25, length 64
08:58:44.941456 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 65312, seq 26, length 64
08:58:45.965521 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 65312, seq 27, length 64
08:58:46.989437 IP 10.1.1.1 > 8.8.8.8: ICMP echo request, id 65312, seq 28, length 64
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```
Now we can see the packets out of the `eth1`.