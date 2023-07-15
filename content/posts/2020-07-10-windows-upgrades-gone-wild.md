---
title: 'Windows Upgrades Gone Wild'
date: Fri, 10 Jul 2020 05:42:00 +0000
draft: false
tags: ['partitions', 'servers', 'windows']
---

I use Windows at home as a gaming platform. I've used a Mac laptop (a 2013 Air I purchased in, if I recall, 2016, and now from my job, a Pro) for everything else since I got it, but it's still difficult to beat Windows for gaming. And let's be honest, Windows 7 was pretty decent for a desktop OS. Windows 10 definitely had and continues to have its growing pains, but I've come around to it. We'll ignore Vista and 8. I maintain that ME was actually not as bad as people make it out to be, but I'm aware that I'm in the minority with that.

But surely this isn't a post about a desktop OS, right? Yeah, I just needed an intro. Windows Server is incredibly popular around the world, and like it or hate it, you should have an inkling of how to use it if you're in the industry. While I am in no way an expert, I have used Windows at home from 3.1, so I know my way around more or less everything except things like GPO and AD - you know, the server stuff. Hey, I'm getting there.

The time had come at work to upgrade our two Windows Bamboo build agents, which had been happily humming along on 2008R2 in EC2 instances. Apparently, security found that troubling. We begrudgingly agreed to upgrade them to 2012R2, and after reading AWS' and Microsoft's documentation on the process, I began in earnest - it turns out if you demonstrate a modicum of competency with Windows, you'll be labeled the Windows Guy. Be ye warned.

The general steps for an in-place upgrade on EC2 are as follows:

1.  Take a snapshot. JUST DO IT.
2.  Stop any services that you might not want running, like Bamboo.
3.  Make a volume from the applicable AMI, e.g. 2012R2 English.
4.  Mount said volume to your instance.
5.  Run setup, select upgrade, and wait a bit.
6.  Restart aforementioned services if they don't automatically do so.

But what if you run into an issue, like not enough space to perform the upgrade? Expand the EBS volume and then grow the partition, I hear you saying. Sure, of course. Unless you (and by you, I mean the people who set these up, without Terraform or any other form of version control) already partitioned the EBS volume into Boot and Bamboo, and expanding the EBS volume adds unallocated space to the end, nearest the Bamboo partition. These are simple volumes, and don't take kindly to non-contiguous space. Short of adding a new EBS volume and shifting the Bamboo partition over (which is honestly not a bad idea, and if I had to build these from scratch, I would just have two EBS volumes to begin with), how do you fix this?

The answer, as with many things, is Linux. Well, also gparted. Note, you can absolutely do this with parted and ntfsresize if you're feeling froggy, but time has taught me that writing clever one-liners isn't always the right answer, especially when you're already familiar with the GUI.

But how do you get a GUI desktop to a Linux EC2 instance? Again, there are options, and I'm not suggesting this is the best way, but it's the way I did it. YMMV. Note, this assumes you're using a Mac.

**DISCLAIMER: IF YOU BREAK PROD BECAUSE YOU WERE FOLLOWING INSTRUCTIONS ON SOME DUDE'S BLOG, YOU DESERVE ALL RIDICULE YOU WILL INEVITABLY RECEIVE.**

1.  Spin up a Linux distro that you're familiar with; I chose Ubuntu 18.04. Give it enough disk space to install some GUI stuff. Make sure it's in the same AZ and VPC (if applicable, alternately you can do this over public IP space) as the Windows instance in need of work.
2.  Shut down the Windows instance, and detach its volume.
3.  Download [XQuartz](https://www.xquartz.org/) and install it.
4.  ssh into the Linux instance and install the following:
    1.  gparted
    2.  $DE\_OF\_YOUR\_CHOICE (I like Xfce)
    3.  xrdp (might be required, depends how you go about this)
5.  Restart the Linux instance - it should tell you to do so anyway.
6.  Attach the Windows volume to the Linux instance.
7.  Launch XQuartz, and then Terminal from within XQuartz.
8.  ssh into the Linux instance from this terminal, using the -Y flag. If this fails, try -X.
9.  Run sudo gparted. Background it if you want the terminal, I guess.
10.  Select the Windows volume you attached earlier.
11.  Shift partitions around to add space to the Boot partition. For example, you could:
    1.  Shrink the 2nd partition from the left.
    2.  Grow the 2nd partition from the right into the unallocated space you added earlier to make up for this loss.
    3.  Grow the 1st partition from the right into the newly abandoned space.
12.  Apply the changes - this will take some time, so get comfortable. Also note that gparted will almost certainly complain about corrupting the MBR, but since you aren't shifting the start sector of the Boot partition, you can ignore this. Don't get me wrong, things will almost certainly fail here, but the MBR is not one of them.
13.  Detach the Windows volume from the Linux instance, and shut down the Linux instance if desired.
14.  Attach the Windows volume to its instance (note, you'll need to specify /dev/sda1 as the mount) and attempt to boot it. If it works, congratulations. If not, keep reading.
    1.  You can check on the boot status from the AWS console by right-clicking on the instance, then navigating to Instance Settings --> Get Instance Screenshot.
    2.  If you see a Windows boot error, you're in luck.
    3.  If you see something else, figure it out, and write your own post.
15.  Shut down the Windows instance, and detach its volume.
16.  In the same AZ, spin up or use an existing Windows instance.
17.  Attach the troubled Windows volume to this functional Windows instance.
18.  In that instance, launch Powershell and execute the following:
    1.  `Invoke-WebRequest https://s3.amazonaws.com/ec2rescue/windows/EC2Rescue_latest.zip -OutFile $env:USERPROFILE\Desktop\EC2Rescue_latest.zip`
    2.  This will download a .zip file to your desktop. Make a directory and unzip it. Or clutter your desktop like a heathen, your call.
19.  Launch the ec2rescue application.
20.  Select Offline Instance.
21.  Select the volume you attached that won't boot.
    1.  If this volume is Offline, ec2rescue will attempt to bring it online. It may take a couple of tries to successfully read it as a Windows installation. You can always manually do so with diskmgmt.msc.
22.  Select Restore.
23.  Select Last Known Good.
24.  Accept any warnings, and apply.
25.  Detach the Windows volume from the Windows instance, and disconnect from that instance.
26.  Re-attach the volume to the problematic Windows instance, and attempt to boot it again.
27.  It may take upwards of 10 minutes to boot the first time - you can check its progress using Step 14 above.
28.  Congratulations! Now upgrade your instance with plenty of free space, and don't forget to like and sub...
29.  If all this fails, well, you _did_ take a snapshot at the beginning, right? Restore from that, and weep the bitter tears of failure.

So, that's that. It took me the better part of an afternoon to figure out and accomplish, but written out, it's probably 1-1.5 hours of work, with much of that being waiting for gparted to shift files. I spent a long time trying to get X11 forwarding working, which was unsuccessful until XQuartz. It just works, with no configurations or environment variables needed. Big thanks to its developers.