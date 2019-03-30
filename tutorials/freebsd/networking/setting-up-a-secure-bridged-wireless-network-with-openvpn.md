# Setting up a Secure Bridged (Wireless) Network with OpenVPN

## Getting Routed OpenVPN working for FreeBSD

Author: kz

This howto is variation off of the Routed tutorial, however we'll do this by example since that is probably easiest to visualize for most people. This example has been tested on FreeBSD 5.4 and OpenVPN 2.0.5 but will likely work on other versions of FreeBSD including FreeBSD 6.x.
The Scenario

Here's the scenario. You have a network like the below

```asciiart
   {Internet}
        |(if0)
[FreeBSD Router]
        |(if1)
        |
   [Switch/Hub] 10.0.0.0/24
     |    |__________
     |               |
 [Wireless Bridge] [Other servers]
```

Now in this scenario, you have no choice but to encrypt your network with WEP or WPA, both of which are notoriously insecure. You can choose to block by mac address but people can still sniff your mac address and in some cases they can switch the mac of their wireless and join your network anyhow, and even if they can't there's a high likelyhood they can sniff your traffic capturing any websites you browse to, and maybe even passwords if they are unencrypted.

OpenVPN offers a solution with this network topology:

```asciiart
{Internet}
    |(if0)
[FreeBSD Router]
   |(if2)  |(if1)
   |       |_________
   |                 |
[Wireless Bridge]  [Other Servers]
  10.0.1.1/24        10.0.0.0/24
```

If this solution you have a private network for your wireless bridge, and a separate network for your other ethernet connected servers.

In the following example we'll set up this network with these assumptions (match them to your own): Our FreeBSD router has the following interfaces:

```conf
if0 at ip 1.2.3.4
if1 at ip 10.0.0.1/24
if2 at ip 10.0.1.1/24
```

On the router you must make sure you have the following in your kernel:

```conf
options BRIDGING
device tap
```

And you need to have these sysctls set (you can put them in `/etc/sysctl.conf`).

```conf
net.link.ether.bridge.enable=1
net.link.ether.bridge_cfg=if1,tap0
```

## Getting keys set up

You can try using the tools in `/usr/local/share/doc/openvpn/easy-rsa` but they don't seem to work that well. Here's a way that does.

First step is to get you SSL keys generated. I'm not going to outline everything here since it's mostly covered in [Basics of using OpenSSL](../security/basics-of-using-openssl.md).

Suffice to say you'll need a Root CA Cert and key (called `cacert.pem` and `cakey.pem`) outlined in [Making your own CA](../security/basics-of-using-openssl.md#Making_your_own_CA).

You'll need a server cert and key as well (see [Making a new Certificate](../security/basics-of-using-openssl.md#Making_a_new_Certificate) which will be like `myserver.crt` and `myserver.key`.

You can follow these steps to make client and server keys:

```shell
openssl req -nodes -new -keyout client1.key -out client1.csr
openssl ca -out client1.crt -in client1.csr -policy policy_anything
```

You'll want to make sure that you replace client1 with something that makes sense also make sure that you make a unique common name (CN) for each client key that you create.

Likewise for the examples below you'll want to name the server keys:

```shell
server.key, server.csr, and server.crt
```

Last you'll need to generate Diffie Hellman parameters.

## Configuring OpenVPN

You can find sample config files in:

```shell
/usr/local/share/doc/openvpn/sample-config-files/
```

We'll start with server.conf. Copy it to

```shell
/usr/local/etc/openvpn/
```

### Creating your Server openvpn.conf file

Here's an example of a full working openvpn.conf. If you read the conf that comes with openvpn it has lots of great explanations for what everything does:

```conf
port 1194
proto udp
dev tap
mode server

ca keys/cacert.pem
cert keys/foo.example.com.crt
key keys/foo.example.com.key
dh keys/dh2048.pem
# Don't put this in the keys directory unless user nobody can read it
crl-verify keys/crl.pem

ifconfig-pool-persist /tmp/ipp.txt
server-bridge 10.0.0.1 255.255.255.0 10.0.0.100 10.0.0.200
push "redirect-gateway local def1"

client-config-dir /usr/local/etc/openvpn/bridge-clients
client-to-client
keepalive 10 120

cipher BF-CBC        # Blowfish (default)
comp-lzo
max-clients 100
user nobody
group nobody

persist-key
persist-tun
verb 3
```

In the above configuration the default route will automatically get pushed to the client, so there's no need to worry about routes. One thing you should also note. In the config

```shell
client-config-dir /usr/local/etc/openvpn/bridge-clients
```

This allows you to explicitly assign addresses to clients based on the CN of their key. So in this example:

```shell
# ls /usr/local/etc/openvpn/bridge-clients
client.example.com
# cat /usr/local/etc/openvpn/bridge-clients/client.example.com
ifconfig-push 10.0.0.5 255.255.255.0
```

We see that once they connect to OpenVPN, their tap device will get assigned 10.0.0.5 to be used on the other network.
A sample client.conf (for FreeBSD, Linux or OS X)

On the client side you'll need the client keys (see above) and the conf. Like the server I store the configs in /usr/local/etc/openvpn and the keys in /usr/local/etc/openvpn/keys. For OS X you can use [tunnelblick](http://tunnelblick.net/) with the configuration below, likewise for Linux and FreeBSD.

```conf
#Begin client.conf
client
dev tap
proto udp
remote 10.0.1.1 1194
resolv-retry infinite
nobind
user nobody
group nobody
persist-key
persist-tun
mute-replay-warnings
ca keys/cacert.pem
cert keys/client.crt
key keys/client.key
cipher BF-CBC
comp-lzo
verb 3
mute 20
```

We can connect to the server by starting this up like the server conf:

```shell
cd /usr/local/etc/openvpn && openvpn client.conf
```

Getting a client working in Windows

Install the openvpn-gui from http://openvpn.se/download.html

Click on Start->OpenVPN->OpenVPN Sample Configuration Files Edit client.opvn

Change the following, make sure the client1 keys reflect the names of the keys you created for this workstation.

```conf
remote 1.2.3.4 1194

ca cacert.pem
cert client1.crt
key client1.key

cipher blowfish
```

Once finished click on File-> Save as... And save it in Program Files/OpenVPN/config name the file `client1.ovpn`

Take the certs you created above and make sure they're in Program Files/OpenVPN/config

Right click on the Openvpn icon at the bottom (it should look like a pair of computer with red screens), and select connect and you should be connected to your network!
Revoking a cetificate

This is how you'll be making your access list. You'll need to make sure that crl-verify is enabled in your server.conf.

To revoke a cert perform these steps:

```shell
openssl ca -revoke client.crt
openssl ca -gencrl -out /etc/ssl/CA/crl/crl.pem
cp /etc/ssl/CA/crl/crl.pem /usr/local/etc/openvpn
```

If you're not running openvpn jailed you can also just create a symlink instead of copying each time.
A note on OS X and Bonjour

To get bonjour/rendezvous/zeroconf working over your bridge you'll need to make some firewall rules to pass bonjour multicast packets. If you have other multicast packets to pass, you can find their multicast addresses here: http://www.iana.org/assignments/multicast-addresses

Using pf (with the example interfaces above) here's what I did to do so:

```conf
pass in quick on if1 dup-to if2 inet proto udp from any to 224.0.0.251 port = 5353
pass in quick on if2 dup-to if1 inet proto udp from any to 224.0.0.251 port = 5353
```

Note that these multicast packets will pass into the private network unencrypted which means any hackers sitting there wondering why they can't route out or do anything, and using tcpdump or other like program will see these broadcasts. If this is a problem you probably shouldn't use it.

## Notes/Caveats

Stress tests on a Soekris 4801 show that you'll likely need a faster server if you plan on serving more than one client. I've found that with 2 clients with a high network load can cause high packet loss. If you want a similar solution using IPSec, follow these instructions.

## References

* Bridged Fbsd Reference: http://www.sigsegv.cx/#FreeBSD-WIN2K-VPN-HOWTO-New-5.html  
* OpenVPN Bridged Howto: http://openvpn.sourceforge.net/bridge.html  
