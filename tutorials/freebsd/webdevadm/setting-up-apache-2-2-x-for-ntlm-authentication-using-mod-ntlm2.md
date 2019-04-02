 # Setting up Apache 2.2.x for NTLM Authentication using Mod NTLM2

First off, this is just an extension of information found at the [Blue Smoke Band](http://www.thebluesmokeband.com/mod_ntlm.php).

We've all been there before: your PHB wants you to develop your web application using .NET because some salesperson told him too, and you don't want to. So, what do you do? You develop it in PHP and the PHB never knows because it integrates seamlessly with your Winders network.

We're going to set up your Apache Server running on your FreeBSD box to use NTLM Authentication to prompt your NTLM aware browsers (Firefox, IE, etc.) for your user's login credentials so they don't have to see that pesky login dialog.

### The Setup

* You're running FreeBSD 6.0 RELEASE or later.
* You have Apache 2.2.x on your server (we are using 2.2.4 with prefork MPM).
* You have PHP 5 installed (we have 5.2.1_3).
* You have downloaded the source for mod_ntlm2 from Sourceforge.

Once you have downloaded the .tgz package, you want to extract it to whatever directory you would like to keep it in. I put mine in /usr/home/dan/downloads/.

```shell
#tar -xzvf mod_ntlm2-0.1.tgz
```

You now have a new directory with all of your build files called mod_ntlm2-0.1.

```shell
#cd mod_ntlm2-0.1
```

You should see the following.

```shell
#ls
ChangeLog       Makefile        TODO            mod_ntlm.h      smbval
Documentation   README          mod_ntlm.c      ntlmssp.inc.c
```

If you try to build the module now, you get:

```shell
# make
apxs -c -o mod_ntlm.so -Wc,-shared mod_ntlm.c
/usr/local/build-1/libtool --silent --mode=compile cc -prefer-pic -O2 -fno-strict-aliasing -pipe    
-I/usr/local/include/apache22  -I/usr/local/include/apr-1   -I/usr/local/include/apr-1 -I/usr/local/include -shared  -c -o
mod_ntlm.lo mod_ntlm.c && touch mod_ntlm.slo
mod_ntlm.c:44: warning: conflicting types for built-in function 'log'
In file included from smbval/rfcnb-util.inc.c:24,
                from mod_ntlm.c:102:
/usr/include/malloc.h:3:2: #error "<malloc.h> has been replaced by <stdlib.h>"
In file included from smbval/session.inc.c:24,
                from mod_ntlm.c:103:
/usr/include/malloc.h:3:2: #error "<malloc.h> has been replaced by <stdlib.h>"
In file included from mod_ntlm.c:105:
smbval/smbencrypt.inc.c:22:21: sys/vfs.h: No such file or directory
In file included from smbval/smblib-util.inc.c:24,
                from mod_ntlm.c:106:
/usr/include/malloc.h:3:2: #error "<malloc.h> has been replaced by <stdlib.h>"
In file included from smbval/smblib.inc.c:23,
                from mod_ntlm.c:107:
/usr/include/malloc.h:3:2: #error "<malloc.h> has been replaced by <stdlib.h>"
apxs:Error: Command failed with rc=65536
.
*** Error code 1
Stop in /usr/home/dan/downloads/mod_ntlm2-0.1.
```

What we need to do now is change some of the C files in the smbval directory, specifically the files mentioned in your error output: session.inc.c, smblib.inc.c, smbencrypt.inc.c, rfcnb-util.inc.c, smblib-util.inc.c.

```shell
#cd smbval
```

Now, if you're good with vi or have more mojo than me, all you need to do is search and replace the following line within session.inc.c, smblib.inc.c, rfcnb-util.inc.c, and smblib-util.inc.c.

```c
#include <malloc.h>
```

With this line:

```c
#include <stdlib.h>
```

And within smbencrypt.inc.c, replace the following line:

```c
#include <sys/vfs.h>
```

With the following lines (this worked for me with some research, I am not a C developer so I don't know if you need both):

```c
#include <sys/param.h>
#include <sys/mount.h>
```

So, we've made all of the correct changes, right? Well, it will compile, but if you try to load the module into Apache you will get an error, specifically an unidentified symbol error. We need to change one more line of code to get this sucker ready, in mod_ntlm.c:

```c
apr_pool_sub_make(&sp,p,NULL);
```
Change this line to:

```c
apr_pool_create_ex(&sp,p,NULL,NULL);
```

We still have one last step. If we try to build and install now, we get:

```shell
# make install
apxs -c -o mod_ntlm.so -Wc,-shared mod_ntlm.c
/usr/local/build-1/libtool --silent --mode=compile cc -prefer-pic -O2 -fno-strict-aliasing -pipe    -I/usr/local/include/apache22   
-I/usr/local/include/apr-1   -I/usr/local/include/apr-1 -I/usr/local/include -shared  -c -o mod_ntlm.lo mod_ntlm.c && touch mod_ntlm.slo
mod_ntlm.c:44: warning: conflicting types for built-in function 'log'
/usr/local/build-1/libtool --silent --mode=link cc -o mod_ntlm.la  -rpath /usr/local/libexec/apache22 -module -avoid-version    mod_ntlm.lo
apxs -i -a -n 'ntlm' mod_ntlm.so
/usr/local/share/apache22/build/instdso.sh SH_LIBTOOL='/usr/local/build-1/libtool' mod_ntlm.so /usr/local/libexec/apache22
/usr/local/build-1/libtool --mode=install cp mod_ntlm.so /usr/local/libexec/apache22/
cp mod_ntlm.so /usr/local/libexec/apache22/mod_ntlm.so
cp: mod_ntlm.so: No such file or directory
apxs:Error: Command failed with rc=65536
.
*** Error code 1
Stop in /usr/home/dan/downloads/mod_ntlm2-0.1.
```

The key is some simple edits in the Makefile. Change the following line:

```shell
APACHECTL=/etc/rc.d/apache
```

To:

```shell
APACHECTL=/usr/local/sbin/apachectl
```

And change:

```conf
install: all
       $(APXS) -i -a -n 'ntlm' mod_ntlm.so
```

To:

```conf
install: all
       $(APXS) -i -a -n 'ntlm' .libs/mod_ntlm.so
```
Okay, Now when we do a make install:

```shell
# make install
apxs -c -o mod_ntlm.so -Wc,-shared mod_ntlm.c
/usr/local/build-1/libtool --silent --mode=compile cc -prefer-pic -O2 -fno-strict-aliasing -pipe    -I/usr/local/include/apache22  
-I/usr/local/include/apr-1   -I/usr/local/include/apr-1 -I/usr/local/include -shared  -c -o mod_ntlm.lo mod_ntlm.c && touch mod_ntlm.slo
mod_ntlm.c:44: warning: conflicting types for built-in function 'log'
/usr/local/build-1/libtool --silent --mode=link cc -o mod_ntlm.la  -rpath /usr/local/libexec/apache22 -module -avoid-version    mod_ntlm.lo
apxs -i -a -n 'ntlm' .libs/mod_ntlm.so
/usr/local/share/apache22/build/instdso.sh SH_LIBTOOL='/usr/local/build-1/libtool' .libs/mod_ntlm.so /usr/local/libexec/apache22
/usr/local/build-1/libtool --mode=install cp .libs/mod_ntlm.so /usr/local/libexec/apache22/
cp .libs/mod_ntlm.so /usr/local/libexec/apache22/mod_ntlm.so
Warning!  dlname not found in /usr/local/libexec/apache22/mod_ntlm.so.
Assuming installing a .so rather than a libtool archive.
chmod 755 /usr/local/libexec/apache22/mod_ntlm.so
[activating module `ntlm' in /usr/local/etc/apache22/httpd.conf]
```

Success! I hope. There still is some configuration left to get your module working. Double check your http.conf and make sure the module was indeed activated. Look for the following line in your .conf:

```conf
LoadModule ntlm_module        libexec/apache22/mod_ntlm.so
```

Since we're already in the `httpd.conf` file, let's configure for NTLM. I grabbed the following example from the aforementioned Blue Smoke Band:

```html
<Location / >
    AuthType NTLM
    NTLMAuth on
    NTLMAuthoritative on
    NTLMDomain <name of domain here>
    NTLMServer <hostname>
    NTLMBackup <hostname>
    Require valid-user
</Location>
```

Obviously, this configuration sets NTLM authentication for the whole site. You can specify specific directories, if you so choose, and also From Sorrell the following advice: make sure you use the hostnames only of your Domain Controllers, not the Fully Qualified Domain Name. If your server can't find them with DNS add them to your local hosts file.

Now, restart your apache server.

```shell
#apachectl restart
```

If you get no error messages, you should be okay. Sorrell at the Blue Smoke Band provided the following test page which I used as well, but of course you can create whatever you would like:

```html
<html>
<head>
<style type="text/css">
body { background-color: #269; }
p { 
	font-size: 18px;
	font-weight: bold;
	color: #cc0;
}
#thedivision {
	width: 50%;
	left: 25%;
	top: 25%;
	height: 50%;
	background-color: #036;
	position: absolute;
	text-align: center;
	padding: 3% 3% 3% 3%;
}
</style>

</head>

<body>

<div id="thedivision">
<p>
Welcome to the official <i>test</i> PHP page.

<p>
If you see this page, and we're assuming you do if you're
reading this sentence, this implies that PHP is set up
correctly. 

<p>
Moreover, you appear to be logged in as:
<br>
<?php
echo $_SERVER['REMOTE_USER'];
?>

</div>

</body>

</html>
```

Use this baby and make sure when you enter your credentials that your Windows Login Name is displayed. If not, you might have other problems...

That should be it! You have now successfully set up NTLM authentication so your Windows users can sign on to your secure intranet site without a pesky login prompt. Good luck and thanks again to Sorrell at Blue Smoke Band for helping guide my way.

Danimal