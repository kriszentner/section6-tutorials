# Configuring Samba3 to be a Windows Domain Member

This simple guide is a mostly accurate way to set up a Samba machine as a domain member in a MicrosoftWindows 2000 or Windows 2003 ActiveDirectory domain. For a REALLY short version, tested with Win2k3, see the Quick 'n' Dirty instructions at the bottom of the page. Samba as an Active Directory Domain Member

The following setup is used:

```conf
192.168.0.1 	test.test.org 	
```

the AD server, hereafter known as "the server"

```conf
192.168.0.200 	mail.test.org 	
```

## Samba3 "Client" Machine

The Samba system is based on a Debian Sarge installtion of Samba3

The following steps are needed to get the system functioning:

1. configure name resolution using either dns or a hosts file 
2. configure samba and winbindd 
3. configure kerberos 
4. testing Samba and winbindd

## Configuring name resolution

Active Directory relies HEAVILY on DNS to resolve not only host names but services they provide as well.

The first step is to configure name resolution for our systems. The kerberos authentication system, which we will configure later on, requires us to be able to do a reverse lookup on an IP address to get a fully qualified domain name (FQDN).

There are two ways to do this. The cheap and nasty method is to use a hosts file on both systems. Hosts based authentication, which is discussed here, is ugly and hacky, and should be avoided at all costs. If you want to do it anyway, you need entries similar to the following.

Samba machine:

```conf
#/etc/hosts
127.0.0.1       localhost localhost.localdomain
192.168.0.1     test.test.org test
192.168.0.200   mail.test.org mail
```

Windows Active Directory server

```conf
#%Systemroot%\System32\drivers\etc\hosts
127.0.0.1       localhost localhost.localdomain
192.168.0.1     test test1.test.org
192.168.0.200   mail mail.test.org
```

The correct method is to setup DNS on the server which can be done through the DNS console in the Administrative Tools section of Windows 2000/2003 Server. (You shouldn't be runing an Active Directory without a well set up DNS; if you don't know how to do it, go away and learn RIGHT NOW). We won't go into the details of setting this up here, but we will specify the Linux side of that here.

A good way to set this up is to have a Linux-based BIND server doing name resolution for your site 'mydomain.tld', just as you normally would; then configure BIND to delegate the special Active Directory sub-domains DomainDnsZones.mydomain.tld and so on to the Windows Server 2003 box. Then, configure Windows Server 2003 DNS to be a caching proxy using the Linux BIND box as its parent, except for the AD sub-domains for which it should be authoritative. All machines can then use the Linux box for DNS. This way, name resolution of normal names stays on good ole reliable Linux where it belongs, the Windows Active Directory crud goes on Windows where it belongs, and everything's happy. If the Windows Server is down, the AD stuff stops working (there's no avoiding that if the PDC is offline); however normal (non-AD) name resolution is unaffected. Thanks to Matthew Sanderson for the tip.

```conf
#/etc/resolv.conf
search      test.org
domain      test.org
nameserver  192.168.0.1
```

## Configuring Samba3 and Winbindd

This part is the easy one, we just create ourselves a default Samba configuration with at least the following entries (Note this is a completely empty and default configuration file, and you may wish to add more. A file share would be handy to add).

/etc/samba/smb.conf
```conf
 [global]
   # general options
   workgroup = THINCLIENT
   netbios name = MAIL
   # winbindd configuration
   # default winbind separator is \, which is good if you
   # use mod_ntlm since that is the character it uses.
   # users only need to know the one syntax
   winbind separator = +
   # idmap uid and idmap gid are aliases for
   # winbind uid and winbid gid, respectively
   idmap uid = 10000-20000
   idmap gid = 10000-20000
   winbind enum users = yes
   winbind enum groups = yes
   template homedir = /home/%D/%U
   template shell = /bin/bash
   # Active directory joining
   # "ads server" is only necessary if your kdc
   # can't be located using /etc/krb5.conf 
   ads server = test.test.org
   security = ads
   # encrypt passwords = yes is now default in Samba3 
   encrypt passwords = yes
   realm = test.org
   # this handles the "ads server = " directive as well 
   password server = test.test.org
```

The important things to pay attention to here are the name of our samba machine (netbios name), the workgroup, and the ActiveDirectory information.
Configure Kerberos5

If your Kerberos setup is good, run `net ads join -U Administrator%password` and it will perform all the ktpass and ktutil stuff on the fly as mentioned in the SAMBA howto . Then you can skip to the winbind section below. If you don't specify %password, it will prompt you on the command line (for the security minded).

Configuring a Kerberos setup is much easier in the long run then generating the key and importing it.
Manual approach

We need to generate a key for our samba machine on the Windows server, and securely import this into our samba machine. To create the keyfile we run the following on the Windows server:

```bash
ktpass -princ host/mail.test.org@.TEST.ORG \
            -mapuser MAIL -pass MAIL1234PASSWORD -out mail.keytab
```

This, and many other tools for managing Kerberos in Windows 2000, are located in the support tools which are directly downloadable from Microsoft.

We then transfer the mail.keytab securely to our samba machine by using something similar to SSH or another secure means. And then on the samba machine we will import the keyfile we just generated by using the ktutil program, which is part of the kerberos distribution. The unix commands for ktutil are as follows:

```
% __ktutil__
ktutil: __rkt mail.keytab__
ktutil: __list__
ktutil: __wkt /etc/krb5.keytab__
ktutil: __q__
```

Consult the Configuring Kerberos5 to use Active Directory for more information.
(Re)starting Samba and Winbindd

First we test our samba configuration and our winbind settings, before we modify our samba startup script.

```
root@host# /etc/rc.d/init.d/samba restart
root@host# /usr/sbin/winbindd
```

For some of our paranoid friends, we can check to see if our winbindd is actually running using

```
root@host# ps fax | grep winbindd
```

Now for a real test, and see if we can get some information off our Active Directory PDC.

```
root@host# /usr/bin/wbinfo -u
```

And we should get a list of users in the format TEST+\<username\>

```
TEST+Administrator
TEST+Guest
..
```

And we can do the same for our list of groups.

```
root@host# /usr/bin/wbinfo -g
TEST+Domain Admins
TEST+Domain Users
TEST+Schema Admins
..
```

We can now use the getent utility to get a unified list of both the local and PDC users and groups. These utilities will generate a list of data similar in format to the `/etc/passwd` and `/etc/group` files respectively.

add following entries in `nssswitch.conf`:

```
passwd:        files winbind
group:         files winbind
```

if you are compiling samba from source then you need to copy following files manually

```
root@host# cp /usr/src/samba-3.0.1/source/nsswitch/pam_winbind.so  /lib/security/
root@host# cp /usr/src/samba-3.0.1/source/nsswitch/libnss_winbind.so /lib/
root@host# cp /usr/src/samba-3.0.1/source/bin/pam_smbpass.so  /lib/security/
```

then run following command to get unified entries

```
root@host# /usr/bin/getent passwd
root@host# /usr/bin/getent group
```

It is now a good idea to test to ensure your Active Directory usernames are valid on the system. Try the following:

```
root@host# chown "TEST+username" filename
```

(where TEST is the active directory short name)

If `wbinfo -u` and `getent passwd` work fine but your chown says this is an unknown user, you probably have NSCD running. You should disable NSCD and restart winbind. See the WINBIND documentation for more information.

After this we can fix up our init.d startup scripts to automate the startup of winbindd and not start NSCD.
Configure PAM and Winbind

Before we do anything at all here, we need to make a backup of our `/etc/pam.d/*` files. And have a linux bootdisk available if possible. If anything goes wrong here, you may not be able to login to your system properly. (So don't reboot or logoff to test, but use a text console)

To have our ActiveDirectory users be able to login to our we have to modify our /etc/pam.d/login. We don't need to modify our /etc/pam.d/samba settings as it is already configured for winbind.

```
#/etc/pam.d/login
#%PAM-1.0
auth        required     pam_securetty.so
auth        sufficient   pam_winbind.so
auth        sufficient   pam_unix.so use_first_pass
auth        required     pam_stack.so service=system-auth
auth        required     pam_nologin.so
account     sufficient   pam_winbind.so
account     required     pam_stack.so service=system-auth
password    required     pam_stack.so service=system-auth
session     required     pam_stack.so service=system-auth
session     optional     pam_console.so
```

After we save this file, we should now be able to login to our linux machine with the username TEST+Administrator, and get ourself a login prompt. Now the system may complain if you do not have the specified home directory created (in this case /home/TEST/Administrator)

## SSH Support

Do the same additions that you made to /etc/pam.d/login to /etc/pam.d/sshd to support logins via SSH.

Congatulations, it should work. If you want to configure further items such as mail and other things you may need to modify the apropriate PAM modules, which will be included in a later article.

## References

* Using Kerberos Clients section of the [Microsoft : Step-by-Step Guide to Kerberos 5](http://www.microsoft.com/windows2000/techinfo/planning/security/kerbsteps.asp)

* [Authentication with ADS](http://mailman.mit.edu/pipermail/kerberos/2002-June/001189.html)

* The winbindd and Active Directory Domain Member sections of the [Samba v3 Documentation](http://au1.samba.org/samba/devel/docs/html/Samba-HOWTO-Collection.html)

## Quick and Dirty setup for Samba 3 and Windows 2003

These are the absolute bare minimum steps to get your Samba server integrated as a member server in an AD controlled domain with Win2k3 as the DC.

1. ENSURE your samba box has an A record and associated PTR in DNS.

2. On your DC, disable signing: Run Domain Controller Policy tool and edit Account Policies -> Security Options -> Microsoft network client: Digitally sign communications (always) Set this to Disabled. Do the same in the Domain Policy tool. Note, you will need to reboot the server for this step, though it won't tell you to. Disable on your samba server as well with the following in smb.conf

```conf
client signing = no
client use spnego = no
```

3. On your samba server, install kerberos5, and edit /etc/krb5.conf. It should contain:

```toml
[libdefaults]
       default_realm = YOUR.ADS.DOMAIN
       dns_lookup_kdc = false
       dns_lookup_realm = false

[domain_realm]
       .your.domain.name=YOUR.ADS.DOMAIN
       your.domain.name=YOUR.ADS.DOMAIN

[realms]
YOUR.ADS.DOMAIN = {
       default_domain = your.domain.name
       kdc = IP.OF.THE.DC
}
```

4. Ensure smb.conf contains

```conf
realm = YOUR.ADS.DOMAIN
workgroup = YOUR
security = ADS
```

5. Get a ticket using kerberos: kinit administrator (enter the administrator password when prompted). The klist command should then list a ticket.

6. Join the domain using 'net ads join'. This should use the credentials in your kerberos ticket.

7. Set up winbind - ensure the following is in smb.conf

```conf
winbind uid = 10000-20000
winbind gid = 10000-20000
winbind enum groups = yes
winbind enum users = yes
```

8. store your winbind credentials with `wbinfo --set-auth-user=DOMAIN\\administrator%password`

9. modify /etc/pam.d/samba (on woody) or the appropriate pam file to add "sufficient" for auth and account using pam_winbind.so. These need to go BEFORE the pam_unix.so calls for samba. My /etc/pam.d/samba is as follows:

```conf
auth            sufficient      pam_winbind.so
auth            required        pam_unix.so nullok
account         sufficient      pam_winbind.so
account         required        pam_unix.so
session         required        pam_unix.so
password        required        pam_unix.so
```

10. Modify /etc/nsswitch.conf with the following:

```
passwd:         winbind compat
group:          winbind compat
shadow:         winbind compat
```

11. Restart samba and winbind.

12. All should work. :) Browse your server and see...
Footnotes

13. %Systemroot% is a variable set by Windows NT and onward to mean "the location where Windows is installed", ie c:\winnt, c:\windows, etc.
