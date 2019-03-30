 # Using Bacula for Tape Backups

## What is Bacula?

Bacula is a very well designed and documented backup systems that is fairly intuitive to use. Some of the many features of Bacula include the ability to schedule backups of multiple platforms across the network, and to store information about those backup jobs in a MySQL database.

Bacula can also handle machines that connect to the network infrequently, such as laptops. Scripts can be run before or after the job a backup job, on the client as well as the server.

The restoration system is also well designed. One can restore from a paticular point in time. Bacula can prompt for which tapes or media that is required to perform the restoration. Bacula even supports browsing files to allow for individual file and directory restoration.
Preparing for installation

Before we cover installing and configuring Bacula, there are a few assumptions made with this article.

* The user has a working installation of FreeBSD 5.x
* The user has installed some sort of backup media (tape or optical drive)
* The user has an updated FreeBSD ports tree installed in /usr/ports
* The user has a working installation of MySQL 4.x server

If any of these requirements are not met, then it will make it difficult to continue with this article (Though theoretically the user could backup to an NFS or SMBFS mounted network drive). Please consult the WIKI for more information on updating Ports with CVSUP and MySQL concepts.

# Components of Bacula

Bacula is a full client-server network backup solution and is composed of several distinct parts:

* Director - The director is the most complex part of the system. It is the daemon that will keep track of all clients and files to be backed up. This daemon is the layer between the clients and the storage daemon.

* Client Daemon - The Client (or File) daemon is the backup agent that runs on each computer which will be backed up by the Director.

* Storage Daemon - The Storage Daemon communicates with the backup device, which may be tape or optical disk.

* Console - The Console is the primary interface used to interact with the Director. Though there are UI subsitutes for the Bacula console, fo rthis article we will concentrate on the console insterface that comes with Bacula

Each Client Daemon will potentially have an entry in the Director configuration file. Other important entries include Jobs and FileSets. A FileSet identifies a set of files to backup. A Job specifies a single FileSet, the type of backup to perform (full, differential, incremental), the schedule of the backup, and what device to use for backups.

## Installing the Bacula Server

The first thing we need to do is install the Bacula Server component. This componenst is located in the FreeBSD ports tree under the following directory:

```shell
/usr/ports/sysutils/bacula-server
```

We can simply change to this directory and compile the Bacula server with MySQL support by running the following command:

```shell
root@host# make with_mysql="YES"
```

Note that we can alternately use make config and choose the MySQL option from the configurations menu. Then we can simply install the port after it has compiled by running the following command:

```shell
root@host# make install
```

## Configuring the MySQL database

We now need to add a MySQL database to which Bacula can log. We can accomplish this task by simply running the following command:

```shell
root@host# mysqladmin create bacula
```

Once the database has been created then we need to grant privileges on the database for the bacula database so that the Bacula daemon has access.

* (Note that by default Bacula runs as root, if you wish to run Bacula as a non provileged user then you will have to grant privileges on the bacula database for the user under which you would like to run Bacula and access the database)

* (Also note that if you wish to use a different database name you will have to alter the MySQL scripts in the Bacula work directory to reflect the changes of the database name)

```shell
root@host# /usr/ports/sysutils/bacula-server/work/bacula-1.36.3/src/cats/grant_mysql_privileges
```

This will grant privileges on the database for the current user. Then we will then need to create the table structure for Bacula. This is also a script in the work directory for the Bacula port:

```shell
root@host# /usr/ports/sysutils/bacula-server/work/bacula-1.36.3/src/cats/make_mysql_tables
```

* Also note that if you changed the root password for the MySQL database, you might get something like the following error when trying to run either of these scripts:

```shell
ERROR 1045: Access denied for user: 'root@localhost' (Using password: NO)
Creation of Bacula MySQL tables failed.
```

If that is the case then you can simply run each script command with the -p mysql option to prompt for the root password of the MySQL database, thus running the scripts and setting up your database.

## Configuring the Bacula Server

Bacula has myriads of configurations options. The goal of this section is to provide minimalistic configuration options as an example to get up and running quickly.
The Storage Device Configuration

Bacula is very configurable when it comes to storage devie definitions. One can backup to anything from floppy disks to mounted network drives. The following is a simple example of a storage configuration:

```conf
################### /usr/local/etc/bacula-sd.conf ###########################
Storage {                             # definition of myself
 Name = computer-sd
 SDPort = 9103                       # Director's port
 WorkingDirectory = "/var/db/bacula"
 Pid Directory = "/var/run"
 Maximum Concurrent Jobs = 20
}
##############################################################################
# List Directors who are permitted to contact Storage daemon
#
Director {
 Name = computer-dir
 Password = "password1"
}
#
# Restricted Director, used by tray-monitor to get the
#   status of the storage daemon
#
Director {
 Name = computer-mon
 Password = "password2"
 Monitor = yes
}
##############################################################################
# Devices supported by this Storage daemon
# To connect, the Director's bacula-dir.conf must have the
#  same Name and MediaType.
#
Device {
 Name = FileStorage
 Media Type = DDS-4
 Archive Device = /dev/sa0
 LabelMedia = yes;                   # lets Bacula label unlabeled media
 Random Access = Yes;
 AutomaticMount = yes;               # when device opened, read it
 RemovableMedia = yes;
 AlwaysOpen = no;
}
###############################################################################
# Send all messages to the Director,
# mount messages also are sent to the email address
# 
Messages {
 Name = Standard
 director = computer-dir = all
}
```

The configuration of the Storage Device is pretty self explanatory.. but if you do have further questions, consult the Bacula documentation site for more information.

## The Director Configuration

The configuration for the Bacula Director daemon is a little more involved. It can contains each Client, Job, FileSet, and Storage Device for potential backups to be run.

The following example shows the relevent entries of the Director Configuration file we want to configure:

```conf
################ /usr/local/etc/bacula-dir.conf ######################
Director {                            # define myself
 Name = computer-dir
 DIRport = 9101                # where we listen for UA connections
 QueryFile = "/usr/local/share/bacula/query.sql"
 WorkingDirectory = "/var/db/bacula"
 PidDirectory = "/var/run"
 Maximum Concurrent Jobs = 1
 Password = "password2"         # Console pa
 Messages = Daemon
}
######################################################################
JobDefs {
 Name = "DefaultJob"
 Type = Backup
 Level = Incremental
 Client = computer-fd
 FileSet = "Full Set"
 Schedule = "WeeklyCycle"
 Storage = FileStorage
 Messages = Standard
 Pool = Default
 Priority = 10
}
######################################################################
# Backup the catalog database (after the nightly save)
Job {
 Name = "BackupCatalog"
 JobDefs = "DefaultJob"
 Level = Full
 FileSet="Catalog"
 Schedule = "WeeklyCycleAfterBackup"
 # This creates an ASCII copy of the catalog
 RunBeforeJob = "/usr/local/share/bacula/make_catalog_backup bacula bacula"
 # This deletes the copy of the catalog
 RunAfterJob  = "/usr/local/share/bacula/delete_catalog_backup"
 Write Bootstrap = "/var/db/bacula/BackupCatalog.bsr"
 Priority = 11                   # run after main backup
}
#######################################################################
# Standard Restore template, to be changed by Console program
Job {
 Name = "RestoreFiles"
 Type = Restore
 Client=computer-fd
 FileSet="Full Set"
 Storage = FileStorage
 Pool = Default
 Messages = Standard
 Where = /tmp/bacula-restores
}
########################################################################
# List of files to be backed up
FileSet {
 Name = "Full Set"
 Include {
   Options {
     signature = MD5
   }
########################################################################
#  Put your list of files here, preceded by 'File =', one per line
#    or include an external list with:
#
#    File = <file-name
#
#  Note: / backs up everything on the root partition.
#    if you have other partitons such as /usr or /home
#    you will probably want to add them too.
#
#  By default this is defined to point to the Bacula build
#    directory to give a reasonable FileSet to backup to
#    disk storage during initial testing.
#
   File = /
   File = /usr
   File = /var
 }
##########################################################################
#
# If you backup the root directory, the following two excluded
#   files can be useful
#
 Exclude {
   File = /proc
   File = /tmp
   File = /.journal
   File = /.fsck
  }
}
#
# When to do the backups, full backup on first sunday of the month,
#  differential (i.e. incremental since full) every other sunday,
#  and incremental backups other days
Schedule {
 Name = "WeeklyCycle"
 Run = Full 1st sun at 1:05
 Run = Differential 2nd-5th sun at 1:05
 Run = Incremental mon-sat at 1:05
}
######################################################################
# This schedule does the catalog. It starts after the WeeklyCycle
Schedule {
 Name = "WeeklyCycleAfterBackup"
 Run = Full sun-sat at 1:10
}
######################################################################
# This is the backup of the catalog
FileSet {
 Name = "Catalog"
 Include {
   Options {
     signature = MD5
   }
   File = /var/db/bacula/bacula.sql
 }
}
# Client (File Services) to backup
Client {
 Name = computer-fd
 Address = clientcomputer.domain.com
 FDPort = 9102
 Catalog = MyCatalog
 Password = "clientpassword"          # password
 File Retention = 30 days            # 30 days
 Job Retention = 6 months            # six months
 AutoPrune = yes                     # Prune expired Jobs/Files
}
# Definiton of file storage device
Storage {
 Name = FileStorage
# Do not use "localhost" here
 Address = server                # N.B. Use a fully qualified name here
 SDPort = 9103
 Password = "password1"
 Device = FileStorage
 Media Type = DDS-4
}
```

## Testing the Tape Drive

It is usually a good idea to test out the medium to which we are going to backup. Fortunately, Bacula comes with a handy utility for testing the tape drive. The `tape` program is a console based utility we can use to perform certain operations on our medium. In this case.. we will use the "test" option to test our tape drive:

```shell
root@host# /usr/local/sbin/btape -c /usr/local/etc/bacula-sd.conf /dev/sa0
btape: butil.c:258 Using device: "/dev/sa0" for writing.
btape: btape.c:335 open_dev /dev/sa0 OK
*test
=== Append files test ===
This test is essential to Bacula.
I'm going to write one record in file 0,
              two records in file 1,
        and three records in file 2
btape: btape.c:380 Rewound /dev/sa0
btape: btape.c:845 Wrote one record of 64412 bytes.
btape: btape.c:847 Wrote block to device.
```

Be sure and type "test" at the btape console. We will then have to wait a bit as this process will wind the tape and test the integrity of the drive as well as the medium. Afterwards we should know that the tape drive and the medium look good for backups. We can exit the btape console by typing quit.

## Starting the Bacula Server

It is now time to start the Bacula daemons and get our first backups going.

```shell
root@host# /usr/local/etc/rc.d/bacula.sh start
```

* (Note that the bacula service might not start if the shell script is still named as a sample file. Simple rename `/usr/local/etc/rc.d/bacula.sh.sample` to `/usr/local/etc/rc.d/bacula.sh`

We can verify the service is running with the following command:

```shell
root@host# ps -aux | grep bacula
root 63416 0.0 0.3 2040 1172 ?? Ss 4:09PM 0:00.01 /usr/local/sbin/bacula-sd -v -c 
root 63418 0.0 0.3 1856 1036 ?? Ss 4:09PM 0:00.00 /usr/local/sbin/bacula-fd -v -c 
root 63422 0.0 0.4 2360 1440 ?? Ss 4:09PM 0:00.00 /usr/local/sbin/bacula-dir -v -c
```

## Installing the Bacula Client Daemon

The Bacula client daemon installation for FreeBSD is much the same as the Server installation. However, the client daemon can be installed on multiple computers to interface with the Bacula server. To install the client daemon on FreeBSD, run the following commands:

```shell
root@host# cd /usr/ports/sysutils/bacula-client
root#host# make install
```

To install the Bacula client daemon on GNU/Linux platforms, consult the documentation of your distribution for more information on the Bacula client package.. or visit the Bacula Downloads page for all distributions, including Windows and Mac OSX.

## Configuring the Bacula Client Daemon

Configuration of the Bacula client is a much simpler process than the Server component configuration. All we need to do is to define a few options for the client daemon to run:

```conf
#################### /usr/local/etc/bacula-fd.conf ##################
Director {
 Name = computer-dir
 Password = "secret"
}
#####################################################################
# Restricted Director, used by tray-monitor to get the
#   status of the file daemon
#
Director {
 Name = computer-mon
 Password = "secret"
 Monitor = yes
}
#####################################################################
# "Global" File daemon configuration specifications
#
FileDaemon {                          # this is me
 Name = computer-fd
 FDport = 9102                  # where we listen for the director
 WorkingDirectory = /var/db/bacula
 Pid Directory = /var/run
 Maximum Concurrent Jobs = 20
}
#####################################################################
# Send all messages except skipped files back to Director
Messages {
 Name = Standard
 director = computer-dir = all, !skipped
}
```

Hopefully this configuration file is self explanatory. We only need to point it to the right Director we configured earlier and use the right password to connect.

* Note that you will need to install the client daemon on the same computer as the server daemon if you wish to backup files locally on the backup server itself.

* Also note that the client daemon installs an extra bacula.sh.sample script that is not need for bacula to startup. Simply remove this sample script and restart bacula

From here we just need to setup the console configuration file and we can start testing backups.

```conf
#################### /usr/local/etc/bconsole.conf ###################
#
# Bacula User Agent (or Console) Configuration File
#
Director {
 Name = computer-dir
 DIRport = 9101
 address = computer.domain.com
 Password = "password"
}
```

This will allow our client console to connect to the Director and take a peek at what is going on.
Running the Bacula Console

We can run the Bacula Console by running the following command:

```shell
root@host# /usr/local/sbin/bconsole -c /usr/local/etc/bconsole.conf
```

The `status all` console command displays the status of each system component and is also a quick and easy way to verify that all components are up and running.

```shell
status all

Using default Catalog name=MyCatalog DB=bacula laptop-dir Version: 1.32c (30 Oct 2003) i386-portbld-freebsd4.8 freebsd 4.8-STABLE Daemon started 02-Nov-2003 12:42, 1 Job run. Last Job *Console*.2003-11-02_13.12.18 finished at 02-Nov-2003 13:12 Files=0 Bytes=0 Termination Status=Error Console connected at 02-Nov-2003 13:12 No jobs are running.

Scheduled Jobs:
Level       Type    Scheduled         Name          Volume
===============================================================================
Incremental Backup  03-Nov-2003 01:05 Client1       *unknown*
Full        Backup  03-Nov-2003 01:10 BackupCatalog *unknown*
====
Connecting to Storage daemon File at computer.domain.com:9103
computer-sd Version: 1.32c (30 Oct 2003) i386-portbld-freebsd4.8 freebsd 4.8-STABLE
Daemon started 02-Nov-2003 12:42, 0 Jobs run.
Device /tmp is not open.
No jobs running.
====
Connecting to Client computer-fd at computer.domain.com:9102
computer-fd Version: 1.36.3 (22 April 2005) i386-portbld-freebsd5.4 freebsd 5.4-RELEASE
Daemon started 26-May-05 19:45, 0 Jobs run since started.
```

From here it is just a matter of learning the console commands at our disposal. Try running the help console command for a little insight to available options. For more information on the Bacula console, visit the [Bacula documentation page](http://www.bacula.org/rel-manual/index.html).

Happy backups, and good luck.
