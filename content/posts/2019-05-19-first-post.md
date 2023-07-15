---
title: 'First Post!'
date: Sun, 19 May 2019 07:05:53 +0000
draft: false
tags: ['short', 'meta']
---

Before Reddit, there was Digg. Before Digg, there was /.

Anyway, moving on. This site aims to share some of the more interesting stuff that I do, and also, a way for me to play with AWS, because #marketability. I'll have to figure out Azure and Google Cloud later.

When the .dev TLD launched, I thought, "self, you should get one of those, because all the cool kids have a website as a shrine to their hubris." After an agonizing decision of whether to use first.last, firstLast, firstInitLast, or firstInit.last, I settled on the latter. My name is Stephan, and I like to pretend I'm a software developer. You can read more about my background in my [About Me](https://sgarland.dev/about-me/) page, but suffice it to say, I've been playing with computers long enough to be able to write an autoexec.bat file without the help of the internet (don't test me, please, it probably won't work).

I had a website for my IT repair side business (it is incredible what people will pay you to run Malwarebytes) on Bluehost, and considered adding this onto it. However, they wanted more money for a second domain, and I also had been toying with the idea of hosting it myself. I was worried about downtime, though, and so looked into EC2. I had some experience with it, and I figured a t2.micro could handle the tiny amount of traffic I got. Then, I saw Lightsail, and decided to make the leap. It's a quarter the cost of Bluehost, and I get the joy of maintaining the environment myself. The plan is to set up load balancing to my server, and potentially a friend's, for fun and profit.

I first tried AWS' multi-site WP instance, but had issues getting the two sites to play nicely. I'm sure it can work, but it wasn't for me. Practically everything on my home server is containerized, so I thought, why not Docker? Delete instance, spin up Debian Stretch. Hallelujah, people have already done the legwork. Thanks to [selloween/docker-multi-wordpress](https://github.com/selloween/docker-multi-wordpress), the process was dead simple. OK, relatively easy. I did have to add SSL Insecure Content Fixer - because WP enjoys a sundry mix of secure and insecure sources - I had to change some folder permissions that the author admitted were wrong but didn't fix, and for my afore-mentioned IT site, I had to add some \\x20 to a tagline to prevent it from barfing up default theme text out of nowhere... Point being, it was surprisingly easy, and I now have two WP sites, with unique domains, running on one tiny VPS - 512 MB RAM, 1 vCPU, 20 GB of storage.

That brings us up the present. Cheers!