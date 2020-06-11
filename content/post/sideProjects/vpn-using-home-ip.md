---
title: "Set up a vpn server that uses home ip as exit ip"
date: 2020-06-11T17:56:48+08:00
draft: false
lastmod: 2020-06-11T17:56:48+08:00
tags: ["network", "wireguard", "vpn", "shadowsocks", "iptables"]
categories: ["side projects"]
---

It all started with my previous bad experience when doing VPN hosting myself...

<!--more-->

If we host VPN servers on GCP/AWS/DO/Linode, you now have all the devices on local network, but the common problems/downsides are e.g. streaming services like Netflix will block you (so you need to turn off VPN a lot of times...); Google search will often challenge you with recaptcha, etc. So I would like to find a way to workaround it :) 

## Expectation / Solution

The VPN servers in the cloud usually have bad IPs, but the IP that we are using at home is usually good and not blocked! So if we can somehow redirect all external traffic going out of VPN servers hosted in the cloud to our home IP, we are all set.

But hosting VPN server at home, which is usually a floating IP and behind NAT, is hard. So we need some workaround for it!

# Architecture

This is my final setup.

```
        VPN server (on GCP)
        /                \ 
       (VPN)          (ss + VPN)
       /                  \
  smartphone         rpi sitting at home 
```

The final solution is actually pretty simple. Client traffic coming from VPN subnet to GCP are redirected through ss tunnel to rpi, then exit to the world from there.

So basically
* Install wireguard and shadowsocks on vpn server and rpi
* Connect vpn server and rpi with wireguard
* Setup shadowsocks server on rpi 
* Setup shadowsocks client on vpn server, connect to shadowsocks server on rpi, and forward traffic to shadowsocks server  

## Install shadowsocks on vpn server and rpi

* `go get -u -v github.com/shadowsocks/go-shadowsocks2`
    * manual is here `https://github.com/shadowsocks/go-shadowsocks2`

## Install wireguard on vpn server and rpi

* `sudo apt install wireguard`
* don't forget to open the port for connection on cloud computing platforms
* allow ip forwarding on servers
    * `sudo sysctl -w net.ipv4.ip_forward=1` 
    * `sudo sysctl -p /etc/sysctl.conf` to reload settings

### Common commands for wireguard

* generate wireguard keys by `wg genkey | tee wg-private.key | wg pubkey > wg-public.key`
* start the wireguard client/server using `sudo wg-quick up wg0`
* stop the wireguard client/server using `sudo wg-quick down wg0`
* check the wireguard connection using `sudo wg show`
    * can do `watch -n 1 sudo wg show`

## Setup wireguard vpn server 

```
VPN server
```

* sample config file, assuming file name is `wg0.conf`
```
# vpn server

[Interface]
Address = 10.200.200.1/24 # subnet
ListenPort = 12345 # external port for incoming connection
PrivateKey = [vpn server's private key]

# if you use -A, it might get installed after the drop rule + log rule!!
PostUp   = iptables -I FORWARD 1 -i %i -j ACCEPT; iptables -I FORWARD 1 -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o [your outgoing interface name] -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o [your outgoing interface name] -j MASQUERADE

[Peer]
# rpi server
PublicKey = [rpi server's public key]
AllowedIPs = 10.200.200.2/32 # peer's ip in subnet
PersistentKeepalive = 30 # good for peers behind NAT
```
* run `sudo wg-quick up ./wg0.conf` in the directory where `wg0.conf` is present

## Setup rpi and connect to vpn server

```
VPN server ---(wireguard connection)--- rpi
```

* sample config file, assuming file name is `wg0.conf`
```
# rpi server

[Interface]
Address = 10.200.200.2/24 # subnet ip 
PrivateKey = [rpi server's private key]
DNS = 1.1.1.1
        
[Peer]
PublicKey = [vpn server's public key]
AllowedIPs = 10.200.200.1/32 # add all subnet ips that might have traffic coming in, so the return traffic can go through
Endpoint = [vpn server ip:port]
PersistentKeepalive = 30 # good for peers behind NAT
```
* run `sudo wg-quick up ./wg0.conf` in the directory where `wg0.conf` is present
    * connection between vpn server and rpi should be made and observable through `wg show`

## Setup shadowsocks server on rpi

```
VPN server ---(wireguard connection)--- rpi(ss server)
```

* run `./go/bin/go-shadowsocks2 -s 'ss://AEAD_CHACHA20_POLY1305:[password]@:[port that server will listen on]' -verbose` to start shadowsocks server on rpi

## Setup shadowsocks client on vpn server and connect it to shadowsocks server on rpi

```
VPN server (ss client) ---(wireguard connection, ss)--- rpi(ss server)
```

* run `./go/bin/go-shadowsocks2 -c 'ss://AEAD_CHACHA20_POLY1305:[password]@[ss server ip:ss server port]' -redir :[port that redirected traffic should go to] -verbose` to setup shadowsocks client on vpn server for traffic redirection 

## Setup iptables on vpn server to do port forwarding on the port that shadowsocks client is listening to

```
VPN server (ss client) ---(wireguard connection, ss)--- rpi(ss server)
```

* iptables script
```
# list all rules
sudo iptables -L -v && sudo iptables -t nat -L -v

# Create new chain
iptables -t nat -N SHADOWSOCKS

# Ignore your shadowsocks server's addresses
# It's very IMPORTANT, just be careful.
iptables -t nat -A SHADOWSOCKS -d 10.200.200.2 -j RETURN

# Ignore LANs and any other addresses you'd like to bypass the proxy
# See Wikipedia and RFC5735 for full list of reserved networks.
# See ashi009/bestroutetb for a highly optimized CHN route list.
iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN

# Anything else should be redirected to shadowsocks's local port
iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-ports 8488

# Apply the rules
iptables -t nat -A PREROUTING -p tcp -j SHADOWSOCKS

# allow wg0 traffic
sudo iptables -I INPUT 1 -i wg0 -p tcp --dport 8488 -j ACCEPT
# sudo iptables -D INPUT 1

# list all rules
sudo iptables -L -v && sudo iptables -t nat -L -v
```
* reference for [the iptables setup](https://manpages.debian.org/testing/shadowsocks-libev/shadowsocks-libev.8.en.html)

## Setup clients e.g. iPhones

* gen keys by `wg genkey | tee wg-private.key |  wg pubkey > wg-public.key`
* create a client config
```
[Interface]
Address = [client ip]/24
PrivateKey = [client's private key]
DNS = 1.1.1.1
        
[Peer]
PublicKey = [vpn server's public key]
AllowedIPs = 0.0.0.0/0 # for passing all traffic through wireguard
Endpoint = [server ip]:[server port]
```
* add a peer in server config, and `down` then `up` the interface
* use `qrencode` to generate qrcode for clients to scan on their wireguard app on phones :) 
    * generate qrcode by `qrencode -t ansiutf8 < [filename]`
    * install qrcode by `sudo apt install qrencode resolvconf`

# Tried but failed solutions

Actually the main problem is how to preserve the destination IP while re-routing the packet to other places. That's why I settled with the wireguard + shadowsocks solution.
