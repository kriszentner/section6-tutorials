 # Creating a FreeBSD Wireless Access Point

Access points are essentially wireless switches or hubs. Just like a switch or a hub, all clients communicate through the access point. FreeBSD allows us to easily create an access point with just very little configuration and just the right hardware

To set up a wireless access point using FreeBSD, you need to have a compatible wireless card. For the article presented here, we are using a Prism 2-based chipset. For a complete list of cards that are supported, consult the man page for wi, or visit the Wireless Network Interface Section of the FreeBSD documentation site.

## Configuring the kernel

Depending on how you wish to set up the access point will determine what options need to be added to the kernel config file. If the wireless network device is being installed on a server that is currently running as a Firewall/NAT, then we only need to compile the wireless device driver into the kernel:

```conf
# Wireless NIC cards
device          wlan            # 802.11 support
device          an              # Aironet 4500/4800 802.11 wireless NICs.
device          awi             # BayStack 660 and others
device          wi              # WaveLAN/Intersil/Symbol 802.11 wireless NICs.
device          wl              # Older non 802.11 Wavelan wireless NIC.
```

Choose the appropriate driver for your card from the list and include the wlan device, then recompile and install your kernel.

For more information on setting up a Firewall/NAT in FreeBSD, consult the articles:

* [Creating a FreeBSD Firewall using PF](../security/setting-up-a-firewall-nat-using-pf)
* [Setting up a Firewall NAT using IPFCreating a FreeBSD NAT/Firewall using IPfilter](../security/setting-up-a-firewall-nat-using-ipf)

If this the wireless network device is going to be installed on a system that does not serve as a Firewall/NAT, then we would want to include the `BRIDGE` option, along with the appropriate wireless device driver in the kernel config file.

```conf
# Ethernet bridging support
option		BRIDGE

# Wireless NIC cards
device          wlan            # 802.11 support
device          an              # Aironet 4500/4800 802.11 wireless NICs.
device          awi             # BayStack 660 and others
device          wi              # WaveLAN/Intersil/Symbol 802.11 wireless NICs.
device          wl              # Older non 802.11 Wavelan wireless NIC.
```

The bridging option will allow the wireless device to communicate with the wired ethernet interface. We must also add a couple of options to the `/etc/sysctl.conf` file in order to establish the bridge between the two interfaces:

```conf
net.inet.ip.forwarding=1
net.link.ether.bridge.enable=1
net.link.ether.bridge.config=wi0,fxp0
```

Be sure and replace fxp0 with whatever wired ethernet interface you are using with your FreeBSD installation. For information on bridging, consult the Bridging Section of the FreeBSD Handbook.

## Configuring the Wireless Interface

The configuration of the wireless interface is fairly straightforward, we just need to add a few more options than if it were a wired ethernet interface. The following is an example ofifconfig options for a wireless interface:

```shell
ifconfig wi0 inet 10.0.0.5 netmask 255.255.255.0 
ifconfig wi0 ssid My_Network channel 11 media DS/11Mbps mediaopt hostap up stationname "My Network"
```

Of course this can all be setup in the `/etc/rc.conf` file so that these settings are retained every time the system boots. From this point, your access point should be up and broadcasting. There are just a couple more options to consider

## Post Configuration

As stated earlier, if the wireless interface is installed in a server that is functioning as a Firewall/NAT, then the bridging option is unecessary. We just need to add a couple of rules to our firewall configuration files to allow traffic to be passed from the wireless interface.

If you are using PF as your Firewall/NAT solution, simply add the following lines to your `/etc/pf.conf` file

```conf
pass in on wi0 from wi0:network to any keep state
pass out on wi0 from any to $wi0:network keep state
```

Replace `wi0` with the appropriate interface name of your wireless card

If you are using IPfilter as your Firewall/NAT solution, then simply add the following lines to your `/etc/ipf.rules` file

```shell
pass in on wi0 from any to any keep state
pass out on wi0 from any to any keep state
```

Again, replace wi0 with the appropriate interface name of your wireless card.

## Administration

Once the access point is configured and operational, we will want to see the clients that are associated with the access point. We can type the following command to get this information:

```shell
root@host# wicontrol -l
1 station:
ap[0]:
        netname (SSID):                 [ My_Network ]
        BSSID:                          [ 00:04:23:60:89:d9 ]
        Channel:                        [ 11 ]
        Quality/Signal/Noise [signal]:  [ 0 / 51 / 0 ]
                                [dBm]:  [ 0 / -98 / -149 ]
        BSS Beacon Interval [msec]:     [ 10 ]
        Capinfo:                        [ ESS ]
```

Now you should have a complete functioning access point up and running. You are encouraged to read more about the [wicontrol](http://www.freebsd.org/cgi/man.cgi?query=wicontrol&sektion=8) and [wi](http://www.freebsd.org/cgi/man.cgi?query=wi&sektion=4) commands for further information.
