---
title: 'Home Networking Is A Dumpster Fire'
date: Mon, 25 May 2020 16:41:13 +0000
draft: false
tags: ['netgear', 'networking', 'unifi', 'docker']
---

Not exactly SRE/DevOps, but I need to write _something_, so here we are. Also, while I receive no kickbacks from anyone, and don't have sponsored links, I am definitely being a cheerleader for Ubiquiti here.

It's not exactly a secret that consumer networking equipment is hot garbage. Back in the day of the WRT54G, it honestly wasn't that bad, but it's hard to say if the reliability was a result of the simpler hardware and software, or the far fewer clients per AP - IoT wasn't really a thing in the early 2000s. Today, with lightbulbs and doorbells requesting an IP address, demands placed on switches and APs are a bit higher.

I've owned quite a few throughout the years, and I'll attempt to remember/rank them.

*   Linksys WRT54G
    *   I remember having absolutely no issues with this, including when it was flashed with aftermarket firmware (DD-WRT, I think) to allow a second unit being set up as a repeater. 802.11g wasn't exactly speedy, but then, neither was most people's internet access.
*   D-Link DIR-655 N300
    *   This worked fine for quite a few years, but I do remember it starting to sporadically and seemingly without reason failing, which caused me to upgrade.
*   Netgear WNDR4500 N900
    *   I had actually forgotten about this until I searched Gmail for "Netgear," and found this mentioned in an archived Hangouts conversation.
    *   I cannot express enough hatred and disdain for this unit. Said archived conversation has me grousing to a friend: "the speed isn't good, even sitting 10 feet from it like I am now.  I get about 10 MBps write speeds sustained to my NAS."
    *   Hot. Freaking. Garbage.
*   Netgear Nighthawk R7000 AC1900
    *   You'd think I would have learned my lesson about Netgear, but as you'll find, I am a slow learner.
    *   This was actually an extremely solid unit for the first couple of years, and then it began exhibiting the random failures I saw several years previously with the D-Link.
    *   There was an unfortunate time (OK, there were at least two, maybe three) where an exploit to obtain root shell was discovered, and not patched for several weeks.
*   Netgear Orbi RBK50 AC3000
    *   See?
    *   Whatever hatred I had for the WNDR4500, multiply it by at least two for this, not least because it was absurdly expensive, and at the time, was the purported best-in-class mesh networking option.
    *   I got a whopping six months out of it before it began forgetting settings, including authentication so that while logged into the router, I had to input my password every 10 seconds or so.
    *   RMA'd both the main unit and satellite, which fixed the settings disappearing, but I still had issue with random slowdowns. Since I work from home, including video conferencing, fast and reliable internet is a necessity, not a luxury.

After having a Zoom conference stutter out to the point of needing to go audio-only, I was fed up, and picked up a Ubiquiti Unifi setup. I opted for a USG for the router, a US-8-60W PoE switch, and a UAP-AC-PRO-US. I skipped the Cloud Key, as I intended to run the controller on my own.

I happened to have a couple hundred feet of CAT6 laying around, so I picked up connectors and boots, along with some keystone plate and jacks. I wanted to stick everything into a closet to try to remove some heat load from the room everything is in now, and to make the cable runs neater. As it turns out, terminating CAT6 is significantly more annoying than I remember CAT5 to be - I assume it's a combination of the larger wires (not always, to be fair, but mine is 23 AWG) and staggered wires that makes it more difficult.

Getting everything set up was a nightmare, largely because I had an undetected Layer 1 fault, and also because of outdated firmware. The tl;dr is that you will almost certainly have to update firmware before they'll join together, and to do that, you have to have a working internet connection. Try as I may, I couldn't get the switch to reach the internet through the router, which did successfully pull an IP address. I wound up downloading the firmware for everything except the router and manually updating.

I initially spun up an EC2 instance to act as the controller, but soon realized I could just host it on my server, and also, that Docker was once again a savior. The controller needs some weird dependencies, including JRE8. While there are some great scripts the community has written to figure these out and load everything correctly, Docker is just so much easier. The only hiccup I had was having to change the container's networking to host so that the inform URL wouldn't default to a Docker IP (could have also probably done macvlan for arguably a better security posture), but aside from that, it was smooth sailing.

So far, I adore it. The UI is great, the configuration settings are endless, the app is actually useful, and of course, the range is great. I have a two story house, and I placed the AP in the center of the second floor, on the ceiling. I've had no issues with 2.4 or 5 GHz bands throughout the house, and if I ever want to add more APs, I gather joining them is painless.

Of course, in the past, I usually loved the new shiny when it was new, but found issues over time. The only one I've noticed so far (which may or may not related to this change) is that Plex will randomly pause, necessitating a stop/resume, not just pause/play. More investigation will be needed to determine if it's a networking, Docker, or other issue.

EDIT: From discussions with a friend, he made me remember something I apparently blacked out on - namely, how to get the controller on a Docker host to see and adopt the Unifi stuff, when you need the Unifi stuff up and running to get the controller working? In my case, the host I'm using has two NICs, so I set up one to connect to the Orbi, still with WAN access, and the other to the Unifi switch. Then, with a laptop plugged into the Unifi switch, I was able to manually update the firmware on all the Unifi devices, and get them adopted by the Docker controller, at which point I could remove the Orbi system entirely.

EDIT2: Also, while doing this, I ran new coax to the modem. I have Spectrum, rated at 400/20, and before, I was seeing right around that, maybe 5-10% better. Initially with the Unifi setup, I was seeing about half the download speed. I reset it, and it came up to expected levels. When in doubt, reboot. After running the new coax, though, I was able to get 530 down! Definitely worth the minor investment.