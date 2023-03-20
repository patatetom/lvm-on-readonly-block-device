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
