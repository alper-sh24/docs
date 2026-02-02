# Extending Linux Partition

On a fres **ubuntu-24.04.3-live-server-amd64** installation, I had a problem with the OS not using the whole available disk space on my system.

## Problem

Running `lsblk` in terminal would output:

```txt
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0 894,2G  0 disk
├─sda1                      8:1    0     1G  0 part /boot/efi
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0 891,2G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0   100G  0 lvm  /
sdb                         8:16   1     0B  0 disk
```

Indicating that I would be using **100gb** of **891,2gb** available.

Running `lvdisplay` also shows:

```txt
--- Logical volume ---
  LV Path                /dev/ubuntu-vg/ubuntu-lv
  LV Name                ubuntu-lv
  VG Name                ubuntu-vg
  LV UUID                yGR2Jq-iSQT-tFrm-686K-cLiY-Il3B-JYVSLD
  LV Write Access        read/write
  LV Creation host, time ubuntu-server, 2026-02-02 14:25:28 +0000
  LV Status              available
  # open                 1
  LV Size                100,00 GiB
  Current LE             25600
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     1024
  Block device           252:0
```

## Solution

Referencing [How to Extend LVM Partition with lvextend Command in Linux](https://www.linuxtechi.com/extend-lvm-partitions/) post by [Pradeep Kumar](https://www.linuxtechi.com/author/pradeep/)

To get all the free available space, I ran `sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv`. It is possible that if this does not work, an alternative would have been `lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`

Then you get this result:

```txt
  Size of logical volume ubuntu-vg/ubuntu-lv changed from 100,00 GiB (25600 extents) to <891,17 GiB (228139 extents).
  Logical volume ubuntu-vg/ubuntu-lv successfully resized.
```

then you finally resize with: `sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv` (alternatively `sudo resize2fs /dev/ubuntu-vg/ubuntu-lv`)

finally, you get:

```txt
Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizing required
old_desc_blocks = 13, new_desc_blocks = 112
The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 233614336 (4k) blocks long.
```

And if you run `lsblk` again:

```txt
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0 894,2G  0 disk
├─sda1                      8:1    0     1G  0 part /boot/efi
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0 891,2G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0 891,2G  0 lvm  /
sdb
```

All is well!
