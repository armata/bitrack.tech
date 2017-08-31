---
layout: post
title: Moving to VyOS (Part 1) - Accessing modem from inside firewall
excerpt_separator: <!--more-->
---

You might be wondering: how do I access my modem if I'm behind my VyOS router/firewall?

[I've had the same setup on pfSense.](https://doc.pfsense.org/index.php/Accessing_modem_from_inside_firewall)

The configuration is actually pretty simple -- here's how I did it.

<!--more-->

Assuming you have a PPPoE connection, it'll be on a virtual interface, leaving the actual interface (ethX) *unused*.

That means you can set a static IP (don't use DHCP here!), add some source NAT and some firewall rules and you should be good to go.

1) Set the IP on the interface

My modem network is `192.168.1.0/24` and the modem is on `192.168.1.1`. I'll give my VyOS router an address of `192.168.1.2`.

```
set interfaces ethernet eth1 address '192.168.1.2/24'
```

2) Create the source NAT rule

We also need NAT so it looks like all the traffic comes out of `192.168.1.2` and not your private network ranges behind VyOS.

```
set nat source rule 100 description 'LAN to MODEM'
set nat source rule 100 outbound-interface eth1 # WAN interface
set nat source rule 100 source address '192.168.10.0/24' # my LAN network
set nat source rule 100 translation address masquerade
```

3) Firewall setup

Since I use a zone-based firewall, I created a special zone called `MODEM`.

[I highly suggest moving to a zone-based firewall; it's much easier to work with]

```
# traffic from LAN to MODEM
set firewall name LAN-MODEM default-action accept

# create the MODEM zone
set zone-policy zone MODEM default-action drop
set zone-policy zone MODEM from LAN firewall name LAN-MODEM
set zone-policy zone MODEM interface eth1

# traffic from MODEM to LAN
set firewall name MODEM-LAN default-action drop
set firewall name MODEM-LAN rule 1 action 'accept'
set firewall name MODEM-LAN rule 1 state established 'enable'
set firewall name MODEM-LAN rule 1 state related 'enable'
set firewall name MODEM-LAN rule 2 action 'drop'
set firewall name MODEM-LAN rule 2 state invalid 'enable'

# modify the LAN zone
set zone-policy zone LAN from MODEM firewall-name MODEM-LAN
```

Of course, don't foget to `commit` and `save`.

That should be it! You should now be able to access `192.168.1.0/24` (MODEM) from `192.168.10.0/24` (LAN).
