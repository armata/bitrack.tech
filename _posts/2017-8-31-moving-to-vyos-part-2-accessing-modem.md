---
layout: post
title: Moving to VyOS (Part 2) - Accessing modem from inside firewall
excerpt_separator: <!--more-->
---

You might be wondering: how do I access my modem if I'm behind my VyOS router/firewall?

The configuration is actually pretty simple -- here's how I did it.

Assuming you have a PPPoE connection, it'll be on a virtual interface, leaving the actual interface (ethX) *unused*.

That means you could set a static IP (don't use DHCP here!), add a bit of source NAT, some firewall rules and you should be able to access the modem from your LAN network.

<!--more-->

## Set the IP on the interface

My modem network is `192.168.1.0/24` and the modem is on `192.168.1.1`. I'll give my VyOS router an address of `192.168.1.2`.

```
set interfaces ethernet eth1 address '192.168.1.2/24'
```

(the /24 here is to create a static route)

## Create the source NAT rule

We also need NAT so it looks like all the traffic comes out of `192.168.1.2` and not your private network ranges behind VyOS.

```
set nat source rule 100 description 'LAN to MODEM'
set nat source rule 100 outbound-interface eth1 # WAN interface
set nat source rule 100 source address '192.168.10.0/24' # my LAN network
set nat source rule 100 translation address masquerade
```

## Configure the firewall

Since I use a zone-based firewall, I created a special zone called `MODEM`.

(I highly suggest moving to a zone-based firewall; it's much easier to work with)

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

## Conclusion

That should be it! You should now be able to access `192.168.1.1` (the modem) from `192.168.10.0/24` (the LAN network).
