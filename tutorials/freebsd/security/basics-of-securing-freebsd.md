# Basics Of Securing FreeBSD

As the Internet becomes less and less of a friendly place, you don't want to be connected without being protected by some sort of firewall. Fortunately, FreeBSD supports two firewalls:ipfw and ipfilter. Here is some information to get you started:

* man ipfw
* FreeBSD Handbook: Chapter 10 -- Security -- Firewalls
* man ipf
* IPFilter and PF resources
* For information on setting up IPfilter, refer to the Section6 Firewall Tutorial.

Good security is always "defense in layers," meaning that if one mechanism fails, there should be a backup mechanism. Even if your system is protected by a firewall, you should also disable all services except for those you absolutely need. On a desktop system, you need very few services.

To see which services are using ports on your system, use the following command:

```conf
sockstat -4
```

Your output will vary, depending upon what settings you selected during the final installation phase of FreeBSD and what ports and packages you have built since then.

## Securing X11

It is very common to see X11 services port 6000 in the output. Unfortunately, there have been many X11 exploits over the years. Fortunately, you don't need to leave port 6000 open in order to use X11 on your system. Don't worry, you'll still have a Window Manager available if you close this port!

There are several ways to close this port; one of the easiest is edit `/usr/X11R6/bin/startx`, find the serverargs line and make the following change:

```conf
serverargs="-nolisten tcp"
```

Once you've saved your changes, start X as a regular user and rerun sockstat. Port 6000 should now be no longer open.

## Securing Sendmail

If you'd like to read up on some of the security issues of leaving port 6000 open, refer to the [Section6 X11 Security tutorial](basics-of-securing-x11.md).

You now have one less service in your sockstat output. You probably still have two ports that deal with email: ports 25 (smtp) and 587 (submission). You don't need port 587 to send/receive email; to close it you can edit /etc/mail/sendmail.cf. Find this line:

```conf
O DaemonPortOptions=Port=587, Name=MSA, M=E
```

Comment it out, and then restart the service:

```conf
killall -HUP sendmail
```

The changes made to `/etc/mail/sendmail.cf` should now take effect. Repeat sockstat and port 587 should no longer show as open.

By default, FreeBSD runs with Sendmail enabled. If you do not wish to use the Sendmail as your MTA, simply make following change to `/etc/rc.conf`

```conf
sendmail_enable="NONE"
```

Afterwards, simply `kill -9 sendmail` and the service will no longer start when the system boots

## Securing Services in `/etc/rc.conf`

Another common service on FreeBSD servers to look for is Portmap. Portmap, in conjuction with NFS is used to shared files between Unix computers. If you do not have a need for this service, then it is wise to disable it. When running sockstat, look for port 111 in the output and disable it by adding the following lines to `/etc/rc.conf`

```conf
nfs_server_enable="NO"
nfs_client_enable="NO"
portmap_enable="NO"
```

Syslog (port 514) will probably also show in your output. You most likely do not want to disable syslog completely, because you will not receive logging messages. However, you do not need to have this port open to do so. In your /etc/rc.conf file, make sure syslog is enabled and add a second line with some options:

```conf
syslogd_enable="YES"
syslogd_flags="-ss"
```

The `-ss` flag will disable logging from remote hosts and close that port, but still allow your localhost to keep its logging capabilities.

If you find any other questionable servcies in your sockstat output, man rc.conf to see if there is an option to disable it. If there isn't, it was most likely started with a startup script that was installed with a package or a port. If this is the case, then check the `/usr/local/etc/rc.d` directory

This will allow you to see which startup scripts have been added to your system. Most packages/ports will install a sample script with a "sample" extension. As long as it ends in "sample," that script will not run at startup. Other packages/ports install a working script that is read at bootup. These startup scripts usually end with an "sh" extension. The easiest way to disable the script is to rename it with a "sample" extension, and then kill the daemon so that its port number no longer shows up in the output of sockstat.

You might also want to consider adding the following options to `/etc/rc.conf`:

```conf
tcp_drop_synfin="YES"
```

This option prevents something known as OS fingerprinting, which is a scan technique used to determine the type of operating system running on a host. If you decide to enable this option, you will also have to rebuild your kernel with the following option included in your kernel configuration file:

```conf
options TCP_DROP_SYNFIN
```

Two related options are:

```conf
icmp_drop_redirect="YES"	
icmp_log_redirect="YES"
```

ICMP redirects can be used to launch a DOS attack, as explained in the ICMP redirect portion of this ARP and ICMP redirection games article.

Be very careful if you decide to include the icmp_log_redirect option, as it will log every ICMP redirect, which has the potential of filling up your logging directory if you ever are the victim of this type of attack.

Another useful option in the /etc/rc.conf is

```conf
accounting_enable="YES"
```

This will enable system accounting. If you're new to system accounting, read the man pages for sa and lastcomm to decide whether this option would be useful to you or not.

Finally, this is a good option to include in /etc/rc.conf:

```conf
clear_tmp_enable="YES" 
```

This will clear the /tmp directory at startup, which is always a good thing.

## Converting passwd hashes to Blowfish

Changing the default algorithm used when encrypting a user's password can be very useful in securing accounts against local exploits. One renowned cipher that provides quick and secure encryption is called Blowfish. To implement Blowfish hashes, edit /etc/login.conf and change the following line:

```conf
:passwd_format=blf:\ 
```

Save the change, then rebuild the login database with the following command:

```conf
cap_mkdb /etc/login.conf 
```

You'll then either have to change all of your users' passwords yourself, or instruct your users to change their passwords themselves so they will get a new Blowfish hash.

Once you're finished, double-check that it worked by checking the contents of the /etc/master.passwd file. All of the passwords for your users should begin with $2.

Finally, configure the adduser utility to use Blowfish whenever you create a new user by editing the /etc/auth.conf file. Make the following change:

```conf
crypt_default=blf 
```

## Changing the banner info

You've probably noticed when you log in to your FreeBSD system that your login prompt reminds you that you are running FreeBSD. And that after you log in, you receive the FreeBSD copyright information, which is followed by the version of FreeBSD and the name of your kernel, and finally, a useful (but rather boring) motd which again reminds you that you are running FreeBSD. You probably already know what version of FreeBSD you are running and might not want to share that information with the rest of the world. And the motd is a good place to remind the rest of the world that they shouldn't be messing with your system anyways.

You can edit the `/etc/motd` file to say whatever you wish to be displayed when one logs in to your system.

The next step is to remove the copyright info. Create an empty copyright file by running the following command:

```conf
touch /etc/COPYRIGHT 
```

Then to change the text that appears at the login prompt, edit the `/etc/gettytab`. Find the line in the `default:\` section that starts with the following:

```conf
:cb:ce:ck:lc 
```

Carefully, change the text between \r\n\ \r\n\r\nr\n: to whatever text you wish to appear. Double-check that you have the right amount of \rs and \ns and save your change.

You can test your changes by logging out and checking the login prompt.

Finally, even though you've edited your motd to remove your version and kernel information, by default FreeBSD will still re-add it to /etc/motd every time you log in. To prevent this behavior, add the following line to /etc/rc.conf:

```conf
update_motd="NO" 
```

This change requires a reboot, so make sure you've first tested your previous changes and have saved all of your work on any other terminals.

There are a few edits that will also restrict logins to your system in the first place. Since these changes modify the behavior of the login program, you'll want to carefully test your changes. Keep one terminal open and go to another terminal to log out and ensure that you can still log in. If for some reason you're unable to log in (this shouldn't happen, but you can't be too careful), you can return to the other terminal and look for errors in the file you just edited.

No one (including you) should log in to your system using the root account for general use. To prevent this from happening, edit /etc/ttys. Once you get past a page's worth of comments, you'll notice a section that goes from ttyv0 to ttyv8. Change the word secure on each of those lines to insecure. This is a file you don't want a typo in, so double-check your changes carefully. Test your change by trying to log in as root on one of your alternate terminals. You should receive a "Login incorrect" message.

## Securing Login Locations

The last edit covered here allows you to restrict who can log in to your system and from where. This is done by editing /etc/login.access.

If you want to prevent all remote logins (meaning you can only log in if you are physically sitting at your system), remove the # from this line: #-:wheel:ALL EXCEPT LOCAL .win.tue.nl and remove the .win.tue.nl so that the line now looks like this:

```conf
-:wheel:ALL EXCEPT LOCAL 
```

If you plan on accessing your system remotely, replace .win.tue.nl with the IP address(es) or hostname(s) of the system(s) you'll be logging in from. If there are multiple addresses, separate them with a single space. If you have only one or two user accounts that you wish to be able to log in to your system, you can prevent all other logins by making the following changes:

```conf
-:ALL EXCEPT user1 user2:ttyv0 ttyv1 ttyv2 ttyv3 ttyv4 
```

Replace user1 user2 with the names of the user accounts to which you wish to give access. Put in as many ttys as you wish to restrict.

Alternatively, you can place the users in a group and give login access to that group. This example adds the users sam, bob, and dave to a group called mygroup and allows only the members of that group to login to the system. First, edit the `/etc/group file` and carefully add this line:

```conf
mygroup:*:100:sam,bob,dave 
```

When you add your own group, make sure you use a GID (in my case, 100) that is not being used by any other lines in your /etc/group file.

Then, change `/etc/login.access` to:

```conf
-:ALL EXCEPT mygroup:ttyv0 ttyv1 ttyv2 ttyv3 ttyv4 ttyv5 
```

It is very important you test this change. Leave one terminal logged in, just in case something goes wrong. Go to another terminal and try to log in as each of the users in your group. That should work. Then try to log in as another user; if need be, create a test account and try to log in as that test account. That login attempt should result in a `Permission denied` message.