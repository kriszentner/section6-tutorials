# What is NFS 

## Description

Modern network operating systems support sharing files and services with other users and computers on the network. Administrators familiar with a Windows environment are used to relying on SMB file sharing. Even though most distributions of Unix support the SMB protocol, they also support another suite of file sharing protocols and services called NFS. NFS may intimidate many who are not familiar with its concepts, but it is really quite simple once one understands the methodology.

Each NFS connection works on a client-server model. One computer is the server and offers filesystems to other systems. This is known as "NFS exporting," and the filesystems that are shared are referred to as "exports." Remote clients can mount these server exports in a manner almost identical to that used to mount local filesystems on a Unix system.

One interesting thing about NFS is its stateless connections. One could reboot a server and theoretically the client will not freeze because a remote file system is not accessible. Though the client will not be able to access files on the server's export list while the server is down, once it returns the client can pick up right where it left off.

In this article we will look at connecting multiple FreeBSD 5.4 systems. Even though each NFS implementation on various distributions of Unix based operating systems may slightly differ, NFS should work between them all while only requiring the occasional tweak.
Configuring NFS

In configuring systems for NFS, both server and clients require NFS support in the kernel. FreeBSD's generic kernel supports NFS, but if we customize our kernel and do not like loading file system support as a module, we need to be sure our kernel configuration includes:

    options         NFSCLIENT               # Network Filesystem Client
    options         NFSSERVER               # Network Filesystem Server
    options         NFS_ROOT                # NFS usable as /, requires NFSCLIENT

First of all, we have the server side. We can enable basic NFS exports with the following rc.conf options:

    rpcbind_enable="YES"
    nfs_server_enable="YES"
    nfs_server_flags="-u -t -n 4"

Secondly, we have the client side. We can configure the NFS client options in the rc.conf with the following example:

    rpcbind_enable="YES"
    nfs_client_enable="YES"

The RPC service provides a mapping service for network ports. Different exports and clients require unique network ports. Clients ask RPC which port they should connect to for their actual mount. The nfs_server_enable option starts nfsd and mountd. mountd just listens for incoming NFS requests on assorted high-numbered network ports, and makes these port numbers available to portmap. When clients talk to RPC and mountd, nfsd actually handles their requests.

Once we reboot, our server should show something like the following amongst it's sockstat output. This shows that server-side NFS is running more or less properly. If we don't see something resembling this, check /var/log/messages for log messages indicating the problem.

    # sockstat -4
    USER     COMMAND    PID   FD PROTO  LOCAL ADDRESS         FOREIGN ADDRESS
    root     nfsd       385   3  tcp4   *:2049                *:*
    root     mountd     383   4  udp4   *:977                 *:*
    root     mountd     383   5  tcp4   *:779                 *:*
    root     rpcbind    316   7  udp4   *:111                 *:*
    root     rpcbind    316   8  udp4   *:957                 *:*
    root     rpcbind    316   9  tcp4   *:111                 *:*

Our client will not show any special sockstat output before network shares are mounted, but running ps -ax |grep nfs will display several nfsiod processes.

Now that our systems are prepared to handle NFS, we need to tell the server which directories it can export. We could just export the entire server, but that is not the most secure option. Clients should have little or no need to remotely mount the server root filesystem. Define allowed exports in the /etc/exports file. This file has a separate line for each hard-drive partition on the server. Each line has up to three components:

* the directories to be exported
* the options on that export
* the clients that can connect

## Exporting filesystems

Each combination of clients and server disk partition can only have one line in the exports file. This means that if /usr/ports and /usr/home are on the same partition, they must be exported in the same line to any one client. We do not have to export the entire partition and we can just as easily share out a single directory within a partition. This directory must be an absolute path and it must not be symbolically linked directories. If I wanted to export my home directory to every host on the Internet, I could use an /etc/exports line that consisted entirely of this:

```
/usr/home/johndoe
```

This example has no options and no host restrictions. Now that we have edited the exports file, we have to tell mountd to re-read it.

```
killall -1 mountd
```

Any problems will appear in our `/var/log/messages` file. For example, if we tried a single entry in `/etc/exports` of `/home/johndoe`, this would fail because `/etc/exports` cannot contain symlinks. FreeBSD puts user home directories under `/usr/home` and uses a symlink to create the `/home` directory. The error log might give a warning like this:

```
Jan 24 07:13:35 server mountd[68]: bad exports list line /home/johndoe
```

## Configuring the NFS client

Now, over on the client side, create the directory /nfsmount. We want to NFS-mount the home directory on the server onto this directory. This looks almost exactly like a standard mount(8) command. mount takes two arguments: the physical device we're using and the mount point. In this case, our physical device is a remote server and the exported filesystem:

```
# mount server:/usr/home/johndoe /nfsmount
#
```
Note that we are assuming that DNS is running on the network and is able to resolve server. If this is not the case, try replacing the hostname itself with the IP address of the server from which we want to mount its exports:

```
# mount a.b.c.d:/usr/home/johndoe /nfsmount
#
```

Of course we would replace a.b.c.d with the actual IP address of the server in question.

Once this finishes, test our mount with `df -h`

```
# df
Filesystem               1K-blocks     Used    Avail Capacity  Mounted on
/dev/ad0s1a                  99183    67411    23838    74%    /
/dev/ad0s1f                5186362  3873110   898344    81%    /usr
/dev/ad0s1e                 198399    21211   161317    12%    /var
procfs                           4        4        0   100%    /proc
server:/usr/home/johndoe  34723447  3886523 28059049    12%    /nfsmount
#
```

Permissions

One thing to note is that NFS uses the same usernames on each side of the connection. My files are owned by johndoe on the server, so they are owned by johndoe on the client. This can be a problem on a large network where users have root access on their own machines. To create a central repository of authorized users, consider Kerberos, NIS, or LDAP. On a small network or a network with limited administrators, this usually is not a problem.

Looking at the contents of /nfsmount, we see that the files are mostly owned by johndoe, with the occasional file owned by root on the remote server. Let us say that there is a file on the remote exports called "fileownedbyroot.txt", and that we want to remove it because the user "johndoe" does not need access to it:

```
# su
# cd /nfsmount
# rm fileownedbyroot.txt
override rw-r--r--  root/johndoe for fileownedbyroot.txt? y
rm: fileownedbyroot.txt: Permission denied
#
```

The issue at hand is if we are the "root" user, then why can we not delete the file?

Even though we are root on the client computer, we are not on the remote server. The server does not trust root on other machines to execute commands as root on the server. It does trust usernames, however. NFS has a special option for handling root; we can map requests from root to any other username. For example, we might say that all requests from "root" on a client will run as "nfsroot" on the server. With careful use of groups, we could allow this nfsroot user to have limited access to things. Use the -maproot option to map root to another user.

```
/usr/home/johndoe -maproot=0
```

We would have to restart mountd again. Since NFS is stateless, old mounts remain in effect. On the client we could remain in the same directory and restarting mountd did not disconnect the client at all. The rm shoule run flawlessly now.
Advanced Configurations

So, what if we want to export another directory on the same partition? For example, suppose we want to export /usr/src to another computer so that we can save space on the hard drive? List the other directories in /etc/exports, right after the first exported directory, separated by a space. While we're at it, we could also NFS-mount /usr/obj. This way, we could runmake buildworld on the faster server computer and then run make installworld on the slow workstation, thus greatly accelerating upgrades when recompiling FreeBSD source. The/etc/exports would now look like this:

```
/usr/home/johndoe /usr/src /usr/obj -maproot=0
```

There are no identifiers between the components of the line. Yes, it would be easier to read if we could put each shared directory on its own line, but we cannot because they are all on the same partition. Restart mountd and mount these filesytems:

```
# mount server:/usr/obj /usr/obj
# mount server:/usr/src /usr/src
```

The slow workstation filesystems would now look like this:

```
# df
Filesystem       	  1K-blocks Used    	Avail Capacity  Mounted on
/dev/ad0s1a          	  99183     67411    	23838     74%    /
/dev/ad0s1f        	  5186362   3873110   	898344    81%    /usr
/dev/ad0s1e         	  198399    21212   	161316    12%    /var
procfs                    4         4        	     0   100%    /proc
server:/usr/home/johndoe  34723447  3886523 	28059049  12%    /nfsmount
server:/usr/obj   	  34723447  3886485 	28059087  12%    /usr/obj
server:/usr/src   	  34723447  3886485 	28059087  12%    /usr/src
#
```

We can also easily restrict NFS mounts by IP address, allowing only certain clients to mount an exported share. Just specify the hostname or IP address at the end of the partition's/etc/exports entry. A client having an IP address of 192.168.1.200 would look like this:

```
/usr/home/johndoe /usr/src /usr/obj -maproot=0 192.168.1.200
```

This quickly becomes very convenient for upgrades. What's nice is that we can also build ports on the server, and install them on the client, by exporting /usr/ports. This is starting to get a little cumbersome however. Eventually /etc/exports will have entries for almost every directory in /usr. We might as well export the whole of /usr and get it over with. This may not be that great of a solution because we might want to mount the exported directories in different places. The exported ports tree should go over /usr/ports, for example, but we don't want the server's home directory overwriting our own. Fortunately, there's an easy solution. The -alldirs option allows us to export a partition, and all the directories beneath it. We must specify a partition when we use alldirs. Multiple options are separated by a comma.

```
/usr -alldirs,maproot=0 192.168.1.200
```

Now we restart mountd again, and all of /usr can be exported and mounted separately. For example, we still had `/usr/src` and `/usr/obj` NFS-mounted. Quick tests with `df` and `ls` show that the mounts are still there, still accessible, and still serving files. Now we can mount arbitrary directories from the server's exported /usr directory.
Performance

Finally, here's a tip on NFS performance. By default, FreeBSD uses conservative NFS mounting options. These work well when trying to interoperate with other distributions of Unix. We can use mount options to augment NFS performance but reduce interoperability somewhat. These options aren't necessary when we're working with one or two clients, but as our NFS installation grows, we'll find them helpful. They may or may not work with other operating systems; it depends on what options those systems support.

First of all, NFS runs over UDP by default. The tcp option tells the client to request a mount over TCP.

Then we want to make the mount interruptible. This means that if the server goes away, for any reason, client programs that are trying to access the export can be interrupted. Otherwise, the client will continue trying to access the export until the filesystem times out. Set this with the intr option.

NFS comes in many versions. The latest one in wide use is Version 3. We can request this with the nfsv3 option.

Finally, we have the size of read and write requests. The defaults are rather small, being well suited to smaller networks. We can set the read and write size to more practical values with the -r and -w options. 32768 seems to be an acceptable value for both.

So, putting this all together, we would want to mount the exports in the following way.

```
# mount -o tcp,intr,nfsv3,-w=32768,-r=32768 server:/usr/home/johndoe /nfsmount
```

This will give us close-to-optimal performance with minimal effort. Further fine-tuning NFS requires testing various options with different network equipment.

## Conclusion

Now that we have NFS running, we can effectively share whatever directories we deem necessary for our network setup. Though it might take a bit of tinkering to get it running, hopefully it is well worth the minor trouble for you to set up.
