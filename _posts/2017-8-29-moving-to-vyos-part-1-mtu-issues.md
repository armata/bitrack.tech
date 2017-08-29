---
layout: post
title: Moving to VyOS: Part 1: PPPoE MTU issues
excerpt_separator: <!--more-->
---

Hi everyone! I've decided to move from pfSense to VyOS for many reasons, some of them being:

* command line interface
* easily exportable configuration
* zone-based firewall
* CLI similarities with EdgeOS
* doesn't break after each update (like pfSense tends to do)

I've followed the [basic setup guide](https://wiki.vyos.net/wiki/User_Guide) but there were some bumps along the road which my blog will cover.

<!--more-->

The issue was that I couldn't access some websites, such as Wikipedia or Speedtest.net. I could access Google though.

Since I have a PPPoE connection, MTU was the culprit. Ethernet interfaces have a MTU of 1500 which doesn't apply to PPPoE interfaces because the PPPoE header takes 8 bytes of space.

Therefore, PPPoE's MTU is 1492. Now this means you have to 'modify' the TCP packets going through your router to have a MTU of **1452** (1492 - 40).

On VyOS this can be fixed with a routing policy. It needs to be applied on the WAN and LAN interface(s).

*On EdgeOS this can be set globally.**

Here are the commands I've used to fix this. Of course, adapt them to match your setup.

```
set policy route MSS-CLAMP rule 1 protocol tcp
set policy route MSS-CLAMP rule 1 set tcp-mss 1452
set policy route MSS-CLAMP rule 1 tcp flags SYN

# WAN interface (incoming?)
set interfaces ethernet eth1 pppoe 0 policy route MSS-CLAMP

# LAN interface (outgoing?)
set interfaces bonding bond0 vif 10 policy route MSS-CLAMP
```

Have a nice day everyone :^)
