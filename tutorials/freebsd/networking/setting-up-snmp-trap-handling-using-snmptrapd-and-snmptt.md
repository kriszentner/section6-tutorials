 # Setting up SNMP trap handling using snmptrapd and snmptt

## Trap Handling
### About

This describes the basics of trap handling. In this article, we'll work with trap handling with snmptt. While this increases complexity by adding another layer to the mix, it also greatly increases the readibility of the incoming traps.

### What's needed

You'll need snmptrapd, snmptt, and MIB's for any devices you want traps sent for. A MIB basically allows the OID of the trap sent into something more descriptive.

Most hardware manufactuers have MIBs available on their site, or you can check out a site like mibdepot.

### Configuration

In this example, we'll be running both snmptrapd and snmptt. This allows dynamic configuration of snmptt, though it is possible to run snmptt in daemon mode.

Change snmptrapd to run with the options -On, this can be done in:

```shell
/etc/init.d/snmptrapd
```

on Redhat by changing this line:

```shell
OPTIONS="-On -s -u /var/run/snmptrapd.pid"
```

In your `snmptrapd.conf` file you'll want it to read simply:

```conf
traphandle default /usr/local/sbin/snmptt
```

In the `snmptt.ini` file I set the following options. Don't set these blindly, make sure they apply to you. 
```conf
mode = standalone 
description_mode = 1
unknown_trap_exec = /etc/snmp/traphandle.sh
```

## Get Your MIB On

Remember those MIB files? Well we're going to do something with them now. I'll take an example to run with.

Lets say you have about 8 Compaq MIB files all starting with "cpq", and you want one conf file that snmptt will handle. You could run a shell script like the following:

```shell
for i in cpq*; do
  snmpttconvertmib --in=/where/your/compaq/mibs/are/$i --out=/etc/snmp/compaq.conf --exec='/etc/snmp/traphandle.sh $r $s "$D"'
done
```

You'll notice above that we used `--exec` in the mib convert process. This will give a command that will be executed when that OID is found. This command could be something like a nagios event handler, or a shell script (which I have up there). You should probably look up which variables you want passed to your traps. There's a whole list in the documentation, but here's explanations for the ones I used:

```conf
$R, $r  - Trap hostname
$s  - Severity
$D - Description text from SNMPTT.CONF or MIB file
```

Afterwards you'd change your snmptt.ini:

```shell
snmptt_conf_files = <<END
/etc/snmp/compaq.conf
```

If you make more conf files you can just add new ones on a new line:

```shell
snmptt_conf_files = <<END
/etc/snmp/compaq.conf
/etc/snmp/ibm.conf
```

## Handling Traps

The shell script used in the --event line during the MIB conversion process could be just a about anything, such as the one below:

```shell
#!/bin/sh
#
echo "$4

Host: $1
Severity: $2

Description:
$3
" | mail -s "$2 Error from: $1" noc@section6.net
```

This will send a simple e-mail out when a trap is recieved.

## References

* http://www.snmptt.org/docs/snmptt.shtml