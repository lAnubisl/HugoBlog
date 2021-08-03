---
title: "My current command to format hard drive for linux NAS"
date: 2021-08-03T21:22:05+03:00
draft: false
Summary: "Put it here to just not forget =) Here is my command to format my hard drives for NAS. I came to this when I bought 2 10TB hard drives for Chia coin mining.
``` bash
sudo mkfs.ext4 /dev/sde1 -T largefile4 -m 0
```"
---

Put it here to just not forget =) Here is my command to format my hard drives for NAS. I came to this when I bought 2 10TB hard drives for Chia coin mining.

``` bash
sudo mkfs.ext4 /dev/sde1 -T largefile4 -m 0
```

The **-T largefile4** flag adjusts the amount of inodes that are allocated at the creation of the file system to 1 inode per 4 MB of free space. The less inodes you have the less free space they take. However the less inodes you have the less files you can store in your file system. It is not a problem in case if you have large files.

The **-m 0** flag prevents super-user reserved blocks. By default, 5% of the filesystem blocks will be reserved for the super-user, to avoid fragmentation and "allow root-owned daemons to continue to function correctly after non-privileged processes are prevented from writing to the filesystem"

[[Source](https://wiki.archlinux.org/title/ext4)]