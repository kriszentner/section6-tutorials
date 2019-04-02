 ## Creating a gateway router with natd and stateful firewall in FreeBSD

* To set up my home freebsd box as a natd/firewall/router I used the following config settings.
* If done right, the whole process can be done by working with 4 files and only 1 reboot.

## /usr/src/sys/i386/conf/NEWKERNEL
Compile a custom kernel with the following options:

*  http://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/kernelconfig-building.html

```conf
options IPFIREWALL
options IPFIREWALL_VERBOSE
options IPDIVERT
```
## /etc/rc.conf

* See /etc/default/rc.conf for a full list of options and short info about each one.

```conf
#Public Interface
ifconfig_fxp0="DHCP"
#These are connected to my two workstations
ifconfig_re0="inet 192.168.0.1 netmask 255.255.255.0"
ifconfig_re1="inet 192.168.1.1 netmask 255.255.255.0"
#To make sure I can route between workstations
router_enable="YES"
#for natd firewall magic
gateway_enable="YES"
natd_enable="YES"
natd_flags="-f /etc/natd.conf"
firewall_enable="YES"
firewall_script="/etc/ipfw.rules"
```

## /etc/natd.conf

Create this file.

```conf
#I'm using a cable modem with a dynamic IP
dynamic yes
interface fxp0 
same_ports yes
unregistered_only
#I want to be able to log into one of my workstations via Remote Desktop from the internet
redirect_port tcp 192.168.1.2:3389 3389
```

## /etc/ipfw.rules

* http://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/firewalls-ipfw.html

```bash
#!/bin/sh
#This comes from the FreeBSD Handbook, I've modified it to fit my needs
cmd="/sbin/ipfw add"
skip="skipto 5000"
pif="fxp0"
ks="keep-state"
dns1="IP address for one of the DNS servers I use"
dns2="IP address for the other DNS server I use"
ws="IP address to my webserver hosted elsewhere"
office="IP address to my office"
```

```bash
/sbin/ipfw -q -f flush

${cmd} 0030 allow all from any to any via lo0  # exclude loopback traffic
 
${cmd} 1000 divert natd ip from any to any in via ${pif}
${cmd} 1010 check-state

# Authorized outbound packets
${cmd} 1200 ${skip} udp from any to ${dns1} 53 out via ${pif} ${ks}
${cmd} 1210 ${skip} udp from any to ${dns2} 53 out via ${pif} ${ks}
${cmd} 1250 ${skip} tcp from any to any out via ${pif} setup ${ks}
${cmd} 1300 ${skip} icmp from any to any out via ${pif} ${ks}
${cmd} 1350 ${skip} udp from any to any out via ${pif} ${ks}
${cmd} 1450 ${skip} all from any to any via re1 #workstation 1
${cmd} 1451 ${skip} all from any to any via re2 #workstation 2
#Remember the RDP forward? Well I only want to be able to do it from my office, all else gets blocked.
${cmd} 1460 ${skip} tcp from ${office} to 192.168.1.2 3389 via fxp0 setup ${ks}

# Deny all inbound traffic from non-routable reserved address spaces
${cmd} 3000 deny all from 192.168.0.0/16  to any in via ${pif}  #RFC 1918 private IP
${cmd} 3010 deny all from 172.16.0.0/12   to any in via ${pif}  #RFC 1918 private IP
${cmd} 3020 deny all from 10.0.0.0/8      to any in via ${pif}  #RFC 1918 private IP    
${cmd} 3030 deny all from 127.0.0.0/8     to any in via ${pif}  #loopback
${cmd} 3040 deny all from 0.0.0.0/8       to any in via ${pif}  #loopback
${cmd} 3050 deny all from 169.254.0.0/16  to any in via ${pif}  #DHCP auto-config
${cmd} 3060 deny all from 192.0.2.0/24    to any in via ${pif}  #reserved for docs
${cmd} 3070 deny all from 204.152.64.0/23 to any in via ${pif}  #Sun cluster
${cmd} 3080 deny all from 224.0.0.0/3     to any in via ${pif}  #Class D & E multicast

# Authorized inbound packets
#Pings from my webserver should be handled by this box.
${cmd} 4100 allow icmp from ${ws} to me in via ${pif}
${cmd} 4110 allow icmp from me to ${ws} out via ${pif}

#ssh from my webserver should go to this box.
${cmd} 4200 allow tcp from ${ws} to me 22 in via ${pif}
${cmd} 4210 allow tcp from me 22 to ${ws} out via ${pif} 

#ssh from my office should go to this box.
${cmd} 4400 allow tcp from ${office} to me 22 in via ${pif}
${cmd} 4410 allow tcp from me 22 to ${office} out via ${pif}

#if it hasnt generated a dynamic rule, or been explicitly allowed, then deny
${cmd} 4500 deny ip from any to any

# This is skipto location for outbound stateful rules
${cmd} 5000 divert natd ip from any to any out via ${pif}
${cmd} 5100 allow ip from any to any
```
