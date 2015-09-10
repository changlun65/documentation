Linux Network Configuration
===========================

## Usbnet networking

* Host USB Network Configuration

    Configuring the host as a gateway If your host has no firewalling rules, you can set the gateway rules by modifying /etc/interfaces file. 
    ```
    auto usb0
    iface usb0 inet static
    address 192.168.100.120
    netmask 255.255.255.0
    up iptables -A POSTROUTING -t nat -s 192.168.100.0/24 -j MASQUERADE
    up iptables -P FORWARD ACCEPT
    up sysctl -w net.ipv4.ip_forward=1
    ```
    Please remember to do the following configuration on device: 
    ```
    route add default gw 192.168.100.120
    echo "nameserver 192.168.100.120" > /etc/resolv.conf
```

* How to mount samba network drive.
	1. Install smbfs package.
	```apt-get install smbfs```
    2. Mount samab drive by smbmount
    ```smbmount //smb-server mount-point -o username=xxx,password=xxx,domain=xxx,rw,noperm```