---
title: Virtualization with kvm and libvirt
layout: default
---

Virtualization with kvm and libvirt
===================================

Using the virsh console:
------------------------

In order to be able to use virsh's console you need to activate a serial console
in the guest system.

* Edit your /etc/inittab and append:
    s0:12345:respawn:/sbin/agetty -L 115200 ttyS0 vt102
* append to your grub.conf kernel line:
    console=tty0 console=ttys0,115200

Moving paritions:
-----------------

I've always used this for moving data to new partitions:

 % cd /mnt/chroot/var
 % find -xdev -depth -print | cpio -padvmB /mnt/chroot/newvar

Using lvm volumes as storage:
-----------------------------

Instead of image files it is possible to use lvm volumes for storage.
In Centos 5.4 selinux prevents libvirt to utilize storage other than
found under /var/lib/libvirt/images/. (This bug is said to be fixed in fedora 10)

For Centos 5.4 we need to tell selinux about our new volume:

    chcon -t virt_image_t /dev/lvm-kvm/*<logical_vol>*
    restorecon /dev/lvm-kvm/*<logical_vol>*

we can make this persisten to come effective after reboot with:

    semanage fcontext -m -t virt_image_t "/dev/lvm-kvm/*<logical_vol>*"

See:
https://bugzilla.redhat.com/show_bug.cgi?id=491245
http://www.linux-archive.org/fedora-selinux-support/283369-selinux-qemu-lvm-issues.html

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


and add something to /etc/rc.local to do this during boot up (see sample scripts below)

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


Costum kernel options:
----------------------

If you build a custom kernel make sure you build it with the 8139cp network driver.
Also the VIRTIO options should be selected (some are still masked experimental)
Look as well for the KVM and PARAVIRT options and see what fits to your setup ...

Virtio device drivers:
----------------------

If you use the virtio drivers for the harddisk remember that the devices are named

vda1
vda2
vdaN

Before using virtio make sure to change grub.conf and /etc/fstab


Cloning a virtual machine:
--------------------------

Cloning a guest with libvirt's virt-clone is quite straightforward.

You can use virt-clone with the --prompt option for convinience. There are
somethings to remember after cloning a existing machine. The networks hardware
address (MAC) will change and the new guest system will not know about it. It
will probably create a new device and you need to make sure it will get started
and configured on startup. So after booting the new machine you need to
*virsh console* into it and reconfigure the network. Places to look are
dependend on the operating system of the guest but may include the following:

* /etc/init.d/
* /etc/sysconfig/
* /etc/conf.d/
* /etc/udev/rules.d/70-persistent-net.rules (for mapping the hwaddr to device name)

And be sure to edit the xml definition file of the new machine and adjust mem
and cpu usage


Sample scripts
--------------



/etc/rc.local

    #ugly workaround to wait for libvirtd to be finished
    #trying to set up routing to virbr1 for the kvm guests here
    #and removing libvirts restrictive iptables rules
    #TODO: find a better place to do this

    #try 5 times to see if virbr1 is active
    #if virbr1 is up set the route
    for i in {1..5}; do
      if /sbin/ifconfig virbr1 >/dev/null 2>&1; then
        echo "virbr1 is up: Set up routes..."
        /sbin/ip route add <guest_ip> dev virbr1 scope link >/dev/null 2>&1
        /sbin/ip route add <guest_ip> dev virbr1 scope link >/dev/null 2>&1
        /sbin/ip route add <guest_ip> dev virbr1 scope link >/dev/null 2>&1
        echo "Done setting up routes for virb1"
        break
      fi
      #wait 2 seconds before the next try
      sleep 2
    done

    #try another 5 times to remove the iptable rules
    for i in {1..5}; do
      if /sbin/iptables -D FORWARD -o virbr1 -j REJECT >/dev/null 2>&1; then
        echo "remove libvirt iptable rule for virbr1"
        break
      fi
      sleep 2
    done

    for i in {1..5}; do
      if /sbin/iptables -D FORWARD -i virbr1 -j REJECT >/dev/null 2>&1; then
         echo "remove libvirt iptable rule for virbr1"
         break
      fi
      sleep 2
    done








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