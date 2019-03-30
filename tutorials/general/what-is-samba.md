# What is Samba?

## What is SMB?

The Server Message Block Protocol (SMB) is a method for client applications on a computer to request services from remote hosts on a network. SMB was first introduced in Windows for Workgroups as an upper layer protocol suite that is carried over lower level networking protocols such as TCP/IP or IPX. This upper layer protocol includes the ability for an application to access resources from a remote server such as shared files and printers. In fact, SMB is the preferred method of sharing and accessing resources for Microsoft operating systems.

Most modern distributions of Unix-like operating systems ship with packages or ports of the SMB client and server libraries, most commonly called Samba. Samba is designed for the ability to share files and printers with other SMB clients on the network, such as Windows computers. This article will take a quick look at installing and configuring Samba for two distributions; FreeBSD and Debian GNU/Linux.

## Installing Samba

Installing Samba is a pretty straight forward process on almost any distribution. For FreeBSD, we would simply compile the ports from the following directory:

```
root@host# cd /usr/ports/net/samba3
root@host# make install
```

On Debian GNU/Linux we would simply install the DEB package from our apt utility by running the following commands:

```
root@host# apt-get install samba-common
root@host# apt-get install samba
```

For information on installing the Samba package on other distributions, please consult the documentation of your specific distribution for more information.
Configuring Samba

Most Samba packages ship with a tool called SWAT, the Samba Web Administration Tool. This services runs a web-based frontend on port 901 of your server and allows you to configure SMB users, shared directories, and printers. For the scope of this particular document, we will manually configure the smb.conf, instead of using the SWAT utility.

Locate the smb.conf file on your system and open it with your favorite editor. On FreeBSD systems, this file is located in the `/usr/local/etc/` directory. On Debian GNU/Linux systems, this files is located in the `/etc/samba` directory. If you do not have a `smb.conf` file, then go ahead and create one.

The following is short and simple example smb.conf file that should get you up and running:

### Samba config file created using SWAT

```conf
# Global parameters
[global]
        workgroup = HOME
        netbios name = FreeBSD
        server string = FreeBSD
        interfaces = 10.0.0.0/24
        bind interfaces only = Yes
        security = SHARE
        preferred master = No
        local master = No 
        domain master = No

[WinShare]
        path = /winshare
        read only = No
        guest ok = Yes

[SecretShare]
        path = /secretshare
        read only = No
        guest ok = Yes
        browseable = No

[Printer]
        path = /tmp
        printable = Yes
```

The global section defines what NetBIOS workgroup this computer belongs in, as well as its NetBIOS browseable name. There are a few other paramets used here, such as the interfaces argument, which allows us to bind Samba to only one interface if the computer in question has multiple network cards. In this case, we have chosen to configure the security for these services as SHARE. There are alternatives to this option based on other servies on the network, such as a Domain Controller for authentication to the shared resources on Samba server. It is also important to note that we want to disable the master options, unless running this server as an emulated Windows Domain Controller. Leaving these options enabled can cause NetBIOS browsing problems with other systems on the network

The other sections actually describe the shared resources available, such as shared directories and a printer. Note the difference between the WinShare and the SecretShare, being that we set the browseable option to No. This is the equvalence of placing a $ symbol at the end of a share name in order to hide it. This share would not be browseable by other clients on the network and must be accessed directly by its resource name.

## Conclusion

At this point you should be able to start you Samba service by issuing the following command for Freebsd:

```bash
root@host# /usr/local/etc/rc.d/samba.sh start
```

Or in Debian GNU/Linux

```
root@host# /etc/init.d/samba start
```

If the service is already started, try restarting it. From here, we should be able to access our Samba server by its resource name. Try browsing for the name from your system, whether using Windows or another Unix client (configured with Samba of course), we should be able to see the browseable NetBIOS name and resources available.

Remember, this article by no means gives enough information to extensively or securely deploy Samba on a network, but is only meant to get a user up and running quickly. For more information on the options available for Samba, please visit the [Samba Web Site](http://www.samba.org).
