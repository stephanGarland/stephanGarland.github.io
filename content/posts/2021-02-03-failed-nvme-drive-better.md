---
title: 'Failed NVMe drive? Better change your upstream DNS resolver.'
date: Wed, 03 Feb 2021 04:19:21 +0000
draft: false
tags: ['dns', 'nvme', 'zfs']
---

This is not entirely an "it's always DNS joke," I promise. But for the record, it is always DNS.

I have an (three, actually - two are boot drives/VM storage) NVMe drive in my server that serves multiple functions - a Plex cache, a ZFS Intent Log (I should get rid of that, as it's shown to have 0% gains for me), and a general scratch disk for anytime I need a fast disk. The disk itself is controlled by Proxmox, partitioned, and I then pass the partitions through to the correct VMs, and mount them via UUID. It's worked well so far.

This afternoon, Plex stopped playing videos. OK, `docker restart plex`. Nope, didn't fix it. Wow, `docker logs plex` has a ton of failures. Let's see if the disk is full or something - `ls /mnt/nvme_plex`. Can't list it? Huh? It still shows up in `lsblk`. Let's check Proxmox. Annnnnd it's gone - missing from `lsblk`, and some I/O errors in `dmesg`. Great.

At this point, I'm suspecting the cheap PCIe adapter over the drive itself, but I wanted to verify. I shut down the node, yanked it out, and after tearing apart my gaming rig, swapped out its secondary NVMe drive with this one. While Windows couldn't read it (ext4), its partition manager did see all three partitions, so the drive was good. On a whim, I put it back into the adapter, re-installed it into the node, and booted it back up. Hey, it worked! Lovely.

Except now the AdGuard container isn't running, and thus the household doesn't have internet. Can't bind to 53/udp. `netstat` shows precisely nothing using it, so that's odd. I eventually figured out that its internal config file had decided to latch onto an old Docker IP address for its bind, which was now in use by something else. Should probably file a bug report or something. But wait, there's more. Like an idiot, I decided now would be a great time to change to DNS-Over-TLS instead of DNS-Over-HTTPS as I had been successfully using. When something is working fine, the obvious answer is to change it until it doesn't.

Guess what happens when Plex doesn't have a PKCS12 token set up (my bad) and you change from DoH to DoT? Apparently, remote connection fails. Who knew? Didn't stop me from spending a solid 30-45 minutes playing around with various other settings before reverting that change. But wait, there's more!

Plex is up, we can watch a movie now, right? Nah, content not found. OK, ZFS is probably messing up - it likes to do that. No kernel module? Weird. Oh, the reboot caused a kernel revision bump to be loaded, guess I have to recompile for the new kernel. An hour later, I found [this bug report](https://bugs.launchpad.net/ubuntu/+source/zfs-linux/+bug/1851314) - did you know that `apt` doesn't always get you the same results as `apt-get`? I didn't. Finally, I can run `zfs import`!

```
This pool uses the following feature(s) not supported by this system:
        org.zfsonlinux:project\_quota (space/object accounting based on project ID.)
        com.delphix:spacemap\_v2 (Space maps representing large segments are more efficient.)
All unsupported features are only required for writing to the pool.
The pool can be imported using '-o readonly=on'.
cannot import 'tank': unsupported version or feature
```

Ah, well then. It turns out [that this is a known bug](https://icesquare.com/wordpress/zfs-on-linux-trouble-this-pool-uses-the-following-features-not-supported-by-this-system-all-unsupported-features-are-only-required-for-writing-to-the-pool/) (I see a trend...) with ZFS On Linux when you update the kernel. Sadly, the solutions in that blog did not fix my woes, but luckily, uninstalling the new kernel and recompiling the ZFS module for the old kernel did. I guess I'll have to be happy with 4.9.0-13. I think I'll live.