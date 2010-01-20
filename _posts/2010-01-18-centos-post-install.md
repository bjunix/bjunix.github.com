---
title: Centos post-install steps
layout: default
---

Centos post-install steps
===========

Cleanup
-------

Following packages will be removed after installation:

    yum erase cups-libs
    #(this also removes ecryptfs-utils, gtk2 and trousers)