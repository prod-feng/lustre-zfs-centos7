# lustre-zfs-centos7
Install Lustre on CentOS 7, with backfstype of zfs

The following lists the steps I used to install Lustre 2.8 and zfs-0.6.4.2 on CentOS Linux release 7.2.1511 (Core). I installed MDT and OST services on a single computer, my laptop, with USB drives as devices. NO Guarrantee!

After the whole installation process, the kernel should have been then upgraded to:
>[root@new-host feng]# uname -a
>Linux new-host 3.10.0-327.3.1.el7_lustre.x86_64 #1 SMP Thu Feb 18 10:53:23 PST 2016 x86_64 x86_64 x86_64 GNU/Linux

The orignal kernel on my laptop is 3.10.0-123.el7.x86_64.

The hard drives I used are actually 3(three) 16GB USB drives,  connected to my computer with a USB hub with 7 ports, which work fine.

First the Linux kernel needs to be updated with Lustre patches. The RPM packages for server and client can be found here: https://downloads.hpdd.intel.com/public/lustre/latest-feature-release/

The list of packages needed to install on server/client from here: https://build.hpdd.intel.com/job/lustre-manual/lastSuccessfulBuild/artifact/lustre_manual.xhtml#table_cnh_5m3_gk


#1). Install Lustre patched kernel:

>]#rpm -ivh kernel-3.10.0-327.3.1.el7_lustre.x86_64.rpm

After completion, reboot to the new  kernel.

#2). Install ZFS package.

The ZFS packages and information can be found here: http://zfsonlinux.org/. I chose DKMS package, as described here: https://openzfs.github.io/openzfs-docs/Developer%20Resources/Custom%20Packages.html

The RPM packages of most recent version zfs-0.6.5.8 seems do not work well with Lustre 2.8(Can not load osd-zfs module, so can not start Lustre at the end). Maybe rebuild the Lustre osd-zfs RPMs can solve the problem, but I chose to install an older versio: zfs-0.6.4.2.

After setting as recommended, it's time to install ZFS:

>]#yum install zfs-0.6.4.2

If everything goes fine, the ZFS should work now. Run following commands to verify:

>]# lsmod |grep zfs

to see if the module has been loaded to the kernel. If so, run:

>]# zpool status

or

>]#zpool list

to chekc whether ZFS works fine. Since no ZFS storage has been made yet, it supposes not to print out anything, and that's fine.

#3). Install Lustre packages.

I chose to install ZFS on step 2 because some of Lustre packages need ZFS. Now install Lustre packages:

>]#rpm -ivh lustre-2.8.0-3.10.0_327.3.1.el7_lustre.x86_64.x86_64.rpm  lustre-modules-2.8.0-3.10.0_327.3.1.el7_lustre.x86_64.x86_64.rpm lustre-osd-zfs-2.8.0-3.10.0_327.3.1.el7_lustre.x86_64.x86_64.rpm lustre-osd-zfs-mount-2.8.0-3.10.0_327.3.1.el7_lustre.x86_64.x86_64.rpm

Also for test purpose, I installed LDISKFS module:

>]#rpm -ivh lustre-osd-ldiskfs-2.8.0-3.10.0_327.3.1.el7_lustre.x86_64.x86_64.rpm lustre-osd-ldiskfs-mount-2.8.0-3.10.0_327.3.1.el7_lustre.x86_64.x86_64.rpm

Other packages I installed include:

>]#rpm -ivh e2fsprogs-1.42.13.wc5-7.el7.x86_64.rpm e2fsprogs-libs-1.42.13.wc5-7.el7.x86_64.rpm libcom_err-1.42.13.wc5-7.el7.x86_64.rpm libss-1.42.13.wc5-7.el7.x86_64.rpm


If everything goes well, the Lustre and ZFS should work. Run command:

>]#lsmod |grep lustre

to check wheter Lustre module has been loaded into the kernel. If not, run:

>]#modprobe lustre

to load Lustre module.

#4). Make lustre storage with LDISKFS.

First I used the most regular LDSIKFS to make the storage:

>]#mkfs.lustre --mdt --mgs  --fsname=mylustre  --index=0  /dev/sdb

```text
   Permanent disk data:
Target:     mylustre:MDT0000
Index:      0
Lustre FS:  mylustre
Mount type: ldiskfs
Flags:      0x65
(MDT MGS first_time update )
Persistent mount opts: user_xattr,errors=remount-ro
Parameters:

checking for existing Lustre data: not found
device size = 14800MB
formatting backing filesystem ldiskfs on /dev/sdb
target name  mylustre:MDT0000
4k blocks     3788800
options        -J size=592 -I 512 -i 2048 -q -O dirdata,uninit_bg,^extents,dir_nlink,quota,huge_file,flex_bg -E lazy_journal_init -F
mkfs_cmd = mke2fs -j -b 4096 -L mylustre:MDT0000  -J size=592 -I 512 -i 2048 -q -O dirdata,uninit_bg,^extents,dir_nlink,quota,huge_file,flex_bg -E lazy_journal_init -F /dev/sdb 3788800
Writing CONFIGS/mountdata

```

Then, install OST:

[root@new-host Downloads]# mkfs.lustre --ost  --fsname=mylustre --mgsnode=192.168.1.5  --index=1  /dev/sdc

```text
   Permanent disk data:
Target:     mylustre:OST0001
Index:      1
Lustre FS:  mylustre
Mount type: ldiskfs
Flags:      0x62
(OST first_time update )
Persistent mount opts: ,errors=remount-ro
Parameters: mgsnode=192.168.1.5@tcp

checking for existing Lustre data: not found
device size = 14800MB
formatting backing filesystem ldiskfs on /dev/sdc
target name  mylustre:OST0001
4k blocks     3788800
options        -J size=400 -I 256 -i 69905 -q -O extents,uninit_bg,dir_nlink,quota,huge_file,flex_bg -G 256 -E resize=4290772992,lazy_journal_init -F
mkfs_cmd = mke2fs -j -b 4096 -L mylustre:OST0001  -J size=400 -I 256 -i 69905 -q -O extents,uninit_bg,dir_nlink,quota,huge_file,flex_bg -G 256 -E resize=4290772992,lazy_journal_init -F /dev/sdc 3788800
Writing CONFIGS/mountdata

```

Looks like everything is fine, then I can try to start my Lustre server.

Befote start, let's add the following lines in to /etc/ldev.conf:
```text
new-host  - mylustre-MDT0000 /dev/sdb
new-host  - mylustre-OST0001 /dev/sdc
```
Also,  in file /etc/modprobe.d/lustre.conf:
```text
options lnet networks="tcp0(wlp4s0)"
```
Here wlp4s0 is my wireless NIC.

Now start Lustre  storage system.

>[root@new-host Downloads]# systemctl start lustre

If no error message, then it's a good sign. Now let me check the Lustre server status. I can see the local mount of the MDT and OST:

>[root@new-host Downloads]# df -h

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       9.8G   68M  9.7G   1% /

...

/dev/sdb         11G   46M  9.5G   1% /mnt/lustre/local/mylustre-MDT0000
/dev/sdc         15G   42M   14G   1% /mnt/lustre/local/mylustre-OST0001
```

OK, it's working. Now I can mount the Luster storage. Since I am trying to mount it on the server, I do not need the cliet software yet.

First, add one line to /etc/fstab file:

>192.168.1.5@tcp:/mylustre /mylustre lustre defaults,_netdev,user_xattr 0 0

Now mount it:

>[root@new-host Downloads]# mount /mylustre/

>[root@new-host Downloads]# df -h

```text
 ...
/dev/sdb                    11G   46M  9.5G   1% /mnt/lustre/local/mylustre-MDT0000
/dev/sdc                    15G   42M   14G   1% /mnt/lustre/local/mylustre-OST0001
192.168.1.5@tcp:/mylustre   15G   42M   14G   1% /mylustre
```

Greate, it'sworking.

Now I stop the Lustre storage to go to next step: Lustre with ZFS.

>]#systemctl stop lustre
 
 
#5). Install Lustre Storage usingg ZFS.
 
Make MDT:
 
>[root@new-host feng]#  mkfs.lustre --mdt --mgs  --backfstype=zfs --fsname=mylustre  --mgsnode=192.168.1.5  --index=0 zfspool/mds /dev/sdb

```text
   Permanent disk data:
Target:     mylustre:MDT0000
Index:      0
Lustre FS:  mylustre
Mount type: zfs
Flags:      0x65
   (MDT MGS first_time update )
Persistent mount opts:
Parameters: mgsnode=192.168.1.5@tcp

checking for existing Lustre data: not found
mkfs_cmd = zpool create -f -O canmount=off zfspool /dev/sdb
mkfs_cmd = zfs create -o canmount=off -o xattr=sa zfspool/mds
Writing zfspool/mds properties
  lustre:version=1
  lustre:flags=101
   lustre:index=0
   lustre:fsname=mylustre
   lustre:svname=mylustre:MDT0000
   lustre:mgsnode=192.168.1.5@tcp
```

Now, OST:

>[root@new-host feng]# mkfs.lustre --ost --backfstype=zfs --fsname=mylustre --mgsnode=192.168.1.5  --index=1 zfspoolost/ost /dev/sdc /dev/sdd

```text
Permanent disk data:
Target:     mylustre:OST0001
Index:      1
Lustre FS:  mylustre
Mount type: zfs
Flags:      0x62
   (OST first_time update )
Persistent mount opts:
Parameters: mgsnode=192.168.1.5@tcp

checking for existing Lustre data: not found
mkfs_cmd = zpool create -f -O canmount=off zfspoolost /dev/sdc /dev/sdd
mkfs_cmd = zfs create -o canmount=off -o xattr=sa zfspoolost/ost
Writing zfspoolost/ost properties
lustre:version=1
   lustre:flags=98
   lustre:index=1
   lustre:fsname=mylustre
   lustre:svname=mylustre:OST0001
   lustre:mgsnode=192.168.1.5@tcp
```

Looks like everything is fine, then I can try to start my Lustre server.


Now I can check the ZFS status and Lustre status as:

>[root@new-host Downloads]# zpool list

```text
NAME         SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
zfspool     14.4G  2.72M  14.4G         -     0%     0%  1.00x  ONLINE  -
zfspoolost  28.8G  2.30M  28.7G         -     0%     0%  1.00x  ONLINE  -
```

Looks OK.

>[root@new-host Downloads]#  lfs df -h
>
```text
UUID                       bytes        Used   Available Use% Mounted on
mylustre-MDT0000_UUID       10.3G       45.9M        9.5G   0% /mylustre[MDT:0]
mylustre-OST0001_UUID       14.0G       41.1M       13.2G   0% /mylustre[OST:1]

filesystem summary:        14.0G       41.1M       13.2G   0% /mylustre
```

Befote start, let's add the following lines in to /etc/ldev.conf:

```text
new-host  - mylustre-MDT0000 zfs:zfspool/mds
new-host  - mylustre-OST0001 zfs:zfspoolost/ost
```

The file /etc/modprobe.d/lustre.conf should be the same as above.

Now, start it:

>[root@new-host feng]# systemctl restart lustre

If no error message, that means the Lustre storage, MDT and OSt services have been started succefully. Let's check:

>[root@new-host feng]# df -h

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       9.8G   68M  9.7G   1% /
...
zfspool/mds      15G  2.7M   15G   1% /mnt/lustre/local/mylustre-MDT0000
zfspoolost/ost   29G  1.8M   29G   1% /mnt/lustre/local/mylustre-OST0001
```

It is good, the MDT and OST storages have been mounted locally. Now, I can mount it now(the fstab is the same as above).


>[root@new-host Downloads]# mount /mylustre/

>[root@new-host Downloads]# df -h

```text
Filesystem                 Size  Used Avail Use% Mounted on
/dev/sda2                  9.8G   68M  9.7G   1% /
...
zfspool/mds                 15G  2.7M   15G   1% /mnt/lustre/local/mylustre-MDT0000
zfspoolost/ost              29G  2.3M   29G   1% /mnt/lustre/local/mylustre-OST0001
192.168.1.5@tcp:/mylustre   29G  2.3M   29G   1% /mylustre
 
 ```
 
It's done.
 
 
 
 
 
