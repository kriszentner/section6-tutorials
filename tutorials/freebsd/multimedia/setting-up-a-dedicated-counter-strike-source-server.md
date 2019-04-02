# Setting up a Dedicated Counter Strike Source Server

Counter Strike: Source is the long awaited update to the original Half Life mod Counterstrike. This update includes all the original maps and missions, with the exception of adding an updated engine support for the new popular Half Life 2 game engine.

## Ports that are needed:

`Linux Base 8: /usr/ports/emulators/linux_base-8`

Various other Linux emulation ports may work for this task.. but for the sake of this tutorial, we will be using Linux Bas 8.

Once installed, be sure and either kldload the linux emulation module or compile the option into your kernel:
```conf
options         COMPAT_LINUX
options         LINPROCFS
```

## Getting Steam

The company who developes and distributes Half-Life now employs a

client-side update tool known as Steam. Steam allows us to connect to one or more of the Half-Life update servers and download the dedicated server files required to run the Counter Strike: Source server. To get the Steam client on our server, simply download it to the directory of your choice by running the following command:

```shell
wget http://www.steampowered.com/download/hldsupdatetool.bin
```

After the client has been downloaded, we can extract it and run it

```shell
chmod +x hldsupdatetool.bin
./hldsupdatetool.bin 
```

After confirming acceptance to the license agreement, the Steam client should be extracted to the same directory in which we downloaded the Hal-Life update tool. We now need to register for a new account with the Steam service.

```shell
./steam -command create -username yourdesiredname -email yourdesiredemail -password yourdesiredpassword -question "your desired question" -answer yourdesiredanswer
```

Of course we would replace the yourdesired options with the actual username, password, and other relevent information. Once your account has been registered, you can login to download the desired game server of your choice.

In this case, we are going to want to pull in Counter Strike: Source, so we would run the following command:

```shell
./steam -command update -game "counter-strike source" -dir /usr/games/cssource -username yourusername -password yourpassword
```

The tool will probably update itself, and then begin downloading the files. Eat a snack or get off your behind for a bit, and take a walk. This can take awhile, for the game files are over 500 megabytes

Note: sometimes the Steam client has problems with posix permissions on UFS2, so if you encounter problems, try creating the directory ~/.steam for which the Steam client can create its update files.

Note: Occasionally the Steam client can crash when attempting to update, though I personally havent debugged it, it seems as if there might be a problem with procfs. If you continually see this issue when attempting to update and cannot circumvent the issue, consider using Steam to download the files from an actual GNU/Linux computer, and then transferring them over to your FreeBSD server later.
Configuration

Once the Counter Strike: Source files have been downloaded, simply change to the directory where we downloaded the game files and edit the following three configuration files to your needs:

* cstrike/cfg/server.cfg
* cstrike/motd.txt
* cstrike/mapcycle.txt

For more information on the syntax and configuration options for these files, please visit the Counter Strike Server Support Site. They have plenty of online documentation and support forums for server administrators starting out.

From here, we should be able to simply run the game with a few command line options to get it started:

```shell
./srcds_run -game cstrike +maxplayers 20 +map de_dust +ip 1.1.1.1 +port 27015
```

Of course we would want to use the correct IP to which we wanted the dedicated server to bind. Also, remember that if you are running the server behind a Firewall/NAT, you must open and/or forward port 27015 TCP and UDP.

From here you should have a fully functional Counter Strike: Source server, so happy gaming!