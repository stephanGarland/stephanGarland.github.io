---
title: On Being Wrong (Again) – Now With Metrics
date: Tue, 30 Apr 2024 13:41:59 -0400
draft: false
tags: ['linux', 'homelab', 'non-tech']
---

Recently, I [wrote about](https://sgarland.dev/posts/2024-04-07-on-being-wrong/) being wrong. Specifically germane to this post, I wrote:

> Being wrong means I probably had some gross misunderstanding of a system's architecture or the operation of a program, and that means I have an opportunity to learn more about it, and hopefully to be able to guide my decisions.

I was involved in an incident where I thought that some EC2 instances had reached disk saturation. This was primarily driven by three factors: referencing an [outdated man page](https://linux.die.net/man/1/iostat), tunnel vision, and missing the units on a graph.

My work has some [i4i-family](https://aws.amazon.com/ec2/instance-types/i4i/) EC2 instances, which have native NVMe drives. As in, locally attached to the host, not over EBS. If you haven't personally experienced the performance difference, it is massive. IOPS are essentially unlimited (not exactly, but at the 3.75 GiB/device level, you get [400K/220K Read/Write](https://aws.amazon.com/ec2/instance-types/i4i/) – presumably with 4K blocks, but it doesn't specify), albeit the data is ephemeral and does not survive a stop/start. This is a fair trade, if you design your system for the eventuality that the hardware may fail. Ours have 4x such drives, in a RAID0 stripe, so we have about 1.6M/880K Read/Write IOPS.

Some queries were taking an absurd amount of time to complete, and running `iostat` on the host, I noticed that the reported `%util` for the array was hovering between 99 – 100%. I had also previously noticed that there were a large amount of blocks being dirtied by other queries running, and that there were buffer waits being reported. My thought was that the queries dirtying blocks were causing this other large query to have to flush/fill the buffer pool, which was saturating the disks. IOWait was quite low, but that's [known to be deceptive](https://www.percona.com/blog/understanding-linux-iowait/), so I ignored that. In hindsight, had I paid closer attention to the `r/s` and `w/s` values, I'd have found that they were far below device limits.

Had I instead referenced [man7.org](https://man7.org/linux/man-pages/man1/iostat.1.html) for `iostat` (or, you know, just ran `man iostat` on the machine), I would've found this:

> Device saturation occurs when this value is close to 100% for devices serving requests serially.  But for devices serving requests in parallel, such as RAID arrays and modern SSDs, this number does not reflect their performance limits.

Annoyingly, I _know_ this, and had I not gotten tunnel vision on what I was sure was the issue, I might have caught myself. Similarly, had I paid closer attention to the AWS-generated graphs, I would've seen that they report in ops/minute, not ops/second. Alas, that wasn't the case. A good reminder to take a deep breath, think about the metrics you're seeing, and ask if they support your hypothesis.

In order to fully disabuse myself of this notion (and to run an experiment), I opted to utilize the NVMe drives in my homelab. In my Proxmox compute cluster, I have three Samsung 983 DCT 1.92 TB drives, one in each server. This is distributed via Ceph in replication mode, and is connected via Mellanox ConnectX3-Pro NICs in a mesh. This provides for 56 Gbps over Infiniband. As I don't yet have [RoCE](https://en.wikipedia.org/wiki/RDMA_over_Converged_Ethernet) working, I'm using [IPoIB](https://docs.nvidia.com/networking/display/mlnxofedv24010331/ip+over+infiniband+(ipoib)), which is much more CPU intensive. I can get about 20 Gbps before saturating a CPU core. Still, that's not too shabby compared to the 1 Gbps I had before. The drives themselves are rated at 580K/52K IOPS Random Read/Write (it's a read-optimized model), or 3.4/2.2 GBps Sequential Read/Write. So, I have a known maximum.

Ceph of course has its own overhead, but since all I really wanted to know was whether I had misinterpreted the output, that didn't matter. A simple test was set up by creating a test pool in Ceph, and in that pool, creating two `rbd` images. They were then mapped to `/dev/image{01,02}`, exposing them to the kernel as normal block devices. A normal `ext4` filesystem was created on each, they were mounted, and then benched with the built-in `rbd bench` toolset. During this, each device was monitored with `iostat`. I first tested one singly, then both simultaneously. Since the underlying device[s] are the same, this should test how `iostat` shows utilization.

Output during a single test run (peak `w/s` window selected):

```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          15.5%    0.0%    8.0%    0.5%    0.0%   75.9%

     r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz Device
    0.00      0.0k     0.00   0.0%    0.00     0.0k nvme0n1

     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz Device
 16597.00    617.4M 10328.00  38.4%    0.14    38.1k nvme0n1

     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz Device
    0.00      0.0k     0.00   0.0%    0.00     0.0k nvme0n1

     f/s f_await  aqu-sz  %util Device
    0.00    0.00    2.28  96.0% nvme0n1
```

```
sgarland@dell01-pve:~$ sudo rbd bench --io-type write image01 --pool=testbench --io-size 8192 --io-total 8GB
bench  type write io_size 8192 io_threads 16 bytes 8589934592 pattern sequential
  SEC       OPS   OPS/SEC   BYTES/SEC
    1    103152    103581   809 MiB/s
    2    199008     99710   779 MiB/s
    3    287488   95961.3   750 MiB/s
    4    361552   90481.2   707 MiB/s
    5    454848   91044.3   711 MiB/s
    6    553520   90072.3   704 MiB/s
    7    656064   91409.9   714 MiB/s
    8    766800     95861   749 MiB/s
    9    880320    103752   811 MiB/s
   10    998896    108808   850 MiB/s
elapsed: 10   ops: 1048576   ops/sec: 100360   bytes/sec: 784 MiB/s
```

Output with two simultaneous runs (peak `w/s` window selected):

```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          19.2%    0.0%    8.6%    0.4%    0.0%   71.8%

     r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz Device
    0.00      0.0k     0.00   0.0%    0.00     0.0k nvme0n1

     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz Device
 21357.00    591.9M  9520.00  30.8%    0.07    28.4k nvme0n1

     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz Device
    0.00      0.0k     0.00   0.0%    0.00     0.0k nvme0n1

     f/s f_await  aqu-sz  %util Device
    0.00    0.00    1.49  98.0% nvme0n1
```

```
sgarland@dell01-pve:~$ sudo rbd bench --io-type write image01 --pool=testbench --io-size 8192 --io-total 8GB
bench  type write io_size 8192 io_threads 16 bytes 8589934592 pattern sequential
  SEC       OPS   OPS/SEC   BYTES/SEC
    1     68848   69139.6   540 MiB/s
    2    139104   69698.4   545 MiB/s
    3    218880   73061.7   571 MiB/s
    4    300704   75254.2   588 MiB/s
    5    382704   76542.9   598 MiB/s
    6    464544   79138.1   618 MiB/s
    7    543344   80846.8   632 MiB/s
    8    614624   79147.7   618 MiB/s
    9    689072   77672.5   607 MiB/s
   10    761792   75877.2   593 MiB/s
   11    836736   74437.3   582 MiB/s
   12    911088   73547.7   575 MiB/s
   13    988256   74725.3   584 MiB/s
elapsed: 13   ops: 1048576   ops/sec: 75523   bytes/sec: 590 MiB/s

sgarland@dell01-pve:~$ sudo rbd bench --io-type write image02 --pool=testbench --io-size 8192 --io-total 8GB
bench  type write io_size 8192 io_threads 16 bytes 8589934592 pattern sequential
  SEC       OPS   OPS/SEC   BYTES/SEC
    1     45952   46151.9   361 MiB/s
    2     95120   47662.6   372 MiB/s
    3    141968   47390.5   370 MiB/s
    4    191536   47935.2   374 MiB/s
    5    240320     48105   376 MiB/s
    6    290560   48920.9   382 MiB/s
    7    340224   49020.1   383 MiB/s
    8    395408   50687.3   396 MiB/s
    9    453680     52428   410 MiB/s
   10    510992   54047.1   422 MiB/s
   11    567488   55384.8   433 MiB/s
   12    630288     58012   453 MiB/s
   13    684544   57826.4   452 MiB/s
   14    744768   58216.8   455 MiB/s
   15    812784   60454.2   472 MiB/s
   16    912272   68955.8   539 MiB/s
   17   1029440   79829.2   624 MiB/s
elapsed: 17   ops: 1048576   ops/sec: 61048   bytes/sec: 477 MiB/s
```

You can clearly see that `iowait` was relatively low for both, and even though there were ~30% more writes/sec at peak for the simultaneous run, `%util` for both is effectively at maximum (96% vs 98%). The same held true for reads. Thus, `%util` is indeed fairly meaningless for devices capable of parallel processing, like NVMe drives. At least I learned something.
