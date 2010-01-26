---
title: Virtualization with kvm and libvirt
layout: default
---

Virtualization with kvm and libvirt
===================================


Setup routed network
--------------------

Add to /etc/sysconfig/network:

    FORWARD_IPV4=true

Create the file /etc/libvirt/qemu/networks/routed_internet.xml

    <network>
      <name>routed_internet</name>
      <uuid>*< enter uuid here - use uuidgen to generate one >*</uuid>
      <forward dev='eth0' mode='route'/>
      <bridge name='virbr1' stp='on' forwardDelay='0' />
      <ip address='*<enter host-ip here>*' netmask='255.255.255.255' />
    </network>

Tell virsh about it:

    virsh net-define /etc/libvirt/qemu/networks/routed_internet.xml
    virsh net-autostart routed_internet
    virsh net-start routed_internet

Do not know if this is needed:

    #iptables -I RH-Firewall-1-INPUT -i virbr1 -j ACCEPT
    iptables -I FORWARD -i virbr1 -j ACCEPT
    iptables -D FORWARD -i virbr1 -j REJECT
    iptables -D FORWARD -o virbr1 -j REJECT
Setup guest IPs
----------------

for every ip run this on the virt host:

    /sbin/ip route add *<guest_ip>* dev virbr1 scope link

Guest network config
--------------------

ip: xxx.xxx.xxx.xxx
netmask: 255.255.255.0 ? 255.255.255.248
broadcast: xxx.xxx.xxx.255
gateway: *<host_ip>*

    ifconfig eth0 xxx.xxx.xxx.xxx netmask 255.255.255.248 up
    route add default gw *<host_ip>*

    vim /etc/resolv.conf

    nameserver *<dns_ip>*

Use virt-install to setup virtual machine
-----------------------------------------

If you have made a change to the XML configuration file, you need to tell KVM to reload it before restarting the VM:

virsh define /etc/libvirt/qemu/mirror.xml


Building a reverse ssh tunnel to connect to the vnc server
----------------------------------------------------------

virt-install binds the vnc server to localhost only (for a good reason). To access
the vnc server we need to build a reverse ssh tunnel and bind it to the vnc port
(:5900 in most cases). The problem is, that I can only build the tunnel in the
direction of the host to my local machine. When my local machine sits behind a
firewall so I cant ssh into I need a connection broker (middle). This has to be a machine
that can be reached from both the vnc host (destination) and my local machine (local)

We have 3 hosts:

destination: the virt host with vnc running on 5900
middle: a connection broker we can access from both machines (need user account here)
local: my local machine

destination --> ssh --> middle <-- vnc <-- local

than we can build the tunnel. From destination we call:

ssh -i -R 10000:localhost:5900 user@middle

Now we can connect to middle:10000 and all data will get redirected to destination:5900.

Be sure not to forget to set a password for the vnc server as everybody can connect
to middle:10000


Sample scripts
--------------

Setup machine for a Gentoo installation boot up with the Gentoo Livecd

    #!/bin/bash

    virt-install \
         --connect qemu:///system \
         --name $1 \
         --ram 500 \
         --file /var/lib/libvirt/images/$1.img \
         --file-size 5 \
         --cdrom /var/lib/libvirt/images/install-amd64-minimal-20091203.iso \
         --network user \
         --force \
         --vnc \
         --accelerate \
         --noautoconsole



Install Centos 5.4 from network location:

    #!/bin/bash
    virt-install \
         --connect qemu:///system \
         --name $1 \
         --ram 500 \
         --file /var/lib/libvirt/images/$1.img \
         --file-size 5 \
         --network user \
         --force \
         --accelerate \
         --noautoconsole \
         --location http://ftp.halifax.rwth-aachen.de/centos/5.4/os/x86_64/ \
         -x "console=ttyS0"

http://fedora.uni-oldenburg.de/releases/12/Fedora/x86_64/os/
http://ftp.halifax.rwth-aachen.de/centos/5.4/os/x86_64/

virt-install \
         --connect qemu:///system \
         --name centos54_base \
         --ram 512 \
         --file /var/lib/libvirt/images/centos54_base.img \
         --file-size 5 \
         --network user \
         --accelerate \
         --noautoconsole \
         --location http://ftp.halifax.rwth-aachen.de/centos/5.4/os/x86_64/ \
         -x "console=ttyS0"