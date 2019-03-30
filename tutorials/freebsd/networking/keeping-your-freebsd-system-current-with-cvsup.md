# Keeping your FreeBSD System Current with CVSup

CVSup is a software package for distributing and updating source trees from a master CVS (current versioning system) repository on a remote server host. The FreeBSD system and port sources are maintained in a CVS repository on central development servers. With CVSup, FreeBSD users can easily keep their own system and port source trees up to date.

## Installation

The easiest way to install CVSup is to use the precompiled net/cvsup package from the FreeBSD packages collection. If you prefer to build CVSup from source, you can use the net/cvsup port instead (If you installed the ports collection during the setup of FreeBSD). But be forewarned: the net/cvsup port depends on the Modula-3 system, which takes a substantial amount of time and disk space to download and build.

Note: If you are going to be using CVSup on a machine which will not have XFree86™ installed, such as a server, be sure to use the port which does not include the CVSup GUI, net/cvsup-without-gui.

## Configuration

A configuration file called the supfile controls CVSup’s operation. There are some sample supfiles in the directory /usr/share/examples/cvsup/.

The information in a supfile answers the following questions for CVSup:

* Which files do you want to receive?
* Which versions of them do you want?
* Where do you want to get them from?
* Where do you want to put them on your own machine?

Let us look at an example configuration file for retrieving update sources for the FreeBSD ports tree. The following illustrates the ports-supfile located in `/usr/share/examples/cvsup`.

```conf
*default host=Cvsup8.FreeBSD.org
*default base=/usr
*default prefix=/usr
*default release=cvs tag=.
*default delete use-rel-suffix
*default compress
ports-all
```

Note the only lines we are concerned with include a * character at the beginning of each line. Any line stat beginning with the # character is commented out and ignored.

The first line tells CVSup from which server it should pull updates. In this case we have configured it to use cvsup8.freebsd.org. For a complete list of FreeBSD source CVSup servers, please refer to the " [CVSup server list](http://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/cvsup.html#HANDBOOK-MIRRORS-CHAPTER-SGML-CENTRAL-CVSUP).

The second line tell CVS where CVS will store information about the collections transferred to you system. In this example, `/usr` will generate information in `/usr/sup`.

The third line tells CVSUP where to store the updated source directory. In this case, we will use `/usr`. A directory of what we are downloading will be created under `/usr`, such `as/usr/ports`.

The fourth line tells CVSup what release tag it is downloading. You can specify source for various releases of operating systems such as RELENG_4 (FreeBSD 4.0 English Release), but in this case we want the updated source for ports. Using the . tag will give us the most updated source for ports and we are not concerned about a specific release of the operating system itself. If we were downloading src instead of ports then we would specify the version of the FreeBSD source files we wish to download.

The fifth line allows CVSup to delete or overwrite the old directory structure and replace it with the update source downloaded.

The sixth line specifies that CVSup should use a compressed file transfer (This is handy if you don’t want to saturate your bandwidth downloading source files).

Finally, the seventh option specifies the source files to download. In this case we are downloading all of the ports distribution. If we were downloading source for the FreeBSD operating System, we would use `src-all`.

Now that the supfile has been saved, simply run cvsup with the specified options to use the specific supfile we just edited with the following command:

```shell
cvsup -g -L 2 /usr/share/examples/cvsup/ports-supfile
```

For information on cvsup switches, please refer to the cvsup man page.

Alternately, you can create a shell script file that runs the updates or even add a scheduled job to FreeBSD’s crontab for periodical updates to your source or ports tree.

## A note on FreeBSD Ports

Using refuse to slim down your ports tree

You'll find that when you cvsup your ports tree you'll find a ton of ports you don't want or need, like for example maybe you don't want all the foreign language ports such as ukranian and vietnamese. You can use a file to make it so your computer doesn't spend time downloading updates to these trees. The file is usually /usr/sup/refuse (as long as you have *default base=/usr in your supfile). A good example of a refuse file excluding non-english ports is in

```shell
/usr/src/share/examples/cvsup/refuse
```

This can also be used for the src tree as well but is not recommended unless you know what you're doing as it can easily break make buildworld.

## Downloading the INDEX file to speed up port builds

You may notice that if you are cvsupping ports and are using the handy tool portinstall that it will take about 2 hours or so to build an INDEX.tmp. This is because ports no longer includes the index file. To remedy this you'll want to cvsup your ports, then:

```shell
cd /usr/ports && make fetchindex && portsdb -u<
```

This will allow future port builds to go smoothly.

## Tracking ports security

You may also notice that when you're building ports you may end up with this error:

```shell
===>  Vulnerability check disabled, database not found
```

This is because ports is looking for a tool called portaudit in security/portaudit. You may want to install it from ports, and run portaudit -Fa to fetch the portaudit database and give you a report on which ports have known vulnerabilities. FreeBSD will also use portaudit as part of its daily security report e-mailed to you (or whomever is assigned the root address).

Thus, based on the above, an appropriate crontab to update your ports tree daily would be:

```cron
0 3 * * * /usr/local/bin/cvsup /root/ports-supfile && cd /usr/ports/ && make fetchindex && portsdb -u && /usr/local/sbin/portaudit -F
```
