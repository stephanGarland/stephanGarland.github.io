---
title: 'Might As Well Jump'
date: Sun, 21 Feb 2021 17:12:26 +0000
draft: false
tags: ['jumpbox', 'security', 'vms']
---

A friend wanted to learn Linux, so I offered to spin up a VM under Proxmox. Done.

Just kidding. I mean, that would work (assuming you handled port forwarding) if you were hitting an IP, but FQDNs are much easier for people to remember. Except ssh isn't based on HTTP, so how do you forward them? One way is with [nginx's stream module](http://nginx.org/en/docs/stream/ngx_stream_core_module.html). Something like this suffices.

```
stream {
  upstream ssh {
    server $DEST_IP:$SSH_PORT;
  }
  server {
    listen $FORWARDED_PORT;
    proxy_pass ssh;
  }
}
```

And that'll handle one person just fine. But what if another friend wants the same thing? Linux is perfectly capable of handling concurrent users, but the point of the VM was that they could break it during learning, and I could restore a snapshot (after giving them time to fix it themselves, of course). I could have multiple ports being forwarded, and direct each person to specify their assigned port, but that wouldn't let me learn anything new. Plus, opening more ports is arguably less than ideal. If IPv6 was everywhere, every VM could just have their own IP, but we aren't there yet. A VPN is of course a great option, and it's something I'm working on. In the meantime, enter `authorized_keys` commands, and jump boxes.

A jump box is basically a small, ideally hardened machine that does nothing but forward connection requests somewhere else. Remove all other services, keep it up to date, and have good logging and audits. That's basically it. You could also use a [Docker container](https://github.com/monsoft/ssh-docker-jumpbox) to accomplish the same thing.

Since I have set up Proxmox images with Packer, creating a new VM for this purpose was painless. I added a user with minimal access, and added the two ssh keys to `~/.ssh/authorized_keys` - with a catch. sshd allows for forced commands to be ran when a given key is used to login. By adding a command string to the beginning of the public key entry, sshd on the jumpbox will forward those users to their respective VMs whenever they log in.

```
command="ssh $USER@$USER_VM_IP" ssh-rsa $PUBLIC_KEY
```

You then set up a single A record pointing to this jump box (NAT'd or otherwise), and tell your users to pass their private key, and that they should use your jumpbox user, e.g. `ssh -i ~/.ssh/id_rsa jump@jumpbox.website.com`.

This is a middle ground as far as security goes. A VPN is better, but we can still improve the security posture. First - and this is recommended for any machine anywhere - disable password authentication and root logins. By forcing the usage of keys, you're severely reducing the possibility of a successful attack. Second, [fail2ban](https://github.com/fail2ban/fail2ban) can further help mitigate this possibility. Third, enabling 2FA is enormously helpful, if you can get all users onboard. [TOTP with PAM](https://github.com/google/google-authenticator-libpam) is a possibility, and one I successfully tested. Finally, and perhaps the least helpful but easiest, is to change the listening port - port scanners are going to hit 22 looking for ssh access. Changing it to something else, ideally numerically higher (port scanning takes time, after all - they probably don't want to sift through thousands of ports looking for a hit), can help here. Security through obscurity is the lowest form of security, but every layer helps. Alongside this, you could use an [ssh tarpit](https://github.com/skeeto/endlessh) or [honeypot](https://github.com/droberson/ssh-honeypot) on 22.