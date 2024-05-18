+++
title = 'Minetest Server Behind CGNAT'
date = 2024-05-11T18:22:35+05:00
draft = false
+++

## Make the Minetest server publicly available

Do you want to play Minetest(or any other game) with your friends on your server but unfortunately your ISP uses CGNAT? In this blog post I will explain how I bypassed this restriction and will show you what I did.

#### Author:
I am Soupborsh, I am not a GNU/Linux professional and can make mistakes, consequently, I am not responsible for any possible damage.

### This will require:
- Remote server(VPS with public ipv4 or ipv6) (*I will refer to it as VPS*)
- Home server with internet connection (*I will refer to it as homeserver*)

**This should work with anything using UDP, even with TCP if you can configure iptables for it*

### My situation:
My ISP uses CGNAT for IPv4 and blocks connections from Internet for IPv6.

VPS OS: Ubuntu 22.04 LTS

## How does this work?

To make sure that people all over the internet will be able to connect to your home server which is behind CGNAT(Carrier Grade NAT) I decided to use WireGuard. This is working like this:

Minetest server <-- homeserver <-- WireGuard tunnel <-- VPS <-- Internet <-- Client(Player)

We need to connect to the VPS using Wireguard. It creates virtual network devices on homeserver and VPS. These have private IPs of other peer, for example, these are outputs of
```ip a``` on homeserver and VPS after configuring everything.

homeserver:
```shell
14: srv: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none 
    inet 10.66.66.2/32 scope global srv
       valid_lft forever preferred_lft forever
    inet6 fd42:42:42::2/128 scope global 
       valid_lft forever preferred_lft forever
```
VPS:
```shell
3: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 8920 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none 
    inet 10.66.66.1/24 scope global wg0
       valid_lft forever preferred_lft forever
    inet6 fd42:42:42::1/64 scope global 
       valid_lft forever preferred_lft forever
```

```10.66.66.2``` and ```fd42:42:42::2``` are private vpn ips of homeserver. On a VPS we will redirect all incoming traffic going to VPS's public ip on port 3000 on udp to ```10.66.66.2``` and ```fd42:42:42::2``` aka homeserver.

## Step by step guide

#### VPS:

#### 1. Creating WireGuard config

 I decided to configure WireGuard using this wireguard-install script:
```shell
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
chmod +x wireguard-install.sh
./wireguard-install.sh
```

It should promt you something like this:

```shell
Welcome to the WireGuard installer!
The git repository is available at: https://github.com/angristan/wireguard-install

I need to ask you a few questions before starting the setup.
You can keep the default options and just press enter if you are ok with them.

IPv4 or IPv6 public address: public_vps_address
Public interface: enp37s0
WireGuard interface name: wg0
Server WireGuard IPv4: 10.66.66.1
Server WireGuard IPv6: fd42:42:42::1
Server WireGuard port [1-65535]: 58838
First DNS resolver to use for the clients: 1.1.1.1
Second DNS resolver to use for the clients (optional): 1.0.0.1
...
Client configuration

The client name must consist of alphanumeric character(s). It may also include underscores or dashes and can't exceed 15 chars.
Client name: soupborsh
Client WireGuard IPv4: 10.66.66.2
Client WireGuard IPv6: fd42:42:42::2
...
Here is your client config file as a QR Code:
...
Your client config file is in /home/soupborsh/wg0-client-soupborsh.conf
If you want to add more clients, you simply need to run this script another time!
```
It automatically tries to detect your public ip and randomy generates everything else
Change something if it is wrong or you do not like this value. I changed only ```public_vps_address``` to my real public ip address(I have used IPv6 but you can use IPv4 too). 

Remember the WireGuard port as well as Server(VPS) and Client(homeserver) IPs, as we will need them later.

#### 2. Configuring iptables

The simplest way I found to redirect traffic from VPS to homeserver is using iptables. Run these commands replacing ```homeserver_private_vpn_ipv4``` ```homeserver_private_vpn_ipv6``` with your actual private WireGuard ipv4 and ipv6 of homeserver(in my case it is 10.66.66.2 and fd42:42:42::2).

```shell
sudo iptables -t nat -A PREROUTING -p udp --dport 30000 -j DNAT --to-destination homeserver_private_vpn_ipv4:30000
sudo ip6tables -t nat -A PREROUTING -p udp --dport 30000 -j DNAT --to-destination [homeserver_private_vpn_ipv6]:30000
```

```shell
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
sudo ip6tables -t nat -A POSTROUTING -j MASQUERADE
```

Make these rules persistent:

```shell
sudo apt install iptables-persistent
```

```shell
sudo sh -c "iptables-save > /etc/iptables/rules.v4"
sudo sh -c "ip6tables-save > /etc/iptables/rules.v6"
```

#### 3. Opening ports

We need to open WireGuard port and port for Minetest(default is 30000) on VPS to be able to connect to it from homeserver. I use ufw as my firewall. Change ```wireguard_port``` to the port of WireGuard we have set up earlier.

```shell
sudo ufw allow wireguard_port/udp
sudo ufw allow 30000/udp
sudo ufw reload
```
You can see which ports are open:
```shell
sudo ufw status
```

```shell
Status: active

To                         Action      From
--                         ------      ----
22/tcp                   ALLOW       Anywhere                  
30000                      ALLOW       Anywhere                  
58838/udp                  ALLOW       Anywhere                  
22/tcp (v6)              ALLOW       Anywhere (v6)             
30000 (v6)                 ALLOW       Anywhere (v6)             
58838/udp (v6)             ALLOW       Anywhere (v6)
```

If it says that status is not active, enable ufw, but be sure that your ssh connection is allowed in it, due to that you can lose your ssh connection to your VPS like that you will have to have to login into your server localy as to regular PC without internet connection.

```shell
sudo ufw enable
```

### Homeserver:
#### 1. We need to copy WireGuard .conf files. It can be done with ```scp```:

```shell
sudo scp user@vps_public_ip:/path/to/conf/interface.conf /etc/wireguard/srv.conf
```
Or like this if you use keys and a custom ssh port:
```shell
sudo scp -i /path/to/private/key -P ssh_port user@vps_public_ip:/path/to/conf/interface.conf /etc/wireguard/srv.conf
```
Change ```/path/to/private/key```, ```ssh_port```, ```user```, ```vps_public_ip``` and ```/path/to/conf/interface.conf``` to yours. Additionally, I set a name for the downloaded config to ```srv.conf``` instead of ```wg0-client-soupborsh.conf```, due to that ```wg0-client-soupborsh.conf``` gave me issues. Rename it to your liking but I think you can use only lowercase English letters without hyphens and other symbols.

#### 2. Connect to WireGuard VPN.

```shell
sudo wg-quick up srv
```
Change ```srv``` to the name of your config file in ```/etc/wireguard/``` without ".conf" at the end.

#### 3. Minetest server!

If you want IPv6 support add this to the end of your ```minetest.conf```:

```toml
ipv6_server = true
```

Start or restart(firstly turn off using ctrl + c) your server:
```shell
minetestserver
```
Or by any other method you use to start and stop your Minetest server.

## I hope everything went well for you and you are enjoying your game!
