# Basics of using OpenSSL
## OpenSSL and the Certificate Dance in FreeBSD

Keep in mind this is just one way to do this, there are a million others with different variables, order, and even using a perl script. This is just one of many that works.


## Preparing to do SSL stuff

First, edit /etc/ssl/openssl.cnf. Edit "dir = ./demoCA" to where you're going to store your certs and keys. I set it to /etc/ssl/CA (remember to create this dir after you're done).

Other vars that I set:

```conf
default_days = 3650
default_bits = 2048
countryName_default = US
stateOrProvinceName_default = Washington
localityName_default = Seattle
0.organizationName_default = My Company
commonName_default = example.com
emailAddress_default = root@example.com
```

If you're stuck on default bits, a good place to read is here: http://www.schneier.com/crypto-gram-0204.html

Also make sure that

```conf
certificate     = $dir/cacert.pem
```

or you'll get errors

if you have another openssl installed you'll want to make sure you're on the same page no matter which one you're using:

```shell
rm /usr/local/etc/openssl.cnf
ln -s /etc/ssl/openssl.cnf /usr/local/etc/openssl.cnf
```

Caveat: Not using -nodes will make your key secure if someone breaks into your box and steals your private keys, but you'll need to worry about typing a password all the time.

## Making your own CA

This is necessary if you're not going to use someone like Verisign to sign your key for you.

```shell
mkdir /etc/ssl/CA
cd /etc/ssl/CA
mkdir certs
mkdir crls
mkdir newcerts
mkdir private
touch index.txt
echo 01 > serial
openssl req -nodes -new -x509 -keyout private/cakey.pem -out cacert.pem -days 3650
```

This makes a certificate authority certificate in cacsr.pem and a certificatekey in private/cakey.pem.


## Create a Certificate Revocation List

```shell
openssl ca -gencrl -out crls/crl.pem
```

## To View the CSR info

```shell 
openssl req -noout -text -in cert.csr
```

## Making a new Certificate

Here's how to generate public and private certs, the optional company name is the server name ie private.example.com.

```shell
cd certs
openssl req -nodes -new -keyout www.example.com.key -out www.example.com.csr
```

Then you'll want to sign this certificate (This is what verisign does too)

```shell
openssl ca -out www.section6.net.crt -in www.section6.net.csr -policy policy_anything
```

So now you have 3 files designated by their common name:
```conf
www.example.com.crt    # The Signed server certificate
www.example.com.key    # The private key for the server cert these files
                       # are what a hacker would want so keep them secure.
www.example.com.csr    # Encrypted private key for the server cert,
                       # and the cert request.
```
## Installing Certs in Apache

This is here because this is a fairly common example everyone uses, but it can be used for other apps too.

Copy www.example.com.key to /usr/local/etc/apache/ssl.key
and copy www.example.com.crt to /usr/local/etc/apache/ssl.key


## Revoking a cetificate

For the paths below you'll want to look up where openssl.cnf says where your crl (cerficate revocation list) is stored. It should be under the "crl" and "crl_dir" derivatives state. On mine it's `$dir/crl` for `crl_dir` and `$dir/crl/pem` where dir is `/etc/ssl/CA`

You'll want to revoke a certificate if you find you no longer need it, it's been compromised, or in the case of openvpn you wish to deny a user access:

```shell
openssl ca -revoke certifcate.crt
openssl ca -gencrl -out /etc/ssl/CA/crl/crl.pem
```

To verify the cert has been revoked:

```shell
cat cacert.csr /etc/ssl/CA/crl/crl.pem > revoke.pem
openssl verify -CAfile revoke.pem -crl_check cerficate.crt
```

You can delete revoke.pem when you're done.
Generating Diffie Hellman parameters

This isn't needed for Apache, but is needed for other apps such as openvpn. Here's how:

You can use the script at:

```shell
/usr/local/share/doc/openvpn/easy-rsa/build-dh
```

or you can do it manually (recommended) with:

```shell
openssl dhparam -out dh1024.pem 1024
```

You can replace 1024 with whatever your preferred keysize is which should be defined in default bits in you openssl.cnf (see 1st section).

## References

* http://www.xs4all.nl/~dorus/linux/openssl.html