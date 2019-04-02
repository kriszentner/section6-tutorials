# Thwarting ssh hackers with swatch+pf
## Caveats

This tutorial is still here for educational purposes. However this method doesn't work so well. You may experience a number of swatch processes in limbo (but not zombied) eventually after using this method.

The best method so far is to do the following:

* Switch the port ssh is on.
* Use this great tool: denyhosts

## Introduction

You may notice these days that there's a lot of ssh hackers out there filling your auth.log up with a bunch of messages like:

```log
Dec 1 12:34:56 server sshd[67743]: Illegal user patrick from 123.123.123.12
```

This can get pretty annoying considering what they're doing is a blatent attack on your server and all you can do is watch what they did. You want to bump back, but how? The best way is to install swatch, a log watcher, and with a little configuration, this will give them one illegal user attempt and they'll be blocked indefinately. This tutorial will assume you are using pf for your filewall. Section6 has a decent tutorial on setting this up.

Keep in mind that if you or people on your server mistype their username a lot, you may want to either add some sort of exceptions list or not install this at all.

## Installing and configuring swatch.

These instructions will be mostly FreeBSD specific, but you can extrapolate how to do it on Linux as well. Install swatch from ports at /usr/ports/security/swatch.

You'll want to figure out who you'll be running swatch as, for reasons of convenience I'll be running it as root, but with some tinkering you can run this as another user or use sudo if it makes you feel better.

First you want to grab the IP address of the offender from the logfile and stick it in a buffer. You'll want to make your .swatchrc file in /root/.swatchrc with this in it:

```conf
#Look for bad ssh attempts, if found, block them!
watchfor /Illegal user/
        exec "/root/bin/addblock $10"
```
Now you'll want to make the addblock script we referenced above, we could put all our commands in multiple exec lines. However, putting quotes inside quotes in exec doesn't work so well, so we'll use a separate shell script. Here is what `/root/bin/addblock` should contain:

```shell
#!/bin/sh
pfctl -t hackers -T add $1
echo $1 >> /root/pf/hackers
logger swatch: $1 caught with bad login. Added to hackers pf table
```

The first line will add the address to the hackers table in pf. In the second line there's a file we are appending the addresses to this is so when we restart pf, it'll remember the old addresses, feel free to put this file anywhere you feel is appropriate. The last line sends a message to syslog about the address addition. Which will end up in /var/log/messages

Now to make sure this starts at boot time, you want to add the following to your `/etc/rc.conf`:

```conf
swatch_enable="YES"
swatch_rules="1"
swatch_1_flags="swatch --config-file=/root/.swatchrc --tail-file=/var/log/auth.l
og --awk-field-syntax --daemon --pid-file=/var/run/swatch.pid"
swatch_1_user="root"
swatch_1_pid="/var/run/swatch.pid"
```

## Configuring pf

You'll want to put the following lines in the add/block rules section of you pf file:

```conf
table <hackers> persist file "/root/pf/hackers"
block in quick on $extif proto { tcp, udp, ipv6, icmp, esp, ipencap } from <hackers> to any
```

## Finishing up

If you've done everything above you can start swatch and enable your new pf rules with:

```shell
pfctl -f /etc/pf.conf
/usr/local/etc/rc.d/swatch.sh start
```

## Caveats

This tutorial is still here for educational purposes. However this method doesn't work so well. You may experience a number of swatch processes in limbo (but not zombied) eventually after using this method. The best method so far is to just switch the port ssh is on.