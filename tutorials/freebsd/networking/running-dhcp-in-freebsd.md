# Running DHCP in FreeBSD

## Running DHCP Client

If you’re interested in running DHCP as a client on FreeBSD your requirements will be very few. Keep in mind that if you plan on running services on your system this set up may not be very advantageous as your address is susceptible to changing every so often. FreeBSD already comes with DHCP (the client is called dhclient). To have DHCP start on FreeBSD, put the following in your `/etc/rc.conf`:

```conf
ifconfig_ed0="DHCP"
```

This doesn’t require any `dhclient.conf` for a default client. The DHCP server will provide all the info you need to your system. Advanced Configuration

You may find yourself in a position where you’re forced to use DHCP, such as those on broadband cable and you want a static IP. For those in this position, like Section 6 Networks, there is an answer.

To run DHCP with most cable Internet providers, you can do the setup as explained above. You’ll notice that your leases are held in the file `/var/db/dhclient.leases`. Here is a standard lease from attbi:

```conf
lease {
  interface "ed0";
  fixed-address 12.230.180.99;
  option subnet-mask 255.255.255.0;
  option routers 12.230.180.1;
  option dhcp-lease-time 345600;
  option dhcp-message-type 5;
  option domain-name-servers 204.127.198.4,63.240.76.4;
  option dhcp-server-identifier 12.242.16.34;
  option broadcast-address 255.255.255.255;
  option host-name "dhcp-223-248";
  option domain-name "attbi.com";
  renew 2 2002/4/30 17:20:31;
  rebind 2 2002/4/30 17:20:31;
  expire 2 2002/4/30 17:20:31;
}
```

You’ll notice some valuable information here. I’ll assume at this point that you have an IP address and you don’t want it to be changed. Create an `/etc/dhclient.conf` and put part of your lease in it like so:

```conf
interface "ed0" {
}
lease {
  interface "ed0";
  fixed-address 12.230.180.99;
  option subnet-mask 255.255.255.0;
  option routers 12.230.180.1;
  option dhcp-message-type 5;
  option dhcp-server-identifier 12.242.16.34;
  option broadcast-address 255.255.255.255;
}
```

Be sure to change ed0 to the device you’re using of course. You’ll notice the whole lease isn’t in here, just the bits that we’d like to stay permanent. This will cause your client to insist on an IP instead of being passive and accepting whatever IP it’s assigned. Now you’ll also notice another annoying side effect from using DHCP with attbi. Your `/etc/resolv.conf` file will get clobbered from time to time. There are two ways around this. You can chflags schg /etc/resolv.conf making the file immutable. The problem with this is that if you run at securelevel 1 or higher this file is permenant until you get out of this mode. This may not be desirable. The second option is to make an /etc/dhclient-enter-hooks file which will write the file with servers you specify. There are probably other ways to get around this, but these are the two I know of. If you go for the second option you’ll want to put something like the following in your /etc/dhclient-enter-hooks file:

```bash
#!/bin/sh
new_domain_name="section6.net"
new_domain_name_servers="3ffe:b80:b54:1::1 10.0.0.1 204.127.198.4"
```

Now `chmod 744 /etc/dhclient-enter-hooks` this file needs to be executable. Restart dhclient, or restart your system if you feel like testing it to see how well it’ll run on a reboot (this is when IP changes occur most frequently). You’ll notice that you’ll have the same IP indefinitely.
