# Setting up a PPTP VPN Server in FreeBSD

If you want to run a PPTP (Point to Point Tunneling Protocol) server on FreeBSD, there are a couple of solutions available. The option discussed here is using the MPD port for just such a service. The current version of MPD we use here at Section6 is 3.8. This of course may vary as to the date of your ports tree.
What’s Needed

* A FreeBSD kernel compiled with “pseudo–device tun” support
* Enable the option sysctl net.inet.ip.forwarding=1 or gateway_enable=“YES” on /etc/rc.conf
* Install MPD on the server you wish to run PPTP services (ports/net/mpd)
* Open and redirect TCP port 1723 and protocol 47 (GRE) if the PPTP server exists behind a firewall

## Configure the the MPD links and configuration file

By default, the ports installation of MPD creates a `/usr/local/etc/mpd` directory which houses the configuration files for mpd, and it also creates mpd.sh startup script in `/usr/local/etc/rc.d`

## Creating the MPD configuration

The first file to look at is called is called mpd.conf (which is represented by a sample file called mpd.conf.sample):

```conf
default:
       load pptp

pptp:
       new -i ng0 pptp pptp                    	## create a new interface of ng0 for the pptp connection
       set iface disable on-demand             	## disable on-deman dialing for this connection
       set iface enable proxy-arp			## enable the arp proxy for the created interface
       set bundle disable multilink			## disable multi link options
       set bundle authname Fred			## define the username for this connection
       set bundle enable encryption			## enable encryption for this connection
       set link yes acfcomp protocomp			## address control and protocol field compression
       set link disable pap				## disable PAP authentication for this link
       set link enable chap				## enable CHAP authentication for this link
       set link keep-alive 10 60			## keep alive settings for idle links
       set ipcp enable vjcomp		           	## enables header compression for the link
       set ipcp ranges 10.0.0.2/32 10.0.0.101/32	## sets IP of PPTP server as well as initial link
       set ipcp dns 10.0.0.1				## sets IP of DNS server to be given to client
       set ipcp nbns 10.0.0.20		        	## sets IP of the WINS server to be given out
       set bundle enable compression			## enables tunnel compression
       set ccp enable mppc				## enables microsoft point-to-point compression
       set ccp enable mpp-e40		        	## 40-bit MPP encryption
       set ccp enable mpp-e128		        	## 128-bit MPP encryption
       set ccp yes mpp-stateless			## enables stateless mode for faster recovery
       set bundle enable crypt-reqd		## require client to have encryption or drop link
```

The first section tells `mpd` to load a connection called `pptp`. Then next section defines the settings for the connection called pptp. To allow other clients to make simultaneous connections, you must define and load additional connections (pptp1, pptp2, etc.. )

## Creating the MPD links

The next file to look at is called `mpd.links` (whisch is also represented by a sample file called `mpd.links.sample`)

```conf
pptp:
set link type pptp	## define the link type protocol as PPTP
set pptp self 10.0.0.2        ## define the IP address  on which MPD will run
set pptp enable incoming      ## define the connection as Incoming
set pptp enable originate     ## enables PPTP connection for communication with the client
```

This entry is a link configuration for the connection that was defined in the mpd.conf file. There must be subsequent link entries for each connection that was defined (pptp1, pptp2, etc. )

## Creating MPD usernames and passwords

The third file to look at is mpd.secrets. This file contains the usernames and passwords of users that have access to the PPTP server as defined in the “authname” configuration for each connection in the mpd.conf file.

```conf
Fred	password_for_fred
Joe	password_for_joe
Bob	password_for_bob
```

These lines are pretty straightforward and understandable. Just make sure after creating the file that only root has read access to this file.

From here, you just need to start MPD from the startup script, `/usr/local/etc/rc.d/mpd.sh start`. You now have a funtional PPTP server on your network. Have your PPTP clients login and test connectivity with the network.
