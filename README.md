#the LXC over MACVLAN inside VirtulBox
=======

##0 config the network configuration of all card in the virtulbox:
--------
you will have to update your VM network settings.
Open the settings panel for the VM, go the the "Network" tab, pull down the
"Advanced" settings. Here, the "Adapter Type" should be pcnet (the full
name is something like "PCnet-FAST III"), instead of the default e1000
(Intel PRO/1000). Also, "Promiscuous Mode" should be set to "Allow All".

If you don't do that, bridged containers won't work, because the virtual
NIC will filter out all packets with a different MAC address. If you are
running VirtualBox in headless mode, the command line equivalent of the above
is modifyvm --nicpromisc1 allow-all. If you are using Vagrant, you can add
the following to the config for the same effect:

	config.vm.provider "virtualbox" do |v|
	  v.customize ['modifyvm', :id, '--nictype1', 'Am79C973']
	  v.customize ['modifyvm', :id, '--nicpromisc1', 'allow-all']
	end

Note: it looks like some operating systems (e.g. CentOS 7) do not support
pcnet anymore. You might want to use the virtio-net (Paravirtualized
Network) interface with those.
1 configure the network for vxlan on the OVS1
```
sudo vi /etc/network/interfaces
```
OVS and LXC and VxLAN begin
```
auto vmbr0
iface vmbr0 inet static
        address 10.0.0.1
        netmask 255.255.255.0
        ovs_type OVSBridge
        post-up ovs-vsctl add-port vmbr0 vxlan1 -- set Interface vxlan1 type=vxlan option:remote_ip=192.168.56.13 option:key=flow ofport_request=4
        post-up ovs-vsctl set-controller vmbr0 tcp:192.168.56.11:6633
        post-up ovs-vsctl set bridge vmbr0 other-config:datapath-id=0000000000000001
OVS and LXC and VxLAN end
```
and restart the networking via /etc/init.d/networking restart

##2 config the lxc container config
```
sudo vi /var/lib/lxc/<CT name>/config
```
the ifup script file:
```
vi /etc/lxc/ifup  #with 0755 mod for exec
```
```
!/bin/bash
BRIDGE='vmbr0'
ovs-vsctl --may-exist add-br $BRIDGE
ovs-vsctl --if-exists del-port $BRIDGE $5
ovs-vsctl --may-exist add-port $BRIDGE $5
```
```
vi /etc/lxc/ifdown   # with 0755 mod for exec
```
```
!/bin/bash
ovsBr='vmbr0'
ovs-vsctl --if-exists del-port ${ovsBr} $5
```

`Network configuration`
```
lxc.network.type = veth
lxc.network.link = lxcbr0 #pay attention
lxc.network.veth.pair = br0vm1 #changed to veth.pair for vxlan
lxc.network.name = eth0
lxc.network.flags = up
lxc.network.script.up = /etc/lxc/ifup # the script file 
lxc.network.script.down = /etc/lxc/ifdown # the scirpt file 
lxc.network.hwaddr = 00:16:3e:cb:c6:50
lxc.network.ipv4 = 10.0.0.101/24#  lxc ct ip address connect to the OVS1
lxc.network.ipv4.gateway=10.0.0.1
```

`MACVLAN  for the MACVLAN `
```
lxc.network.type = macvlan
lxc.network.macvlan.mode = bridge
lxc.network.flags = up
lxc.network.link = mvlan0
lxc.network.name = eth1
lxc.network.hwaddr = 00:16:3e:41:11:65
lxc.network.mtu = 1500
lxc.start.auto = 1
lxc.start.delay = 5
```
and now the vxlan tunnel must be OK
next for the lxc ct to access  internet through MACVLAN


```
sudo ip lin add mvlan0 link enp0s9 type macvlan mode bridge
ifconfig mvlan0 10.0.2.200 up
ip -d link show mvlan0
` `` 
```
38: mvlan0@enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
    link/ether ae:3b:f1:84:d7:f0 brd ff:ff:ff:ff:ff:ff
 ```
##3 start the lxc 
```
sudo lxc-start -n vm1
sudo lxc-attach -n vm1
```
```
ifconfig eth1 10.0.2.100 
```

4 modify the route of lxc ct and host
the route of host
```
ubuntu@box2:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.2.2        0.0.0.0         UG    0      0        0 enp0s3
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 vmbr0
10.0.2.0        0.0.0.0         255.255.255.0   U     0      0        0 enp0s3
10.0.2.100      0.0.0.0         255.255.255.255 UH    0      0        0 mvlan0
10.0.3.0        0.0.0.0         255.255.255.0   U     0      0        0 lxcbr0
192.168.56.0    0.0.0.0         255.255.255.0   U     0      0        0 enp0s8
192.168.57.0    0.0.0.0         255.255.255.0   U     0      0        0 enp0s9
```
note, that should add a route to the lxc ct on the mvlan0, and del the 10.0.0.0/8 route on the mvlan0
the route of the lxc ct
```
root@vm1:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.2.2        0.0.0.0         UG    0      0        0 eth1
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.0.0.0        0.0.0.0         255.0.0.0       U     0      0        0 eth1
```
modify the gw to the same as the host, so that can access the network through NAT of the virtulBOX.





