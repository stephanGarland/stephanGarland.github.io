---
title: On Being Wrong
date: Sun, 07 Apr 2024 21:05:25 -0400
draft: false
tags: ['linux', 'homelab', 'non-tech']
---

I wrote previously about how some things should just work, should not need mucking about with, and how I shifted my NAS' duties to TrueNAS Scale. I must now inform you that I was incorrect, and have shifted back to running it manually on Debian.

I'm a creature of habit. I also like knowing how things work, how to fix them when they don't, and how to improve them. While TrueNAS (Scale, anyway; I assume Core also) does begrudgingly allow you to ssh in, the MOTD is a warning banner that they guarantee nothing once you've touched it outwith their blessed API path. I can't fault them for this, as users are prone to breaking things. This is probably doubly so for the average TrueNAS user, based on reading through their forums. No offense to anyone who is technically competent and uses it (I personally know at least two such people), but it is designed as an appliance, and the fact that you _don't_ need to know how to navigate a CLI is a feature.

For me, though, the fact that I couldn't easily add in 3rd party tooling without fear of breaking things eventually drove me batty. The fact that one can simply `zfs export tank` and reattach it to another system certainly makes it less painful. So, I set about finishing the rewrite of [my fork](https://github.com/stephanGarland/ansible-initial-server) of Packer + Ansible scripts to create Debian VMs. It's a complete rewrite, does quite a bit more, and as of this writing, is barely in a `0.0.1` state as I would declare it. It works-ish, as evidenced by the NAS currently running, but I'm not yet ready to make the repo public.

But this post isn't intended to be technical – that will be later, as this also coincided with a major homelab overhaul / emergent repairs. This post is to discuss being wrong, why it should be embraced, and why it's so frustating when others do not embrace this belief.

I'm wrong a lot. I mean, not _that_ often (hi, future employers), but often enough to remind me that I'm far from perfect. My favorite way to be wrong is when I'm running experiments to test a hypothesis. Being wrong means I probably had some gross misunderstanding of a system's architecture or the operation of a program, and that means I have an opportunity to learn more about it, and hopefully to be able to guide my decisions. The other kind of wrong, of course, is when one states something to be true, and someone who knows better corrects you. Ideally they do so with kindness, but even in the best of cases, it can feel humiliating, as though the technical persona you've carefully cultivated is now destroyed. Everyone will see that you're a fraud, will no longer trust you to be the SME, and will think lesser of you. Bosses will turn a cold shoulder at promo time, and in general, your life will fall apart.

If this has ever been your actual experience, I am terrifically sorry, and hope you're in a better situation now. In my experience, this is in fact not how this usually plays out, especially if you figure out the correct answer and then come back later to demonstrate this (with sources, of course). If others care at all, they're more likely to respect you more for demonstrating humility and grace, and an intellectual tenaciousness to find the right answer, and to spread that newfound knowledge.

In contrast, I've worked with people who could not, would not be incorrect. To say this is frustrating and difficult is a massive understatement. Not only do you grow to not trust them, they grow resentful at your tiresome corrections, and you grow resentful at the extra work being placed upon you for something you knew the answer to last week.

The next time you're wrong about something, own it. Find out where you went wrong, correct your knowledge, and thank the person who gave you a reason to read docs.
