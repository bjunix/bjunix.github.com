---
title: Running Gentoo as a kvm guest
layout: default
---

virt-install \
    --connect qemu:///system \
    --name gentoo_base \
    --ram 512 \
    --file /var/lib/libvirt/images/gentoo_base.img \
    --file-size 10 \
    --cdrom /var/lib/libvirt/images/install-amd64-minimal-20091203.iso \
    --network user \
    --force \
    --vnc \
    --accelerate \
    --noautoconsole



