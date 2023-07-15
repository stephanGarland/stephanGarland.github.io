---
title: 'Server Healthy Check'
date: Sat, 27 Jul 2019 06:44:21 +0000
draft: false
tags: ['automation', 'bash', 'short']
---

This will be quite short, but I wanted to put some new content out there.

Healthy checks are a vital part of any organization, be it your homelab, a small office network, or a datacenter. Knowing the status (availability, load, temperature, etc.) of a server is critical in not only being aware of its health, but also of the potential need to scale.

For my home server use, the only thing I'm really concerned with is availability, mainly because my toddler delights in pushing the power button. Sadly, my version of iDRAC doesn't allow the button to be remapped. I could disconnect it, but that leads to annoyances when actually having to use the button, so...

This is a very simple script that pings the server, and if it doesn't reply, uses [Pushbullet](https://www.pushbullet.com/) to tell my phone something's wrong. You can use any push service you want; I have no affiliation with Pushbullet, but it's free and easy.

```
#!/bin/bash

if ping -t 5 -c 1 YOUR_SERVER_IP_ADDRESS &> /dev/null
then
        :
else
        curl --silent -u """YOUR_PUSHBULLET_API_KEY"":" \
                        -d type="note" -d body="SERVER_NAME didn't respond to a ping" \
                        -d -title="Server Down" 'https://api.pushbullet.com/v2/pushes'
fi
```

Insert this as a cron job at your desired interval - I chose 15 minutes. Also, important note, you have to be running this on something physically other than the server you're checking. I have a Raspberry Pi Zero W powered from my UPS that's running Pi-Hole, so I threw the script on there. Anything that's always-on will work. You'll probably want to have ssh access to whatever you choose so that if there is a deliberate or sustained outage, you can login to disable the script.

The beauty/limit of it is that since this is checking connectivity via the LAN, a WAN problem (said toddler discovering the power switch for the router, for instance) won't trigger it (in most cases; I guess the switch functionality could fail while retaining WAN access), so your phone won't blow up should that occur. Similarly, if your internet access upstream of the router is down, you won't be informed. You could check for both of these by substituting the private IP address with something obtained via a dynamic DNS service or the like, in which case it would check both WAN and LAN connectivity.

Future updates may check things like HDD and CPU temps, or extended CPU load, but for now, this meets my needs.