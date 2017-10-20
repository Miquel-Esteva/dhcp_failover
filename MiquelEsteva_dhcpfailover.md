Miquel Esteva Thomas 16/10/2017
# DHCP FAILOVER

The target is to get a test system with two dhcp servers (primary and secondary) that are serving the same pool to a client.

## VIRTUAL ENVIRONTMENT

This practice will be implemented using a KVM virtualization system under Fedora (any other linux system can be used).

### INSTALL KVM (libvirtd, virt-manager)

[root@i14 isx41536245]# dnf -y install libvirtd virt-manager

### CREATE AND INSTALL MAIN FEDORA QCOW2 TEMPLATE

Seguimos los pasos del virt-manager para instalar fedora23 a través de PXE. 

### CREATE INCREMENTAL QCOW2 DISKS (dhcpd1, dhcpd2 and client)

[root@i14 isx41536245]# cd /var/lib/libvirt/images/
-rw-------. 1 root root 10739318784 Oct 13 12:00 fedora24.qcow2

[root@i14 images]# qemu-img create -f qcow2 dhcp_server1.qcow2 -b fedora24.qcow2
Formatting 'dhcp_server1.qcow2', fmt=qcow2 size=10737418240 backing_file=fedora24.qcow2 encryption=off cluster_size=65536 lazy_refcounts=off refcount_bits=16

[root@i14 images]# ll
total 2099024
-rw-r--r--. 1 root root      197120 Oct 13 12:03 dhcp_server1.qcow2
-rw-------. 1 root root 10739318784 Oct 13 12:00 fedora24.qcow2

[root@i14 images]# qemu-img create -f qcow2 dhcp_server2.qcow2 -b fedora24.qcow2
Formatting 'dhcp_server2.qcow2', fmt=qcow2 size=10737418240 backing_file=fedora24.qcow2 encryption=off cluster_size=65536 lazy_refcounts=off refcount_bits=16

[root@i14 images]# qemu-img create -f qcow2 client.qcow2 -b fedora24.qcow2
Formatting 'client.qcow2', fmt=qcow2 size=10737418240 backing_file=fedora24.qcow2 encryption=off cluster_size=65536 lazy_refcounts=off refcount_bits=16

### CREATE SERVERS AND CLIENT KVM MACHINES



### CREATE AND CONFIGURE PRACTICE NETWORK ON VIRTUAL MACHINES

Para dhcp_server1 i dhcp_server2 creamos dos redes virtuales:

	1) virtual network 'practica': isolated network, internal and host routing only.
	2) virtual network 'default': NAT --> con ésta tendremos acceso a internet. 

## DHCP SERVER 1

### NETWORK CONFIGURATION

DHCP1

vim ifcfg-ens9

TYPE="Ethernet"
BOOTPROTO="none"
DEFROUTE="no"
IPV4_FAILURE_FATAL="yes"
IPV6INIT="no"
NAME="ens9"
DEVICE="ens9"
ONBOOT="yes"
PEERDNS="no"
PEERROUTES="no"
IPADDR="172.16.1.100"
PREFIX="24"

### DHCPD SERVICE INSTALLATION

[root@i14 isx41536245]# dnf -y install dhcp

### DHCPD CONFIGURATION

vim /etc/dhcp/dhcpd.conf

failover peer "dhcp-failover" {
        primary;
        address 172.16.1.100;
        port 647;
        peer address 172.16.1.200;
        peer port 647;
        max-response-delay 180;
        max-unacked-updates 10;
        mclt 1800;
        split 128;
        load balance max seconds 3;
}

subnet 172.16.1.0 netmask 255.255.255.0 {
        option domain-name-servers 172.16.1.1;
        option broadcast-address 172.16.1.255;
        option subnet-mask 255.255.255.0;
        option routers 172.16.1.1;
        pool {
                failover peer "dhcp-failover";
                range 172.16.1.220 172.16.1.230;
        }
}

[root@dhcp_server1 ~]# systemctl restart dhcpd

## DHCP SERVER 2

### NETWORK CONFIGURATION

vim ifcfg-ens9

TYPE="Ethernet"
BOOTPROTO="none"
DEFROUTE="no"
IPV4_FAILURE_FATAL="yes"
IPV6INIT="no"
NAME="ens9"
DEVICE="ens9"
ONBOOT="yes"
PEERDNS="no"
PEERROUTES="no"
IPADDR="172.16.1.200"
PREFIX="24"

[root@dhcp_server2 ~]# systemctl restart NetworkManager

### DHCPD SERVICE INSTALLATION

[root@i14 isx41536245]# dnf -y install dhcp

### DHCPD CONFIGURATION

failover peer "dhcp-failover" {
        secondary;
        address 172.16.1.200;
        port 647;
        peer address 172.16.1.100;
        peer port 647;
        max-response-delay 180;
        max-unacked-updates 10;
        load balance max seconds 3;
}

subnet 172.16.1.0 netmask 255.255.255.0 {
        option domain-name-servers 172.16.1.1;
        option broadcast-address 172.16.1.255;
        option subnet-mask 255.255.255.0;
        option routers 172.16.1.1;
        pool {
                failover peer "dhcp-failover";
                range 172.16.1.220 172.16.1.230;
        }
}

[root@dhcp_server2 ~]# systemctl restart dhcpd


Desactivamos firewalld

[root@dhcp_server1 ~]# systemctl stop firewalld.service 
[root@dhcp_server1 ~]# systemctl disable firewalld.service 
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.

## DHCP CLIENT

### NETWORK CONFIGURATION

/etc/sysconfig/network-scripts/ifcfg-ens3

TYPE=Ethernet
BOOTPROTO=dhcp
DEFROUTE=yes
PEERDNS=yes

## TESTING ENVIRONTMENT

Ejecutamos un dhclient -v para obtener una ip del rango:

Nos asigna la dirección 172.16.1.225/24

DHCP1

[root@dhcp_server1 ~]# systemctl status dhcpd
● dhcpd.service - DHCPv4 Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2017-10-16 10:03:27 CEST; 21min ago
     Docs: man:dhcpd(8)
           man:dhcpd.conf(5)
 Main PID: 1669 (dhcpd)
   Status: "Dispatching packets..."
   CGroup: /system.slice/dhcpd.service
           └─1669 /usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid

Oct 16 10:23:34 dhcp_server1 dhcpd[1669]: failover peer dhcp-failover: Both servers normal
Oct 16 10:23:34 dhcp_server1 dhcpd[1669]: balancing pool 5ea9c47a80 172.16.1.0/24  total 11  free 6  backup 5  lts 0  max-own (+/-)1
Oct 16 10:23:34 dhcp_server1 dhcpd[1669]: balanced pool 5ea9c47a80 172.16.1.0/24  total 11  free 6  backup 5  lts 0  max-misbal 2
Oct 16 10:24:01 dhcp_server1 dhcpd[1669]: DHCPDISCOVER from 52:54:00:c1:fb:9c via ens9
Oct 16 10:24:02 dhcp_server1 dhcpd[1669]: DHCPOFFER on 172.16.1.225 to 52:54:00:c1:fb:9c (dhcp_client) via ens9
Oct 16 10:24:02 dhcp_server1 dhcpd[1669]: DHCPREQUEST for 172.16.1.225 (172.16.1.100) from 52:54:00:c1:fb:9c (dhcp_client) via ens9
Oct 16 10:24:02 dhcp_server1 dhcpd[1669]: DHCPACK on 172.16.1.225 to 52:54:00:c1:fb:9c (dhcp_client) via ens9
Oct 16 10:24:55 dhcp_server1 dhcpd[1669]: reuse_lease: lease age 53 (secs) under 25% threshold, reply with unaltered, existing lease for 172.16.1.225
Oct 16 10:24:55 dhcp_server1 dhcpd[1669]: DHCPREQUEST for 172.16.1.225 from 52:54:00:c1:fb:9c (dhcp_client) via ens9
Oct 16 10:24:55 dhcp_server1 dhcpd[1669]: DHCPACK on 172.16.1.225 to 52:54:00:c1:fb:9c via ens9

DHCP2

[root@dhcp_server2 ~]# systemctl status dhcpd
● dhcpd.service - DHCPv4 Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2017-10-16 10:23:34 CEST; 1min 42s ago
     Docs: man:dhcpd(8)
           man:dhcpd.conf(5)
 Main PID: 1566 (dhcpd)
   Status: "Dispatching packets..."
    Tasks: 1 (limit: 512)
   CGroup: /system.slice/dhcpd.service
           └─1566 /usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid

Oct 16 10:23:34 dhcp_server2 dhcpd[1566]: failover peer dhcp-failover: I move from startup to normal
Oct 16 10:23:34 dhcp_server2 dhcpd[1566]: balancing pool e797318ab0 172.16.1.0/24  total 11  free 6  backup 5  lts 0  max-own (+/-)1
Oct 16 10:23:34 dhcp_server2 dhcpd[1566]: balanced pool e797318ab0 172.16.1.0/24  total 11  free 6  backup 5  lts 0  max-misbal 2
Oct 16 10:23:34 dhcp_server2 dhcpd[1566]: failover peer dhcp-failover: peer moves from communications-interrupted to normal
Oct 16 10:23:34 dhcp_server2 dhcpd[1566]: failover peer dhcp-failover: Both servers normal
Oct 16 10:24:01 dhcp_server2 dhcpd[1566]: DHCPDISCOVER from 52:54:00:c1:fb:9c via ens9: load balance to peer dhcp-failover
Oct 16 10:24:02 dhcp_server2 dhcpd[1566]: DHCPREQUEST for 172.16.1.225 (172.16.1.100) from 52:54:00:c1:fb:9c via ens9: lease owned by peer
Oct 16 10:24:55 dhcp_server2 dhcpd[1566]: reuse_lease: lease age 53 (secs) under 25% threshold, reply with unaltered, existing lease for 172.16.1.225
Oct 16 10:24:55 dhcp_server2 dhcpd[1566]: DHCPREQUEST for 172.16.1.225 from 52:54:00:c1:fb:9c via ens9
Oct 16 10:24:55 dhcp_server2 dhcpd[1566]: DHCPACK on 172.16.1.225 to 52:54:00:c1:fb:9c via ens9

[root@dhcp_server1 ~]# systemctl stop dhcpd
[root@dhcp_client ~]# ifconfig ens3 down

[root@dhcp_client ~]# ifconfig ens3 up

Ahora ha sido dhcp_server2 quién nos ha ofrecido la IP: 

DHCP1 APAGADO:

[root@dhcp_server1 ~]# systemctl status dhcpd
● dhcpd.service - DHCPv4 Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: man:dhcpd(8)
           man:dhcpd.conf(5)

Oct 16 10:23:34 dhcp_server1 dhcpd[1669]: balanced pool 5ea9c47a80 172.16.1.0/24  total 11  free 6  backup 5  lts 0  max-misbal 2
Oct 16 10:24:01 dhcp_server1 dhcpd[1669]: DHCPDISCOVER from 52:54:00:c1:fb:9c via ens9
Oct 16 10:24:02 dhcp_server1 dhcpd[1669]: DHCPOFFER on 172.16.1.225 to 52:54:00:c1:fb:9c (dhcp_client) via ens9
Oct 16 10:24:02 dhcp_server1 dhcpd[1669]: DHCPREQUEST for 172.16.1.225 (172.16.1.100) from 52:54:00:c1:fb:9c (dhcp_client) via ens9
Oct 16 10:24:02 dhcp_server1 dhcpd[1669]: DHCPACK on 172.16.1.225 to 52:54:00:c1:fb:9c (dhcp_client) via ens9
Oct 16 10:24:55 dhcp_server1 dhcpd[1669]: reuse_lease: lease age 53 (secs) under 25% threshold, reply with unaltered, existing lease for 172.16.1.225
Oct 16 10:24:55 dhcp_server1 dhcpd[1669]: DHCPREQUEST for 172.16.1.225 from 52:54:00:c1:fb:9c (dhcp_client) via ens9
Oct 16 10:24:55 dhcp_server1 dhcpd[1669]: DHCPACK on 172.16.1.225 to 52:54:00:c1:fb:9c via ens9
Oct 16 10:30:35 dhcp_server1 systemd[1]: Stopping DHCPv4 Server Daemon...
Oct 16 10:30:35 dhcp_server1 systemd[1]: Stopped DHCPv4 Server Daemon.



DHCP2:

[root@dhcp_server2 ~]# systemctl status dhcpd
● dhcpd.service - DHCPv4 Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2017-10-16 10:23:34 CEST; 8min ago
     Docs: man:dhcpd(8)
           man:dhcpd.conf(5)
 Main PID: 1566 (dhcpd)
   Status: "Dispatching packets..."
    Tasks: 1 (limit: 512)
   CGroup: /system.slice/dhcpd.service
           └─1566 /usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid

Oct 16 10:23:34 dhcp_server2 dhcpd[1566]: failover peer dhcp-failover: Both servers normal
Oct 16 10:24:01 dhcp_server2 dhcpd[1566]: DHCPDISCOVER from 52:54:00:c1:fb:9c via ens9: load balance to peer dhcp-failover
Oct 16 10:24:02 dhcp_server2 dhcpd[1566]: DHCPREQUEST for 172.16.1.225 (172.16.1.100) from 52:54:00:c1:fb:9c via ens9: lease owned by peer
Oct 16 10:24:55 dhcp_server2 dhcpd[1566]: reuse_lease: lease age 53 (secs) under 25% threshold, reply with unaltered, existing lease for 172.16.1.225
Oct 16 10:24:55 dhcp_server2 dhcpd[1566]: DHCPREQUEST for 172.16.1.225 from 52:54:00:c1:fb:9c via ens9
Oct 16 10:24:55 dhcp_server2 dhcpd[1566]: DHCPACK on 172.16.1.225 to 52:54:00:c1:fb:9c via ens9
Oct 16 10:30:35 dhcp_server2 dhcpd[1566]: peer dhcp-failover: disconnected
Oct 16 10:30:35 dhcp_server2 dhcpd[1566]: failover peer dhcp-failover: I move from normal to communications-interrupted
Oct 16 10:31:51 dhcp_server2 dhcpd[1566]: DHCPREQUEST for 172.16.1.225 from 52:54:00:c1:fb:9c via ens9
Oct 16 10:31:51 dhcp_server2 dhcpd[1566]: DHCPACK on 172.16.1.225 to 52:54:00:c1:fb:9c (dhcp_client) via ens9


