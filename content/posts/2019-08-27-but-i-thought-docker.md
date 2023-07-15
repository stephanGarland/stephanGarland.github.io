---
title: 'But I thought Docker fixed dependency hell...'
date: Tue, 27 Aug 2019 05:57:58 +0000
draft: false
tags: ['automation', 'bash', 'docker', 'mongo']
---

For a school project, I had to install [mongoDB.](https://www.mongodb.com/) OK, no big deal, docker pull mongo. Since I run a home server, and the purpose of this mongo installation was to create an app that uses it, I figured installing it there and making it available to the rest of my group would be beneficial/helpful. Hmm, might want to also install some kind of GUI management tool - [nosqlclient](https://www.nosqlclient.com/) looks like it'll work. It's also containerized, so even better.

A short while later, I've got both of them humming along, except I can't seem to get the shell built-in to nosqlclient to talk to my mongo server. Weird. I confirm that I can run shell within the mongo server's container.

I hit up the dev in Slack, paste my logs, and he says he can't see the problem, and that it should be working. He says that the login string that nosqlclient uses to connect to a db is the same that's being used to connect the shell, so my results make no sense. Terrific. All I can see in the log is that the connection is getting reset, with the exception being thrown from [line 606 of node/net.js.](https://github.com/nodejs/node/blob/master/lib/net.js#L606) OK, that's calling tryReadStart(). Line 551. Read from a socket, or throw an exception. Not much help.

After an embarrassing amount of time, it occurs to me I should look at the logs for the mongo server. Oh, this is more helpful: "34348 cannot translate opcode 2010." [Line 120 of mongo/message.h](https://github.com/mongodb/mongo/blob/master/src/mongo/rpc/message.h#L120) says that's using networkOp. Well, there's an enum for NetworkOp on line 47. Jackpot!

```
 // dbCommand_DEPRECATED = 2008,      // These were used during 3.2 development, but never in a
    // dbCommandReply_DEPRECATED = 2009, // stable release.
    // dbCommand = 2010,      // These were used for intra-cluster communication in 3.2, but never
    // dbCommandReply = 2011, // by any driver. Deprecated in 3.6 by OP_MSG and removed in 4.2.
```

As it turns out, while mongo:latest is, of course, the latest version of mongodb, nosqlclient née mongoclient bundles an older version (3.2, to be precise) of mongo for its utilities, and as seen above, the two are incompatible.

Time to fork. nosqlclient/.docker/install-mongo.sh is responsible for getting the mongo utilities. There it is, pulling v.3.4.2. Find the new location's URL, and... great, the tarball has some kind of hash in the root directory, so the script as-is fails. Ineffectually play with various combinations of mv and find/exec before reading the manpage for tar, and discover that strip --1 drops the root directory. Nifty.

So now we're pulling the correct version of mongo. Let's build the docker image. Several minutes later, all done, so let's redeploy mongoclient. Shell still fails to execute. Logfile is just pointing to node/net.js again, so that's no help. Log into a shell in mongoclient, poke around. Oh, mongo can't execute, because it's missing libssl and libcrypto. To be fair, basically every distro warns you about manually installing tarballs instead of using their package manager. What had happened was, nosqlclient is using Debian Jesse, aka Debian 8. mongo v4.2.0 is built for Debian Stretch, aka Debian 9. Guess what got updated in Stretch? libssl and libcrypto.

* nosqlclient/.docker/install-deps.sh
* curl libssl1.1\_1.1.1.0k-1~deb9u1\_amd64.deb && dpkg -i && rm
* Rebuild docker image
* Redeploy mongoclient
* Login, attempt shell
* Success!

Things I learned/reaffirmed:

1.  Docker may have eliminated dependency hell, but that doesn't mean that building the images is worry-free. Someone has to pay the piper.
2.  Porting in future packages to older versions of the OS is fraught with problems.
3.  When in doubt, dig through the stack trace, and start reading code.

In closing, I'd also like to reaffirm my love for the open source community. Instead of calling tech support, waiting to get percolated up to someone actually competent, and then waiting for a patch to be delivered, become the patch. Talk directly to the dev who wrote it. Read the source code, read manpages, read Stack Overflow. Find it, fix it, submit a PR. All gratis. Absolutely incredible.