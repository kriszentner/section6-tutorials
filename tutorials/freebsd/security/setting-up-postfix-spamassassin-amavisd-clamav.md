# Setting up Postfix Spamassassin Amavisd Clamav
## Fighting Spam - Setting up Postfix+Amavisd+Spamassassin+Razor+Clamav
author: kz  

## Section 1: Installing what's necessary

Before starting let's install the following ports:

```
/usr/ports/mail/postfix/
/usr/ports/security/amavisd-new/
/var/ports/mail/p5-Mail-SpamAssassin/
/var/ports/security/clamav/
```

## Section 2: Setting Up Postfix
An Overview of What's Going On

To give you an idea of what we're trying to accomplish here, this is a rough idea of how the postfix system works when we're done:

```asciiart
              -----------------
incoming mail | Postfix MTA   |local or
on port 25    |               |remote delivery
       ------>|               |--------->
              |    port 10025 |   |
              ----------^------   |
                |       |         v
                |       |
          ------v-------|---     /dev/null or quarantine
          |  port 10024    |
          |                |
          | Amavisd Filter |
          ------------------
```

* Postfix recieves a mail from another MTA via port 25.
* Postfix sends it to Amavisd via port 10024.
* Amavisd processes it and delivers it back to postfix via port 10025.
* Postfix delivers the e-mail locally or remotely as usual, or takes other measures if it's a virus/spam.


## Changes in main.cf

Here is the section in `main.cf` relevant for configuring antispam on Postfix

```conf
header_checks = regexp:/usr/local/etc/postfix/header-checks

smtpd_client_restrictions =
   check_client_access hash:/usr/local/etc/postfix/blackwhite.map,
   reject_non_fqdn_hostname,
   reject_non_fqdn_sender,
   reject_unknown_sender_domain,
   permit_mynetworks,
   reject_rbl_client list.dsbl.org,
   reject_rbl_client sbl.spamhaus.org,
   reject_rbl_client relays.ordb.org,
   reject_rbl_client bl.spamcop.net,
   reject_rbl_client dun.dnsrbl.net,
   permit

smtpd_sender_restrictions =
   check_sender_access hash:/usr/local/etc/postfix/blackwhite.map,
   reject_unknown_sender_domain,
   reject_non_fqdn_sender,
   permit

smtpd_recipient_restrictions =
   check_recipient_access hash:/usr/local/etc/postfix/blackwhite.map,
   reject_non_fqdn_hostname,
   reject_non_fqdn_sender,
   reject_non_fqdn_recipient,
   reject_unknown_sender_domain,
   permit_mynetworks,
   reject_unauth_destination,
   permit

### Tarpit those bots/clients/spammers who send errors or scan for accounts
smtpd_error_sleep_time = 60
smtpd_soft_error_limit = 60
smtpd_hard_error_limit = 10
```

## Black/Whitelisting at the Server

Right now would be a good time to take domains or individual addresses that you know e-mail you frequently or you want to be able to e-mail you and put them in `blackwhite.map`. The format is like this:

```conf
gooddomain.com OK
myfriend@soemplace.com OK
spammer@weruintheinternet.com REJECT
```

When you're done, issue the command:

```conf
postmap blackwhite.map
```

## Changes in master.cf

At this point you can change the smtp services at the top to optimize your mail server. We can assume that we won't be doing any virus checking for the internal network which in this case is 10.0.0.1, the external net is 1.2.3.4, this is also where we will point postfix to use amavis which we will comment out for now.

At this point you have two filtering options if you use Postfix 2.1 or above:

```conf
    use -o content_filter=smtp-amavis:[127.0.0.1]:10024 for post-queuefiltering.
    use -o smtpd_proxy_filter=smtp-amavis:[127.0.0.1]:10024 for pre-queue filtering.
```

I'm using pre-queue filtering below since I have a fairly low amount of mail traffic. If you choose pre-queue filtering, note well the option to make this work in the amavisd.conf below. You'll want to leave this commented until you are all ready to test out amavis, or you may lose mail.

```conf
1.2.3.4:smtp      inet  n       -       n       -       -       smtpd
#	-o content_filter=smtp-amavis:[127.0.0.1]:10024
10.0.0.1:smtp           inet  n       -       n       -       -       smtpd
        -o smtpd_client_restrictions=permit_mynetworks,reject
127.0.0.1:smtp          inet    n       -     n       -       -       smtpd
        -o smtpd_client_restrictions=permit_mynetworks,reject

Now you'll want to edit your /usr/local/etc/postfix/master.cf and add this to the bottom, this accepts the mail back from amvis when it's done:

	smtp-amavis unix -      -       -     -       2  smtp
	    -o smtp_data_done_timeout=1200
	    -o disable_dns_lookups=yes
	127.0.0.1:10025 inet n  -       -     -       -  smtpd
           -o content_filter=
           -o local_recipient_maps=
	    -o relay_recipient_maps=
 	    -o smtpd_restriction_classes=
 	    -o smtpd_client_restrictions=
	    -o smtpd_helo_restrictions=
	    -o smtpd_sender_restrictions=
	    -o smtpd_recipient_restrictions=permit_mynetworks,reject
	    -o mynetworks=127.0.0.0/8
	    -o strict_rfc821_envelopes=yes
```

## Changes in /etc/aliases and Finishing the Postfix Configuration

Next step is to add the following to `/etc/aliases`:

```conf
virusalert: root 
```

Then issue the command:

```conf
postalias /etc/aliases
```

At this point you can start postfix and see if it's running on ports 25 and 10025. Just make sure you commented the content_filter or smtpd_proxy_filter bit in the master.cf.

```shell
# postfix stop && postfix start
# sockstat -l4 |grep 25|grep master
root     master     51589 11 tcp4   1.2.3.4:25            *:*
root     master     51589 14 tcp4   10.0.0.1:25           *:*
root     master     51589 17 tcp4   127.0.0.1:25          *:*
root     master     51589 95 tcp4   127.0.0.1:10025       *:*
```

You should see something like the two lines above. If not there's something not configured right in your `master.cf`. Otherwise, if you see this you're done with postfix for now.

## Section 3: Setting up Amavisd-new

Now let's edit the amavisd.conf:

```conf
cd /usr/local/etc/
cp amavisd.conf-dist amavisd.conf
```

There's a million things to tweak here. I highly suggest patiently going through the whole thing so you don't end up coming back to it many times later. I'll list some important bits to fill in, and ones I found useful.

While you're going through this file, if you just copied it like I did in the example above, you may want to delete a lot of the comments you won't use for easier scanning and readability.

```conf
$mydomain = 'example.com';
$TEMPBASE = "$MYHOME/tmp"; # this uses a good chunk of space on busier sites
                           # so make sure you have it.
#Make sure the below are uncommented even though they are default
$forward_method = 'smtp:127.0.0.1:10025';  # where to forward checked mail
$notify_method = $forward_method;          # where to submit notifications

# The below is fine for a home server, but for a company you might want to bump
# it to 10 or more. Also like the comments say, it *must* match the value in
# your master.cf:
# smtp-amavis unix -      -       -     -       2  smtp
 
$max_servers = 2;

# VERY IMPORTANT IF YOU ARE USING POST FILTERING
# Set the below to 0 if you are using smtpd_proxy_filter,
# Set to 1 (default) if you are using content_filter.
$insert_received_line = 0;       # behave like MTA: insert 'Received:' header

# For viruses I really recommend just discarding the e-mail. So many viruses
# these days have forged recipients that sending a bounce warning does more
# harm than good. If you want to discard only viruses that fake sender then
# leave $final_virus_destiny at BOUNCE and be sure to update
# $viruses_that_fake_sender_re as new virii appear.

$final_virus_destiny      = D_DISCARD;  # (defaults to D_BOUNCE)
$final_banned_destiny     = D_BOUNCE;  # (defaults to D_BOUNCE)
$final_spam_destiny       = D_DISCARD;  # (defaults to D_REJECT)

# If you get a lot of spam you'll want to make sure you have space for the
# dir below, and you may want to clean it periodically.
# Something like this in crontab should work for weekly:
# 0 1 * * 0       /usr/bin/find /var/virusmails -ctime +7 -exec /bin/rm {} \;

$QUARANTINEDIR = '/var/virusmails';

@av_scanners = (
### http://clamav.elektrapro.com/
['Clam Antivirus-clamd',
   \&ask_daemon, ["CONTSCAN {}\n", '/var/amavis/clamd'],
   qr/\bOK$/, qr/\bFOUND$/,
   qr/^.*?: (?!Infected Archive)(.*) FOUND$/ ],
);

@av_scanners_backup = (
);
```

now lets create some directories

```shell
# cd /var/amavis
# mkdir tmp clamav .spamassassin .razor
# touch .spamassassin/user_prefs 
# chown -R vscan:vscan .razor .spamassassin clamav tmp
```

## Section 4: Setting up clamav

edit /usr/local/etc/clamav.conf

```conf
User vscan
LocalSocket /var/amavis/clamd
```

edit /usr/local/etc/freshclam.conf

```conf
DatabaseOwner vscan
```

As Root:

```shell
# chown -R vscan:vscan /var/log/clamav
# chown -R vscan:vscan /var/run/clamav
# chown -R vscan:vscan /usr/local/share/clamav
# /usr/local/bin/freshclam --datadir=/usr/local/share/clamav
```

You'll also want to put the above command in your crontab, the below

 will make it run nightly: 

```cron
0 1 * * * /usr/local/bin/freshclam --datadir=/usr/local/share/clamav
```

Edit `/etc/group` and add `vscan` to the mail group.

That's all you need to do for clamav

## Section 5: Setting up razor

Here's some simple steps to set up razor, make sure you replace postmaster@example.com with your e-mail address.

```shell
# su - vscan
$ razor-admin -create
$ razor-admin -discover
$ razor-admin -register -user postmaster@example.com
```

## Section 6: Setting up spamassassin

This is the part that probably takes the most tuning if you really want effective spam blocking. Below is my local.cf. You can take the sample in /usr/local/etc/mail/spamassassin or you can use mine, or you can combine the two. I've added a bit of custom stuff at the bottom that may or may not be relevant to you, so I strongly suggest you go through this.

```conf
skip_rbl_checks 1
# By default SpamAssassin runs the Realtime Blackhole List checks. 
# It's better to turn this option off.

use_bayes 1
# This turns Bayesean Learning on. 0 turns it off.
	
bayes_path /var/amavis/.spamassassin/bayes
# Bayesean database location.

use_razor2 1
# Tells SA that we want to use Razor version 2

use_dcc 0
# In case you want DCC.

use_pyzor 0
# Tells SA that we don't want to use Pyzor

dcc_add_header 1
# DCC header in case you want it.

dns_available yes
# If you are sure you have DNS access set it to "yes".

header LOCAL_RCVD Received =~ /\S+\.section6.net\s+\(.*\[.*\]\)/
score LOCAL_RCVD -50
# This checks "Received: from...." lines in the message header.
# Set .domain.com to your domain so outgoing mail will not be tagged as
# spam. Unless you are a spammer of course. In case you are I strongly urge
# you to use this option.

## Optional Score Increases
score DCC_CHECK 4.000
score RAZOR2_CHECK 2.500
score BAYES_99 5.300
score BAYES_90 4.500
score BAYES_80 4.000
# For scores have a look at /usr/local/share/spamassassin/50_scores.cf
# file.
score HTML_FONT_INVISIBLE 3
score HTML_FONTCOLOR_UNKNOWN 2
score ORDER_NOW 1.5
score CLICK_BELOW 1
score LIMITED_TIME_ONLY 1
# This rule might be extreme but html only spams get through too easy.
# In other words, if you can't take the time to write something and are
# posting an image only, then you're 86'd!
score HTML_IMAGE_ONLY_02 2
score HTML_IMAGE_ONLY_04 2
score OFFERS_ETC 2
score HTML_LINK_CLICK_HERE 1
score LINES_OF_YELLING 1
```

I also downloaded some custom rules from http://www.emtinc.net/spamhammers.htm and put them in /usr/local/etc/mail/spamassassin. More of these add-ons make spamassassin do more work, but they also block more spam. Use your judgement.

Another thing to do, is to make your own custom ham rules. For example, if you work for a company such as Apple, you may wish to hamify words like Apple, Macintosh, PowerPC, or whatever. If this is just for yourself then put your name in there (as long as it's not part of your e-mail address), or other common words people might use with you.
Setting up Bayesian Learning with Spamassassin

If you're like me you probably have a folder with a ton of spam in it. You can i use this for the bayesian learner in Spamassassin (which is activated in the above config). I also have an mbox folder with a ton of regular mail with no spam in it.

So here's what I do:

```shell
sa-learn --spam --mbox -p /var/amavis/.spamassassin/user_prefs ~/Mail/spam
sa-learn --ham --mbox -p /var/amavis/.spamassassin/user_prefs ~/Mail/mbox
sa-learn --rebuild -p /var/amavis/.spamassassin/user_prefs
```

## Section 7: Finishing up and testing

At this point you'll want to test spamassassin to make sure it's working. try saving an email and running this command on it:

```shell
spamassassin -t < mail.txt
```

If it works then Spamassassin is good to go. If it takes a really long time (longer than 10 seconds) like it did on my Pentium 166 you might want to use a faster machine for this.

From here you can start amavisd from the command line and see if it's running.

```shell
# /usr/local/sbin/amavisd
# ps auxww|grep amavisd
vscan       761  0.0  0.7 28556  632  ??  Ss   11:13PM   0:12.83 amavisd (master) (perl)
vscan     10526  0.0  0.0 29412   12  ??  I     1:00PM   0:08.47 amavisd (child) (perl)
vscan     10527  0.0  0.0 29676    0  ??  IW   -         0:00.00 amavisd (child) (perl)
```

If you don't see it in the process list then you should have got some errors on the command line when you tried to start it. 

Now lets start clamd using the same process as above:

```shell
# /usr/local/sbin/clamd
# ps auxww|grep clamd
vscan      3141  0.0  0.6 21868  576  ??  Ss   12:11PM   0:03.36 clamd
```

If you didn't see it in the process list check `/var/log/clamav/clamd.log` to see if there were any errors.

At this point if you have both amavis running, clamd is go, go back and uncomment the content_filter or smtp_proxy_filter line in /usr/local/etc/postfix/master.cf and reload postfix:

```shell
# postfix reload
```

Now send yourself an e-mail while checking `/var/log/maillog` to make sure there's no errors and you're good to go!

## References

* [A tutorial I based this one from](http://ezine.daemonnews.org/200309/postfix-spamassassin.html)
* [Another tutorial like this one using OpenBSD](http://www.flakshack.com/anti-spam/)
* [A tutorial like this one with MySQL and emphasis on implimentation in an ISP](http://www.marlow.dk/?target=postfix)
* [Spamassassin Blacklists](http://www.stearns.org/sa-blacklist/)
* [Jennifer's Spamassassin rules](http://www.emtinc.net/spamhammers.htm)
* [Spamassassin Rules Wiki](http://www.exit0.us/index.php)
* [Postfix After-Queue Content Filter](http://www.postfix.org/FILTER_README.html)
* [Postfix Before-Queue Content Filter](http://www.postfix.org/SMTPD_PROXY_README.html)