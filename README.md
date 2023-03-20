# LVM on readonly block device


in all the cases below, `udev` includes a rule that allows the block device to be switched to read-only mode as soon as it is discovered, i.e. even before it is used by other processes :

```console
# cat /etc/udev/rules.d/09-rodev.rules 
KERNEL!="ram*", SUBSYSTEM=="block", RUN+="/usr/local/sbin/rodev /dev/%k"

# cat /usr/local/sbin/rodev
#!/usr/bin/bash
[ ! "${1}" ] && echo "error: device missing" >/dev/stderr && exit 1
[ "${2}" ] && echo "error: too many devices" >/dev/stderr && exit 1
[[ "${1}" =~ ^/dev/sda ]] && exit
chmod 440 "${1}"
blockdev --setro "${1}"
```



## Linux kernel 5.10.83-1-lts

`block/blk-core.c` is patched like this :

```diff
--- 5.10.19.original/blk-core.c 2023-03-15 13:44:20.176929833 +0100
+++ 5.10.19.me/blk-core.c	2023-03-15 13:44:02.353596114 +0100
@@ -706,7 +706,7 @@
        "Trying to write to read-only block-device %s (partno %d)\n",
  bio_devname(bio, b), part->partno);
  /* Older lvm-tools actually trigger this */
- return false;
+ return true;
  }
 
  return false;
  ```
  
  > see [here](https://github.com/vitaly-kamluk/Linux-write-blocker/tree/master/kernel) and [here](https://github.com/msuhanov/Linux-write-blocker) for more informations.


here are the test performed, the checksums are constant throughout the test: :

```console
# uname -r
5.10.83-1-lts

# lsblk -o+fstype 
NAME     MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS       FSTYPE
loop0      7:0    0  1.1G  1 loop                    
`-rootfs 254:0    0  1.1G  1 crypt /run/live/rootfs  squashfs
sda        8:0    0    4G  0 disk                    
|-sda1     8:1    0  2.8G  0 part  /mnt              ntfs
`-sda2     8:2    0  1.2G  0 part  /run/live/bootmnt vfat
sdb        8:16   0  256M  1 disk                    
|-sdb1     8:17   0   83M  1 part                    LVM2_member
|-sdb2     8:18   0   83M  1 part                    LVM2_member
`-sdb3     8:19   0   83M  1 part                    LVM2_member

# ll /dev/sdb*
br--r----- 1 root disk 8, 16 Mar 20 13:41 /dev/sdb
br--r----- 1 root disk 8, 17 Mar 20 13:41 /dev/sdb1
br--r----- 1 root disk 8, 18 Mar 20 13:41 /dev/sdb2
br--r----- 1 root disk 8, 19 Mar 20 13:41 /dev/sdb3

# flush(){ sync && sysctl -q vm.drop_caches=3; }

# flush && sha1sum /dev/sdb*
c96ad903be2573c805c5968df4bfdaa2c1e3adb4  /dev/sdb
a886843c556fcf8971b68d26c3bea9e734271af5  /dev/sdb1
27d5e7f8ccf998f40efeac1d2ab1b004227f717f  /dev/sdb2
7ccb5b44f0fb7c35be986612054db244f7213188  /dev/sdb3

# dd if=/dev/zero of=/dev/sdb1 count=1
dd: writing to '/dev/sdb1': Operation not permitted
1+0 records in
0+0 records out
0 bytes copied, 0.00704442 s, 0.0 kB/s

# flush && sha1sum /dev/sdb*
c96ad903be2573c805c5968df4bfdaa2c1e3adb4  /dev/sdb
a886843c556fcf8971b68d26c3bea9e734271af5  /dev/sdb1
27d5e7f8ccf998f40efeac1d2ab1b004227f717f  /dev/sdb2
7ccb5b44f0fb7c35be986612054db244f7213188  /dev/sdb3

# pacman -Rsn lvm2
error: target not found: lvm2

# rm -rf /etc/lvm 

# pacman -S lvm2
resolving dependencies...
looking for conflicting packages...

Packages (2) thin-provisioning-tools-0.9.0-1  lvm2-2.03.14-2

Total Download Size:   1.99 MiB
Total Installed Size:  7.81 MiB

:: Proceed with installation? [Y/n] 
:: Retrieving packages...
 lvm2-2.03.14-2-x86_64                         1547.2 KiB   571 KiB/s 00:03 [###########################################] 100%
 thin-provisioning-tools-0.9.0-1-x86_64         491.0 KiB   592 KiB/s 00:01 [###########################################] 100%
 Total (2/2)                                   2038.2 KiB   576 KiB/s 00:04 [###########################################] 100%
(2/2) checking keys in keyring                                              [###########################################] 100%
(2/2) checking package integrity                                            [###########################################] 100%
(2/2) loading package files                                                 [###########################################] 100%
(2/2) checking for file conflicts                                           [###########################################] 100%
(2/2) checking available disk space                                         [###########################################] 100%
:: Processing package changes...
(1/2) installing thin-provisioning-tools                                    [###########################################] 100%
(2/2) installing lvm2                                                       [###########################################] 100%
:: Running post-transaction hooks...
(1/4) Reloading system manager configuration...
(2/4) Reloading device manager configuration...
(3/4) Arming ConditionNeedsUpdate...
(4/4) Updating linux initcpios...

# pvdisplay 
  --- Physical volume ---
  PV Name               /dev/sdb1
  VG Name               test.lvm
  PV Size               <83.01 MiB / not usable <3.01 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              20
  Free PE               3
  Allocated PE          17
  PV UUID               mQR1eS-K5Zf-5lfl-Dupc-lRhN-Yfpf-7Vl36V
   
  --- Physical volume ---
  PV Name               /dev/sdb2
  VG Name               test.lvm
  PV Size               <83.01 MiB / not usable <3.01 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              20
  Free PE               3
  Allocated PE          17
  PV UUID               As2RrD-Cev0-hhB7-yohn-r5ym-P34m-MFV0M9
   
  --- Physical volume ---
  PV Name               /dev/sdb3
  VG Name               test.lvm
  PV Size               <83.01 MiB / not usable <3.01 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              20
  Free PE               3
  Allocated PE          17
  PV UUID               khaOAc-YEX4-bi0u-fdtq-doG9-Wuok-B2kCS8
   
# flush && sha1sum /dev/sdb*
c96ad903be2573c805c5968df4bfdaa2c1e3adb4  /dev/sdb
a886843c556fcf8971b68d26c3bea9e734271af5  /dev/sdb1
27d5e7f8ccf998f40efeac1d2ab1b004227f717f  /dev/sdb2
7ccb5b44f0fb7c35be986612054db244f7213188  /dev/sdb3

# vgdisplay 
  --- Volume group ---
  VG Name               test.lvm
  System ID             
  Format                lvm2
  Metadata Areas        3
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               0
  Max PV                0
  Cur PV                3
  Act PV                3
  VG Size               240.00 MiB
  PE Size               4.00 MiB
  Total PE              60
  Alloc PE / Size       51 / 204.00 MiB
  Free  PE / Size       9 / 36.00 MiB
  VG UUID               sttYAJ-VmEf-biZh-C8Uh-jUCK-orFO-WVNX6t
   
# flush && sha1sum /dev/sdb*
c96ad903be2573c805c5968df4bfdaa2c1e3adb4  /dev/sdb
a886843c556fcf8971b68d26c3bea9e734271af5  /dev/sdb1
27d5e7f8ccf998f40efeac1d2ab1b004227f717f  /dev/sdb2
7ccb5b44f0fb7c35be986612054db244f7213188  /dev/sdb3

# vgchange -ay
  1 logical volume(s) in volume group "test.lvm" now active

# flush && sha1sum /dev/sdb*
c96ad903be2573c805c5968df4bfdaa2c1e3adb4  /dev/sdb
a886843c556fcf8971b68d26c3bea9e734271af5  /dev/sdb1
27d5e7f8ccf998f40efeac1d2ab1b004227f717f  /dev/sdb2
7ccb5b44f0fb7c35be986612054db244f7213188  /dev/sdb3

# vgrename test.lvm lvm.test
  Error writing device /dev/sdb1 at 19968 length 3584.
  WARNING: bcache_invalidate: block (6, 0) still dirty.
  Failed to write metadata to /dev/sdb1.
  Failed to write VG lvm.test.

# flush && sha1sum /dev/sdb*
c96ad903be2573c805c5968df4bfdaa2c1e3adb4  /dev/sdb
a886843c556fcf8971b68d26c3bea9e734271af5  /dev/sdb1
27d5e7f8ccf998f40efeac1d2ab1b004227f717f  /dev/sdb2
7ccb5b44f0fb7c35be986612054db244f7213188  /dev/sdb3
```


## Linux kernel 6.1.15-1-lts

here are the same test performed, the checksums are identical to the previous ones in inputs but **have changed during the test** :

```console
# uname -r
6.1.15-1-lts

# lsblk -o+fstype
NAME     MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS       FSTYPE
loop0      7:0    0  1.6G  1 loop                    
`-rootfs 254:0    0  1.6G  1 crypt /run/live/rootfs  squashfs
sda        8:0    0  1.9G  0 disk                    
|-sda1     8:1    0  103M  0 part  /mnt              ntfs
`-sda2     8:2    0  1.8G  0 part  /run/live/bootmnt vfat
sdb        8:16   0  256M  1 disk                    
|-sdb1     8:17   0   83M  1 part                    LVM2_member
|-sdb2     8:18   0   83M  1 part                    LVM2_member
`-sdb3     8:19   0   83M  1 part                    LVM2_member

# ll /dev/sdb*
br--r----- 1 root disk 8, 16 Mar 20 15:07 /dev/sdb
br--r----- 1 root disk 8, 17 Mar 20 15:07 /dev/sdb1
br--r----- 1 root disk 8, 18 Mar 20 15:07 /dev/sdb2
br--r----- 1 root disk 8, 19 Mar 20 15:07 /dev/sdb3

# flush(){ sync && sysctl -q vm.drop_caches=3; }

# flush && sha1sum /dev/sdb*
c96ad903be2573c805c5968df4bfdaa2c1e3adb4  /dev/sdb
a886843c556fcf8971b68d26c3bea9e734271af5  /dev/sdb1
27d5e7f8ccf998f40efeac1d2ab1b004227f717f  /dev/sdb2
7ccb5b44f0fb7c35be986612054db244f7213188  /dev/sdb3

# dd if=/dev/zero of=/dev/sdb1 count=1
dd: writing to '/dev/sdb1': Operation not permitted
1+0 records in
0+0 records out
0 bytes copied, 0.00159138 s, 0.0 kB/s

# flush && sha1sum /dev/sdb*
c96ad903be2573c805c5968df4bfdaa2c1e3adb4  /dev/sdb
a886843c556fcf8971b68d26c3bea9e734271af5  /dev/sdb1
27d5e7f8ccf998f40efeac1d2ab1b004227f717f  /dev/sdb2
7ccb5b44f0fb7c35be986612054db244f7213188  /dev/sdb3

# pacman -Rsn lvm2
checking dependencies...
:: e2fsprogs optionally requires lvm2: for e2scrub

Packages (2) thin-provisioning-tools-1.0.2-1  lvm2-2.03.19-1

Total Removed Size:  8.97 MiB

:: Do you want to remove these packages? [Y/n] 
:: Processing package changes...
(1/2) removing lvm2                                                         [###########################################] 100%
(2/2) removing thin-provisioning-tools                                      [###########################################] 100%
:: Running post-transaction hooks...
(1/3) Reloading system manager configuration...
(2/3) Reloading device manager configuration...
(3/3) Arming ConditionNeedsUpdate...

# rm -rf /etc/lvm

# pacman -S lvm2
resolving dependencies...
looking for conflicting packages...

Packages (2) thin-provisioning-tools-1.0.2-1  lvm2-2.03.19-1

Total Download Size:   2.76 MiB
Total Installed Size:  8.97 MiB

:: Proceed with installation? [Y/n] 
:: Retrieving packages...
 lvm2-2.03.19-1-x86_64                         1830.3 KiB   496 KiB/s 00:04 [###########################################] 100%
 thin-provisioning-tools-1.0.2-1-x86_64         994.9 KiB   562 KiB/s 00:02 [###########################################] 100%
 Total (2/2)                                      2.8 MiB   517 KiB/s 00:05 [###########################################] 100%
(2/2) checking keys in keyring                                              [###########################################] 100%
(2/2) checking package integrity                                            [###########################################] 100%
(2/2) loading package files                                                 [###########################################] 100%
(2/2) checking for file conflicts                                           [###########################################] 100%
(2/2) checking available disk space                                         [###########################################] 100%
:: Processing package changes...
(1/2) installing thin-provisioning-tools                                    [###########################################] 100%
(2/2) installing lvm2                                                       [###########################################] 100%
:: Running post-transaction hooks...
(1/4) Reloading system manager configuration...
(2/4) Reloading device manager configuration...
(3/4) Arming ConditionNeedsUpdate...
(4/4) Updating linux initcpios...

# pvdisplay 
  --- Physical volume ---
  PV Name               /dev/sdb1
  VG Name               test.lvm
  PV Size               <83.01 MiB / not usable <3.01 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              20
  Free PE               3
  Allocated PE          17
  PV UUID               mQR1eS-K5Zf-5lfl-Dupc-lRhN-Yfpf-7Vl36V
   
  --- Physical volume ---
  PV Name               /dev/sdb2
  VG Name               test.lvm
  PV Size               <83.01 MiB / not usable <3.01 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              20
  Free PE               3
  Allocated PE          17
  PV UUID               As2RrD-Cev0-hhB7-yohn-r5ym-P34m-MFV0M9
   
  --- Physical volume ---
  PV Name               /dev/sdb3
  VG Name               test.lvm
  PV Size               <83.01 MiB / not usable <3.01 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              20
  Free PE               3
  Allocated PE          17
  PV UUID               khaOAc-YEX4-bi0u-fdtq-doG9-Wuok-B2kCS8
   

# flush && sha1sum /dev/sdb*
c96ad903be2573c805c5968df4bfdaa2c1e3adb4  /dev/sdb
a886843c556fcf8971b68d26c3bea9e734271af5  /dev/sdb1
27d5e7f8ccf998f40efeac1d2ab1b004227f717f  /dev/sdb2
7ccb5b44f0fb7c35be986612054db244f7213188  /dev/sdb3

# vgdisplay 
  --- Volume group ---
  VG Name               test.lvm
  System ID             
  Format                lvm2
  Metadata Areas        3
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               0
  Max PV                0
  Cur PV                3
  Act PV                3
  VG Size               240.00 MiB
  PE Size               4.00 MiB
  Total PE              60
  Alloc PE / Size       51 / 204.00 MiB
  Free  PE / Size       9 / 36.00 MiB
  VG UUID               sttYAJ-VmEf-biZh-C8Uh-jUCK-orFO-WVNX6t
   

# flush && sha1sum /dev/sdb*
c96ad903be2573c805c5968df4bfdaa2c1e3adb4  /dev/sdb
a886843c556fcf8971b68d26c3bea9e734271af5  /dev/sdb1
27d5e7f8ccf998f40efeac1d2ab1b004227f717f  /dev/sdb2
7ccb5b44f0fb7c35be986612054db244f7213188  /dev/sdb3

# vgchange -ay
  1 logical volume(s) in volume group "test.lvm" now active

# flush && sha1sum /dev/sdb*
9ce7c5288aea317d83f9f62abc97ecb6547876ca  /dev/sdb
2f5344f5bde920d1998b26999668b9b035c67555  /dev/sdb1
8bd428ec90d69967d4264051c0ed8c13f67a38df  /dev/sdb2
ead9e3b31ae5929b7181bc464497a0b5a832d868  /dev/sdb3

# vgrename test.lvm lvm.test
  Error writing device /dev/sdb1 at 19968 length 3584.
  WARNING: bcache_invalidate: block (6, 0) still dirty.
  Failed to write metadata to /dev/sdb1.
  Failed to write VG lvm.test.

# flush && sha1sum /dev/sdb*
9ce7c5288aea317d83f9f62abc97ecb6547876ca  /dev/sdb
2f5344f5bde920d1998b26999668b9b035c67555  /dev/sdb1
8bd428ec90d69967d4264051c0ed8c13f67a38df  /dev/sdb2
ead9e3b31ae5929b7181bc464497a0b5a832d868  /dev/sdb3

# vgchange -an
  0 logical volume(s) in volume group "test.lvm" now active

# flush && sha1sum /dev/sdb*
22d8aa2dbc57deb5ae11d1fc82dc27f1a6c1bdfe  /dev/sdb
c613bf2a8c92e4c7e94c60af6139ade64203c5a2  /dev/sdb1
2e23fa0857326c07977aa171a4c354ca26db4e2e  /dev/sdb2
6a17c8f40bea416fe93ff3bdc11701adc7115494  /dev/sdb3
```
