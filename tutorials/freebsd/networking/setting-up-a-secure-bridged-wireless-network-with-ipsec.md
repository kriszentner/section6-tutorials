# Setting up a Secure Bridged (Wireless) Network with IPSec

## Getting IPsec working for FreeBSD

This tutorial was created in persuing a secure wireless home network. It has some similarities to the OpenVPN setup, however you'll end up with two networks instead of one bridged one. It has been tested with a Soekris 4801 Router and OS X clients.
The Scenario

Here's the scenario. You have a network like the below

```asciiart
   { Internet }
         |
   [    Soekris    ]
    |           |
 10.0.1.1    10.0.0.1
    |           |
{Wireless}    {LAN}
    |
{laptop}
10.0.1.10
```

Now in this scenario, you have no choice but to encrypt your network with WEP or WPA, both of which are notoriously insecure. You can choose to block by mac address but people can still sniff your mac address and in some cases they can switch the mac of their wireless and join your network anyhow, and even if they can't there's a high likelyhood they can sniff your traffic capturing any websites you browse to, and maybe even passwords if they are unencrypted.

If this solution you have a private network for your wireless bridge, and a separate network for your other ethernet connected servers.

In the following example we'll set up this network with these assumptions (match them to your own): Our FreeBSD router has the following interfaces:

```conf
if0 at ip 1.2.3.4
if1 at ip 10.0.0.1/24
if2 at ip 10.0.1.1/24
```

On the router you must make sure you have the following in your kernel:

```conf
options IPSEC         #IP security
options IPSEC_ESP     #IP security (crypto; define w/ IPSEC)
```

or

```conf
options FAST_IPSEC
device crypto
```

Note that FAST_IPSEC is not known to work with IPv6, and that this setup was tested only with IPSEC and IPSEC_ESP, but it should work with either.

## Setting up Racoon

Racoon is an IKE key management daemon. You can opt not to use it, and just add static keys yourself in the ipsec rules, but racoon will make things much more secure as it switches keys during the session every so often.

To generate your secret key you can use something like:

```shell
jot -r -s "" -c 24 35 122
```

On the server: Put this in `/usr/local/etc/racoon/psk.txt`

```conf
10.0.1.10      "secret"
```

When finished

```shell
chmod 0600 /usr/local/etc/racoon/psk.txt
chown root:wheel /usr/local/etc/racoon/psk.txt
```

For an example of a working racoon.conf look [here](../networking/setting-up-a-secure-bridged-wireless-network-with-ipsec.md).

In some howtos you'll see bits about setting up IPsec rules. By setting

```conf
generate_policy on;
```

It'll create policies on the fly, which is an advantage if you have multiple clients, however you won't get the side side effect of blocking packets that aren't IPsec, so this is where pf (or a firewall of your choice) comes in...

## Setting Up PF

You'll want to set up NAT on your wireless vpn if you want packets to go out:

```conf
nat on $extif from $vpnif:network to any -> ($extif)
```

These rules should be sufficient in keeping people out while allowing ipsec, and racoon:

```conf
pass in quick on $vpnif inet proto udp from any to any port 500
pass in quick on $vpnif proto esp
pass out quick on $vpnif all
block in quick on $vpnif all
```

## Setting up an OS X Client

You can get a client working with this configuration using the [IPSecuritas](http://www.lobotomo.com/products/IPSecuritas) tool. Once you start racoon with the above configuration, set up IPSecuritas like so:

| General | |
|-----------|-|
| **Parameter** | **Setting** |
| Mode of Operation | Host to Anywhere |
| Remote IPSec Device | 10.0.0.1 |
| Exchange Mode	| Main, Aggressive |
| Proposal Check | Obey |
| Noce Size	| 16 |  

| Phase 1 | |
|---------|-|
| **Parameter** | **Setting** |
| Lifetime | 28800 |
| DH Group | Mod1024 (2) |
| Encryption | 3DES |
| Authentication | SHA1 |


| Phase 2 | |
|---------|-|
| **Parameter** | **Setting** |
| Lifetime | 28800|
| PFS Group | Mod1024 (2)
| Encryption | DES, 3DES, AES128, Blowfish
| Authentication | HMAC MD5, HMAC SHA1


| ID/Auth | |
|---------|-|
| **Parameter** | **Setting** |
| Local Identifier |Address |
| Remote Identifier | Address |
| Preshared Secret | secret |


| Options | |
|---------|-|
| **Parameter**	| **Setting** |
| IPSec/IKE Options | IPSec DOI, SIT_IDENTITY_ONLY, 
Initial Contact, Generate Policy |
|General Options | Establish IKE immediately |


## Notes/Caveats

Stress tests on a Soekris 4801 with 2 OS X clients shows IPSec working great with minimal load. With more clients it may be necessary to buy a card like this [one](http://www.soekris.com/vpn1401.htm) to offload some of the work the processor is doing. If you're using Bonjour you may need to do a few tricks to get the packets from one networking working on the other.

## References

* [Handbook entry on IPsec](http://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/ipsec.html%7CFreeBSD)
* [Securing your wireless with IPsec (single client)](http://www.onlamp.com/pub/a/bsd/2004/10/21/wifi_ipsec.html)

racoon.conf
```conf
path include "/usr/local/etc/racoon";
#include "remote.conf";

# the file should contain key ID/key pairs, for pre-shared key authentication.
path pre_shared_key "/usr/local/etc/racoon/psk.txt";

# "log" specifies logging level.  It is followed by either "notify", "debug"
# or "debug2".
#log debug;

# "padding" defines some padding parameters.  You should not touch these.
padding
{
       maximum_length 20;      # maximum padding length.
       randomize off;          # enable randomize length.
       strict_check off;       # enable strict check.
       exclusive_tail off;     # extract last one octet.
}

# if no listen directive is specified, racoon will listen on all
# available interface addresses.
listen
{
        isakmp 10.0.1.1;
}

# Specify various default timers.
timer
{
        # These value can be changed per remote node.
        counter 5;              # maximum trying count to send.
        interval 20 sec;        # maximum interval to resend.
        persend 1;              # the number of packets per send.

        # maximum time to wait for completing each phase.
        phase1 30 sec;
        phase2 15 sec;
}
remote anonymous
{
        exchange_mode main,aggressive;
        doi ipsec_doi;
        situation identity_only;

        #my_identifier asn1dn;
        my_identifier address;

        nonce_size 16;
        initial_contact on;
        proposal_check obey;    # obey, strict, or claim
        generate_policy on;

        proposal {
                encryption_algorithm 3des;
                hash_algorithm sha1;
                authentication_method pre_shared_key;
                dh_group 2;
        }
}

sainfo anonymous
{
        pfs_group 2;
        encryption_algorithm 3des, cast128, blowfish 448, des, rijndael ;
        authentication_algorithm hmac_sha1, hmac_md5;
        compression_algorithm deflate;
}
```