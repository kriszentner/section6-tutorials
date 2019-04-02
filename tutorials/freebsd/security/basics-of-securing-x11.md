# Basics Of Securing X11

## Introduction

X11 can pose a potential security risk. Through a network, anyone can connect to an open X display, read the keyboard, dump the screen and windows and start applications on the unprotected display. This article deals with some of the aspects of X windows security. This is in no way a complete guide on the subject, but rather a basic introduction.
How open X displays are found

An open X display is in formal terms an X server that has its access control disabled. Disabling access control is normally done with the xhost command.

```shell
$ xhost + 
```

This allows connections from any host. A single host can be allowed connection with the command

```shell
$ xhost + XXX.XXX.XXX.XXX 
```

Where X is the IP address or host-name. Access control can be enabled by issuing the following command:

```shell
$ xhost - 
```

In this case no host but localhost can connect to the display. If the display runs in the 'xhost -' state, you are safe from programs that scan and attach to unprotected X displays. You can check the access control of your display by simply typing xhost from a shell.

Anyone with a bit of knowledge of Xlib and sockets programming can write an X exploit tool in very little time. The task is normally accomplished by probing the port that is reserved for X11, port 6000. If anything is listening on that port, the tool can call XOpenDisplay("IP-ADDRESS:0.0"), which will return a pointer to the display structure if the target display has its access control disabled. If access control is enabled, XOpenDisplay returns 0 and reports that the display could not be opened.

```text
Xlib: connection to "display:0.0" refused by server
Xlib: Client is not authorized to connect to Server
```

The probing of port 6000 is necessary because of the fact that calling XOpenDisplay() on a host that runs no X server will simply hang the calling process.

## The localhost problem

Running your display with access control enabled by using 'xhost -' will guard you from XOpenDisplay attempts through port 6000, but there is a way one can bypass this protection. If he/she can log into your host, he/she can connect to the display of the localhost. The exploit is fairly simple. By issuing a few lines, one can dump the screen of the host 'target':

```shell
$ rlogin target
$ xwd -root -display localhost:0.0 > ~/stolen.xwd
$ exit
$ xwud -in ~/stolen.xwd
```

Here we have a screendump of the root window of the X server target.

Of course, the intruder must have an account on your system and be able to log into the host where the specific X server runs. On sites with a lot of X terminals, this means that no X display is safe from those with access. If you can run a process on a host, you can connect to any of its X displays.

Every Xlib routine has the Display structure as it's first argument. By successfully opening a display, one can manipulate it with every Xlib call available. For an intruder, the most 'important' ways of manipulation is grabbing a window and keystrokes.

## Snooping techniques - dumping windows

The most natural way of stealing a window from an X server is by using the X11R6 utility xwd. Here is a small excerpt from the xwd manual page

```text
Xwd is an X Window System window dumping utility. Xwd allows Xusers to
store window images in a specially formatted dump file. This file can then be
read by various other X utilities for redisplay, printing, editing, formatting,
archiving, image processing, etc. The target window is selected by clicking the
pointer in the desired window. The keyboard bell is rung once at the beginning
of the dump and twice when the dump is completed. </tt> Basically, xwd will
dump X windows into a format readable by another program, xwud. Here is an
excerpt from the xwud manual page. 

Xwud is an X Window System image undumping utility. Xwud allows X users to
display in a window an image saved in a specially formatted dump file, such as
produced by xwd(1).
```

Both the entire root window, a specified window (by name) can be dumped, or a specified screen. As a security measure xwd will beep the terminal it is dumping from, once when xwd is started, and once when it is finished (regardless of the xset b off command). But with the source code available, it is a matter of small modification to compile a version of xwd that does not generate a beep or otherwise identify itself - on the process list. If we wanted to dump the root window or any other window from a host, we could simply pick a window from the process list, which often gives away the name of the window through the -name flag.

```shell
$ xwd -root localhost:0.0 > file 
```

Here, the output is redirected to a file, and then read with the xwud utility.

```shell
$ xwud -in file 
```

Xterm windows are a bit of a different beast. You can not specify the name of an xterm and then dump it. They are blocked from X_Getimage used by xwd.

```shell
$ xwd -name xterm 
```

This will will result in an error. However, the entire root window (with Xterms and all) can still be dumped and watched by xwud.

## Snooping techniques - reading keyboard

If you can connect to a display, you can also log and store every keystroke that passes through the X server. A program circulating the net, called xkey, does this trick. A kind of higher-level version of the infamous ttysnoop.c. I wrote my own, who could read the keystrokes of a specific window ID (not just every keystroke, as my version of xkey). The window ID's of a specific root-window, can be acquired with a call to XQueryTree(), that will return the XWindowAttributes of every window present. The window manager must be able to control every window-ID and what keys are pressed down at what time. By use of the window-manager functions of Xlib, KeyPress events can be captured, and KeySyms can be turned into characters by continuous calls to XLookupString.

You can even send KeySym's to a Window. An intruder may therefore not only snoop on your activity, he can also send keyboard events to processes, like they were typed on the keyboard. Reading/writing keyboard events to an xterm window opens new horizons in process manipulation from remote. Luckily, xterm has good protection techniques for prohibiting access to the keyboard events.

## Xterm - Secure keyboard option

A lot of passwords are typed in xterm windows. It is therefore crucial that the user has full control over which processes can read and write to an xterm. The permission for the X server to send events to an Xterm window, is set at compile time. The default is false, meaning that all SendEvent requests from the X server to an xterm window is discarded. You can overwrite the compile-time setting with a standard resource definition in the .Xdefaults file:

``conf
xterm*allowSendEvents True 
``

Or by selecting Allow Sendevents on the Xterm Main Options menu. (Accessed by pressing CTRL and the left mouse button But this is NOT recommended by the man page. Read access is a different thing. One Xterm mechanism for hindering other X clients from reading the keyboard transactions of sensitive data is the XGrabKeyboard() call. Only one process can grab the keyboard at any one time. To activate the Secure Keyboard option, choose the Main Options menu in your Xterm window (CTRL+Left mouse button) and select Secure Keyboard. If the colors of your xterm console becoming inverted, the keyboard has now been 'grabbed' and no other X client can read the input.

## Trojan X clients - Xlock and X login managers

One of the most suitable programs for installing a password-grabbing trojan than xlock. With a few simple adjustments of thegetPassword() routine in xlock.c, the password of every user using the trojaned xlock can be stored in a file for later use by an intruder.

If a user has a world writable home directory and a ./ in his or her PATH environment variable, he or she is vulnerable to this kind of attack. Getting the password is achieved by placing the trojaned Xlock in the user's home directory and waiting. The functionality of the original Xlock is contained in the trojaned version and can even tidy up and destroy itself after one succesfull attempt, and the user will not know that his or her password has been captured.

Xlock, like every password-prompting program, should be regarded with suspicion if it shows up in places it should not be, like in your own home directory.

Spoofed X based login managers however are a bit more tricky for the intruder to accomplish. The intruder must simulate the login screen of the display manager run by XDM. The only way to ensure that you get the proper XDM login program is to manually chack for alternate binaries on the system and then restart the X11 session.

## X Security tools - xauth MIT-MAGIC-COOKIE

To avoid unathorized connections to your X display, the command xauth for encrypted X11 connections is widely used. When you login using a display manager, `xdm` creates a file called.Xauthority in your homedir. This file is binary, and readable only through the xauth command.

```shell
$ xauth list 
```
You should receive output resembling the following:

```shell
your.display.ip:0 MIT-MAGIC-COOKIE-1 73773549724b76682f726d42544a684a

display name     authorization type               key
```

The .Xauthority file sometimes contains information from older sessions, but this is not important, as a new key is created at every login session. To access a display with xauthactive - you must have the current access key.

If you want to open your display for connections from a particular user, you must inform that user of your key. He or she must then issue the command

```shell
$ xauth add your.display.ip:0 MIT-MAGIC-COOKIE-1 73773549724b7668etc. 
```

Now, only that user (including yourself) can connect to the display. Xauthority is simple and powerful, and eliminates many of the security problems with X11.