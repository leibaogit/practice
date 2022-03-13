# Nmcli

nmcli is a command-line tool for controlling NetworkManager and reporting network status. It can be utilized as a replacement for nm-applet or other graphical clients. nmcli is used to create, display, edit, delete, activate, and deactivate network connections, as well as control and display network device status

<https://developer.gnome.org/NetworkManager/stable/nmcli.html>

## Terms

- Connection: Collections of data (Layer2 details, IP addressing, etc.) that describe how to create or connect to a network.

## To view the available interfaces on the system, issue a command as follows

```bash
$ nmcli con show
NAME         UUID                                  TYPE            DEVICE
System enp2s0  9c92fad9-6ecb-3e6c-eb4d-8a47c6f50c04  802-3-ethernet  enp2s0
System enp1s0  5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03  802-3-ethernet  enp1s0
```

## To create an 802.1Q VLAN interface on Ethernet interface enp1s0, with VLAN interface VLAN10 and ID 10, issue a command as follows

```bash
$ nmcli con add type vlan ifname VLAN10 dev enp1s0 id 10
Connection 'vlan-VLAN10' (37750b4a-8ef5-40e6-be9b-4fb21a4b6d17) successfully added.
```

No con-name was given for the VLAN interface, the name was derived from the interface name by prepending the type, so here we can get the connection name as `vlan-VLAN10`:

```bash
$ nmcli con show
NAME                 UUID                                  TYPE      DEVICE
vlan-VLAN10          22bb0f91-fcb7-46fd-9877-ea9a6dfc68e0  vlan      --
```

And a network profile/script is created:

```bash
$ cat /etc/sysconfig/network-scripts/ifcfg-vlan-VLAN10
VLAN=yes
TYPE=Vlan
PHYSDEV=enp1s0
VLAN_ID=10
REORDER_HDR=yes
GVRP=no
MVRP=no
HWADDR=
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=vlan-VLAN10
UUID=22bb0f91-fcb7-46fd-9877-ea9a6dfc68e0
DEVICE=VLAN10
ONBOOT=yes
```

You can also specify a connection name with the con-name option as follows:

```bash
$ nmcli con add type vlan con-name VLAN12 dev enp1s0 id 12
Connection 'VLAN12' (b796c16a-9f5f-441c-835c-f594d40e6533) successfully added.
```

## Update a connection

```bash
$ nmcli connection modify vlan-VLAN10 802.mtu 1496
```

## view all the parameters associated

```bash
$ nmcli connection show vlan-VLAN10
```


# Delete a bridge
nmcli connection delete id bridge-br606
