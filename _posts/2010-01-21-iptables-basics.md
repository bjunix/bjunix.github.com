---
title: Iptables basics
layout: default
---

Great article about iptables:
http://wiki.centos.org/HowTos/Network/IPTables

Built-in chains:

INPUT (for packets destined to local sockets)
FORWARD (for packets being routed  through the box)
OUTPUT (for locally-generated packets leaving the box).

Built-in target:

ACCEPT
DROP
.....


Clear iptables ruleset
----------------------

    #set default policies to ACCEPT everything
    iptables -P INPUT ACCEPT
    iptables -P FORWARD ACCEPT
    iptables -P OUTPUT ACCEPT

    #flush all rules
    iptables -F

    #delete all chains
    iptables -X

    #delete default tables
    iptables -t nat -F
    iptables -t nat -X
    iptables -t mangle -F
    iptables -t mangle -X



Basic rules
-----------

    #Allow all incoming for the loopback device *lo*
    iptables -A INPUT -i lo -j ACCEPT

    #Accept packets from already established connections
    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

    #Open up port 22 (ssh)
    iptables -A INPUT -p tcp --dport 22 -j ACCEPT

    #Allow ping
    iptables -A INPUT -p icmp --icmp-type 8 -j ACCEPT

    #Set default policy to DROP. If none of the above rules have matched drop the packet
    iptables -P INPUT DROP

    #Do not restrict routed (forwared) traffic
    iptables -P FORWARD ACCEPT

    #Allow all outgoing connections
    iptables -P OUTPUT ACCEPT



