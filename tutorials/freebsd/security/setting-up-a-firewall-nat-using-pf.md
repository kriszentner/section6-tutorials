# Setting up a Firewall NAT using PF

## Creating a FreeBSD Firewall using PF
author: TBONIUS & kz  
Revised: 12/10/2004  

PF is probably one of the best firewalls available. Here we will discuss what's needed to create your own firewall. PF is also part of the FreeBSD base system starting with version 5.3. If you're using a version below 5.3 you will need to use the port (and your kernel requirements will be different).

## Requirements

* A 486 processor or better with 128mb or RAM, two network cards, and a 500M drive (1G preferred)
* A working FreeBSD installation (of course)
* An ethernet hub or switch
* Two Cat5 ethernet cables

## Configuring the kernel

You will also need to add the following options to your kernel and recompile a new kernel

```conf
device          pf                      #PF OpenBSD packet-filter firewall
device          pflog                   #logging support interface for PF
device          pfsync                  #synchronization interface for PF
```

If you Want ALTQ you will also need these

```conf
#ALTQ Support
options         ALTQ
options ALTQ_CBQ        # Class Bases Queueing
options ALTQ_RED        # Random Early Drop
options ALTQ_RIO        # RED In/Out
options ALTQ_HFSC       # Hierarchical Packet Scheduler
options ALTQ_CDNR       # Traffic conditioner
options ALTQ_PRIQ       # Priority Queueing
```

## Configuring /etc/rc.conf

Configure your network interfaces in `/etc/rc.conf`. One will be an internal interface and one will be your external. In the following example, ed0 is the internal interface, ed0 is the internal. You will also need to allow your freebsd server to be a router. Sample:

```conf
ifconfig_fxp0="inet 1.2.3.4 netmask 255.255.255.0" #Our external interface
ifconfig_ed0="inet 10.0.0.1 netmask 255.255.255.0"    #Our internal interface
pf_enable="YES"
pflogd_enable="YES"
gateway_enable="YES"
```

Optionally you could configure fxp0 to use a dynamic address if your ISP requires:

```conf
ifconfig_fxp0="DHCP"
```

## Configuring the Firewall Rules

The file `/etc/pf.conf` contains all firewall rules for PF. You'll note that the rules must be in a certain order or `pfctl` will complain. The following example derived from a working pf.conf. The examples contained within should be enough to get you started.

```conf
# define macros for each network interface
extif = "xl0"
intif = "fxp0"
tcp_services = "{ 22, 443 }"

# define our networks
intnet  = "{ 10.0.0.0/24, 192.168.0.0/24 }"
extaddr = "1.2.3.4"
natone  = "10.0.0.2"
nattwo  = "10.0.0.3"

icmp_types = "echoreq"
allproto = "{ tcp, udp, ipv6, icmp, esp, ipencap }"
privnets = "{ 127.0.0.0/8, 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }"
bittorrent = "{ 6881, 6882, 6883, 6884, 6885, 6886, 6887, 6888, 6889 }"

set loginterface $extif

# Normalizes packets and masks the OS's shortcomings such as SYN/FIN packets 
# [scrub reassemble tcp](BID 10183) and sequence number approximation 
# bugs (BID 7487).
scrub on $extif reassemble tcp no-df random-id

#############
# NAT Rules #
#############
nat on $extif from $intif:network to any -> ($extif)

#HTTP, HTTPS, to natone
rdr on $extif proto tcp from any to any port 80 -> $natone
rdr on $extif proto tcp from any to any port 443 -> $natone

#SSH to natone
rdr on $extif proto tcp from any to any port 22 -> $natone

#Bittorrent to nattwo
rdr on $extif proto tcp from any to any port $bittorrent -> $nattwo

# The spamd interface, we redirect bad IP's to the spamd port, keep in mind
# the table has a limit so don't put thousands of addresses in there!
table <spammers> persist file "/usr/local/etc/spamnets"
rdr on $extif inet proto tcp from <spammers> to any port 25 -> 127.0.0.1 port 8025

###########
# END NAT #
###########

block log
pass quick on lo0 all

#This is necessary to pass to spamd
pass quick proto tcp from any to $privnets port 8025

#"Block drop in quick" will kill the rdr rules above for the privnet
block drop in on $extif from $privnets to any
block drop in on $extif from any to $privnets

################################
# Begin Selective Port Opening #
################################

#For a Mail server
pass in on $extif proto tcp from any to any port 25 flags S/SA
 
#WebServer, HTTPS, 8000
pass in on $extif proto tcp from any to any port 80 flags S/SA
pass in on $extif proto tcp from any to any port $tcp_services flags S/SA synproxy state
#pass in on $extif proto tcp from any to $natone port 80 flags S/SA keep state

# DNS server 
pass in on $extif proto {tcp, udp} from any to any port 53

# Allow specific servers to ident me (synproxy breaks ident)
table <ircservers> persist file "/usr/local/etc/ircservers"
pass in quick on $extif proto tcp from <ircservers> to any port 113  flags S/SA
 
###############
 # Basic Rules #
###############

pass in inet proto icmp all icmp-type $icmp_types keep state

#Lets keep the local net free
pass in  on $intif from $intif:network to any keep state
#Allow fw to establish connections to internal net
pass out on $intif from any to $intif:network keep state

#Pass out TCP UDP, ICMP and ipv6
pass out on $extif proto ipv6 all
#This doesn't work, maybe needs altq?
pass out on $extif proto tcp all modulate state flags S/SA
#pass out on $extif proto { tcp, udp, icmp } all keep state
pass out on $extif all keep state
```

Before blindly using all the above rules keep in mind that this isn't the best ruleset. In this example I'm allowing all traffice passed out. Ideally, you would filter outgoing traffic as well, especially if windows boxes are on your network as if they got a virus they would be free to use any outgoing ports for communication.

### A note on scrub

Above you'll notice an interesting rule:

```conf
scrub on $extif reassemble tcp no-df random-id
```

This will fix packets with illegal TCP options such as SYN/FIN. For more information and options you can set check out the documentation.

### A note on synproxy

You'll notice that one of the rules was passed with flags S/SA synproxy state. The synproxy is discussed further here, and its main purpose is to defend agains SYN floods. However, I've found with services such as IDENT it can break. Using it on a web server I've found certain people (including most on SBC-Yahoo's network) are unable to reach it. Keep this in mind if you enable it on one of your services.  

## Using spamd

This rulset also shows how to run spamd as part of your firewalling system. Spamd can be found in ports at ports/mail/spamd. There are several ways to do so but I have a table which holds a list of IP addresses in `/usr/local/etc/spamnets` the table is read with one address per line, shell style comments (#) are allowed, as are CIDR blocks (like /24).

Several lists can be be obtained from the internet for populating a spamnet table. If your mailserver doesn't do any business with countries like China or Korea, you're better off honeypotting them. Section6 has recieved many Chinese hackers falling into its honeypot. You can obtain a list of Chinese and Korean IP blocks go here. Other RBL's can be foundhere, however be advised. A very large table can crash your filter. I have yet to find the exact limit to what a table can be, but a four megabyte list will crash it fairly consistantly.

### List management

* To add contents of a file to a table named spammers in pf:

```shell
cat file | pfctl -t spammers -T replace -f -
```

* To manually add a netblock to a table named spammers:

```shell
pfctl -t spammers -T add 200.0.0.0/8. 
```

To view the spammers table:

```shell
pfctl -t spammers -T show
```

## Syslogging

If you want to see honeypot activity separate from the pool of messages in `/var/log/messages` you can append the following to the bottom of you `/etc/syslog.conf`

```conf
!spamd
*.*                                             /var/log/spamd.log
```

## Conclusion

PF can be a most effective packet filtering firewall for FreeBSD. Its ease of ruleset creation and solid state session filtering can effectively secure your network against unwanted intruders.

## Resources

* FreeBSD PF port - http://pf4freebsd.love2party.net
* PF FAQ - http://www.openbsd.org/faq/pf
* PF tutorials and links - https://solarflux.org/pf