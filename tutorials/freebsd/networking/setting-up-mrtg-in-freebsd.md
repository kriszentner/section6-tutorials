 # Setting up MRTG in FreeBSD

You may want to set up MRTG for monitoring much like we have done here at Section 6 Networks. This will show you how.
Whats Needed

* Install Apache on the server you wish to display MRTG stats from. (ports/www/apache2)
* Install MRTG on the server you wish to display MRTG stats from. (ports/net/mrtg)
* Install SNMP on the machines you wish to monitor (ports/net/net-snmp)

Configure SNMPD on your machines to be monitored

This step is not necessary if you don’t wish to customize snmp communities and if security isn't an issue. Otherwise, this can be done one of two ways. Either:

* Run snmpconf and answer all the questions

or

* Create your own config file in /usr/local/share/snmp/snmpd.conf

MRTG only supports SNMPv2, so v3 authentication is not an option.

## Creating Your Own Config

Here is a sample snmpd.conf created with the aide of snmpconf feel free to edit it to your needs and cut and paste.

```conf
syslocation: Systemland, USA
sysservices 0
syscontact	admin@yourlan.com

               #community	#hosts allowed
rwcommunity	private 	trusted.host.com
rocommunity	everyone 	10.0.0.0/24
```

There are probably more you can put in here, but this is a basic configuration. rwcommunitydefines who can write to the host’s MIBs (the derivatives that show info, you can view all the mibs with snmpwalk. The rocommunity shows who can view your SNMP tree. This defaults to public so you may want to make it something harder to guess.

## Creating the MRTG Config

This only needs to be done on the host your using MRTG. It comes with a handy tool called cfgmaker, however this tool will only create configs to monitor traffic. To monitor other things like CPU usage we’ll have to get a little tricky. For now lets just make a basic config using the following commands:

```shell
# cd /usr/local/etc/mrtg
# cfgmaker --output=mrtg.cfg everyone@host1.host.com everyone@host2.host.com
```

In this example everyone is the community name, used from the config file above. If you didn’t make a config file you can use public as your community. There is a lot that you can do with this config file and the more you want to customize it the more time it will take. 

## Cron MRTG

So, you’ve at least set the WorkDir value of MRTG and you’ll need to run it every five minutes. Be warned that MRTG spews a lot of .png and log files so you won’t want WorkDir to be your top level html directory. Make a separate one for it. You’ll want to run

```shell
/usr/local/bin/mrtg /usr/local/etc/mrtg/mrtg.cfg
```

three times to get the log errors out of the way. If you see any other errors on the fourth time you run it you’ll have to fix your config file. Once you find you don't have any errors cron MRTG by doing the following as root:

```shell
crontab -e
```

and put the following line in there:
```cron
*/5 * * * * /usr/local/bin/mrtg /usr/local/etc/mrtg/mrtg.cfg
```
