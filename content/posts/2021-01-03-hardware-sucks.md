---
title: 'Hardware Sucks'
date: Sun, 03 Jan 2021 20:20:51 +0000
draft: false
tags: ['hardware', 'proxmox', 'supermicro', 'virtualization', 'vms']
---

It's coarse and rough and irritating and... it sucks. There's a reason why cloud providers are wildly popular. Scaling in AWS is incredibly easy; not so much when you're rolling your own. God help you if you're doing this by cobbling together disparate groups of enterprise and consumer hardware. Oh wait, that's exactly what I'm doing.

I've had a Linux box since 2016 or so. I mean, I dual-booted every distro known to man with Windows when I was a teenager (including bootstrapped Gentoo, and I managed to get a crotchety HP printer to work with it), but I didn't have a dedicated Linux box until then. I had a Synology DS413 in 2012, but busybox barely counts. In 2016, the IT manager at my employer was kind enough to donate a Dell T310 to me that the company no longer had use for. After buying an H200 HBA and flashing it to support pass-through, I installed Debian and was off to the races. A friend convinced me to learn Docker, and everything after that just kind of came naturally.

While running containers in a host OS is fine, it does have some limitations. Since the host was also running everything else for me, like DNS, any changes I made or experiments could potentially drop connectivity for everyone - not to mention Plex going down being an emergency. While containers help immensely, at some point, you may need to restart the host for one reason or another. Enter virtualization.

Now, that T310 isn't exactly modern. It's got a single X3430 (Lynnfield-era), and 16 GB of RAM. I did have it absolutely filled to the brim with spinning disks, via helpful 5 1/4" drive bay adapters. It held 4x 8 TBs, 2x 10 TBs, and 2x 2.5" SSDs I had in software RAID1 for boot drives. But performance was rather lacking - my ancient Macbook Air beats it in single-threaded applications. All this to say, it wasn't an ideal candidate for loading up with VMs.

Also, and this is mostly for nerd cred, I wanted a rack. So, so badly. The closet with the server and networking gear is sized for a half-rack, so that's what I got - a [Startech 25U](https://www.startech.com/en-us/server-management/4postrack25u) open rack. Expandable depth, solid construction. Due to what is apparently a well-known fluke with Amazon's ordering system for this model, I ordered one and received a pallet, but that's another story.

What to fill it with? I settled upon, for reasons which mostly boil down to "I found a motherboard being sold locally," a [Supermicro X9DR3-LN4F+](https://www.supermicro.com/products/motherboard/Xeon/C600/X9DR3-LN4F_.cfm), paired with a [CSE-826](https://www.supermicro.com/en/products/chassis/2U/826/SC826E26-R1200LPB) chassis. That is a Sandy/Ivy Bridge-era motherboard, in SM's self-titled EEATX size AKA giant. Dual LGA 2011 sockets, 24x DIMM slots, 4x PCI-E 16x slots, 1x PCI-E 8x slot, 1x PCI-E 4x slot, and quad gigabit LAN. The board came with E5-2630 v1s, and 64 GB of PC3-12800. I bought two off-brand 512 GB ADATA NVMe SSDs that had good reviews, slapped them into PCIe adapters, and then discovered that the board couldn't boot from NVMe drives. Luckily, there are clever people on the internet who modded the BIOS to support such things.

My next struggle was cooling. The board came with passive coolers, and the case had a wall of three 80mm fans. Quite a normal arrangement for servers, but not ideal for noise. My first thought was to replace the passive CPU coolers with Supermicro's active coolers, to lessen the need for the fan wall. In retrospect, this was needless, as they put out quite a bit more noise than the 80mm fans do (who knew?). I ultimately went with [Noctua NH-L9x65](https://noctua.at/en/nh-l9x65) coolers, which do in fact fit in a 2U chassis. Barely, so it's best to flip the heatsink fan to pull from the board and exhaust up to the lid, but they do fit. CPU temps in an entirely-too-warm room (high 80s to low 90s \[F\]) with low loading are in the high 60s to low 70s (C)\* mid 50s now that I've raised the fan wall RPM up somewhat. TCase for my CPU is 82 C, so I'm comfortable with that for now. I ran a 100% load test, and they seemed to go fairly asymptotic near 80 C. Exhausting the closet is a future project, since lead-acid batteries don't really enjoy higher temps.

\* I am aware of the lunacy of mixing units, and while I would love to adopt metric for everything, I grew up thinking about human temperatures in Fahrenheit, and computer temperatures in Celsius. If you tell me your hard drives are idling at 129 F, I'll have to do rough estimates in my head, but if you say they're at 54 C, I'll tell to consider improving cooling.

With that settled, I moved onto upgrading the CPUs to v2s. I knew I didn't want/need 2697 v2s, as they're a. incredibly overpriced b. have massive TDP, so I settled on 2680 v2s. 12c/24t, 2.7/3.5 GHz base/boost, and 115 W TDP. OK, so only a 15 W TDP savings, but they're also about 66% the cost of 2697 v2s. The Noctua coolers I bought are rated for 84 W, but I YOLO'd it with the thought that I rarely am stressing my server, and my recent load average with VMs running, albeit mostly idling, shows that -  4.24,4.33,4.37. Oh, I also threw the 80mm fans onto a [Noctua PWM controller.](https://noctua.at/en/na-fc1) This wasn't strictly required, as the noise with the fans set to what Supermicro euphemistically calls Optimal wasn't bad, but I was going for quiet.

Supermicro's site for my motherboard confidently states that v2 CPUs are supported with BIOS version >= 3.0. Not a problem, mine was v3.3. I popped the new CPUs in, carefully applied Arctic Silver and mounted the Noctuas, and... nothing. No VGA output, no beep codes, and BIOS code 00 in IPMI. Huh. I tried a single CPU, then the other one. I yanked out every expansion card, swapped RAM around, still nothing. Put the old CPUs back in, and it booted fine. Thinking that perhaps my modified BIOS was to blame, I flashed to stock, but it was the same story. It wasn't until I asked for help on r/homelab (fine folks there, really - a tremendous source of knowledge) that someone told me to check the board's hardware revision - I had 1.01. Guess what Supermicro won't tell you unless you ask? That early board revisions didn't support v2 CPUs. Their legendary customer support was actually extremely helpful, but regretfully wasn't able to arrange to upgrade my board out of warranty, as they had a five year limit on that support, and it was from 2012. Onwards to eBay, where I sourced an identical board with 1.20a revision.

Then, it still didn't boot. Ah, the BIOS is original, v1 something. Oh, here's this buried warning on Supermicro's site to upgrade the IPMI firmware first if you have a v1 BIOS - good thing I found that. Ugh, why isn't IPMI coming up? Ah, it's jumpered out. Oh good, now it doesn't want to boot at all. After multiple cycles of NVRAM reset, reseating CPUs and DIMMs, I finally got it to boot again, brought up IPMI, flashed it, then flashed the BIOS. All is well.

Next up, this Dell R515 I bought to ingest backups. RAID is not a backup, nor ZRAID, and while I have important things backed up locally and in the cloud, I wanted to be able to back up the multiple TB of media I have. To that end, basically any 2U+ chassis with LFF slots would do. Performance and noise weren't a concern, since it will only power on once a day to pull ZFS snapshots. It took hours of fiddling and reseating things to get it to recognize its iDRAC Express; I never could get it to see the iDRAC Enterprise. However, it now refuses to read drives connected to the H200 I flashed and modified to live in its storage slot. Even when flashed back to stock Dell firmware, I'm getting nothing. Given that I also had some cable errors, my next attempt will be to replace those, as I find it doubtful that the card itself is bad since I've had no problems with any of the flashing tools finding it.

Software Sucks Too, Just Not As Much
------------------------------------

Whew, so the hardware is (more or less) up - how about some software? I'm running [Proxmox VE](https://www.proxmox.com/en/proxmox-ve) on the Supermicro. I chose it because it's free, without limitations like ESXi's free version, and is basically Debian. I don't know if you've caught on yet, but I heart Debian. Here's my dashboard:

![](/images/2021-01-03-hardware-sucks/0.png)

Proxmox Dashboard

**VM**

**Distro**

**Description**

100

[Debian](https://www.debian.org/)

General purpose dev server for whatever I want to play with - nothing critical will be hosted here.

101

[FreeNAS](https://www.freenas.org/)

I wanted to try it out, and also wanted to abstract away some of the difficulties of ZFS. This may have been been a mistake. EDIT: It was, see endnote.

102

[k3OS](https://k3os.io/)

This is Rancher's take on Kubernetes. I'm still playing with it. I also plan on trying out [Talos](https://www.talos-systems.com/).

103

[RancherOS](https://rancher.com/docs/os/v1.x/en/overview/)

All of my services are containerized, and live here. I really like the idea of RancherOS - everything in the system is in a container. EDIT: See endnote.

104

[Debian](https://www.debian.org/)

This is a minimal distro that does nothing but host my LogicMonitor collector. I initially tried Alpine, but it was lacking certain tools I needed to install the collector, and, well, I heart Debian. Separating this was crucial, as it allows me to restart other things without worrying about losing monitoring.

VM Listing

You may have noticed some oddly named disks above. Because I like to live dangerously, my method of changing my disks from ext4+mergerfs+SnapRAID to ZFS was to copy everything to a LVM of 14 TB disks, set up the zpool, then transfer them back. First of all, it turns out that rsync over gigabit isn't really that fast when you're talking about nearly 30 TB of stuff. I was seeing around 85 MBps. Transferring them back, which is between VMs in the same chassis, I'm seeing even worse - around 70 MBps. I blame FreeNAS (the first transfer was my Debian host to a Debian VM). That little note about FreeNAS in the VM descriptions? Yeah... I'm not overly taken with BSD. First of all, obviously, it's not Linux. This became painfully obvious when I tried to use `lsblk` and found it didn't exist. Where are my drives?! What do you mean BSD doesn't natively support LVM? What is a `geom`? Ugh ugh ugh. I never was able to get it to see data on the drives, which may be FreeNAS' fault, or may be FreeBSD - I'm not sure. I gather from FreeNAS forums that they strongly recommend you never touch the shell, which is horrifying. The GUI is gorgeous, I'll admit, but being warned away from interacting with the shell is awful. I have half a mind just to spin up another Debian (ayyyyy) VM, load ZoL, and import the zpool. Since I'm not using any of the other features that FreeNAS offers, like jails and a Type 2 hypervisor, I'm not seeing a lot of benefit.

Related, I'm having issues with disks - I passed them through Proxmox with `qm set $VM_ID /dev/disk/by-id/$ID`, but I've since read that this is considered less than ideal for FreeNAS/ZFS. The solution is to pass the controllers through, which I'm fine with doing - as mentioned, my Proxmox boot drives are NVMe, so any spinning disks are solely dedicated to storing media. I intend to buy another NVMe drive, probably 1+ TB, and use it as a scratch disk for VMs. Plex can benefit from having its library on a SSD, for example, and of course databases and the like can as well. I still have plenty of PCIe slots left, but if I wanted to, the motherboard supports bifurcation, so I could throw everything onto one card, if it will physically fit. After everything is transferred back to the zpool, I'll do that.

Containers
----------

All of my containers (see below) are currently running in RancherOS, and were instantiated via a docker-compose.yml file.

```
[rancher@rancheros ~]$ docker ps | grep -v rancher
CONTAINER ID        IMAGE                             COMMAND                  CREATED             STATUS                  PORTS                              NAMES
25afac09c362        linuxserver/plex:latest           "/.r/r /init"            20 hours ago        Up 20 hours                                                r-Media-plex-1-1144f240
43b2257e92e1        linuxserver/transmission:latest   "/.r/r /init"            22 hours ago        Up 22 hours                                                r-Media-transmission-1-08aee443
42312c06b37a        linuxserver/radarr:latest         "/.r/r /init"            22 hours ago        Up 22 hours                                                r-Media-radarr-1-bd467570
3712083f1060        tautulli/tautulli:latest          "/.r/r ./start.sh py…"   22 hours ago        Up 22 hours (healthy)                                      r-Media-tautulli-1-92e86f33
80b8804ce18e        linuxserver/ombi:latest           "/.r/r /init"            22 hours ago        Up 22 hours                                                r-Media-ombi-1-88bbd363
3e3ed82361d9        linuxserver/sonarr:latest         "/.r/r /init"            22 hours ago        Up 22 hours                                                r-Media-sonarr-1-ef9cd635
```

This works fine, really, but I'd like to shift everything into Kubernetes. This is mostly because I use it at work, and the more practice I can get with it, the better. Also, a [CRD was recently created](https://github.com/kubealex/k8s-mediaserver-operator) that instantiates all of these resources in one fell swoop, so that's tempting. I mean, importing the docker-compose.yml into Rancher is also basically one fell swoop, but still - shiny toys.

Future Work
-----------

I want to play with [pfSense](https://www.pfsense.org/) (ugh, more BSD). I want to add a Quadro for transcoding Plex streams. I _need_ to add nginx, traefik, or something back into my stack so I can stop hitting ports when I want to pull up a page. I should add OpenVPN so I don't have to rely only on TLS and passwords to access those services. I should use Packer or something similar to prebake my images. I used cloud-init to bring up RancherOS, and could do so for the others, but my Debian dev box has quite a lot of things I like installed - having them ready to go would be ideal. Finally, I intend to repurpose the T310 to learn Windows virtualization with Hyper-V. While I don't especially enjoy Windows, especially the Server varieties, it's quite prolific. The more knowledge, the better.

Endnote
-------

I couldn't deal with BSD, at least not it in combination with a forced GUI. I replaced it with [openmedivault](https://www.openmediavault.org/), which I've used before and am comfortable with. It doesn't care if you do things via the shell. Did I mention it's Debian? It was more than happy to mount the transfer LVM and the zpool (OK, I wound up creating a new one because it was a bit iffy with some of the things FreeNAS threw onto the drives), and copies are now taking place at essentially top speed for a [nominally 5400 RPM](https://www.extremetech.com/computing/314745-western-digital-caught-misrepresenting-hdd-rpm-speeds-too) drive, around 120 MBps.

I also got tired of dealing with GUIs in general, and got rid of OMV. My entire stack is now straight Debian.
