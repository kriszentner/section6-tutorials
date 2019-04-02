# Setting up a Dedicated Battlefield 1942 Server

Battlefield 1942 is a World War II multiplayer tactical game. Even though the binaries available for the dedicated server are meant to be run on most distributions of GNU/Linux, it also runs on FreeBSD with Linux emulation. This tutorial will guide you in setting up the dedicated server and is current for FreeBSD 5.x, assuming you know how to install from the FreeBSD ports tree.

## Ports that are needed:

Linux Base 8: /usr/ports/emulators/linux_base-8

After the port installation of Linux Base 8 has completed, download the latest Linux Dedicated Server from the following URL:

http://www.3dgamers.com/dl/games/battlefield1942/bf1942_lnxded-1.6-rc2.run.html

Note that at the time of this article, the version of the dedicated server is 1.6.

To install the Dedicated Server, simply run the following command:

```shell
root@host # sh bf1942_lnxded-1.6-rc2.run
```

Running the shell script will present you with a couple of options and license agreements. Simply choose "yes" to the license agreements and proceed with the installation by choosing the desired directory location in which you wish to install the dedicated server; ie: /usr/local/

Note that the installation automatically creates the bf1942 subdirectory under your target directory; ie: /usr/local/bf1942. Also note that the installation of the PunkBuster software will not allow connections from clients with invalid CD keys

After the installation of the Dedicated Server, it is now time to copy over the game files from a client computer to the server directory in which you installed. Be sure and update the client files with the newest patch for the game. At the time of this article, the newest patch is version 1.6.19; which is available at the following URL:

http://www.3dgamers.com/dl/games/battlefield1942/bf1942_patch_v1.6.19.exe.html

Note that if you are running the Macintosh version because it is currently running at version 1.6, so no update should be necessary.

Once the installtion has been updated to 1.6.19, simply copy over the contents of the Battlefield 1942 directory to the Dedicated Server's /usr/local/bf1942 directory (or to whatever directory in which you installed the Dedicated Server)

Now you can edit a couple of configuration files to your liking. These configuration files are located in the bf1942 Dedicated Server installation directory under mods/bf1942/settings.

Here is an example of a couple of configuration files:

```conf
===serversettings.con===
game.serverName "Cool Battlefield Server"
game.serverDedicated 1
game.serverGameTime 0
game.serverMaxPlayers 32
game.serverScoreLimit 0
game.serverInternet 1
game.serverNumberOfRounds 3
game.serverSpawnTime 20
game.serverSpawnDelay 3
game.serverGameStartDelay 20
game.serverGameRoundStartDelay 10
game.serverSoldierFriendlyFire 100
game.serverVehicleFriendlyFire 100
game.serverTicketRatio 100
game.serverAlliedTeamRatio 1
game.serverAxisTeamRatio 1
game.serverCoopAiSkill 75
game.serverCoopCpu 20
game.serverPassword ""
game.serverReservedPassword ""
game.serverNumReservedSlots 0
game.setServerWelcomeMessage 0 ""
game.serverBandwidthChokeLimit 0
game.serverMaxAllowedConnectionType CTLanT1
game.serverAllowNoseCam 1
game.serverFreeCamera 0
game.serverExternalViews 1
game.serverAutoBalanceTeams 0
game.serverNameTagDistance 50
game.serverNameTagDistanceScope 300
game.serverKickBack 0.000000
game.serverKickBackOnSplash 0.000000
game.serverSoldierFriendlyFireOnSplash 100
game.serverVehicleFriendlyFireOnSplash 100
game.serverIP 10.0.0.1
game.serverPort 14567
game.gameSpyLANPort 0
game.gameSpyPort 0
game.ASEPort 0
game.serverHitIndication 1
game.serverTKPunishMode 1
game.serverCrossHairCenterPoint 1
game.serverDeathCameraType 1
game.serverContentCheck 1
game.serverEventLogging 0
game.serverEventLogCompression 0
game.objectiveAttackerTicketsMod 100
game.serverPunkBuster 0
game.serverUnpureMods ""
 
===maplist.con===
game.addLevel berlin GPM_CQ bf1942
game.setCurrentLevel berlin GPM_CQ bf1942
```

From here, you simply need to create a symbolic link in the bf1942 from the static binary for the dedicated server:

```shell
root@host # ln -s bf1942_lnxded.static bf1942_lnxded
```

If you do not create the symlink, then the server cannot start properly with Linux emulation in FreeBSD

Once the symlink has been created, create a startup script for the Dedicated Server:

```shell
#! /bin/sh
exec ./bf1942_lnxded $@
```

Note that if you are running the Dedicated Server from behind a NAT and/or firewall, be sure TCP and UDP port 14567 is open and forwarded appropriately
Caveats

The Linux Dedicated Server will only read lower-case filenames. All file names encountered at runtime are lower-cased before a filesystem access is attempted. You should therefore make sure all files are lower-case when installing third-party modifications and maps.

To aid you with this there is an included shell script called fixinstall.sh which recursively changes the case of files and directories from the directory where it's run.

You can simulate the actions of the script with these options:

```shell
root@host # ./fixinstall.sh --pretend
```

When you're certain it looks good run the conversion:

```shell
root@host # ./fixinstall.sh --verbose
```

Installing the PunkBuster software during installation is meant to keep client computers with cheats enabled off the Dedicated Server. Unfortunately this also affects client computers with invalid CD keys. If you wish to share your copy of Battlefield 1942 with your friends and play on your dedicated server, refrain from installing the PunkBuster software during the installation of the Dedicated Server.

You can find more information about running dedicated servers in the Battlefield 1942 web forum located at the following URL:

http://www.bf1942.lightcubed.com