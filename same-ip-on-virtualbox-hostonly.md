# Issue description
In virtualbox, I cloned several Ubuntu OS VMs with link. I added hostonly network interface for each cloned VM. But all the IP address on the hostonly interface are the same, even the MAC addresses are different. 

# Solution
Change the machine ID on VMs:
```
sudo rm -f /etc/machine-id
sudo dbus-uuidgen --ensure=/etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo dbus-uuidgen --ensure
reboot
```
