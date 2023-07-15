---
title: 'Refrigerator Monitoring for Fun and Profit'
date: Thu, 17 Sep 2020 04:26:56 +0000
draft: false
tags: ['monitoring', 'rpi']
---

I'm not sure if this is common, but it should be - a second fridge. While ours ostensibly began its life for holding cakes (my wife is a home baker), it's morphed into extra food, beer, and the like. It lives in our garage, which isn't an ideal space for a fridge with respect to its lifespan, but it's the only reasonable spot in the house we can fit it.

This is all fine and dandy, until the GFCI outlet it's plugged into trips. It's not like we go out to it frequently, so it may be hours or even a day before the issue is noticed. This is less than ideal for food preservation. Why is it plugged into a GFCI outlet? NEC 210.8(A)(2) requires any outlets in a garage to be GFCI. There is no exception for appliances. Builders get away with it in the kitchen by locating the refrigerator outlet more than 6 feet from the edge of the sink, and thus easing into an exception to the rule.

Refrigerators, you see, often don't like GFCI outlets. Usually, it's the auto-defrost feature of the freezer that can leak some current, and cause the GFCI to rightfully trip, but the inductive kick of the compressor cycling can also sometimes cause it. It's also possible, of course, that the outlet itself has aged and is failing, and that was my first attempted fix. I even replaced the downstream, non-GFCI (but GFCI-protected) outlet in the minute chance that it was causing the trips. Alas, no. And with that, we finally get into tech.

My first thought was to leverage one of the many smart plugs I have laying around with [IFTTT](https://ifttt.com/), but unfortunately my plugs (TP-Link - I love their smart home products, other than this edge case) don't communicate their status back to the controller, so it can't read it to alert upon loss of connectivity. To be clear, their Kasa app is aware of a device's reachability, so the ability exists, but isn't implemented.

My next thought was to buy a smart outlet with Zigbee or Z-Wave, but I realized that I don't have a controller (I have Echoes, but not the Echo Plus, which can talk to Zigbee devices), so that was out.

I then remembered that I had an old Raspberry Pi Zero W kicking around doing nothing. When brewing beer, it acts as a relay for my Tilt hydrometer to push logging data to the cloud, but otherwise, it does nothing. I could write a simple script like [I did here](https://sgarland.dev/posts/2019-07-27-server-healthy-check/) and... wait, I work for a monitoring company. Duh.

I loaded up [Hypriot](https://blog.hypriot.com/), as I'm running that on my RPi 4 and love it - if I ever want to put stuff into Docker on this, I'm ready to go. I realized after perusing Hypriot's site that they support cloud-init, so I [shamelessly stole](https://github.com/hypriot/flash/tree/master/sample) wrote up some [YAML](https://pastebin.com/kes45avm), flashed the card, booted it up, and... hooray! I do have a lingering issue getting my runcmd script to execute, but I'll solve that. It obtained a DHCP lease, installed packages, and imported my SSH key, so I'm happy enough.

Next step, configuring it in LogicMonitor. Add Device, and whoops, snmpd isn't installed. Back to YAML, add that in, try again. OK, sorted. From there, it was a simple matter of choosing a datapoint to alert on (I opted for Ping Loss Percent, but Host Status would have also been suitable), creating an escalation chain, and testing it out. For my existing devices, I have my escalation chain to only run during business hours. If this website goes down in the middle of the night, I don't care enough to wake up and fix it. Similarly, if my server drops out, it can wait. For the fridge, however, the large amount of food and drink is worth the annoyance, so it's set to alert 24/7. Between the polling interval and other latencies, there's about a 2 minute delay between power loss and alert, which is fine.

Regarding alert fatigue, the fridge has only tripped offline twice in the past month or two, so this isn't a common occurrence. I'd just like to know in real-time when it happens, instead of stumbling across the problem hours later. Full laziness would be rigging up a curved rod on a cam to reset the outlet when it receives an outage notification, but that's beyond the scope of this (for now).

UPDATE: The day after writing this, I received two alerts at night. Luckily (as they were spurious), I slept through them - vibrate apparently isn't enough to wake me (fear not, employer, PagerDuty startles me awake with the smooth sounds of a barbershop quartet). This is problematic, so I went investigating.

```
# /var/log/syslog
Sep 18 01:05:02 fridge-watch wpa_supplicant[354]: wlan0: Trying to associate with SSID 'Garland IOT'
Sep 18 01:05:03 fridge-watch wpa_supplicant[354]: wlan0: CTRL-EVENT-ASSOC-REJECT bssid=00:00:00:00:00:00 status_code=16
Sep 18 01:05:04 fridge-watch wpa_supplicant[354]: wlan0: Trying to associate with SSID 'Garland IOT'
Sep 18 01:05:04 fridge-watch wpa_supplicant[354]: wlan0: CTRL-EVENT-ASSOC-REJECT bssid=00:00:00:00:00:00 status_code=16
Sep 18 01:05:06 fridge-watch wpa_supplicant[354]: wlan0: Trying to associate with SSID 'Garland IOT'
Sep 18 01:05:06 fridge-watch wpa_supplicant[354]: wlan0: CTRL-EVENT-ASSOC-REJECT bssid=00:00:00:00:00:00 status_code=16
Sep 18 01:05:06 fridge-watch wpa_supplicant[354]: wlan0: CTRL-EVENT-SSID-TEMP-DISABLED id=0 ssid="Garland IOT" auth_failures=1 duration=10 reason=CONN_FAILED
```

In retrospect, this is unsurprising. There are many references to this specific problem on RPi forums, albeit with the problem usually not resolving itself. In this case, I opted to change the alert escalation interval to require 20 minutes of sustained loss before sounding the alarm. The longest I saw in the logs was about 5 minutes, so that should be more than enough time for the Pi to re-associate. Status code 16 for auth failure is due to a timeout failure during the auth sequence, which could be correlated to signal issues.  
  
UPDATE 2: It again disassociated, this time for ~22 minutes, or until I picked up the Pi. I'm unsure if me moving the Pi allowed it to obtain a better signal, or if it was merely a coincidence. I've since disabled power saving on the Pi, and will continue to monitor.