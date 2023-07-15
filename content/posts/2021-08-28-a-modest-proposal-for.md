---
title: 'A Modest Proposal for minimizing downtime and adding more blinkenlights to ye olde 25U rack'
date: Sat, 28 Aug 2021 18:34:44 +0000
draft: false
tags: ['short', 'homelab']
---

In case you haven't been following my work (I absolutely can't blame you, as this blog is self-congratulatory at best), [I have a 25U rack](https://sgarland.dev/posts/2021-01-03-hardware-sucks/) with some stuff in it. In short, it consists of a Supermicro 2U X9, and a ZFS pool, with an almost-identical second Supermicro 2U X9 that exists as a cold spare, and a ZFS backup target. There's also a Dell R620 1U that I've done absolutely nothing with yet beyond installing Proxmox. And, of course, networking gear.

All applications are Dockerized, are orchestrated with Docker Compose, and live in a VM. All VMs save for some experimental ones are templated Debian, with configuration handled via Packer and Ansible. Replacing a VM is trivial.

The main purpose of the homelab, beyond learning things, is hosting, indexing, and serving media. Hooray Plex. It also does DNS adblocking, which means that if the container, VM, or node goes down, the household is without media, and without internet. This is less than ideal.

There are also some boot issues I'm still working through - namely, the ZFS pool is handled by its own VM, which has the HBA passed through in Proxmox. It mounts the disks using autofs, and exposes the array via NFS. The NAS VM needs adequate time to boot up, with its own delay for loading ZFS modules, before attempting to export the share with NFS. This was solved with a systemd dependency, but I then found that if the Docker VM comes up quickly enough, the Plex container will attempt to mount the shares which don't yet exist. I further tuned the boot delay in Proxmox to hopefully address this. However, all of this is somewhat annoying and brittle.

My current plan, subject to change, is to pick up two more R620s. I'll create a Proxmox cluster on them, then create a Ceph cluster - this will store VMs. I'll get some Mellanox 40G NICs, and run a mesh network between the three to handle Corosync and Ceph. For media storage, I'll stick with ZFS. I can either convert one of the SM 2Us to a JBOD and connect it via an SFF-8088, or keep the motherboard - perhaps downgrading the CPUs to be as power-efficient as possible - and present it as an iSCSI target or NFS share.

I think this approach has promise, and if I opt to implement it, I'll of course write about my experience, challenges, successes, etc.

EDIT: It dawned on me that my R620 has a combo 10GBe SFP / 1GBe RJ45 NIC already, with dual ports for each. I don't need 40G networking. I can get an 8 port 10G switch for ~$250, and set up VLANs on my main GBe switch. One 10G network for Ceph, one for Proxmox <--> ZFS iSCSI communication (think I'm going that route), then one 1G network for corosync, and the other for WAN.

Additionally, I'll set a watchdog up on my RPi or something that, if failure of the cluster is detected, sends a command to my Unifi Security Gateway to change the DNS forwarding address to 1.1.1.1/8.8.8.8 - or to itself. That way, in the event of extended power failure or something else where I need to maximize runtime, I can maintain internet access for phones without having all the servers up. I could also have it either spin up a Plex container on my soon-to-be-only NAS, or if I joined it into the forthcoming k8s cluster, that would all be taken care of. I could have the DB as a PVC.