# LVM on readonly block device


in all the cases below, `udev` includes a rule that allows the block device to be switched to read-only mode as soon as it is discovered, i.e. even before it is used by other processes :

```bash
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
