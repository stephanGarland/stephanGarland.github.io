---
title: 'Testing VM immutability, secrets storage again, and a brief ode to ZFS'
date: Thu, 23 Dec 2021 23:41:02 +0000
draft: false
tags: ['proxmox', 'zfs']
---

I've had my dev VM upgraded to Debian 11 for a few months, and have fiddled with from time to time. It's not that I distrust Debian stable in the slightest to not be stable, but I also am not going to just throw prod a `sudo apt full-upgrade` (yes, with updating sources first...) and hope for the best. As I didn't experience any issues with workflow with the upgraded VM, I decided it was time to [bake some new images.](https://sgarland.dev/2021/07/03/baking-vms-to-perfection/)

Also, I've been making edits to my IaC without, uh, testing it. Bad, I know. Not quite as bad as making local edits, forgetting that you didn't commit them, and then running the Ansible play and watching the changes go away - not that I've done that, absolutely not. So, it was time to see how immutable my VMs are, and also to see how well ZFS played with a major kernel upgrade. You may recall [I previously had issues](https://sgarland.dev/2021/02/02/failed-nvme-drive-better-change-your-upstream-dns-resolver/) with that.

Aside from dealing with a mountain of various small issues, like typos, YAML hell, forgetting a package, I got it done. After exporting my zpool from the old NAS VM, I imported it to the new one and... wow. It worked. I was also able to later add a number of commands to my post-Packer Ansible plays to handle ZFS command delegation, add an SSH key (more on that later), a PGP key for SOPS, etc. I am more than a little impressed with myself, not gonna lie, but not as impressed as I am with the \*nix community in general, and the amazing products they make.

Re: ZFS, before doing this upgrade cycle, I was fiddling on the NAS and had some test files in a directory structure that mimicked my main pool. You may already see where this is going. I apparently had changed into the actual directory of the pool, as I discovered when a `rm -rf ./$folder` took far longer than I thought. `zfs rollback tank/$folder` and everything was back to normal. It just works. It's absolutely incredible.

So, secrets. I'm using [SOPS](https://github.com/mozilla/sops) for secret files that need to stay on a VM, like the IPMI password to handle powering my backup node on and off. This works fine for local scripts, but remote (like pushing keys onto a server) throws a wrench into that plan. While Ansible does have a SOPS module, I found through a lot of painful tinkering that it doesn't support GPGv2, which is what most GPG installs are. Not wanting to deal with simultaneous GPGv1/v2 installs, I then found [Ansible Vault.](https://docs.ansible.com/ansible/latest/user_guide/vault.html) This worked to put the SSH key (needed for rclone to send zfs backups to my backup node) onto the NAS VM, and also to add the GPG key for SOPS to decrypt the IPMI creds. I get GPG to import the key with an `ansible.builtin.shell` task, and then `rm` the file after it's been imported.

While I'm still inching closer to a k8s setup of some kind, in the interim it's nice to know that I'm able to handle upgrades fairly seamlessly.