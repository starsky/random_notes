# Introduction
I use quite successfull LVM setup on my laptop computer. I have Dell Latitude 7280 with Gentoo Linux. 
I have one phisical volume
```
~# pvdisplay

  --- Physical volume ---
  PV Name               /dev/nvme0n1p3
  VG Name               latitude_disc
  PV Size               931,13 GiB / not usable 3,69 MiB
  Allocatable           yes 
  PE Size               4,00 MiB
  Total PE              238369
  Free PE               79137
  Allocated PE          159232
```

And one volume group

```
~# vgdisplay

 --- Volume group ---
  VG Name               latitude_disc
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  113
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                5
  Open LV               5
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <931,13 GiB
  PE Size               4,00 MiB
  Total PE              238369
  Alloc PE / Size       159232 / 622,00 GiB
  Free  PE / Size       79137 / <309,13 GiB
```

# Logical Groups
The interesting thing starts in logical volumes:
```
~# lvs
LV             VG            Attr       LSize   Pool Origin Data%   
cache          latitude_disc -wi-ao----  15,00g                                                    
distfiles      latitude_disc -wi-ao----   5,00g                                                                  
gentoo         latitude_disc -wi-ao----  22,00g                                                    
home_encrypted latitude_disc -wi-ao---- 570,00g    
```

I have a root file system on `gentoo` lv, it has 22G. I keep also separete `cache` partition mounted on `/var/tmp`, and
`distfiles` partition mounted on `/var/cache/distfiles`.
As you may know Gentoo is compiling all its packages.
Thus `/var/tmp` is used as temp during the compilation.
`/var/cache/distfiles` is the placa where source files for packages that are being installed/upgraded are downloaded.

The `home_encrypted` partition is `/home` encrypted with LUKS.
I also hava `/boot` partition with ext2 file system outside of LVM.
One of the big advantage of LVM is that it allows resizing of logicala volumes, in the way that underlying file system sees it as continous space. Thus resizing is really easy operation that you can do even without rebooting the system. 

What I usually do I do not allocate whole space of my disc. But first I allocate some minimum space to `/` and `/home` partitions, and I resize them when needed. Cause at the beginning I do not know right size for my partitions.

# Snapshots
But the greates functionality that comes with LVM are snapshots.
How does it work? You can create a snapshot of given parition eg.:
```
lvcreate -L10G -s -n gentoo_update /dev/latitude_disc/gentoo
```
Here I create a snapshot named `gentoo_update` for `gentoo` logical volume. And I set 10G size for the snapshot. Now when you type `lvs`
```
  LV             VG            Attr       LSize   Pool Origin Data%                                                     
  gentoo         latitude_disc owi-aos---  22,00g                                                    
  gentoo_update  latitude_disc swi-a-s---  10,00g      gentoo 0,01
```
You can see that origin for `gentoo_update` is `gentoo` and that `0.01%` of the data was modified. 
Because the snapshot is basically keeping track of what changed with respect to origin volume. While the origin volume is left unchanged.

Once you have the snapshot you can do two useful things:

## Full partition backup
You can backup origin partition. You can do it on mounted read/write partition, because the origin partition is frozen, and changes are saved to the snapshot logical group.

It is maybe slightly non-ituitive, but you can mount the origin partition using path to the snapshot eg.
```
mount /dev/latitude_disc/gentoo_update /mnt
```
In the `/mnt` you would see the origin partition.
You can use this to backup the partition
```
dd if=/dev/latitude_disc/gentoo_update of=/some_backup_location/backup.img status=progress bs=64K conv=noerror,sync
```

## Checkpoint before system update
If you sometimes worry if your system would boot after the update. Or you are planning to apply some config changes that you are not sure about. You can use the snapshot mechanism.
Before applying any changes create the snapshot. Then change config or update system. You can reboot the system anc check if everything works fine. If you are sure you can remove the snapshot and merge all changes to the origin paritition.
```
lvremove /dev/latitude_disc/gentoo_update
```
Or if something went wrong you can revert changes and go back to the original state before the update.
```
lvconvert --mergesnapshot /dev/latitude_disc/gentoo_update
```
I know that mergesnapshot sounds counter intuitive, but it is like this.
After running this command all the changes will be reverted after the reboot in cae of root partition.

## Why I use such partitioning scheme
Now you may wonder why do I keep `/var/tmp` and `/var/cache/distfiles` as separate logical volumes.
I create a snapshot of the root partition befere each system update. Now during the update process the source files are downloaded to `/var/cache/distfiles` and the compilation temp files are saved to `/var/tmp`. If those folders were on the root partition, all the changes would have been stored to the snapshot delta. This would require me to allocate more space for the snapshot.

## Useful links
While working with LVM snapshots I found these links useful:
1. http://www.voleg.info/lvm2.html
2. https://www.claudiokuenzler.com/blog/1070/lvm-restore-logical-volume-merging-snapshot-will-occur-on-next-activation