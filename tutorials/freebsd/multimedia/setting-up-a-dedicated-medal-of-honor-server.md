# Setting up a Dedicated Medal Of Honor Server

Medal of Honor : Allied Assault is a World War II multiplayer first person shooter game. Even though the binaries available for the dedicated server are meant to be run on most distributions of GNU/Linux, it also runs on FreeBSD with Linux emulation. This tutorial will guide you in setting up the dedicated server and is current for FreeBSD 5.x, assuming you know how to install from the FreeBSD ports tree.

```conf
seta sv_timeout "120" //amount of time before assuming a disconnected state
seta sv_precache "1"
seta sv_fps "20" //max frame rate to clients - increasing will raise pings
seta sv_maxRate "8000"
seta sv_maxclients "16"
seta g_maxGameClients "0"
seta sv_privateClients "6"
seta sv_allowDownload "1"
seta sv_reconnectlimit "3"
seta sv_zombietime "1"
seta g_inactivity "180"
seta g_forcerespawn "30"
seta g_syncronousclients "0"
seta sv_chatter "1"

// Server Network Settings

set sv_flood_waitdelay "10" 
//not too sure on this, possibly time before flooder is allowed to type again (default)
set sv_flood_persecond "4" 
//messages per second to be considered a flood ??(default)
set sv_flood_msgs "4" 
// ?? (default)
net_noipx "1" 
//Disallows IPX connections, TCP only (network protocol)

// Logs (You must create the log directory if you want logging)

seta g_log "logs/games.log"
seta logfile "1"
seta g_logsync "0"

// Extras

seta sv_maxPing "0"
seta sv_minPing "0"
seta sv_floodProtect "1"

// Server Passwords

seta rconPassword "password"
seta g_password ""
seta sv_privatePassword "password"

// Game Type Settings - ATTN-May be overwritten by MOH config file below
// Set the type of game: 1=Deathmatch 2= Team match 3 = OBJ 4 = Roundbased

seta g_gametype "1"
seta timelimit "0"
seta fraglimit "0"

// seta capturelimit "6"
seta sv_gamespy "0" // Show the server in gamespy

// Game Play Default Settings
//seta g_gravity "800"
//seta g_knockback "1000"
//seta g_quadfactor "3"
//seta g_speed "320"
//seta g_weaponRespawn "5"
//seta g_weaponTeamRespawn "30" //respawn time in seconds for team games

// dmflags -- flags that can be set in the dmflags variable.
// to add multiple flags, total the dmflags you want in the var below
// DF_NO_HEALTH 1
// DF_NO_POWERUPS 2
// DF_WEAPONS_STAY 4
// DF_NO_FALLING 8 
// DF_INSTANT_ITEMS 16
// DF_SAME_LEVEL 32
// DF_NO_ARMOR 2048 
// DF_INFINITE_AMMO 16384 
// DF_NO_FOOTSTEPS 131072
// DF_ALLOW_LEAN 262144
// DF_OLD_SNIPERRIFLE 524288
// This will show Rifle Grenade but give you a shotgun
// DF_GERMAN_SHOTGUN 1048576
seta dmflags 1310720 // shotgun + lean

// Match Settings

seta g_doWarmup "1"
seta g_warmup "20"

// Team Preferences
seta g_teamAutoJoin "0"
seta g_teamForceBalance "1"

// seta g_friendlyFire "0"
seta g_teamdamage "0" // FF on or Off 1 = on

// Voting
seta g_allowVote "1"

// Start the Maprotatin/Game
seta sv_maplist "dm/mohdm1 dm/mohdm2 dm/mohdm3 dm/mohdm4 dm/mohdm2 dm/mohdm1 
dm/mohdm5 dm/mohdm6 dm/mohdm7"

map dm/mohdm2
```

Of course you would want to modify the server.cfg file to fit your needs. The final part is actually starting the server. It is important to start the game with the correct options for it to actually load a map and start the server. Here is an example os some startup options for the server:

```shell
./mohaa_lnxded +exec server.cfg +map dm/mohdm1
```

If you already configured which maps you want to load in your server.cfg file, then you do no need to pass the +map dm/mohdm1 option during startup. However, it is important that you acually call the server.cfg with the +exec server.cfg option for the server to actually run correctly. Just make sure the server.cfg file exists in the main subdirectory of the Medal of Honor: Allied Assault directory

If you wish to run the Medal of Honor: Spearhead expansion dedicated server, the same configuration applies. You must update your Spearhead expansion to version 2.15. The MacOSX version already ships as version 2.15, so the patches are for Windows installations only. These patches can be found at the following URL:

http://moh.filefront.com/sort.files?cat=1549&ref=1462

You can download the Spearhead dedicated server from the following URL:

http://www.section6.net/help/spearhead_lnxded_2.15-beta5.tar.bz2

Once downloaded to the server's Medal of Honor directory, simply extract the tar ball with the following command:

```shell
root@host # tar -xjf spearhead_lnxded_2.15-beta5.tar.bz2
```

This places the fgameded.so file directly into the mainta subdirectory, if you extracted it in the base game directory of Medal of Honor. From here, just copy the same server.cfg file to themainta directory and startup the server with similar command options:

```shell
./spearhead_lnxded +exec server.cfg +map dm/mohdm1
```

From here you should be able to connect to your server from other Medal of Honor: Allied Assault game clients. If you are running your server behind a firewall and/or NAT, just be sure and open and/or forward TCP and UDP port 12203.