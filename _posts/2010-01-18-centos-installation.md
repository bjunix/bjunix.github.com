---
title: hello world
layout: default
---

Centos installation
===================

As the host runs a software RAID1 configuration, centos needs the mdadm tool.
Per default Centos installs exim and therefor mysql as a dependency. To overcome this
problem Centos needs to be installed with *kickstart*.

    Source:

    "The solution is to include postfix (or sendmail) into your kickstart configfile.
    That resolves the smtpdaemon dependency for mdadm and it keeps you from getting mysql."
    (http://lists.centos.org/pipermail/centos/2007-October/045851.html)

Centos generates a kickstart file in the /root dir after normal installation.
This file served as a template for the kickstart installation.



Additional packages
-------------------

We started with the most basic installation (Packagegroup @Core). For daily sysadmin
work we need to install some additional packages.

    yum install man screen vim-enhanced


Install virtualization utilities
--------------------------------

Now we install the tools for virtualization. We will use a kvm virtualization via
qemu. Centos standart management tools for virtualization are libvirt (providing
virsh) and virt-install.

kvm-qemu-img kmod-kvm libvirt python-virtinst
