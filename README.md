# torpi
Setup a Raspberry Pi 4 as a Transparent TOR Proxy with DNS

## Overview
The Raspberry Pi 4 has two network interface (wlan0 and eth0)
Wlan0 will be designated as the connection to the internet (i.e. connect to an Wireless Access Point)
Eth0 will be a connection to a laptop, switch, or whatever for clients
The interface setup can be flip-flopped

## Download the Ubuntu 20.04 LTS image and write to memory card for Pi
https://ubuntu.com/download/raspberry-pi

## Update/Configure system and install TOR
```
ssh ubuntu@<PI_ETH0_IP_ADDR>
sudo su -
passwd
apt update
apt upgrade
useradd -m -U torpi
hostnamectl set-hostname torpi
passwd torpi
apt install tor
userdel -r -f ubuntu
sysctl -w net.ipv4.conf.all.forwarding=0
```

## Remove UFW and install iptables-persistent
```
ufw disable
apt remove ufw
apt purge ufw
apt install iptables-persistent
```

## Setup Netplan Configuration
```
vim /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        eth0:
            addresses: [192.168.222.1/24]
    version: 2
    wifis:
        wlan0:
            dhcp4: true
            access-points:
                "myaccesspoint":
                    password: "supersecretpassword"
```

## Finalize System setup
```
netplan generate
netplan apply
ssh torpi@192.168.222.1
reboot now
```

## Configure TORRC
```
vim /etc/tor/torrc
Log notice file /var/log/tor/notices.log
VirtualAddrNetworkIPv4 10.192.0.0/10
AutomapHostsSuffixes .onion,.exit
AutomapHostsOnResolve 1
TransPort 192.168.222.1:9040
DNSPort 192.168.222.1:5353
```

## Configure IPTABLES v4 Rules
TODO: add more grainular rules in INTERWEBZ CHAIN TO LOCK DOWN ONLY TOR traffic
```
vim /etc/iptables/rules.v4
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A PREROUTING -i eth0 -p tcp -m tcp --dport 22 -j REDIRECT --to-ports 22
-A PREROUTING -i eth0 -p udp -m udp --dport 53 -j REDIRECT --to-ports 5353
-A PREROUTING -i eth0 -p udp -m udp --dport 5353 -j REDIRECT --to-ports 5353
-A PREROUTING -i eth0 -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -j REDIRECT --to-ports 9040
COMMIT
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]
:INTERWEBZ - [0:0]
:PROXYIN - [0:0]
:PROXYOUT - [0:0]
:LOGDROP - [0:0]
# INPUT CHAIN
-A INPUT -i lo -j ACCEPT
-A INPUT -i wlan0 -j INTERWEBZ
-A INPUT -i eth0 -j PROXYIN
-A INPUT -j DROP
# FORWARD CHAIN
-A FORWARD -j DROP
# OUTPUT CHAIN
-A OUTPUT -o lo -j ACCEPT
-A OUTPUT -o wlan0 -j INTERWEBZ
-A OUTPUT -o eth0 -j PROXYOUT
-A OUTPUT -j DROP
# INTERWEBZ CHAIN
-A INTERWEBZ -j ACCEPT
# PROXYIN CHAIN
-A PROXYIN -p tcp -m tcp --dport 22 -m state --state NEW -j LOG --log-prefix "NEW-SSH-IN: "
-A PROXYIN -s 192.168.222.0/24 -d 192.168.222.1/32 -i eth0 -p tcp -m tcp --dport 22 -j ACCEPT
-A PROXYIN -s 192.168.222.0/24 -d 192.168.222.1/32 -p udp -m udp --dport 5353 -j ACCEPT
-A PROXYIN -s 192.168.222.0/24 -d 192.168.222.1/32 -p tcp -m tcp --dport 9040 -j ACCEPT
-A PROXYIN -j LOGDROP
# PROXYOUT CHAIN
-A PROXYOUT -s 192.168.222.1/32 -d 192.168.222.0/24 -o eth0 -p tcp -m tcp --sport 22 -m state --state RELATED,ESTABLISHED -j ACCEPT
-A PROXYOUT -s 192.168.222.1/32 -d 192.168.222.0/24 -o eth0 -p udp -m udp --sport 5353 -m state --state RELATED,ESTABLISHED -j ACCEPT
-A PROXYOUT -s 192.168.222.1/32 -d 192.168.222.0/24 -o eth0 -p tcp -m tcp --sport 9040 -m state --state RELATED,ESTABLISHED -j ACCEPT
-A PROXYOUT -j LOGDROP
# LOGDROP CHAIN
-A LOGDROP -m limit --limit 5/min -j LOG --log-prefix "DROP: " --log-level 7
-A LOGDROP -j DROP
COMMIT
```

## Enable TOR service and Check Client
```
systemctl enable tor
systemctl start tor
```

With a client device connected to eth0 of the Raspberry Pi 4, go to 
https://check.torproject.org/

