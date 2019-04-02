# Creating a FreeBSD Jail

## Preface

This tutorial is largely academic. At the time of this writing, tools to administrate and create jails were nonexistant. Now, there are a host of them including ezjail, and jailuser. My thanks to the creators of these tools to simplify the once time intensive process. If you wish to still go the hard route. Keep reading.

## Introduction

Jails are a great way to secure your processes to a virtual system. Though they have more overhead than chroot, (which basically just restricts the root of a process) a jail uses a virtual machine to house your process or processes. This means that far more restrictions can be placed on the jail, and there's no "breaking out" as can be done with chroot (see links in references).

A few notes first of all. It's very true what they say in the man page about it being easier to make a fat jail, and scale down to a thin one than vice versa. A few weeks of research (and many make worlds) have helped me discover that.

If you're using a version previous to FreeBSD 7.2, your only option will be ipv4 support. IPv6 support exists in versions 7.2 and beyond.

## Jail Creation Techniques

From what I've seen there are three primary ways of creating jails.

### MiniBSD

I've heard reports of people using [https://neon1.net/misc/minibsd.html MiniBSD] to do this, but I haven't had much luck with it, and I have yet to see a howto explaining how they made it work, it's a great idea of making an initial thin jail but there's a million things that can go wrong since it's very minimal and the service(s) you are trying to run may have dependancy issues.

### Using /stand/sysinstall

Other howtos tell to use /stand/sysinstall to go out to the net, download the system binaries, and install specific distributions from the installer. I've had little luck with this as well since you run into the problem of not having an interface set up for the installer to use. There's probably a way to do this but none of the howtos I tried did a very good job of explaining how.

### Using make world

This is the way I'll use here in this tutorial and the way explained in the manpage. You can customize the make file to scale down your distribution and set some optomization flags for your system. The primary drawback is the time it takes to build the world which can be hours depending on your system.
Getting services to not listen to *

First off, we should make sure we get the system so that we have nothing listening on *, to check what what we need to modify issue this command

```shell
sockstat|grep "\*:[0-9]"
```

This should give you a synopsys of all the processes and ports you need to trim down. Here are some hints with your ipv4 addr being 10.0.0.1 and your ipv6 addr being 2002::7ea9

sshd:

* edit /etc/ssh/sshd_config
* change ListenAddress derivative

```conf
ListenAddress 10.0.0.1
ListenAddress 2002::7ea9
```

httpd
* edit /usr/local/etc/apache/httpd.conf (and ssl.conf for https)
* change Listen derivative

```conf
Listen 10.0.0.1:80
Listen [2002::7ea9]:80
```

slapd

* edit /etc/rc.conf
* change slapd_flags

```conf
slapd_flags='-h "ldapi://%2fvar%2frun%2fopenldap%2fldapi/ ldap://10.0.0.1/ ldap://127.0.0.1/ ldap://[2002::7ea9]/"'
```

inetd

* edit /etc/rc.conf
* change inetd_flags=

```conf
inetd_flags="-wW -a yourhost.example.com"
```

mysql

* edit /etc/my.cnf

```conf
bind-address=10.0.0.1
```

```shell
postfix edit /usr/local/etc/postfix/main.cf
```

* change inet_interfaces

```conf
inet_interfaces = [2002::7ea9], 10.0.0.242
```

samba (this will get you most of the way there)

* edit /usr/local/etc/smb.conf
* change the following:

```conf
interfaces = 10.0.0.242/24 127.0.0.1
socket address = 10.0.0.242
bind interfaces only = yes
```

note: if you don't need wins lookups and netbios name translation

```log
you can safely disable nmbd. There doesn't seem to be a way for nmb to not listen to *:138 anyhow.
```

To disable nmb go to /etc/rc.conf and replace samba_enable="YES" with smbd_enable="YES"

openntpd (xntpd listens on all and cannot be changed)

*edit /usr/local/etc/ntpd.conf
listen on 10.0.0.1
listen on 2002::7ea9

syslogd

* edit /etc/rc.conf
```conf
syslogd_flags="-s -s" #For no listening
syslogd_flags="-a 10.0.0.1"
```

bind

* edit your named.conf (may be in /var/named/etc/named.conf)
* In the options section:

```conf
listen-on { 10.0.0.242; };
listen-on-v6 port 53 { 2002:d8fe:10f1:6:202:b3ff:fea9:7ea9; };
query-source address 10.0.0.242 port *;
query-source-v6 address 2002:d8fe:10f1:6:202:b3ff:fea9:7ea9 port *;
```

Unrealircd

    In the listen section:

```conf
listen[::ffff:10.0.0.1]:6667
listen[2002::7ea9]:6667
```

    In the "set { dns {" section

```conf
bind-ip 10.0.0.242;
```

Building your jail for the first time
Creating an appropriate make.conf

You'll need to run make world (or make installworld) to create your jail. If you don't want to install the whole kitchen sink you can use the make.conf below. You can put it in your jail for future use and it'll be used by future port builds inside your jail. One thing I've noticed is that make installworld doesn't seem to respect and MAKE_CONF or __MAKE_CONF variables passed to it so we'll just put it in /etc/make.conf for now.

Lets first back our current make.conf up:

```shell
cp /etc/make.conf /etc/make.conf.bak
```

And new one in there. Keep in mind, depending on what you want to use this jail for you may want to modify this make.conf. For me this has worked on building a variety of services from ports (inside the jail). I like to name the below file make.conf.jail and copy it to make.conf, then copy make.conf.bak back to make.conf when I'm done building the jail.

```conf
NO_ACPI=       true    # do not build acpiconf(8) and related programs
NO_BOOT=       true    # do not build boot blocks and loader
NO_BLUETOOTH=  true    # do not build Bluetooth related stuff
NO_FORTRAN=    true    # do not build g77 and related libraries
NO_GDB=        true    # do not build GDB
NO_GPIB=       true    # do not build GPIB support
NO_I4B=        true    # do not build isdn4bsd package
NO_IPFILTER=   true    # do not build IP Filter package
NO_PF=         true    # do not build PF firewall package
NO_AUTHPF=     true    # do not build and install authpf (setuid/gid)
NO_KERBEROS=   true    # do not build and install Kerberos 5 (KTH Heimdal)
NO_LPR=        true    # do not build lpr and related programs
NO_MAILWRAPPER=true    # do not build the mailwrapper(8) MTA selector
NO_MODULES=    true    # do not build modules with the kernel
NO_NETCAT=     true    # do not build netcat
NO_NIS=        true    # do not build NIS support and related programs
NO_SENDMAIL=   true    # do not build sendmail and related programs
NO_SHAREDOCS=  true    # do not build the 4.4BSD legacy docs
NO_USB=        true    # do not build usbd(8) and related programs
NO_VINUM=      true    # do not build Vinum utilities
NO_ATM=        true    # do not build ATM related programs and libraries
NO_CRYPT=      true    # do not build any crypto code
NO_GAMES=      true    # do not build games (games/ subdir)
NO_INFO=       true    # do not make or install info files
NO_MAN=        true    # do not build manual pages
NO_PROFILE=    true    # Avoid compiling profiled libraries

# BIND OPTIONS
NO_BIND=               true    # Do not build any part of BIND
NO_BIND_DNSSEC=        true    # Do not build dnssec-keygen, dnssec-signzone
NO_BIND_ETC=           true    # Do not install files to /etc/namedb
NO_BIND_LIBS_LWRES=    true    # Do not install the lwres library
NO_BIND_MTREE=         true    # Do not run mtree to create chroot directories
NO_BIND_NAMED=         true    # Do not build named, rndc, lwresd, etc.
```

## Building the Jail

Now for actually building your jail...

I'm defining JAILDIR here because I'm going to use it in a shellscript style example throughout the rest of this howto.

```shell
# Let's first make some directories
JAILDIR=/home/jail
mkdir -p $JAILDIR/dev
mkdir -p $JAILDIR/etc
mkdir -p $JAILDIR/usr/tmp
chmod 777 $JAILDIR/usr/tmp

cd /usr/src/

# You can replace the below with make installworld if you've built your
# world previously
make buildworld
make installworld DESTDIR=$JAILDIR
cd /usr/src/etc
cp /etc/resolv.conf $JAILDIR

make distribution DESTDIR=$JAILDIR NO_OPENSSH=YES NO_OPENSSL=YES
cd $JAILDIR

# At this point we'll mount devfs, and then hide the unneeded devs
mount_devfs devfs $JAILDIR/dev
devfs -m $JAILDIR/dev rule -s 4 applyset

# Create a null kernel
ln -s dev/null kernel

# Quell warnings about fstab
touch $JAILDIR/etc/fstab

# Use our existing resolv.conf
cp /etc/resolv.conf $JAILDIR/etc/resolv.conf

# Copy our settings for ssl
mkdir -p $JAILDIR/etc/ssl
mkdir -p $JAILDIR/usr/local/openssl
cp /etc/ssl/openssl.cnf $JAILDIR/etc/ssl
cd $JAILDIR/usr/local/openssl/
ln -s ../../../etc/ssl/openssl.cnf openssl.cnf
```

Make a decent rc.conf:

```conf
hostname="jail.example.com"    # Set this!
ifconfig_em0="inet 10.0.0.20 netmask 255.255.255.255"
defaultrouter="10.0.0.1"        # Set to default gateway (or NO).
clear_tmp_enable="YES"  # Clear /tmp at startup.
# Once you set your jail up you may want to consider adding a good securelevel:
# Same as sysctl -w kern.securelevel=3
kern_securelevel_enable="YES"    # kernel security level (see init(8)),
kern_securelevel="3"
```

You'll also want to make an alias on your interface for the ip above so we'll do something like:

```shell
ifconfig em0 10.0.0.20 netmask 255.255.255.255 alias
```

Now you'll want to have devfs inside your jail, so to get it working for the first time do this:

```shell
mount_devfs devfs $JAILDIR/devfs
```

And finally, copy your original make.conf back.

```shell
cp /etc/make.conf.bak /etc/make.conf
```

## Starting the jail for the first time

OPTIONAL (but probably necessary): You'll want to mount /usr/ports and /usr/src so you can install ports inside your jail, unless you have another way you want to do this (such as downloading packages).

```shell
mount_nullfs /usr/ports $JAILDIR
mount_nullfs /usr/src $JAILDIR
```

Now we can start our jail

```shell
jail $JAILDIR jail.example.com 10.0.0.20 /bin/sh
```

Once inside the jail you'll want to start services:

```shell
/bin/sh /etc/rc
```

While you're here you'll want to edit your password file since if someone breaks into your jail, and starts cracking it you won't want them to have the same passwords as your root system has. Also remove all users you don't need in the jail:

```shell
vipw
passwd root
```

From here, assuming all went well you can do something like:

```shell
cd /usr/ports/security/openssh
make install clean
```
And build your port(s) inside your jail. Once you're finished be sure to unmount the directories so a compromised jail can't build more ports.

Note that if you try to start your jail with just:

```shell
jail $JAILDIR jail.example.com 10.0.0.20 /bin/sh /etc/rc
```

but you have no services/daemons/programs set to run, the jail will simply start and then exit since there's nothing running inside.

## Getting it to start automatically

You'll now need to put your settings in /etc/rc.conf First put the alias you jail has in there:

```conf
ifconfig_em0_alias0="inet 10.0.0.20 netmask 0xffffffff"
```

## Editing the rc.conf

For those of you that are looking to make your own rc script, I don't recommend it. I've found issues getting devfs rules to be applied with the a script, and really this way is much easier. It's also the standard way and you can attach to jails later on quite easily without using screen (read below).

Here's the standard `rc.conf` way of getting your jail to run at startup:

```conf
jail_enable="YES"        # Set to NO to disable starting of any jails
jail_list="cell"            # Space separated list of names of jails
jail_set_hostname_allow="NO" # Allow root user in a jail to change its hostname
jail_socket_unixiproute_only="YES" # Route only TCP/IP within a jail

jail_cell_rootdir="/usr/home/prison/cell"
jail_cell_hostname="cell.example.com"
jail_cell_ip="10.0.0.20"
jail_cell_exec_start="/bin/sh /etc/rc"
jail_cell_devfs_enable="YES"
jail_cell_devfs_ruleset="devfsrules_jail"
```

## Jail maintenance

Of course from time to time you may have to upgrade ports in your jail, or the world in the jail itself. This isn't a big deal either. Instead of using jail (which makes its own IP address and everything) we can use chroot instead which is similar since all we're using is a simple shell and then we'll be done with it.

First mount the dirs so they're accessable in the chroot:

```shell
mount_nullfs /usr/ports $JAILDIR
mount_nullfs /usr/src $JAILDIR
```

Connect to your jail: find the jail id of the jail you are running with jls:

```shell
#jls
   JID  IP Address      Hostname                      Path
    1  10.0.0.20       cell.example.com              /usr/home/prison/cell
```

Now connect to it using the JID:

```shell
jexec 1 /bin/sh
```

To upgrade your world:

```shell
cd /usr/src
make buildworld
make installworld
```

NOTE: If you've just done make buildworld previously you can do make installworld and install all the newly compiled binaries again.

To build a port:

```shell
cd /usr/ports/sysutils/example
make install clean
```

NOTE: You may also want to install portupgrade to make port management easier.

When you're done just exit:

```shell
exit
```

## Integrating Portaudit

You'll notice that portaudit security check only checks the root server, but none of the jails. There are many ways around this, but here's one:

Create a shell script in a place you keep custom shell scripts. We'll use `/root/bin/metaportaudit.sh`

```shell
#!/bin/sh

JAILDIR=/usr/home/prison/
JAILS="irc www mysql"
TMPDIR="/tmp"

# First lets audit the root server
/usr/local/sbin/portaudit -a

# Now Lets create temp files of ports in the jails,
# audit the root server all jails
# and delete the temp files
cd $TMPDIR
for jail in $JAILS; do
  echo ""
  echo "Checking for packages with security vulnerabilities in jail \"$jail\":"
  echo ""
  ls -1 $JAILDIR/$jail/var/db/pkg > $TMPDIR/$jail.paf
  /usr/local/sbin/portaudit -f $TMPDIR/$jail.paf
  rm $TMPDIR/$jail.paf
done
```

Now lets edit `/usr/local/etc/periodic/security` on about line 55
you'll want to change:

```shell
echo
echo /usr/local/sbin/portaudit -a |
         su -fm "${daily_status_security_portaudit_user:-nobody}" || rc=$?
```

to

```shell
echo
echo /root/bin/metaportaudit.sh -a |
         su -fm "${daily_status_security_portaudit_user:-nobody}" || rc=$?
```

## Jails in Linux

Now you may think "well I have to use Linux, because xapplication only works on Linux!" Well there's hope. You can mess around with the bsdjail patch, or you can install vserver (which has packages in Debian). There's a great tutorial on vserver in Debian here:  
[Running_Vservers_on_Debian](http://www.section6.net/wiki/index.php/Running_Vservers_on_Debian)

## References

* [Creating a Jail Server](http://chxo.com/gww/asparagus/notes/Creating_A_Jail_Server.html)
* [Jail Tools Cookbook](http://www.the-labs.com/FreeBSD/JailTools/cookbook.html)