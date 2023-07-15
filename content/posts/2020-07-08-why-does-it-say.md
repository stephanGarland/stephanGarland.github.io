---
title: 'Why does it say paper jam when there is no paper jam?!'
date: Wed, 08 Jul 2020 19:36:52 +0000
draft: false
tags: ['mergerfs', 'docker']
---

OK, this is actually about Docker and disk space, but close enough. While trying to download ultra high definition Linux ISOs, I encountered an error from Transmission, running in Docker: "No space left on device." Odd, considering I was pretty sure I had a few TB free. Let's see if the logs have anything useful to say (also, yes, I checked around the lines in question to see if there was other information):

```
❯ docker logs transmission-LINUX_ISOs 2>&1 | grep space
[2020-07-07 20:58:16.730] Couldn't open "/downloads/incomplete/LINUX_ISO.part": No space left on device (/home/buildozer/aports/community/transmission/src/transmission-3.00/libtransmission/fdlimit.c:195)
[2020-07-07 20:58:16.730] LINUX_ISO tr_fdFileCheckout failed for "/downloads/incomplete/LINUX_ISO.part": No space left on device (/home/buildozer/aports/community/transmission/src/transmission-3.00/libtransmission/inout.c:95)
[2020-07-07 20:58:16.730] LINUX_ISO No space left on device (/downloads/4K/LINUX_ISO) (/home/buildozer/aports/community/transmission/src/transmission-3.00/libtransmission/torrent.c:574)
```

I followed the source code listed, and found that it's just trying to allocate file descriptors, which makes sense. OK, I was pretty sure I had available space, but let's check.

```
❯ dfd
# alias dfd="df -h | awk '/1:2:3:4|data|Filesystem/'"
Filesystem      Size  Used Avail Use% Mounted on
1:2:3:4          29T   25T  4.1T  86% /media/merged
/dev/sdf1       7.1T  6.7T  455G  94% /media/data2
/dev/sde1       7.2T  6.6T  636G  92% /media/data1
/dev/sdh1       7.3T  6.4T  905G  88% /media/data4
/dev/sdg1       7.3T  4.8T  2.1T  70% /media/data3
```

OK, how about inodes?

```
❯ df -hi
Filesystem     Inodes IUsed IFree IUse% Mounted on
udev             2.0M   504  2.0M    1% /dev
tmpfs            2.0M  1.1K  2.0M    1% /run
/dev/md0          14M  890K   14M    7% /
tmpfs            2.0M     1  2.0M    1% /dev/shm
tmpfs            2.0M     3  2.0M    1% /run/lock
tmpfs            2.0M    17  2.0M    1% /sys/fs/cgroup
1:2:3:4          1.4G   94K  1.4G    1% /media/merged
/dev/sdf1        462M  5.8K  462M    1% /media/data2
/dev/sde1        466M   42K  466M    1% /media/data1
/dev/sdc1        292M    12  292M    1% /media/par1
/dev/sdh1        233M   36K  233M    1% /media/data4
/dev/sdg1        233M   12K  233M    1% /media/data3
/dev/sdb1        292M    12  292M    1% /media/par2
tmpfs            2.0M    11  2.0M    1% /run/user/1000
```

Wait, what if the docker daemon is out of room?

```
❯ df -h /var/lib/docker
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0        218G   32G  176G  16% /
```

Guess I can try restarting the container... nope, that didn't fix it either.

I found some articles discussing container limits with earlier versions of Docker, but I'm using the latest, with overlay2, and besides, the container's size isn't my issue - I think.

Let's check volume mounts; maybe there are a bunch of stuck torrents cluttering up /incomplete or something.

```
❯ docker inspect transmission-LINUX_ISOS | jq -r '.[].Mounts'
[
  {
    "Type": "bind",
    "Source": "/$HOME/.transmission-LINUX_ISOS",
    "Destination": "/config",
    "Mode": "",
    "RW": true,
    "Propagation": "rprivate"
  },
  {
    "Type": "bind",
    "Source": "/media/merged/LINUX_ISOS",
    "Destination": "/downloads",
    "Mode": "",
    "RW": true,
    "Propagation": "rprivate"
  },
  {
    "Type": "bind",
    "Source": "/media/merged/LINUX_ISOS/incomplete",
    "Destination": "/incomplete",
    "Mode": "",
    "RW": true,
    "Propagation": "rprivate"
  },
  {
    "Type": "bind",
    "Source": "/media/merged/LINUX_ISOS/torrents",
    "Destination": "/watch",
    "Mode": "",
    "RW": true,
    "Propagation": "rprivate"
  }
]
```

```
# Yes, you could just ls the volume mount as well
❯ docker exec -it transmission-LINUX_ISOS /bin/bash
root@1fa057d96dcf:/# ls /incomplete
```

I got as far as joining LinuxServer.io's Discord and typing out my symptoms and what I had done, when, as typing often does, I hit upon a thought: I'm using [mergerfs](https://github.com/trapexit/mergerfs) to join my data disks. I have minfreespace quotas set up for the data disks. What if one of those was at or near the minimum?

```
❯ tail -1 /etc/fstab
/media/data* /media/merged fuse.mergerfs defaults,nonempty,allow_other,use_ino,minfreespace=500G 0 0
```

What did my initial df show again?

```
/dev/sdf1       7.1T  6.7T  455G  94% /media/data2
```

Ah ha. Now, what's interesting is that even at a second glance, there shouldn't be an issue. I have a JBOD of 4x data disks under mergerfs, and for this particular file, it was being saved to /media/merged/movies/4K/ (I may as well drop the charade of Linux ISOs). That particular path only exists on /media/data1, which showed as 636G free. However, /media/merged/movies/incomplete, the temporary location for torrents, only exists on - you guessed it - /media/data2. Thus, ENOSPC was returned, and the transfer stopped. Amusingly, mergerfs' README [covers this issue](https://github.com/trapexit/mergerfs/tree/b1f30b703f794513f64b54af6a3b4f7553134a48#why-do-i-get-an-out-of-space--no-space-left-on-device--enospc-error-even-though-there-appears-to-be-lots-of-space-available) - that'll teach me to not turn to documentation earlier.

The short-term fix is to rebalance, with `mergerfs.balance`, which runs rsync to distribute files around more evenly. The medium-term fix is to add `moveonenospc` to my mount options. The long-term fix is to move off of mergerfs, because while it and SnapRAID are great tools for what they are, they have shortcomings, and raise some annoyances.