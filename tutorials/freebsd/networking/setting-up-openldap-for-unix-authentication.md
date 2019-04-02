# Setting up OpenLDAP for UNIX Authentication

## Server Tasks

Edit slapd.conf and make sure you have the nis and cosine schema (others are optional):

```conf
#X.500 RFC1274 COSINE Pilot Schema
include         /usr/local/etc/openldap/schema/cosine.schema
#For Addressbooks
include         /usr/local/etc/openldap/schema/inetorgperson.schema
#For Authentication
include         /usr/local/etc/openldap/schema/nis.schema
```

Now edit the BDB database specifications:

```conf
suffix          "dc=example,dc=com"
rootdn          "cn=root,dc=example,dc=com"
# Cleartext passwords, especially for the rootdn, should
# be avoid.  See slappasswd(8) and slapd.conf(5) for details.
# Use of strong authentication encouraged.
# Use 'slappasswd -h {MD5} -s password' to generate
rootpw          {MD5}X03MO1qnZdYdqyfeuILPmQ==
```

### Setting up SSL for openldap

Create an ssl directory in `/usr/local/etc/openldap`

```shell
# mkdir /usr/local/etc/openldap/ssl
# cd /usr/local/etc/openldap/ssl
openssl req -newkey rsa:1024 -nodes -keyout ldapkey.pem -keyform PEM -out ldapreq.pem -outform PEM
```

This will generate certs based on the ca set in `/etc/ssl/openssl.cnf`

```shell
# openssl ca -in ../ssl_serverkeys/ldapreq.pem
```

Now copy the cert from your newcerts dir (specified in /etc/ssl/openssl.cnf) to the openldap area. You may need to cd to this directory and see what the name of the last cert was:

```shell
# cp -rp /etc/ssl/CA/newcerts/05.pem /usr/local/etc/openldap/ssl/ldap.example.com.pem
```

Now set the following in `/usr/local/etc/openldap/slapd.conf`

```conf
TLSCACertificateFile /etc/ssl/cacert.pem
TLSCertificateFile /etc/openldap/ssl/ldap.example.com.pem
TLSCertificateKeyFile /etc/openldap/ssl/private/ldapkey.pem
TLSCipherSuite HIGH
```

This should be all that's necessary to get ssl working! Keep in mind clients will need the cacert.pem file as well.
Setting ldap.conf

Edit `/usr/local/etc/openssl/ldap.conf`, if you are going to use nss_ldap on the openldap server, this is also /usr/local/etc/nss_ldap.conf, otherwise exclude the nss_base* stuff:

```conf
uri ldap://ldap.example.com/
base "dc=example,dc=com"
port 389
binddn cn=pamclient,ou=SystemAccounts,dc=example,dc=com
bindpw clientpassword
scope sub
pam_password exop
## The immediately following command is optional. Only if you want SSL.
ssl start_tls
TLS_CACERT /etc/ssl/cacert.pem
nss_base_passwd ou=accounts,dc=example,dc=com?one
nss_base_group ou=groups,dc=example,dc=com?one
```

### Adding your initial entry

You 'll need to add an inital entry to your directory. The simplest way to do this is using ldapadd.

ldapadd only takes LDIF files so lets make one called `initial.ldif`:

```conf
dn: dc=example,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: Example Solutions
dc: example

# super user node
dn: cn=root,dc=example,dc=com
objectclass: organizationalRole
objectclass: simpleSecurityObject
cn: root
description: LDAP administrator
userPassword: {CRYPT}cmc1S152QEw/U
```

Now lets throw it into the ldap directory:

```shell
ldapadd -x -D "cn=root,dc=example,dc=com" -W -f initial.ldif
```

Now lets use phpldapadmin to edit entries, this will make things massively easier:

install it and edit the config so that:

```conf
$servers[$i]['name'] = 'Example LDAP';
$servers[$i]['host'] = '127.0.0.1';
$servers[$i]['base'] = 'dc=example,dc=com';
$servers[$i]['auth_type'] = 'config';
$servers[$i]['login_dn'] = 'root';
$servers[$i]['login_pass'] = 'secret';
```

reflects the account you made above, we'll change this later

add the attribute `simpleSecurityObject` for root so that it has a password

create the ou accounts create your first user with posixAccount keep in mind that the default:

```conf
shadowExpire = -1
```

renders the account invalid until the next password change. To make the password never expire, set to 0.

Also create the pamclient account described in `ldap.conf`

then change the config back so that:

```conf
$blowfish_secret = 'somelongrandomstringthatshardtoguess';
$servers[$i]['auth_type'] = 'session';
$servers[$i]['login_dn'] = ;
$servers[$i]['login_pass'] = ;
$servers[$i]['login_attr'] = 'uid';
$servers[$i]['login_string'] = 'uid=<username>,ou=accounts,dc=example,dc=com';
$servers[$i]['default_hash'] = 'blowfish';
$servers[$i]['login_class'] = 'posixAccount';
$servers[$i]['auto_uid_number_search_base'] = 'ou=accounts,dc=example,dc=com';
```

### Broadcasting LDAP servers via DHCP

This option is supposed to work on OS X though I haven't really seen it do so. Still, if you want to broadcast ldap here's how... In your dhcpd.conf add these options:

```conf
option ldap-server code 095 = ip-address;
option ldap-server ldap.example.com;
```

When you're done messing with the server

After you're done migrating all the services and you're sure everything is in good working order, you may want to set the syslog detail down a bit from the standard "debug". You can set this in `/etc/rc.conf` under

```shell
slapd_flags=""
```

Your choices are in order:

```shell
 emerg, alert, crit, err, warning, notice, info, and debug.
```

For example:

```shell
slapd_flags='-s notice -h "ldapi://%2fvar%2frun%2fopenldap%2fldapi/ ldap://0.0.0.0/ ldaps://0.0.0.0/"'
```

## Client Tasks
### On FreeBSD

Install

```shell
/usr/ports/security/pam_ldap
/usr/ports/net/nss_ldap
```

Migrating the passwd file

Download the migration tools:

```shell
fetch http://www.padl.com/download/MigrationTools.tgz
```

and extract them:

```shell
tar -zxvf MigrationTools.tgz
cd MigrationTools-46
./migrate_passwd /etc/master.passwd accounts.ldif
```

From here you'll have a rough ldif file of your users however it makes some assumptions. First that your users are all in ou=People,dc=padl,dc.com and it also seems to have trouble migrating the home directory. The upside is that all your passwords get migrated so hopefully a few tricky edits

You'll have this entry:

```conf
dn: uid=test,ou=People,dc=padl,dc=com
uid: test
cn: test
objectClass: account
objectClass: posixAccount
objectClass: top
userPassword: {crypt}$1$QA4igP/F$arqauwas193iraeunawj/
uidNumber: 1001
gidNumber: 1001
homeDirectory:
```

And you'll want to turn it into something like this:

```conf
dn: uid=test,ou=accounts,dc=example,dc=com
uid: test
givenName: Test
sn: User
cn: Test User
userPassword:: {crypt}$1$QA4igP/F$arqauwas193iraeunawj/
uidNumber: 1001
gidNumber: 1001
shadowMin: 0
shadowMax: 999999
shadowWarning: 7
shadowInactive: -1
shadowFlag: 0
objectClass: top
objectClass: person
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
shadowExpire: 0
homeDirectory: /home/test
loginShell: /bin/csh
```

Once you're done you can import this ldif like your initial one:

```shell
# ldapadd -x -D "cn=root,dc=example,dc=com" -W -f accounts.ldif
```

### Initial File Modifications

Edit /etc/nsswitch.conf

```conf
group: files ldap
group_compat:
hosts: files dns
networks: files
passwd: files ldap
passwd_compat:
shells: files
```

Edit /usr/local/etc/openldap/ldap.conf

```conf
uri ldap://ldap.example.com/
base "dc=example,dc=com"
port 389
binddn cn=pamclient,ou=SystemAccounts,dc=example,dc=com
bindpw clientpassword
scope sub
pam_password exop
## The immediately following command is optional. Only if you want SSL.
ssl start_tls
TLS_CACERT /etc/ssl/cacert.pem
nss_base_passwd ou=accounts,dc=example,dc=com?one
nss_base_group ou=groups,dc=example,dc=com?one
```

Make sure that the `/etc/ssl/cacert.pem` from the server exists where you specify it above. You'll also want to symlink this to `nss_ldap.conf`.

```shell
ln -s /usr/local/etc/openldap/ldap.conf /usr/local/etc/nss_ldap.conf
```

alternatively you can set:

```conf
tls_checkpeer no
```

if you don't want to bother with the certificate.
Modify files in `/etc/pam.d`

First edit /etc/pam.d/ldap

```conf
login   auth    sufficient      /usr/local/lib/pam_ldap.so
```

In `/etc/pam.d` you must add the following to any services, sshd for example (represented by the files) that you wish ldap to use. Add the following line before the #auth section:

```conf
# auth
auth            required        pam_nologin.so          no_warn
auth            sufficient      pam_opie.so             no_warn no_fake_prompts
auth            requisite       pam_opieaccess.so       no_warn allow_local
#auth           sufficient      pam_krb5.so             no_warn try_first_pass
#auth           sufficient      pam_ssh.so              no_warn try_first_pass
auth           sufficient      /usr/local/lib/pam_ldap.so no_warn try_first_pass
auth            required        pam_unix.so             no_warn try_first_pass
```

## Getting passwd to change the password in LDAP

Here's /etc/pam.d/passwd

```conf
password  sufficient        pam_unix.so             no_warn try_first_pass nullok
password  sufficient  /usr/local/lib/pam_ldap.so  use_first_pass
```

Edit /usr/src/usr.bin/passwd/passwd.c

Remove this part:

```c
      errx(1,
"Sorry, `passwd' can only change passwords for local or NIS users.");
```

And replace it with this:

```c
/* XXX: Green men ought to be supported via PAM. */
         fprintf(stderr, "Now you can change LDAP passwordi via PAM\n");
```

then make your you're in the /usr/src/usr.bin/passwd/ dir and:

```shell
make install
```

## Integrating with saslauthd

If you use saslauthd (with imap for example) You'll need to do the following:

Compile cyrus-sasl-saslauthd with WITH_OPENLDAP=yes

then, you'll need to make a file: `/usr/local/etc/saslauthd.conf` to use your pamclient password you defined above:

```conf
ldap_servers: ldap://ldap.example.com=/
ldap_bind_dn: cn=pamclient,ou=SystemAccounts,dc=example,dc=com
ldap_bind_pw: clientpassword
ldap_version: 3
ldap_search_base: dc=example, dc=com
ldap_verbose: on
ldap_debug: 3
```

A regular sasl config file in `/usr/local/lib/sasl2/smtpd.conf` should work ok:

```conf
pwcheck_method: saslauthd
mechlist:plain login crammd5 digestmd5
```

Afterwards you can test it:

```shell
# testsaslauthd -u test -p secretpassword
0: OK "Success."
```

## On OS X

Open up Utilities -> Directory Access

* Click on the lock if it's locked and get admin access
* Check off and select LDAPv3
* Click "Configure"
* Click "New" and add the name of the ldap server
  * If your LDAP server is SSL, check off encrypt
  * Check "Use for authentication"
  * Check "Use for contacts"
* Click on the "Search & Mappings" button at the top.
* On "Access this LDAPv3 server using...Select "RFC 2307 (Unix)"
* Make Click on "Users" on the right, and set your search base dc=example,dc=com (or your search base)
* Click on Security and check "Use authentication when connecting"
* Set your Distinguished Name to

```conf
cn=pamclient,ou=SystemAccounts,dc=example,dc=com" 
```

and the relevant password.

* You may want to set "Disable clear text passwords" if it's not already set.
* Check off clear text passwords
* Click on OK (you may need to restart)

## References

### Openldap Hints

* http://home.frognet.net/~aalug/docs/ldap.pdf
* http://www.flatmtn.com/computer/Linux-HordeImp.html
* http://linsec.ca/ldap_address_book.php
* http://www.cultdeadsheep.org/FreeBSD/docs/Quick_and_dirty_FreeBSD_5_x_and_nss_ldap_mini-HOWTO.html
* http://books.blurgle.ca/read/chapter/4
* http://www.auug.org.au/saauug/events/2005/meetings/ldap/ssl.html (more on ldapssl)

### Auth

* http://www.saas.nsw.edu.au/solutions/ldap-auth-pam.html

### ACL

* http://www.yolinux.com/TUTORIALS/LinuxTutorialLDAP-BindPW.html

### LDIF

* http://webhelp.ucs.ed.ac.uk/direct/ldifspec.htm