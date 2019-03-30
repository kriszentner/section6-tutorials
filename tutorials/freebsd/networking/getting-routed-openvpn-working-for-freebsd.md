
# Getting Routed OpenVPN working for FreeBSD

Author: kz

First you'll have to choose what configuation you want to run, bridging or routing. The differences are [here http://openvpn.sourceforge.net/faq.html#bridge2]. This tutorial will cover setting up routed since it's the easiest to configure and most scalable.

Note that if you choose routing, all the machines on the vpn you want to connect to *must* have a route back to the vpn server, if if your VPN tunnels are using 10.0.0.0/24 and your vpn internal interface is 172.20.10.3, you'll need to route add 10.0.0.0/24 172.20.10.3

If all your machines have a default route to one machine, and you're running openvpn on that machine you have nothing to worry about.

## Getting keys set up

You can try using the tools in /usr/local/share/doc/openvpn/easy-rsa but they don't seem to work that well. Here's a way that does.

First step is to get you SSL keys generated. I'm not going to outline everything here since it's mostly covered in Basics of using OpenSSL.

Suffice to say you'll need a Root CA Cert and key (called cacert.pem and cakey.pem) outlined in Making your own CA.

You'll need a server cert and key as well (see Making a new Certificate which will be like myserver.crt and myserver.key.

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

## Creating your Server server.conf file

You'll likely want to modify "local" to your outside IP address, in this example we'll use the following configuration:

```conf
external ip: 1.2.3.4
internal ip: 10.0.0.1
internal network: 10.0.0.0/24
VPN pool: 10.0.1.0/24
DNS server: 10.0.0.2
```

Here's an example of a full working server.conf. If you read the conf that comes with openvpn it has lots of great explanations for what everything does:

```conf
#Begin server.conf
local 1.2.3.4
port 1194
proto udp
dev tun

ca keys/cacert.pem
cert keys/server.crt
key keys/server.key # This file should be kept secret
dh keys/dh1024.pem
# Don't put this in the keys directory unless user nobody can read it
crl-verify crl.pem

#Make sure this is your tunnel address pool
server 10.0.1.0 255.255.255.0
ifconfig-pool-persist ipp.txt
#This is the route to push to the client, add more if necessary
push "route 10.0.0.0 255.255.255.0"
push "dhcp-option DNS 10.0.0.2"
keepalive 10 120
cipher BF-CBC #Blowfish encryption
comp-lzo
user nobody
group nobody
persist-key
persist-tun
status openvpn-status.log
verb 6
mute 20
```

On any servers themselves you want to add routes to the openvpn server if that is not their default route:

```shell
Linux: route add -net 10.0.1.0/24 gw 10.0.0.1
FreeBSD: route add -net 10.0.1.0/24 10.0.0.1
```

Where `10.0.0.1` is the internal IP of the vpn server on your intranet.

To start it up we use:

```shell
cd /usr/local/etc/openvpn && openvpn server.conf
```

## A sample client.conf (for FreeBSD or Linux)

On the client side you'll need the client keys (see above) and the conf. Like the server I store the configs in /usr/local/etc/openvpn and the keys in /usr/local/etc/openvpn/keys. Below is a sample conf that'll work with the above server conf.

```conf
#Begin client.conf
client
dev tun
proto udp
remote 1.2.3.4 1194
nobind
user nobody
group nobody
persist-key
persist-tun
ca keys/cacert.pem
cert keys/client1.crt
key keys/client1.key
cipher BF-CBC 
comp-lzo
verb 3
mute 20
```
We can connect to the server by starting this up like the server conf:

```shell
cd /usr/local/etc/openvpn && openvpn client.conf
```

## Getting a client working in Windows

Install the openvpn-gui from http://openvpn.se/download.html

Click on Start->OpenVPN->OpenVPN Sample Configuration Files Edit client.opvn

Change the following, make sure the `client1` keys reflect the names of the keys you created for this workstation.

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

## Revoking a certificate

This is how you'll be making your access list. You'll need to make sure that crl-verify is enabled in your server.conf.

To revoke a cert perform these steps:

```shell
openssl ca -revoke client.crt
openssl ca -gencrl -out /etc/ssl/CA/crl/crl.pem
cp /etc/ssl/CA/crl/crl.pem /usr/local/etc/openvpn
```

If you're not running openvpn jailed you can also just create a symlink instead of copying each time.

## References

* Bridged Fbsd Reference: http://www.sigsegv.cx/FreeBSD-WIN2K-VPN-HOWTO-New-5.html
* OpenVPN Bridged Howto: http://openvpn.sourceforge.net/bridge.html
