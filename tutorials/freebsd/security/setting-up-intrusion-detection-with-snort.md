# Setting up Intrusion Detection with Snort

This tutorial is current for FreeBSD 5.x and assumes you understand how to setup FreeBSD, install ports from the /usr/ports/ tree, and that you understand the basics of IP networking including firewalls and NAT. If you do not understand these topics, refer to the PF/NAT documentation article.
Ports that are needed:


* Apache 2.x : /usr/ports/www/apache2
* PHP4 (with_GD=″YES″) : /usr/ports/lang/php4
* MySQL 4.x server : /usr/ports/databases/mysql41-server
* MySQL 4.x client : /usr/ports/databases/mysql41-client
* Adodb 4.x : /usr/ports/databases/adodb
* PHPlot 4.x : /usr/ports/graphics/phplot
* Acid 0.9x : /usr/ports/security/acid
* Snort 2.x (with_MYSQL=″YES″) : /usr/ports/security/snort


## Setting up a MySQL database for Snort logging

Snort has the option to log to many different kinds of databases, in this case we will use MySQL. The first step in setting this up is to create a system account for which Snort will run as. The most important factor in doing so is to assign the user no shell access for the account during the adduser process.

```shell
Enter username [^[a- z0-9_][a-z0-9_-]*$]: snort
Enter full name [ ]: Snort User
Enter shell csh date no sh tcsh [csh]: no
```

After adding the user to the system, you will want to configure a MySQL account for the user `snort`. As root, you have access to MySQL server by simply running the `mysql` command. (you may wish to change the MySQL root account password for extra security measure) From the MySQL client session, we need to first add the snort database with the following command:

```sql
mysql> create database snort;
```

Once the database has been created, we need to add the snort user and permissions for the user on the 'snort' database.

```sql
mysql> grant all privileges on snort.* to 'snort'@'localhost'
-> identified by ‘password’ with grant option;
```

Once the user has been added with the proper permissions, then you can exit the MySQL client session by simply typing 'quit'. Now we can import the Snort Database tables from a sql file that comes distributed with Snort.

```shell
/usr/ports/security/snort/work/snort-2.1.1/schemas/create_mysql
```

This is the usual location of the sql file. Copy this file to a convenient location such as /tmp and edit the file. Simply add this line to the beginning of the file:

```shell
use snort;
```
You can substitute the name here for whatever name you used when creating the database in the MySQL client session. Once this line has been added, and the file has been saved, import the file running the following command:

```shell
root@host # mysql < create_mysql
```

Afterwards, run another MySQL client session and verify the existence of the tables within the newly created database by running the following commands.

```shell
mysql> use snort;
mysql> show tables;
  acid_ag
  acid_ag_alert
  acid_event
  acid_ip_cache
  data
  detail
  encoding
  event
  icmphdr
  iphdr
  opt
  reference
  reference_system
  schema
  sensor
  sig_class
  sig_reference
  signature
  tcphdr
  udphdr
```

The presence of these tables indicates that the create_mysql successfully imported its structure into the MySQL database. (note the rest of these tables get populated with the initialization of Acid) Now we can begin setting up the snort service itself.

## Setting up Snort

Like many other ports, snort places a sample configuration file in /usr/local/etc and a sample startup script in /usr/local/etc/rc.d. Starting with the configuration file, renamesnort.conf.example to snort.conf and open it in your preferred editor. Section 1 contains definitions for your network:

```conf
var HOME_NET 10.0.0.0/24
var EXTERNAL_NET any
var DNS_SERVERS 10.0.0.1
var SMTP_SERVERS 10.0.0.2</pre> 
```

These options are pretty self explanatory. HOME_NET defines the address range of your internal network; while EXTERNAL_NET defines the range of the outside network (in this case it is set to any). Other defined servers are optional for monitoring and alerting purposes. It is highly recommended that you at least define the active DNS, SNMP, and SMTP servers for your network so that you can make full functionality of the alerting features of snort.

Section 2 contains the definitions for traffic inspections that snort performs called 'preprocessors'. In depth discussion about these 'preprocessors' and their functions is beyond the scope of this document. Leaving the default configured processors should suffice in beginner usage of Snort for now. More in depth discussion of Snort preprocessors can be found at the following URL:

http://www.snort.org/docs/

Section 3 of the snort.conf file sets the configuration for the output of Snort. In this case you would want to choose MySQL as your output logging method, so setup the following option:

```conf
Output database log, mysql, user=snort password=password dbname=snort host=localhost
```

Of course you would want to substitute password for the actual password you setup for your 'snort' user when creating it from the MySQL client session. From here you should save yousnort.conf file and create a `/var/log/snort` directory for portscan logs.

The final step is to edit the snort.sh startup script to run the snort service as the user we created earlier. It is not recommended to let snort run as root as this could be a potential security risk. Open the /usr/local/etc/rc.d/snort.sh file change startup line for snort to read like the following:

```shell
${PREFIX}/bin/snort -Dqc ${PREFIX}/etc/snort.conf > -i –u snort –g snort /dev/null && echo –n " snort"
```

Substitute the UID and GID name of the account you created for the local system earlier. Afterwards, simply run the snort.sh startup script in order to start snort logging. Sometimes Snort has issues with specific rules files that have incorrect information or are formatted incorrectly, so double check /var/log/messages for any Snort startup errors.

## Setting up ACID

ACID is a php analysis front-end for MySQL. This handy little package provides a web interface to show alerts, statistics, and reports about traffic being captured by Snort. ACID is quick and easy to setup.

By default, ACID installs its content into `/usr/local/www/acid`. We just need to configure one file, acid_conf.php.

```php
$DBtype = 'mysql';
$alert_dbname = 'snort';
$alert_host = 'localhost';
$alert_user = 'snort';
$alert_password = 'password';
$Chartlib_path = '/usr/local/lib/php/phplot';
$portscan_file='/var/log/snort/portscan.txt';
```
From here, you should be able to create a symbolic link from your web content directory to the /usr/local/www/acid directory, and then access the content via the URL :

```shell
http://whatever_your_hostname_is/acid
```