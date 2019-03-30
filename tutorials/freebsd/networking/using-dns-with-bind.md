# Using DNS with BIND
## Setting up BIND 9 on FreeBSD
author: kz  
revised: 12/1/2004  

## Introduction

Here is a sprint through DNS. It is in no way meant to explain the intricacies of DNS as well as a book (like DNS and BIND by O’Reilly publishing). It is however, meant to get you running DNS quickly with a rudimentary knowledge of how it all works.

Many of these configurations will run on Bind 8 as well. However they have not been tested. If you can I highly suggest running version 9 for the following reasons:

* Bind9 has a number of security enhancements over Bind8.
* Bind9 and it’s tools have full support of IPv6.
* It comes with the current stable version of FreeBSD (5.3 at the time of this writing).

## Enabling Named

You'll want to enable bind in your `/etc/rc.conf` so that the startup script will know you want it:

```conf
named_enable="YES"
named_chrootdir="/var/named"
named_chroot_autoupdate="YES"
```

## Chrooting Bind9

You may ask, “Why should I chroot Bind?”. First and foremost you need to assume that there is a possibility that someone will hack your services somehow. If you set up a chroot for Bind in the manner described below you’ll have a server running as nonroot in it’s own “sandbox” so if someone does break in, the worst they’ll be able to do is manipulate the files in the directory your bind server is confined to. For this reason you may wish to back up your zone files at some point.

If you're running the latest FreeBSD the below is really not needed. The startup script will do everything for you with the exception of making a named.conf and named.root. So what you'll want to do is make a working named.conf in `/var/named/etc/namdb/named.conf` and do:

```shell
cp /usr/src/etc/namedb/named.root /var/named/etc/namedb/named.root
```

From here we can activate the startup script and assuming your named.conf is good (which can be checked with named-checkconf) you can start named with

```shell
/etc/rc.d/named start
```

## Creating the chroot envirnonment (deprecated)

If you're letting the startup script create your chroot environment as above this section can be skipped. Continue on to Finishing Touches.

You’ll first need to create the directories for your chrooted environment by doing the following:

```shell
mkdir -p /var/named/dev /var/named/etc /usr/named/var/run 
```

Next you’ll need to place the appropriate files into the dirs you just created:

```shell
cp /etc/named/named.root /var/named/named.root
cp /etc/localtime /var/named/etc/localtime
```

If you have any existing conf or zone files you’ll want to move them into /var/named as well. Be sure to change the directory option in your existing named.conf (see below).

Since Bind will be in it’s own nearly autonomous environment you’ll need to make some device entries as well by doing the following:

```shell
mknod /var/named/dev/null c 2 2
mknod /var/named/dev/random c 2 3
```
And last, we’ll have to let Bind own all the files and directories we created

```shell
chown -R bind:bind /var/named/*
```

## Finishing Touches

Okay so the environment is created, now we just have to set up syslog and `/etc/rc.conf`.

```shell
/etc/rc.d/syslogd stop
/etc/rc.d/syslogd start
```

The syslogd script will detect that you are running named, and that it needs to add an additional socket. This will restart syslogd with an additional logging socket in /var/named/dev. Be sure that syslogd is running in your `/etc/rc.conf` if it’s not already:

```conf
syslogd_enable="YES"
```

## Setting up Bind9 for IPv4

I’ll show you how to set up a bind server for a NAT and Internet server. Of course if you don’t wish this to be visible on the Internet you can choose to have it blocked via firewall. Many of you will probably be doing bind9 setups for the standard v4 addresses. This may not be the most helpful tutorial out there, I’m only giving a background for Ipv6 setups if you know how to set up IPv4 bind feel free to skip this. Also I’ll assume you are using the chrooted environment as explained above so take that into consideration.
Setting up rndc

The rndc tool is useful for controlling named locally or remotely, some of its functions can give you useful stats as well. It’s worth setting up. So here’s how.

Run `rndc-confgen -a`. This will drop the file rndc.key into `/etc/namedb/`. You’ll notice the key looks something like this:

```conf
key "rndc-key" {
       algorithm hmac-md5;
       secret "5CKK3LlNDdkxshC5gmnzYQ==";
};
```

Place the key in your `/var/named/etc/namedb/named.conf`. It’ll work if you put the rndc specifications at the head of the file like this:

```conf
controls {
  inet 127.0.0.1 allow { localhost; } keys { rndc-key; };
};
key "rndc-key" {
  algorithm hmac-md5;
  secret "5CKK3LlNDdkxshC5gmnzYQ==";
};
```

This allows only the localhost to have access to controlling named with rndc, this however, can be modified.

Place the key in your `/etc/rndc.conf`. Here is a sample of this file with the above sample key.

```conf
options {
        default-server  localhost;
        default-key     "rndc-key";
};

server localhost {
        key     "rndc-key";
};

key "rndc-key" {
        algorithm hmac-md5;
  	secret "5CKK3LlNDdkxshC5gmnzYQ==";
};
```

Start named or `killall -HUP named` for the changes to take effect if you have named set up already. If not, then they will take effect when you do start named after it has been configured.

## Setting up the named.conf

First you’ll need a config file. I’ll attempt to run through this and explain by example how it was done on Section 6 Networks so that you may learn and adopt it to your own needs.
### Set up options on your `/etc`:

```conf
Options {
        directory "/";
        listen-on { 1.2.3.4; };
        recursion no; #Make it so people can only look up records on this host
        version "";
        pid-file "/var/run/named/pid";
        rrset-order {
           class IN type A name "www.example.com" order random;
        };
};
```

This well set up your main directory as the chrooted directory. Since named is chrooted, it’s root is /var/namedb/ if you set it up from the example.

### Explanation of options

* Don't set recursion no; if you're setting up dns for an intranet.
* version ""; will make it so hackers can't probe what version you're using, making you a less likely target.
* `rrset-order` is how you set up round robin dns. If you want round robin dns for all multiple A records in your zones you can simply do this:

```conf
rrset-order {
   order random;
};
```

Other choices besides random include cyclic, and fixed

### Set up your zone info:

```conf
zone "." {
       type hint;
       file "named.root";
};
zone "section6.net" {
       type master;
       file "zones/db.section6.net";
      notify yes;
      allow-transfer { 216.7.11.132; 64.71.191.27; 212.100.224.176; 66.37.215.46; };
};
zone "0.0.10.in-addr.arpa" {
       type master;
       file "zones/db.0.0.10.in-addr.arpa";
};
zone "89.67.45.123.in-addr.arpa" {
       type master;
       file "zones/db.13.180.230.12.in-addr.arpa";
};
// Provide a reverse mapping for the loopback address 127.0.0.1
zone "localhost" {
    type master;
    file "zones/db.localhost";
};
zone "0.0.127.in-addr.arpa" {
    type master;
    file "zones/db.0.0.127.in-addr.arpa";
    notify no;
};
```

Okay so that’s a lot. Basically we have the root hints file, a mandatory file for looking up DNS unknowns, and one forward zone: section6.net which show name to address mappings. We have 2 address to name mappings, one for our private net 10.0.0.0/24 and one for our outside IP, 123.45.67.89. Notice the format of these zones, it’ll become more clear later. We also have the mandatory zone files for localhost, our computer which are localhost and 127.0.0.1. You’ll notice that section6.net is the only one with `notify yes` and `allow transfer` options. notify yes says to notify the transfer hosts upon any changes. We have notify no set on a majority of the zones as they are for internal use only.allow transferspecfies the hosts allowed to transfer your information to theirs.

## The Zone Files

For each defined zone in your named.conf you’ll need a corrosponding zone file detailing the forward or reverse info for that zone. Below is a sample IPv4 forward zone file for section6.lan. The file name, according to named.conf is `db.section6.lan`.

```conf
$ORIGIN section6.net.
$TTL 1d
section6.net. IN SOA    syndie.section6.net. root.syndie.section6.net. (
                       1               ; Serial
                       10800           ; Refresh after 3 hours
                       3600            ; Retry after 1 hour
                       604800          ; Expire after 1 week
                       86400 )         ; Minimum TTL of 1 day

               IN NS   syndie.section6.net.
;
section6.net    IN MX 10 syndie.section6.net.
;
@               IN A    10.0.0.1
localhost       IN A    127.0.0.1
syndie          IN A    10.0.0.1
vpn             IN A    10.0.0.2
schism          IN A    10.0.0.5
test            IN A    10.0.0.20
ganymede        IN A    10.0.0.42
web             IN A    10.0.0.99
gabrielle       IN A    10.0.0.242
;
hades           IN CNAME        syndie
ns              IN CNAME        syndie
mail            IN CNAME        syndie
ftp             IN CNAME        ganymede
```

You’ll notice a few things here. The `$ORIGIN` bascially tells named which domain to tack onto the records so we don’t have to write the whole thing out each time. The . at the end of the domain is **important** if you forget it things will break. There are 2 strings after SOA (Start of Authority). One tells who the SOA is for the domain, the other names the contact (root.syndie.section6.lan = root@syndie.section6.lan). A records point to IP’s, CNAMEs are essentially aliases for A records. Part of good practice in to not have a CNAME pointing to another CNAME. The MX record tells mail exchangers to to send mail to for that domain. The number after it is the priority, 1 being highest.

Now for an example of a *reverse* zone file. This is for the zone `0.0.10.in-addr.arpa`.

```conf
$TTL 1d
0.0.10.in-addr.arpa. IN SOA syndie.section6.net. root.syndie.section6.net. (
                       1        ; Serial
                       10800    ; Refresh after 3 hours
                       3600     ; Retry after 1 hour
                       604800   ; Expire after 1 week
                       86400 )  ; Minimum TTL of 1 day

@               IN NS syndie.section6.net.

1               IN PTR syndie.section6.net.
2               IN PTR vpn.antithesist.net.
5               IN PTR schism.section6.net.
20              IN PTR test.section6.net.
42              IN PTR ganymede.section6.net.
99              IN PTR web.section6.net.
242             IN PTR gabrielle.section6.net.
```

Many of the declarations here are the same as was in the forward file. All the records here are PTR records. Since you have 0.0.10.in-addr.arpa defined (which translates to 10.0.0.x). You just need to put the “x” value for each of the addresses. Again, don't forget the . at the end of each full domain name or it won’t work.

## Setting up Bind9 for IPv6

Some additional notes about IPv6 DNS. There are 2 competing formats for A style records in IPv6: AAAA and A6. Since I have yet to see an A6 record in the wild I’ll refer you to the above mentioned document if you wish to set it up. I will, however, detail how to set up the 2 formats for reverse DNS (nibble and bitstream). IPv6 zones in the named.conf

I have set up my IPv6 forward records in the same zone file that the IPv4 records are located, this works but some may not want to do this. You can create a subdomain to differentiate you IPv4 and IPv6 records, but there is no harm in making both records in the same file, or even 2 identical names each pointing to one IPv6 and one IPv4 record. It has worked alright for me so far. Here is the IPv6 specific information as appeneded to my named.conf example above.

```conf
// IPv6 zone files
// ==========
//
// First, load the zone for the IPv6 loopback address.
//
//The new current way of reverse (Bitstream)
zone "\[x0000000000000000/64].ip6.arpa" {
       type master;
       file "zones/db.0000:0000:0000:0000.ip6.arpa";
       allow-transfer {none;};
};
//The old (depreciated) reverse (Nibble format)
zone "0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.int"  {
       type master;
       file "zones/db.0000:0000:0000:0000.ip6.int";
       allow-transfer {none;};
};
zone "\[x200104701f000222/64].ip6.arpa" {
       type master;
       file "zones/db.2001.470.1f00:222.ip6.arpa";
};
zone "2.2.2.0.0.0.f.1.0.7.4.0.1.0.0.2.ip6.int" {
	type master;
       file "zones/db.2001.470.1f00:222.ip6.int";
};
```

As you can see I have the first 4 groups of 16 bits (also known as a /64 since 16 * 4 = 64) defined in the zone files. The first two entries define my localhost, the second 2 define my public address range 2001.470.1f00.222/64. Most of this part of the file is fairly self explanatory.
