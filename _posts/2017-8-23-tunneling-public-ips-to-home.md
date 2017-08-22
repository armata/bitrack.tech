---
layout: post
title: Tunneling public IPs from data center to home with pfSense and OpenVPN
---

So you'd like to host servers at home and you have a dynamic IP.

No problem, right? Just fire up the trusty Dynamic DNS and you should be good to go.

You could also use a Nginx reverse proxy on your VPS and OpenVPN in routing/tun mode. But what if you want to retain the original source IP of your traffic? Or what if you need a static IP and not a domain?

Thankfully, there's a far better solution. You could create a L2 bridge between your VPS/dedicated server network and your pfSense box.

Not only your IP will never change, you'll be able to have multiple of them and all data will be encrypted.

The VPS I'm using is the 3€/month one from OVH. It can have up to 16 additional IPs.

Of course, you could do this on a dedicated server with a gigabit connection and with more IPs.

# VPS/dedicated server configuration

## OpenVPN server config

First, you'll need to configure OpenVPN in bridge/tap mode. Here's my server configuration (server.conf).

```
port 1194

proto udp

dev tap0

ca ca.crt
cert server.crt
key server.key

dh dh2048.pem

keepalive 10 120

tls-server
tls-auth ta.key 0

auth SHA256

cipher AES-256-CBC

user nobody
group nogroup

persist-key
persist-tun
```

The only important part here is the `dev tap0` line. This will tell the server to run in bridge/tap mode and it will use existing tap0 interface.

I'm not going to cover the certificate/key creation part but [here's a good tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04) that should get you going.

Basically, it's the same as if you were configuring a normal OpenVPN tun server.

## Network configuration

We need to bridge the `tap0` and `eth0` (or whatever interface you use to connect to the internet) to `br0`.

Here's my `/etc/network/interfaces` file. Make sure to check the commenents!

```
# The loopback network interface
auto lo
iface lo inet loopback

auto br0
iface br0 inet static
    address YOUR_VPS_IPV4_ADDRESS
    netmask 255.255.255.255
    # ovh-specific; check your 'normal' eth0 config and edit this accordingly
    post-up /sbin/ip route add 164.132.192.1 dev br0
    post-up /sbin/ip route add default via 164.132.192.1
    pre-down /sbin/ip route del default via 164.132.192.1
    pre-down /sbin/ip route del 164.132.192.1 dev br0
    dns-nameserver 213.186.33.99
    dns-search ovh.net
    # the important part -- this will create the tap0 tunnel and bridge it
    pre-up openvpn --mktun --dev tap0
    bridge_ports eth0 tap0
    bridge_fd 3
```

After you do all the necessary edits, you should restart your VPS/dedicated server.

# Configuring pfSense

## OpenVPN client configuration

After you've added all the necessary CAs/certs/keys to pfSense, we'll have to configure the OpenVPN client.

Again, this part will not be covered -- just make sure the settings are the same as on your server.

Obviously, you'll have to set device mode to tap here.

## Interface configuration

Go to `Interface Assignments` and create an interface with `ovpnc1` as your network port.

Grab one of your additional IPs you've got from your VPS/dedi host and set it as the interface IP.

Now go to Virtual IPs and add that same IP. You can go ahead and add the other ones if you have them.

Go back to the interface configuration and set the gateway. This one should be the same as your gateway in your VPS config (example: 164.132.192.1).

Make sure to set *Use non-local gateway through interface specific route.* in gateway config if necessary.

That should be it!
