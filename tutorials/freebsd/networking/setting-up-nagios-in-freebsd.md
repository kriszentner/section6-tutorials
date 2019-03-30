# Setting up Nagios in FreeBSD

Nagios is a system and network monitoring application. It watches hosts and services that you specify and creates event alerts based on the services it monitors. Nagios used to be the NetSaint project until recently. Although the NetSaint site is still up, all future development for the application will be done on Nagios.
Requirements

* MySQL 4x server : /usr/ports/databases/mysql40-server
* MySQL 4x client : /usr/ports/databases/mysql40-client
* Apache 1.3x : /usr/ports/www/apache13
* Net-SNMP : /usr/ports/net-mgmt/net-snmp
* Nagios : /usr/ports/net-mgmt/nagios (WITH_MYSQL="YES")

## SNMP Configuration

SNMP must be installed and configured on the systems that you wish to monitor with Nagios. Configuration for SNMP is fairly straightforward, although this step is not necessary if you do not wish to customize SNMP communities or if security is not an issue for you. At Section6, we always recommend taking the more secure path. On Unix based systems there are two ways to alter your SNMP configuration.

* Run snmpconf and answer the questions presented
* Create your own config file in `/usr/local/share/snmp/snmpd.conf`

## Creating Your Own Config

Here is a sample snmpd.conf created with the aide of the snmpconf utility.

```conf
syslocation: Systemland, USA
sysservices 0
syscontact	admin@yourlan.com

            #community	#hosts allowed
rwcommunity	private 	trusted.host.com
rocommunity	everyone 	10.0.0.0/24
```

There are probably more you can put in here, but this is a basic configuration. `rwcommunity` defines who can write to the host’s MIBs (the derivatives that show info, you can view all the mibs with snmpwalk. The `rocommunity` shows who can view your SNMP tree. This defaults to public so you may want to change it to something different for security purposes.

If you are running alternate Operating Systems on your network, consult the documentation for your distribution to find out how to install and configure SNMP

## Nagios Configuration

Now that both Nagios and the plugins are installed from ports, we are almost ready to start monitoring our servers. However, Nagios will not even start before we configure it properly.

Let's start by taking a look the sample configuration files in the `/usr/local/etc/nagios` directory:

```conf
cgi.cfg-sample
checkcommands.cfg-sample
contactgroups.cfg-sample
contacts.cfg-sample
dependencies.cfg-sample
escalations.cfg-sample
hostgroups.cfg-sample
hosts.cfg-sample
misccommands.cfg-sample
nagios.cfg-sample
resource.cfg-sample
services.cfg-sample
timeperiods.cfg-sample
```

Since these are sample files, the Nagios authors added a `.cfg-sample` suffix to each file. First, we need to copy or rename each file to a `.cfg` extension, so that the software can use them properly. (If you don't change the configuration filenames, Nagios will still try to access them with the `.cgi` extension, and not be able to find them. The authors want to ensure that everyone create their own custom configuration files.)

Before renaming the sample files, it is ususally a wise idea to back them up, just in case we need to refer to them later.

```shell
root@host:/usr/local/etc/nagios/etc # mkdir sample
root@host:/usr/local/nagios/etc # cp *.cfg-sample sample/
```

Here is a handy command for renaming all the files to `.cfg` extensions:

```shell
root@host:/usr/local/etc/nagios/etc# for i in *cfg-sample; do mv $i 
`echo $i | sed -e s/cfg-sample/cfg/`; done;
```

The following is what you should end up with in the directory.

```shell
cgi.cfg
checkcommands.cfg
contactgroups.cfg
contacts.cfg
dependencies.cfg
escalations.cfg
hostgroups.cfg
hosts.cfg
misccommands.cfg
nagios.cfg
resource.cfg
sample/
services.cfg
timeperiods.cfg
```

First we will start with the main configuration file, `nagios.cfg`. You can pretty much leave everything as is, because the Nagios installation process will make sure the file paths used in the configuration file are correct. There's one option, however, that you might want to change. The `check_external_commands` is set to 0 by default. If you would like to be able to change the way Nagios works, or directly run commands through the Web interface, you might want to set this to 1. There are still some other options you need to set in `cgi.cfg` to configure which usernames are allowed to run external commands.

In order to get Nagios running, you will need to modify all but a few of the sample configuration files. Configuring Nagios to monitor your servers is not as difficult as it looks; sometimes the best approach to configuring Nagios properly the first time is to use the debugging mode of the Nagios binary. You can run Nagios in this mode by running:

```shell
root@host # nagios -v /usr/local/etc/nagios/nagios.cfg
```

This command will parse the configuration files and verbosely report any errors that were found. Start fixing the errors one by one, and run the command again to find the next error. For our purposes, disable all hosts and services definitions that come with the sample configuration files and merely use the files as templates for our own hosts and services. We will keep most of the files as is, and remove the following

```shell
hosts.cfg
services.cfg
contacts.cfg
contactgroups.cfg
hostgroups.cfg
dependencies.cfg
escalations.cfg
```

We will not be going into the more advanced configuration that requires using `dependencies.cfg`

and `escalations.cfg`, so just remove these two files so that the sample configuration in these do not stop Nagios from starting up. Still, Nagios requires that these files are present in the etc directory, so create two empty files and name them `dependencies.cfg` and `escalations.cfg` by running the following as root.

```shell
root@host:/usr/local/etc/nagios # touch dependencies.cfg 
root@host:/usr/local/etc/nagios # touch escalations.cfg
```

## Configuring Monitoring

from: http://www.onlamp.com/pub/a/onlamp/2002/09/26/nagios.html?page=1

We first need to add our host definition and configure some options for that host. You can add as many hosts as you like, but we will stick with one host for simplicity.
Contents of hosts.cfg
```conf
# Generic host definition template
define host{
# The name of this host template - referenced i
name                            generic-host
# Other host definitions, used for template recursion/resolution
# Host notifications are enabled
notifications_enabled           1
# Host event handler is enabled
event_handler_enabled           1
# Flap detection is enabled  
flap_detection_enabled          1
# Process performance data
process_perf_data               1
# Retain status information across program restarts
retain_status_information       1
# Retain non-status information across program restarts
retain_nonstatus_information    1
# DONT REGISTER THIS DEFINITION - ITS NOT A REAL HOST,
# JUST A TEMPLATE!
register                        0
}

# Host Definition
define host{
# Name of host template to use
use                     generic-host
host_name               domain.com
alias                   My Registered Domain
address                 www.domain.com
check_command           check-host-alive
max_check_attempts      10
notification_interval   120
notification_period     24x7
notification_options    d,u,r
}
```

The first host defined is not a real host but a template from which other host definitions are derived. This mechanism can be seen in other configuration files also and makes configuration based on a predefined set of defaults a breeze.

With this setup we are monitoring only one host , `www.domain.com` to see if it is alive. The `host_name` parameter is important because this server will be referred to by this name from the other configuration files.

Now we need to add this host to a hostgroup. Even though we will keep the configuration simple by defining a single host, we still have to associate it with a group so that the application knows which contact group (see below) to send notifications to.

Contents of `hostgroups.cfg`

```conf
define hostgroup{
hostgroup_name  domain-servers
alias           Domain Servers
contact_groups  domain-admins
members         domain.com
}
```

Above, we have defined a new hostgroup and associate the 'domain-admins' contact group with it. Now let's look into the contactgroup settings.

Contents of `contactgroups.cfg`

```conf
define contactgroup{
contactgroup_name       domain-admins
alias                   Domain.Com Admins
members                 bob, bill
}
```

We have defined the contact group `domain-admins` and added two members `bob` and `bill` to this group. This configuration ensures that both users will be notified when something goes wrong with a server that 'domain-admins' is responsible for. (Individual notification preferences can override this). The next step is to set the contact information and notification preferences for these users.

Contents of `contacts.cfg`
```conf
define contact{
contact_name                    bob
alias                           Bob Buddy
service_notification_period     24x7
host_notification_period        24x7
service_notification_options    w,u,c,r
host_notification_options       d,u,r
service_notification_commands   notify-by-email,notify-by-epager
host_notification_commands      host-notify-by-email,host-notify-by-epager
email                           bob@domain.com
pager                           domian-admin@localhost.localdomain
}

define contact{
contact_name                    bill
alias                           Bill Buddy
service_notification_period     24x7
host_notification_period        24x7
service_notification_options    w,u,c,r
host_notification_options       d,u,r
service_notification_commands   notify-by-email,notify-by-epager
host_notification_commands      host-notify-by-email
email                           bill@domain.com
}
```

In addition to providing contact details for a particular user, the `contact_name` in the contacts.cfg is also used by the cgi scripts (i.e the Web interface) to determine whether a particular user is allowed to access a particular resource. Although you will need to configure .htaccess based basic http authentication in order to be able to use the Web interface, you still need to define those same usernames as seen above, before the users can access any of the resources even after they are logged in with their username and passwords. Now that we have our hosts and contacts configured, we can start configuring individual services on our server to be monitored.

Contents of `services.cfg`
```conf
# Generic service definition template
define service{
# The 'name' of this service template,
# referenced in other service definitions
name    generic-service  
# Active service checks are enabled
active_checks_enabled  1 
# Passive service checks are enabled/accepted
passive_checks_enabled  1 
# Active service checks should be parallelized 
# (disabling this can lead to major performance problems)
parallelize_check  1  
# We should obsess over this service (if necessary)
obsess_over_service  1  
# Default is to NOT check service 'freshness'
check_freshness   0  
# Service notifications are enabled
notifications_enabled  1 
# Service event handler is enabled
event_handler_enabled  1 
# Flap detection is enabled
flap_detection_enabled  1 
# Process performance data
process_perf_data  1 
# Retain status information across program restarts
retain_status_information 1  
# Retain non-status information across program restarts
retain_nonstatus_information 1  
# DONT REGISTER THIS DEFINITION - 
# ITS NOT A REAL SERVICE, JUST A TEMPLATE!
register   0
}

# Service definition
define service{
# Name of service template to use
use    generic-service
host_name   host1.domain.com
service_description  HTTP
is_volatile   0
check_period   24x7
max_check_attempts  3
normal_check_interval  5
retry_check_interval  1
contact_groups   domain-admins
notification_interval  120
notification_period  24x7
notification_options  w,u,c,r
check_command   check_http
}

# Service definition
define service{
# Name of service template to use
use    generic-service

host_name   host1.domain.com
service_description  PING
is_volatile   0
check_period   24x7
max_check_attempts  3
normal_check_interval  5
retry_check_interval  1
contact_groups   domain-admins
notification_interval  120
notification_period  24x7
notification_options  c,r
check_command   check_ping!100.0,20%!500.0,60%
}
```

Using the above setup, we are configuring two services to be monitored. The first service definition, which we have called HTTP, will be monitoring whether the Web server is up and notifies us if there's a problem. The second definition monitors the ping statistics from the server and notifies us if the response time increases too much and if there's too much packet loss which is a sign of network trouble. The commands we use to accomplish this are `check_http` and `check_ping` which were installed with the Nagios plugins. Please take your time to get familiar with all other plugins that are available and configure them similarly to the above definitions. You can also write your own plugins to do custom monitoring. For instance, there's no plugin to check if Tomcat is up or down. You could simply write a script that loads a default jsp page on a remote Tomcat server and returns a success or failure status based on the presence or lack of a predefined text value (i.e "Tomcat is up") on the page. (In such a case you would need to add a definition for this custom command in your

`checkcommand.cfg` file which we have not touched)

## Starting Nagios

Now that we have configured the hosts and the services to monitor, we are ready to fire up Nagios and start monitoring. We will start Nagios using the init script.

```shell
root@host # /usr/local/etc/rc.d/nagios.sh start
```

If everything went smoothly, Nagios should now be running. The following command will show you whether Nagios is up and running and the process ID associated with it, if it is indeed running.

```shell
root@host # /usr/local/etc/rc.d/nagios.sh status
  PID TTY          TIME CMD
  22645 ?        00:00:00 nagios
```

## The Web Interface

Although Nagios has already started monitoring and is going to send us the notifications if and when something goes wrong, we need to set up the Web interface to be able to interactively monitor services and hosts in real time. The Web interface also gives a view of the big picture by making use of graphics and statistical information.

Sure enough, we need to have a Web server already set up in order to be able to access the Nagios Web interface. As recommended per this article, we should be running the Apache Web server. We will use the same configuration that is included in the Nagios documentation.

Addition to `httpd.conf`
```conf
ScriptAlias /nagios/cgi-bin/ /usr/local/share/nagios/cgi-bin/
<Directory "/usr/local/share/nagios/cgi-bin/">
AllowOverride AuthConfig
Options ExecCGI
Order allow,deny
Allow from all
</Directory>

Alias /nagios/ /usr/local/share/nagios/
<Directory "/usr/local/share/nagios">
Options None
AllowOverride AuthConfig
Order allow,deny
Allow from all
</Directory>
```

This configuration creates a Web alias `/nagios/cgi-bin/` and directs it to the cgi scripts in your Nagios `cgi-bin` directory. Assuming your main Web site is set up at http://127.0.0.1, you will be able to access the Nagios Web interface at http://127.0.0.1/nagios/ . At this point, the Nagios Web interface should come up properly, but you will notice that you might not be able to access any of the pages. You may receive an error message that looks like the following.

```shell
It appears as though you do not have permission to view information for any of the 
hosts you requested...If you believe this is an error, check the HTTP server authentication 
requirements for accessing this CGI and check the authorization options in your CGI configuration 
file.
```

This is a security precaution that is designed to only allow authorized people to be able to access the monitoring interface. The authentication is handled by your Web server using Basic HTTP Authentication (i.e. `.htaccess`). Nagios then uses the credentials for the user who has logged in and matches it with the contacts.cfg contact_name entries to determine which sections of the Web interface the current user can access.

Configuring `.htaccess` based authentication is easy provided that your Web server is already configured to use it. Please refer to the documentation for your Web server if it's not configured. We will assume that our Apache server is configured to look at the .htaccess file and apply the directives found in it.

First, create a file called `.htaccess` in the `/usr/local/share/nagios/cgi-bin` directory. If you would like to lock up your Nagios Web interface completely, you can also put a copy of the same file in the `/usr/local/share/nagios directory`.

Put the following in this .htaccess file.

```conf
AuthName "Nagios Access"
AuthType Basic
AuthUserFile /usr/local/nagios/etc/htpasswd.users
require valid-user
```

When you're adding your first user, the password file that .htaccess refers to will not be present. You need to run the `htpasswd` command with the `-c` option to create the file.

```shell
htpasswd -c /usr/local/nagios/etc/htpasswd.users bob
New password: ******
Re-type new password: ******
Adding password for user bob
```

For the rest of your users, use the `htpasswd` command without the `-c` option so as not to overwrite the existing one. After you add all of your users, you can go back to the Web interface which will now pop up an authentication dialog. Upon successful authentication, you can start using the Web interface. Notice that your users will only be able to access information for servers that they are associated with in the Nagios configuration files. Also, some sections of the Web interface will be disabled for everyone by default. If you would like to enable those, take a look at `cgi.cfg` file. For instance, in order to allow the user `bob` to access the `Process Info` section, uncomment the 'authorized_for_system_information' line and add 'bob' to the list of names delimited by commas.

This is all you need to install and configure Nagios to do basic monitoring of your servers and individual services on these servers. You can then fine tune your monitoring system by going through all of the configuration files and modifying them to match your needs and requirements. Going through all plugins will also give you a lot of ideas about what local and remote services you can monitor. Nagios also comes with software that can be used to monitor a server's disk and load status remotely. Finally, Nagios comes with so many features that no single article could explain all of it. Please refer to the official documentation for more advanced topics at the following URL:

http://www.nagios.org
