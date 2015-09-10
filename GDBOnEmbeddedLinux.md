GDB on Embedded Linux
=====================

## Build GDB with python support
[QtCreator Build Gdb](https://wiki.qt.io/QtCreator_Build_Gdb)
[Qt: Setting Up Debugger](http://doc.qt.io/qtcreator/creator-debugger-engines.html)
[Debugging Webkit](https://trac.webkit.org/wiki/Debugging%20With%20LLDB%20or%20GDB)
[Debugging WebKitGTK+](http://trac.webkit.org/wiki/WebKitGTK/Debugging)

[kdevelop.git](https://quickgit.kde.org/?p=kdevelop.git&a=tree&h=feadb8367534c3dffd24b5d6925439deaa5019f9&hb=1e325646d96700a30b7113f07ae77e014a8dbeb0&f=debuggers/gdb/printers)

## Remote debugging via usbnet
1. Build usbnet module (g_ether.ko) from linux source tree.
	**Device Drivers>USB support>USB Gadget Support>**
    ```
    <*> Support for USB Gadgets
    USB Peripheral Controller (S3C high speed(2.0, dual-speed) USB OTG device) ---> 
    S3C high speed(2.0, dual-speed) USB OTG device 
    <M> USB Gadget Drivers 
    <M> Ethernet Gadget (with CDC Ethernet support)
    [*] RNDIS support
    ```
2. Insert module (g_ether.ko) into the kernel on the target board.
    ```
    :/mnt/mmc1p1> insmod g_ether.ko 
    ether gadget: using random self ethernet address
    ether gadget: using random host ethernet address
    usb0: Ethernet Gadget, version: May Day 2005
    usb0: using s3c-udc, OUT ep1-bulk IN ep2-bulk STATUS ep3-int
    usb0: MAC f2:b8:16:03:9d:de
    usb0: HOST MAC ba:49:bd:ab:56:87
    usb0: RNDIS ready
    Registered gadget driver 'ether'
    ```
3. Setup the network interface via ifconfig.
    ```
    :/mnt/mmc1p1> ifconfig usb0 192.168.1.121 netmask 255.255.248.0
    :/mnt/mmc1p1> ifconfig -a
    usb0 Link encap:Ethernet HWaddr F2:B8:16:03:9D:DE 
    inet addr:192.168.1.121 Bcast:192.168.7.255 Mask:255.255.248.0
    UP BROADCAST MULTICAST MTU:1500 Metric:1
    RX packets:0 errors:0 dropped:0 overruns:0 frame:0
    TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
    collisions:0 txqueuelen:1000 
    RX bytes:0 (0.0 B) TX bytes:0 (0.0 B)
    ```
4. Insert usbnet related modules in host. (x86/x64)
    ```
    sudo modprobe usbnet
    sudo modprobe cdc_ether
    sudo modprobe zaurus (No needed)
    ```
    **[UPDATED]**
    The issue of multiple network interfaces showing up whenever the device is rebooted/power-cycled is due to the g_ether driver randomly setting the host and device mac addresses. This problem is remedied by passing the same host and device mac addresses during the modprobe of g_ether.

    For example: modprobe g_ether host_addr=46:0d:9e:67:69:eb dev_addr=46:0d:9e:67:69:ec
     
     **[UPDATED]**
    When you connect a usbnet device to a Linux host, it normally issues a USB hotplug event, which will ensure that the usbnet driver is active. In most GNU/Linux distributions you shouldn't even notice whether the driver needed loading. It should just initialize, so that you can immediately use the device as a network interface. If it doesn't, then you probably didn't configure this driver (or its modular form) into your kernel build. To fix that, rebuild and reinstall as appropriate; at this time you might also want to upgrade to a recent kernel.

    Once that driver starts using that USB device, you'll notice a message like this in your syslog files, announcing the presence of a new usb0 (or usb1, usb2, etc) network interface that you can use with ifconfig and similar network tools.
    ```usb0: register usbnet usb-00:02.0-1.3, Belkin, eTEK, or compatible, c2:91:27:cb:d7:29```
    * setup the network interface via ifconfig, it should be connected with target board now.
    * modifications for /etc/network/interfaces if needed.
    ```
    auto usb0
iface usb0 inet static
    address 192.168.100.1
    netmask 255.255.255.0
    network 192.168.100.0
    gateway 192.168.100.200
    ```
    **[UPDATED]**
    It seems NetworkManager of Ubuntu will reset ip in few minutes after connected, we need to ignore usb0 by the following solution: **Ignore specific devices** Sometimes it is desired, that network manager ignores some devices and do not try to get an IP.
    * First you have to find out the Hal UDI (e.g. with lshal):
    ```
    ...
    info.product = 'Networking Interface'  (string)
    info.subsystem = 'net'  (string)
    info.udi = '/org/freedesktop/Hal/devices/net_00_1f_11_01_06_55'  (string)
    linux.hotplug_type = 2  (0x2)  (int)
    linux.subsystem = 'net'  (string)
    net.interface = 'usb0'  (string) 
    ...
    ```
    * Add the udi to /etc/NetworkManager/nm-system-settings.conf:
    ```
    [keyfile]
      unmanaged-devices=/org/freedesktop/Hal/devices/net_00_1f_11_01_06_55
    ```
    * Multiple devices can be specified, delimited by semicolons:
    ```
    [keyfile]
      unmanaged-devices=/org/freedesktop/Hal/devices/net_00_1f_11_01_06_55;/org/freedesktop/Hal/devices/net_00_2c_6d_e2_08_af
    ```
	You do not need to restart NetworkManager for the changes to take effect.

	**[Kubuntu UPDATED]**
    ```
    [keyfile]
        unmanaged-devices=/org/freedesktop/Hal/devices/usb_device_525_a4a2_noserial_if0
    ```
    
## Remote debugging via NFS
1. Install nfs related packages on host. (x86/x64)
    ```
    $ sudo apt-get install nfs-common
    $ sudo apt-get install nfs-kernel-server
    ```
2. 安裝完之後，接下來設定NSF環境設定如下:
	設定分享目錄: 執行命令:sudo gedit /etc/exports 最後一行加入 /home/mihoxp/Nfs *(rw,sync,no_root_squash)
    ```
    # /etc/exports: the access control list for filesystems which may be exported
    #        to NFS clients.  See exports(5).
    #
    # Example for NFSv2 and NFSv3:
    # /srv/homes       hostname1(rw,sync) hostname2(ro,sync)
    #
    # Example for NFSv4:
    # /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt)
    # /srv/nfs4/homes  gss/krb5i(rw,sync)
    /home/mihoxp/Nfs *(rw,sync,no_root_squash)
    ```
    ```
    其中參數說明如下：
    rw－－－讀/寫權限。如果設定只讀權限，則設為ro。但是一般情況下，為了方便交互，要設置為rw；
    sync－－數據同步寫入內存和硬盤;
    no_root_squash－－此參數用來要求服務器允許遠程系統以它自己的root特權存取該目錄;就是說如果用戶是root，那麼他就對這	個共享目錄有root的權限。
    很明顯，該參數授予了target board很大的權利。安全性是首先要考慮的，可以採取一定的保護機制，在下面會講一下保護機制。
    如果使用默認的root_squash，target board自己的根文件系統可能有很多無法寫入，所以運行會受到極大的限制。
    在安全性有所保障的前提下，推薦使用 no_root_squash參數。
    ```
3. 重啟portmap daemon portmapper（端口映射）服務，是NFS所本身需要的.
	```sudo /etc/init.d/portmap restart```
4. 重啟NFS Server 執行/etc/init.d目錄下的程序nfs，重啟NFS Server.
    ```
    sudo cd /etc/init.d
    sudo exportfs -rav
    sudo ./nfs-kernel-server restart

    Stopping NFS kernel daemon                                           [ OK ] 
    Unexporting directories for NFS kernel daemon...           [ OK ] 
    Exporting directories for NFS kernel daemon...exportfs: /etc/exports [1]: 
    Neither 'subtree_check' or   'no_subtree_check' specified for export "*:/home/mihoxp/Nfs". Assuming default behaviour ('no_subtree_check').
    NOTE: this default has changed since nfs-utils version 1.0.x         [ OK ]
    Starting NFS kernel daemon                    [ OK ]
    ```
5. Enable NFS support in embedded linux and mount network folder for development.
	* Enable kernel nfs support
		**File system> Network File Systems> NFS System support, Provide NFSv3 client support**
    * setup network interface via ifconfig
	* mount -t nfs 192.168.1.120:/home/ralph/workarea/ebook/nfs mount_point
		**Note: -o nolock could avoid using portmap in embedded platform.**

## Remote Debugging via serial
1. Configuring getty to be once from respawn in the /etc/inittab file.
    ```
    1:2345:respawn:/sbin/getty 115200 ttySAC0
    ```
    ```
    1:2345:once:/sbin/getty 115200 ttySAC0
    ```
2. Launching gdbserver with serial port specified. nohup - run a command immune to hangups, with output to a non-tty.
    ```
    nohup gdbserver /dev/ttySAC0 QBookApp -qws
    ```
3. You have to terminate the minicom or terminal software. This is because the serial line will be occupied by gdb.
4. Launch gdb as usual, except specifying serial port and baud rate.
    ```
    cgdb -d arm-none-linux-gnueabi-gdb -b 115200 bin/debug/QBookApp
    ```
    ```
    (gdb) target remote /dev/ttyS0
    ```