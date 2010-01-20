---
title: Centos installation
layout: default
---

Centos installation
===================

As the host runs a software RAID1 configuration, centos needs the mdadm tool.
Per default Centos installs *exim* and therefor *mysql* as a dependency. To overcome this
problem Centos needs to be explicitly installed with *postfix*.

    Source:
    "The solution is to include postfix (or sendmail) into your kickstart configfile.
    That resolves the smtpdaemon dependency for mdadm and it keeps you from getting mysql."
    (http://lists.centos.org/pipermail/centos/2007-October/045851.html)

Centos generates a kickstart file in the /root dir after normal installation.
This file served as a template for the kickstart installation.

With a normal graphical installation you could achive the same with the following:
From Packagegroup Server choose Mailserver. At the Optional Packages dialog select
only the *postfix* package.


Update Installation
-------------------

After successful installation we will bring the packages up-to-date

    yum update


Additional packages
-------------------

We started with the most basic installation (Packagegroup @Core). For daily sysadmin
work we need to install some additional packages.

    yum install man screen vim-enhanced


Cleanup
-------

Following packages will be removed after installation:

    yum erase cups-libs
    #(this also removes ecryptfs-utils, gtk2 and trousers)

Get rid of ix86 packages as we run on 64bit system exclusivly.

    yum erase \*.i\?86

To keep it that way, we edit the /etc/yum.conf and append a line:

    exclude = *.i?86


Optional: Backup the basic installation
-----------------------------------------

Now its time to make a backup of this basic installation. It's wise to save the
disk partition information so it will be packed up in the image and you can
restore from it later:

    sfdisk -d /dev/sda > /root/sfdisk.dump


Boot the machine into a rescue environment with something like *partimage*
available and make an iso out of it, copy it somewhere safe and feel happy.

Add admin user
----------------------------

    adduser admin -G wheel
    passwd admin

    su - admin
    ssh-keygen -q -b 2048 -t rsa -f ~/.ssh/id_rsa
    install -m 0600 ~/.ssh/id_rsa.pub ~/.ssh/authorized_keys
    exit

Now get the private key over to your management machine. From the machine run:

    scp root@host:/home/admin/.ssh/id_rsa ~/.ssh/*<somefile_name>*

Do this step before next because we will disable ssh password auth there.

Setup sshd
----------

    cat >> /etc/ssh/sshd_config <<"EOF"
    Port 22
    Protocol 2
    PermitRootLogin no
    PasswordAuthentication no
    ChallengeResponseAuthentication no
    AllowGroups wheel
    EOF



Install virtualization utilities
--------------------------------

Now we install the tools for virtualization. We will use a kvm virtualization via
qemu. Centos standart management tools for virtualization are libvirt (providing
virsh) and virt-install.

    yum install kmod-kvm kvm-qemu-img kvm python-virtinst


Start libvirtd and add to startup

    service libvirtd start
    chkconfig --add libvirtd

Set default vnc password:

    vi /etc/libvirt/