# Solaris-Zones
How to create Zones &amp; Zone Clusters in Solaris

# Solaris version
```sh
uname -n
cat /etc/release
```

## List of disks in the server
`iostat -En`

#### To know where each slice is mounted
`cat /etc/vfstab`


#### Find the CD-ROM device name using the iostat command.
###### Ref [1] - https://docs.oracle.com/cd/E10926_01/doc/owb.101/b12150/appbcdmount.htm
```sh
iostat -En
## c0d0             Soft Errors: 0 Hard Errors: 0 Transport Errors: 0
## Model: VBOX HARDDISK   Revision:  Serial No: VB8abc1378-4a46 Size: 17.18GB <17179803648 bytes>
## Media Error: 0 Device Not Ready: 0 No Device: 0 Recoverable: 0
## Illegal Request: 0
## c1t0d0           Soft Errors: 0 Hard Errors: 0 Transport Errors: 0
## Vendor: VBOX     Product: CD-ROM           Revision: 1.0  Serial No:
## Size: 2.25GB <2254110720 bytes>
## Media Error: 0 Device Not Ready: 0 No Device: 0 Recoverable: 0
## Illegal Request: 8 Predictive Failure Analysis: 0
## The CD-ROM device name is c1t0d0.
## Mount the CD-ROM
mount -r -F hsfs /dev/dsk/c1t0d0s0 /cdrom
```

### Create State Database Replicas for the SOLARIS VOLUME MANAGER
###### Ref [1] - http://docs.oracle.com/cd/E18752_01/html/816-4520/tasks-metadevices-25.html#addtasks-17877
###### Adding Two State Database Replicas to the Same Slice
```sh
metadb -a -f c0t5000C5000A914A9Bd0s3
metadb -a -f c0t5000C50012BC6A0Bd0s6
```

### Create soft partitions
##### Lets say you want to store all zone data in the `Global Zone` & all `Non-Global Zone` separately. You can create meta devices from different disk slices. You MAY also create a mirror/stripe RAID using SVM for your zone paths and application data. Here we will just use slices, named as metadisk `d[1-3]` for OS data & corresponding application metadisks named as `d[1-3][1-3]`.
```sh
metainit d1 -p c0t5000C5000A914A9Bd0s3 30g
metainit d2 -p c0t5000C5000A914A9Bd0s3 30g
metainit d3 -p c0t5000C5000A914A9Bd0s3 30g

metainit d11 -p c0t5000C50012BC6A0Bd0s6 48g
metainit d22 -p c0t5000C50012BC6A0Bd0s6 100g
metainit d33 -p c0t5000C50012BC6A0Bd0s6 130g
```

### Create the UFS filesytems for root and data
```sh
newfs /dev/md/rdsk/d1
newfs /dev/md/rdsk/d2
newfs /dev/md/rdsk/d3

newfs /dev/md/rdsk/d11
newfs /dev/md/rdsk/d22
newfs /dev/md/rdsk/d33
```

### Mount the filesystems under /export
```sh
cd /export
mkdir webzone appzone dbzone
mount -F ufs -o logging /dev/md/dsk/d1 /export/webzone
mount -F ufs -o logging /dev/md/dsk/d2 /export/appzone
mount -F ufs -o logging /dev/md/dsk/d3 /export/dbzone
```

### To finish, make sure to add under /etc/vfstab
```sh
/dev/md/dsk/d1 /dev/md/rdsk/d1 /export/webzone ufs 2 yes logging
/dev/md/dsk/d2 /dev/md/rdsk/d2 /export/appzone ufs 2 yes logging
/dev/md/dsk/d3 /dev/md/rdsk/d3 /export/dbzone ufs 2 yes logging
```



### Creating a Striped Metadevice of Two Slices With a 32 Kbyte Interlace
###### The striped metadevice, d10, consists of a single stripe (the number 1) made of two slices (the number 2). The -i option sets the interlace to 32 Kbytes. (The interlace cannot be less than 8 Kbytes, nor greater than 100 Mbytes.) If interlace were not specified, the striped metadevice would use the default of 16 Kbytes. The system verifies that the Concat/Stripe object has been set up.
```sh
metainit d10 1 1 c0t5000C5000A914A9Bd0s3 -i 32k
## d10: Concat/Stripe is setup

metattach d10 c0t5000C50012BC6A0Bd0s6
## d10: component is attached
```

##### If YOU wanted to check status of the metadb and the size of the new disk slice
```sh
metastat
metastat -p d10
## d10 2 1 /dev/dsk/c0t5000C5000A914A9Bd0s3 \
##          1 /dev/dsk/c0t5000C50012BC6A0Bd0s6
metadb -i
```

### Create an UFS filesystem on this metadevice
###### Ref [1] - https://setaoffice.com/2010/08/08/creating-a-svm-metadevice-and-an-ufs-filesystem/
###### Ref [2] - http://docs.oracle.com/cd/E18752_01/html/817-5093/fscreate-1.html#fscreate-22821
`newfs /dev/md/rdsk/d10`

### Deleting metaslice, db, state
```sh
# metaclear d10
# metadb -d -f /dev/dsk/c0t5000C5000A914A9Bd0s3
# metadb -d -f /dev/dsk/c0t5000C50012BC6A0Bd0s6
```

### List all the zones in the server
```sh
zoneadm list vi
zoneadm list cv
```

### Check the name of the zoneadm
`zonename`

### Creating a zone
####### Ref [1] - http://www.datadisk.co.uk/html_docs/sun/solaris_zones.htm
####### Ref [2] - https://docs.oracle.com/cd/E36784_01/html/E36871/zonecfg-1m.html#REFMAN1Mzonecfg-1m

```sh
zonecfg -z webzone
## webzone: No such zone configured
## Use 'create' to begin configuring a new zone.
zonecfg:webzone> create
zonecfg:webzone> set zonepath=/zones/webzone
zonecfg:webzone> set autoboot=true

zonecfg:webzone> add net
zonecfg:webzone:net> set address=10.10.0.7
zonecfg:webzone:net> set physical=igb0
zonecfg:webzone:net> end

zonecfg:my-zone3> add fs 
zonecfg:my-zone3:fs> set dir=/webdata
zonecfg:my-zone3:fs> set special=/dev/md/dsk/d11
zonecfg:my-zone3:fs> set raw=/dev/md/rdsk/d11
zonecfg:my-zone3:fs> set type=ufs
zonecfg:my-zone3:fs> add options [logging, nosuid]
zonecfg:my-zone3:fs> end

zonecfg:webzone> info
## zonename: webzone
## zonepath: /zones/webzone
## brand: native
## autoboot: true
## bootargs:
## pool:
## limitpriv:
## scheduling-class:
## ip-type: shared
## hostid:
## inherit-pkg-dir:
##         dir: /lib
## inherit-pkg-dir:
##         dir: /platform
## inherit-pkg-dir:
##         dir: /sbin
## inherit-pkg-dir:
##         dir: /usr

zonecfg:webzone> verify
zonecfg:webzone> commit
zonecfg:webzone> end
zonecfg:webzone> exit
```
	
### Check state of the newly created zone
```sh
zoneadm list -cv
zoneadm -z webzone install
zoneadm -z webzone ready
zoneadm -z webzone boot
```

### Login to the zone console
`zlogin -C webzone`

### Use zlogin to Shut Down a Zone
`zlogin webzone shutdown -i 0`

### How to Exit a Non-Global Zone
`zonename# ~.`

### Halting, Uninstalling & Deleting a zone
```sh
oneadm –z webzone halt
zoneadm –z webzone uninstall -F
zoneadm –z webzone delete -F
```

### Delete a Zone Configuration
`zonecfg -z webzone delete -F`

### Export current zone configuration
```sh
cd /export && mkdir zone-configs
zonecfg -z webzone export -f /export/zone-configs/webzone.cfg
```

### Automating Zone Creation

#### Cloning a zone from configuration file
```sh
zonecfg -z appzone -f /export/zone-configs/appzone.cfg
zonecfg -z dbzone -f /export/zone-configs/dbzone.cfg
```

##### Enable ssh root login in Solaris 10 (NOT ADVISABLE)
##### Change the file /etc/ssh/sshd_config with PermitRootLogin yes to replace PermitRootLogin no
##### Restart the services
`svcadm restart svc:/network/ssh:default`




