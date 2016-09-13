# Virtualbox Guest Addtitions Installation on Solaris 11.3 VM on x86 Host

## Mount CD-ROM in Solaris 11.3
Get the device name,  device name associated with the CD drive is usually in this format c**x**t**y**d**z**s**n**. Use the `iostat` command to determine it,

```sh
root@solaris-node-01:~# iostat -En
c1t0d0           Soft Errors: 0 Hard Errors: 0 Transport Errors: 0
Vendor: ATA      Product: VBOX HARDDISK    Revision: 1.0  Serial No:
Size: 26.84GB <26843545600 bytes>
Media Error: 0 Device Not Ready: 0 No Device: 0 Recoverable: 0
Illegal Request: 4 Predictive Failure Analysis: 0 Non-Aligned Writes: 0
*c2t0d0*           Soft Errors: 0 Hard Errors: 0 Transport Errors: 0
Vendor: VBOX     Product: CD-ROM           Revision: 1.0  Serial No:
Size: 0.06GB <59262976 bytes>
Media Error: 0 Device Not Ready: 0 No Device: 0 Recoverable: 0
Illegal Request: 0 Predictive Failure Analysis: 0 Non-Aligned Writes: 0
root@solaris-node-01:~#
```
Here, 
 - *-E* displays all device error statistics &,
 - *-n* shows the names in descriptive format.
 
We are interested in only the logical device name though, _slice 0_ is the default.   

#### Create the mount point for the media
```sh
cd /tmp
mkdir guestAdditions
chmod a+rw /tmp/guestAdditions
```

### Mount the media
```sh
mount -F hsfs -o ro /dev/dsk/c2t0d0s0 /tmp/guestAdditions
```

:+1: You have just mounted your media, Try `ls -l /tmp/guestAdditions` to access your media

```
root@solaris-node-01:/tmp/guestAdditions# ls -l /tmp/guestAdditions/
total 102510
dr-xr-xr-x   2 root     root        2048 Aug 16 23:22 32Bit
dr-xr-xr-x   2 root     root        2048 Aug 16 23:22 64Bit
-r-xr-xr-x   1 root     root         647 Jul 23 01:50 AUTORUN.INF
-r-xr-xr-x   1 root     root        6381 Feb  5  2016 autorun.sh
dr-xr-xr-x   2 root     root        2048 Aug 16 23:22 cert
dr-xr-xr-x   2 root     root        4096 Aug 16 23:22 OS2
-r-xr-xr-x   1 root     root        4824 Oct 21  2015 runasroot.sh
-r-xr-xr-x   1 root     root     8109503 Aug 17 00:18 VBoxLinuxAdditions.run
-r-xr-xr-x   1 root     root     17738240 Aug 17 00:19 VBoxSolarisAdditions.pkg
-r-xr-xr-x   1 root     root     16351360 Aug 17 00:22 VBoxWindowsAdditions-amd64.exe
-r-xr-xr-x   1 root     root     9993520 Aug 17 00:18 VBoxWindowsAdditions-x86.exe
-r-xr-xr-x   1 root     root      268640 Aug 17 00:17 VBoxWindowsAdditions.exe
```

_To sucessfully mount it (I would like to stress it again that directory /tmp/guestAdditions should exist and have proper permissions for operation to succeed)_

## Install Virtualbox Guest Additions
```sh
pkgadd -d VBoxSolarisAdditions.pkg
```

You should see something like this at the end,

> Installation of <SUNWvboxguest> was successful.

