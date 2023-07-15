---
title: 'Filesharing the difficult way'
date: Sat, 08 Jun 2019 23:01:35 +0000
draft: false
tags: ['bash', 'filesharing', 'plex', 'scripting', 'sysadmin']
---

UPDATE: See end for an update on the method I settled on.

I run a home server, primarily as a NAS using Plex. It's an old Dell T310 (X3430 @ 2.4 GHz, 16 GB RAM, 25 TB usable disk with n+1 parity, RAID1 SSD boot drives) which I've shoved HDDs into. It's also my Linux environment when Mac just doesn't quite cut it, and I don't feel like starting a VM on the Air. Anyway, having Plex, eventually other people somehow find out that you have a fairly hefty amount (17 TB and climbing) of media, and Plex can share its libraries, so why not?

One thing Plex does not do, is allow for downloads. You can sync a file for offline viewing, but you can't just straight-up download the file like you could from an FTP server or the like. This is probably for legal reasons on their part, as Plex has a nudge-nudge-wink-wink relationship with the type of media usually used, so I can't really blame them. What to do, then, if someone wants to get a file to show someone else?

There are a variety of ways to answer this, and they depend on the other person's computer savviness. I could host a very simple webserver restricted to a specific folder - Python can do it natively - but that then opens it up publicly unless I add authentication. An rsync script could be used, but the other person in this instance has Windows and a Synology NAS, so it'd have to run on Busybox, and rsync often doesn't like it when the connecting computers have different versions, so that's out. Plus, ugh, what a pain. I could create a torrent or magnet file for each and distribute that; now that I think about it, that may actually be an easier solution than what I intended to write about. Nevermind, charge onward.

I wound up creating a guest account with a chroot, and automating file linking with two scripts. I use [fish](http://fishshell.com/), so you'll have to do a bit of tweaking to make this POSIX-compliant if you want to replicate it. Or, you know, use fish. Auto-completion and man page grepping is amazing, try it.

```
#! /usr/bin/fish

read --prompt "echo 'Movies or TV? (m/t) '" -l media_type
read --prompt "echo 'How many hours to look back? '" -l hours

#echo "Backing up /etc/passwd..."
#cp /etc/passwd /home/my_user_name/backup_etc_passwd

#echo "Enabling guest login..."
#sudo sed -i 's,/home/guest:/usr/sbin/nologin,/home/guest:/bin/bash,g' /etc/passwd

#echo "Checking sed's work..."
#set sed_work (awk -F ':' '{if($1 == "guest") print $NF}' /etc/passwd)
#switch $sed_work
#case '/bin/bash'
#        echo "bash successful"
#case '*'
#        echo "Error, please check /etc/passwd and/or ~/backup_etc_passwd"
#end

echo "Making symlinks in /media/merged/{media}..."
switch $media_type
        case m
                sudo find /media/merged/movies -type f -newermt -$hours+hours \
                        -exec ln -s '{}' /media/merged/transfer_folder \;
        case t
                sudo find /media/merged/tv -type f -newermt -$hours+hours  \
                        -exec ln -s '{}' /media/merged/transfer_folder \;
end

echo "Bind mounting hardlink folder to home/guest as RO..."
sudo mount -o bind,ro /media/merged/transfer_folder /home/guest

echo "All done. Please run disable_guest.fish when complete."
```

You'll notice a lot of commented-out blocks. That's discussed why in the next block. As to how it works, I ask the user (me) what kind of media they're looking for, and how far back to look, since usually I'm fulfilling this request shortly after obtaining the requested files. I make a backup of /etc/passwd just in case, then run sed with -i on it, replacing the /home/guest:/usr/sbin/nologin line with /home/guest:/bin/bash (filthy casuals don't get fish). Check it just to make sure with awk, passing in ':' as the delimiter, and then checking the last field ($NF) for the line with $1 matching guest. This is now expected to be /bin/bash. Next, make symlinks using find -exec. Getting the $hours portion to work correctly was a bit of a puzzle, but using the integer globbed with 'hours' worked well. Note, if your shell doesn't include newermt, you'll need to use mtime, and a negative integer in minutes instead. The exec line then creates symlinks for each resultant line spat out by find. Finally, do a read-only bind mount of transfer\_folder (my Windows upbringing bleeds through...) to guest's home.

I learned something about sshd\_config while writing this, and almost all of the previous block was then made pointless. I discovered that, in fact, the selected shell in /etc/passwd only affects ssh logins; moreover, the pertinent portions of my /etc/ssh/sshd\_config prevents the user from ssh'ing anyway.

```
Match Group filetransfer
        ChrootDirectory %h
        X11Forwarding no
        AllowTcpForwarding no
        ForceCommand internal-sftp
```

As you can see, I have a group named filetransfer, of which guest is a \[the\] member. ForceCommand internal-sftp ensures the only thing they can do is run sftp, and ChrootDirectory %h locks them to their home directory. So, yeah, sed/awk not required.

Skipping ahead a bit, I'm doing a mount -o bind,ro for the directory I created to temporarily store the files I'm sharing. This, to me, seemed the easiest way to share the files without opening up any other permissions.

Then we get to the symlinks. My server currently runs [mergerfs](https://github.com/trapexit/mergerfs) + [SnapRAID](https://www.snapraid.it/), which honestly can be a bit annoying, but I started out with disparate disk sizes, so it was the best solution at the time. I intend to move on to ZFS soon, but for the time being, know that /media/merged is treated as one giant disk, but is in fact made up of four disks. Symlinks are good, symlinks are great, but they have one glaring flaw for this use case: when the user drops into their chrooted ~, with transfer\_folder bound to it, they see symlinks which believe they are in /. When they then try to GET them, they get a cannot stat error, because the symlinks don't point to the actual files.

OK, so hardlinks, right? inode-level links that are for all intents and purposes the original file. Yep, that works great, until one of the files is on a different disk than transfer\_folder. Turns out hardlinks can't cross disk boundaries (well, not without help), and mergerfs sits atop filesystems, not partitions. While a program or user writing to /media/merged/x doesn't need to know what disk a file is going to, mergerfs does, and will take care not to split any files across disks. Were I using LVM or a true RAID, this would not be an issue, since you're at the block level then.

The disable script is essentially just a umount to /home/guest, and rm for all of the links created. It also has a lot of commented-out stuff, as above, that is no longer necessary.

So, there you have it. A somewhat-kludgy method of distributing files, which sometimes breaks with mergerfs. I will look more into automating torrent creation and serving, as I think this would be the easiest solution, avoiding symlinks, hardlinks, guest accounts, and the like.

Things I learned/realized/reasserted:

1.  ISPs need to hurry up with DOCSIS 3.1 so symmetric gigabit cable can be a thing.
2.  Symlinks, when placed into a chroot, will assume their target is at /.
3.  Hardlinks can't cross physical disks without block-level management like LVM.
4.  /etc/passwd default shells don't affect sftp, but on the other hand, ForceCommand in /etc/ssh/sshd\_config can probably fix your problem.
5.  I could have just used cp -av and avoided most of this hassle, but that is decidedly uncool, and also a tremendous waste of IOPS, which is definitely something a SOHO user needs to agonize over.

UPDATE: I tried torrenting. I can automate torrent creation with mktorrent, and serving with qbittorrent, transmission-cli, or anything else similar. However, I found issues getting the announce to public trackers - it was hit or miss. Some files would be visible within minutes, some within an hour, some never at all. It dawned on me that I have two SSDs of reasonable size in RAID 1 as my boot device, and I use barely any of it (unless a rogue container decides to start blowing up /var). So I set up a transfer\_folder inside my /home, with root:root owning it so the chroot still functions. While I still can't use hardlinks or symlinks for the same reasons discussed above, I can just run rsync -aP on the files and deal with the delay. Less than ideal, perhaps, but it's guaranteed to work.