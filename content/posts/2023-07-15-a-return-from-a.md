---
title: A Return From A Lengthy Hiatus
date: Sat, 15 Jul 2023 19:27:15 -0400
draft: false
tags: ['short', 'meta']
---

## It's been awhile

Specifically, it's been 484 days since I last posted. Things have changed a bit. In short:

* The site is hosted on GitHub Pages instead of an EC2
* The site uses Hugo instead of WordPress
* The site is fronted with bunny.net instead of Cloudflare
* I'm now a Database Reliability Engineer

## What happened?

Yak-shaving, mostly. I'll have ChatGPT turn it into a run-on sentence for fun:

I, with the intention of upgrading my homelab's Kubernetes stack, which is/was running k3os, forever stuck at v1.21, and re-doing my Helm charts because they lacked any proper templating and failed to save me any time, had the thought of hosting this site on said homelab, yet I kept postponing the latter due to the former, as I desired to transition to Talos Linux for the k8s stack, despite its lack of support for Longhorn, and my unwillingness to tackle the complexities of Ceph once more.

Oh yeah, ChatGPT happened too.

### What did you end up using?

As of this writing, the homelab is running k3os as before for all hosted apps, but I also have Talos Linux running in parallel (albeit without a workload). I also have Ceph running successfully, with a trio of Samsung PM983 1.92 TB drives. The difference is that I've shifted the complexity of Ceph onto Proxmox, instead of fighting with it myself. Also, as you may recall, last time I was trying to use Rook. I've since gained some opinions on Kubernetes which I'll share later.

As to the blog, as mentioned, this is hosted via GitHub Pages. I don't remember if it supported a custom domain name when I first looked at it; I vaguely recall that being a reason I didn't want to use it. Regardless, it now does, and it only took me the better part of a day to get everything going. Most of that was because I was insistent on writing a script to fix some of the mistakes made by the Wordpress exporter, instead of doing so by hand like a plebe.

## No, what happened to your job?

I was an SRE, and kept getting pulled into database incidents because I was decent at troubleshooting them. I found out that I really enjoy them, and we opened up a spot for a DBRE, so I made a lateral move. I'll write more about databases soon, I'm sure.

## Will you stop writing again?

Probably, but hopefully not for nearly two years next time.

## What did you learn from all of this?

Making things overly complex is both normal and good, as long as you eventually recognize that most of it is unnecessary and a mistake, and that you really just want things to consistently work well. Tear it down and make it simpler, but retain the knowledge from all of the battles you fought getting to that point.

