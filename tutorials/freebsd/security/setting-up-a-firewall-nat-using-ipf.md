# Setting up a Firewall NAT using IPF

## Creating a FreeBSD NAT router/firewall
author: kz  

This tutorial assumes you are using this for a broadband connection. Note that NAT is not supported by IPv6. There are so many addresses it should never become necessary.

## What's needed

* A PC that is a 486 or better with a 2 network cards, and a 500M drive (1G preferred).
* A hub
* 2 Cat5 Ethernet Cables
* A working FreeBSD installation (see the install section of the handbook)


## Be sure you have Ipfilter in your kernel

You'll need to do a kernel compile and include ipfilter. If you've never compiled a kernel before, take a look at the Kernel Compile Section of the handbook. You'll need to add:

```conf
options		IPFILTER
options		PFIL_HOOKS
```

to your kernel configuration file. options: PFIL_HOOKS is only necessary if you are using FreeBSD 5.2x or higher.

## Configuring the /etc/rc.conf

Configure your network interfaces in `/etc/rc.conf`. One will be an internal interface and one will be your external. In the following example, fxp0 is the internal interface, ed0 is the internal. You will also need to allow your freebsd server to be a router. Sample:

```conf
ifconfig_fxp0="inet 10.0.0.1  netmask 255.255.255.0" #Our internal interface
ifconfig_ed0="inet 1.2.3.4 netmask 255.255.255.0"    #Our external interface
gateway_enable="YES"
ipfilter_enable="YES"
ipnat_enable="YES"
```

## Create Your NAT Rules

You will need to put the following in your `/etc/ipnat.rules` file:

```conf
#   Dev  Inside IP     Local Inet IP
map ed0 10.0.0.0/24 -> 0.0.0.0/32 portmap tcp/udp 40000:65000
map ed0 10.0.0.0/24 -> 0.0.0.0/32
```

The first line specifies which port range for NAT usage. You may adjust this if you know what you're doing. Otherwise this is a good default. Also, if you have a static ip you can replace `0.0.0.0` with your static ip address in the above and below examples. Keep in mind that ed0 is your external interface. This example will give your network IP addresses in the range of `10.0.0.2 - 10.0.0.254`.

## Port forwarding

If you have NAT set up, but you want to forward ports to, for example, a web server behind your NAT router with an address of `10.0.0.20`, you can add a rule such as this one to your rules file.

```conf
#Redirection of HTTP
rdr ed0 0.0.0.0/0 port 80 -> 10.0.0.20 port 80
```

## Create Your Filter Rules

Here is a sample `/etc/ipf.rules` file:

```conf
#Basic ruleset
block in all with frag

#Only I can pass packets out on the external interface (This is ok for NAT)
pass out quick on ed0 proto tcp from 1.2.3.4 to any keep state
pass out quick on ed0 proto udp from 1.2.3.4 to any keep state
pass out quick on ed0 proto icmp from 1.2.3.4 to any keep state
pass out quick all

block in on ed0 proto icmp all
pass in on ed0 proto icmp from any to any icmp-type echo
pass in on ed0 proto icmp from any to any icmp-type echorep
block in on ed0 proto icmp from any to any icmp-type unreach code 3

#Block all other non-established connections
block in quick on ed0 proto tcp from any to any flags S/SA
```

This sample shows 1.2.3.4 as the external ip and ed0 as the external interface. This is a very basic firewall file. You may wish to visit the Official IP Filter Site for more details.

## Load the rules

After you are done editing the rules and if you are running a kernel with IP Filter support you can (re)load your ipnat and ipfilter rules respectively with these commands:

```shell
ipnat -CF -f /etc/ipnat.rules
ipf -F a -f /etc/ipfilter.rules
```
