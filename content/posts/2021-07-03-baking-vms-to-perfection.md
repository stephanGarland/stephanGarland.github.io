---
title: 'Baking VMs to Perfection'
date: Sat, 03 Jul 2021 17:53:54 +0000
draft: false
tags: ['packer', 'proxmox', 'vms']
---

I've now accomplished one of my [previously-mentioned desires](https://sgarland.dev/2021/01/03/hardware-sucks/), namely, to use Packer to make VMs for Proxmox. After much battling with YAML and esoteric bash commands, I have succeeded in being able to spawn endless VMs, ready to go just how I like them. As with many projects I've done, this was thanks to someone else's hard work; I merely customized it.

The repo\[s\] are in [two](https://github.com/stephanGarland/ansible-initial-server) [parts](https://github.com/stephanGarland/packer-proxmox-templates), and for my forks, only the Debian template has been customized. It comes with templates for Alpine and Ubuntu as well, but I've not done any work to them.

Essentially, `ansible-initial-server` installs a vanilla Debian VM, installs whatever packages are desired, adds files (`/etc/motd`, `/etc/sudoers`, etc.), and generally customizes the install however is desired - for example, setting up sshd for pubkey auth only. This repo is called by `packer-proxmox-templates`, which uses Packer to create a template that Promxox can use from this newly-created VM. I heavily modified the build script, and added a TUI option using Dialog to make configuring it easier.

I have two basic templates, with a couple of options:

```
"base_type": {
    "prod": {
        "zfs_support": bool,
        "zsh_customized": bool
    }
    "dev": {
        "zfs_support": bool,
        "zsh_customized": bool
    }
}
```

Dev and Prod are shorthand for the number and types of packages installed. Basically, dev installs dev-y things like Docker, k8s, Hashicorp's suite, and various Python libraries I use a lot. In retrospect, since my Docker host runs my prod environment (everything I run is in containers, and handled with Docker Compose), I should probably either mover Docker by default to Prod, or add it as an option, like ZFS and Zsh.

ZFS installs ZoL packages, along with autofs, Samba, and NFS. Why both? I spent an afternoon trying to get Plex to see the Samba share, and for the life of me could not. NFS just worked. OK, it didn't; I also had to set up autofs, because ZFS hadn't loaded when `mount` tried to mount the drives, which made NFS upset. Autofs mounts on-demand, which NFS seems to respect, and everything works. I try not to question the magic after a certain amount of time wasted.

ZSH customization adds Oh My Zsh, my `.zshrc` file, and powerlevel10k (and its config file, `.p10k`) - my preferred Zsh theme. Honestly the only reason I wouldn't use it would be for something like a k8s worker node. If there's a chance I'll be ssh'ing into the VM, I want my environment set up how I like it.

Once they're set up, I have a separate repo that has both initial and maintenance activities, again using Ansible. The initial activity polls my Proxmox hypervisor given a VM ID, and returns its assigned IP address. That's then used to set the IP address and hostname per a file. After that, I can install any packages (it's a little bit of a pain to make new templates, so I tend to only do so every few+ months, and sub in Ansible plays if I decide I need a package universally installed before then), update VMs regularly, etc.

I have successfully tested the ephemerality of the VMs with my NAS VM. Export the ZFS pool, instantiate a new VM, import the pool, done. The Docker host would be the same, given that everything is containerized. My Plex cache is mounted on a partition of an NVMe drive, so I don't even lose my library if I wipe the VM and start over.

All in all, I couldn't be happier with this setup. It allows me to try things out in my dev VM without fear of breaking anything, and then applying those changes to everything if desired. Similarly, if I needed to recreate all of my VMs, it could be accomplished in a few hours. It's not _entirely_ automated (the aforementioned hypervisor script requires entering VM IDs manually), but for my purposes it's good enough - for now.

Future work will probably be changing to a k8s cluster. Docker Compose is slick, and truthfully does everything I need, but having homelab k8s practice can only help my professional career. Plus, it gives the other two idle nodes in my rack a purpose in life.