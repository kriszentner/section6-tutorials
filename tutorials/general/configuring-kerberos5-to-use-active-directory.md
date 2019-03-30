# Configuring Kerberos5 to use Active Directory

This article compliments the Configuring Samba3 to be a Windows Domain Member article. This page decribes how to configure Kerberos on a Unix computer to talk to an Windows 2000 or 2003 ActiveDirectory Server.
Required Software

On Debian Linux Systems, these packages are called:

* `libkrb5`
* `krb5-user`
* `krb5-config`

On FreeBSD Systems, the required packages are called:

* `/usr/ports/net/krb5`

If you are running another Unix distribution and are not sure how to obtain these files, please consult the documentation of your distribution to learn more about its package management system.

## Configuring Kerberos

We may need to create directories for the log files. Also note that Kerberos useage is case sensitive, thus the sections which are in uppercase have to be in uppercase or we will experience problems.

Here is an example Kerberos 5 config file from Debian GNU/Linux:

/etc/krb5.conf

```toml
[logging]
   default = FILE:/var/log/krb5/libs.log
   kdc = FILE:/var/log/krb5/kdc.log
   admin_server = FILE:/var/log/krb5/admin.log
[libdefaults]
   ticket_lifetime = 24000
   default_realm = TEST.ORG
   default_tgs_enctypes = des-cbc-crc des-cbc-md5
   default_tkt_enctypes = des-cbc-crc des-cbc-md5
   forwardable = true
   proxiable = true
   dns_lookup_realm = true
   dns_lookup_kdc = true
[realms]
   TEST.ORG = {
     kdc = test.test.org:88
     default_domain = test.org
   }
[domain_realm]
   .test.org = TEST.ORG
   test.org = TEST.ORG
[kdc]
   profile = /var/kerberos/krb5kdc/kdc.conf
[pam]
   debug = false
   ticket_lifetime = 36000
   renew_lifetime = 36000
   forwardable = true
   krb4_convert = false
```

The second part of setting up the kerberos section is to make sure that kerberos is defined in our /etc/services file. It should contain a line along the following.

```conf
kerberos    88/tcp    kdc kerberos5 krb5  # Kerberos v5
kerberos    88/udp    kdc kerberos5 krb5  # Kerberos v5
```

Most modern ditributions ship with these entries in the `/etc/services` file so this really should not be an issue, but it never hurts to double check.
Testing the Kerberos Configuration

You can use kinit to test your kerberos setup by issuing a ticket from the KDC.

```shell
root@host# kinit Administrator@TEST.ORG
```

This will prompt you for a password and return success if it succeeds. If you get an error `KDC has no support for encryption type`, you need to re-set the password for the windows user "Administrator" (as in this example). Just reset your password on a Windows client using the "Active Directory Users and Computers" tool to your original password.
