---
title: HOWTO Migrate (Postgres) Lightsail --> RDS
date: Sat, 24 May 2025 10:45:48 -0400
draft: true
tags: ['aws', 'cloud', 'db', 'postgresql']
---

A former co-worker reached out to ask if I was interested in helping him do a DB migration
for a client of his, who was running an app in Amazon Lightsail. If you aren't familiar,
[Lightsail](https://aws.amazon.com/lightsail/) is a VPS / kind-of-PaaS offering that
offers compute, block storage, load balancing, RDBMS, etc. for a flat monthly fee. While
it's an attractive offering for people who (understandably) don't want to deal with the
complexity of connecting various AWS services together, it's not the most performant,
especially on the DB side. They don't discuss specifications in terms of hardware generation,
but if I had to guess, it's around 4th gen (e.g. m4.?large). More importantly, they max
out at 2 vCPU / 8 GiB RAM, and 240 GiB storage, with no ability to add more storage. Additionally,
they stopped supporting newer versions of Postgres past 12.

Inexplicably, there is also no easy path to move from Lightsail to RDS. There is a somewhat
convoluted method to move your compute to a standard EC2, but for the DB, the official
method is to stop all traffic, then do a logical dump / restore. That will take quite a long
time to read everything out to a file (the disks aren't the fastest in the world), during which
your site is down hard. By the time you might think you need to move on beyond Lightsail,
you probably have enough traffic where that amount of downtime is unacceptable.

Enter [Postgres Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html).
While physical replication requires the same major version, logical replication can generally cross
versions without any issues, since it's copying the contents, rather than the file layout. The basic
idea is that you create a [publication](https://www.postgresql.org/docs/current/logical-replication-publication.html),
which other DB clusters can [subscribe](https://www.postgresql.org/docs/current/logical-replication-subscription.html)
to, with various options. You can choose to publish an entire database, specific tables, and
even to the granularity of specifying row-level conditions with `WHERE` clauses. You can also have
multiple publications, independent of one another.

By default, when you create a subscription, a replication slot is created on the publisher. This slot
handles all changes from the publisher to the subscriber, and their existence will also cause Postgres
to hold WAL until it's been consumed by any subscribers. By default, a subscriber will receive all
data in the table (`copy_data=true`), regardless of its age. During this initial synchronization,
any changes to the tables are being buffered so that when the initial sync is complete, they can be
replayed, and the subscriber brought up to date. This is extremely useful, and is a feature that MySQL
unfortuanately doesn't have. However, this also means that you need to have enough free disk space on
the publisher to be able to buffer all changes while the subscriber is pulling changes. If you have
very little disk space left on a Lightsail DB, and you have a lot of writes, this can prove problematic.
Worse, since reading data from the heap can (if your DB is large enough - and with a max of of 8 GiB of RAM
on Lightsail, most DBs will be) evict in-use pages from buffers, prod traffic will suffer, latency will
skyrocket, and the DB's performance in general will be poor.

One solution to this problem, which I chose to do, is to have an intermediary DB acting as a buffer.
Lightsail, like most other managed DB offerings, has snapshots. Additionally, while creating a
subscription will automatically create a replication slot, you can also manually create them,
and then specify them for use with a later subscription. The tl;dr of the entire process is this:

* Create an RDS instance, using whatever size you'd like
  * Ensure that you create a custom parameter group
  * Change `wal_level` to `logical`, and `rds.logical_replication` to `1`
  * Consider changing other parameters, such as `max_logical_replication_workers`, `max_parallel_apply_workers_per_subscription`, etc.
* On the Lightsail instance, change `wal_level` to `logical`
  * NOTE: this is a static parameter, so this *does* require a small amount of downtime when rebooting the DB
* Create a logical replication slot on the Lightsail DB, which will begin buffering WAL
* Create a publication on the Lightsail DB
* Snapshot (or use an automatic snapshot, as long as your snapshot and replication slot times sufficiently overlap) the DB
* Launch a new Lightsail DB using that snapshot, and once it's up, create a subscription to the Lightsail DB
* Verify that the new DB is pulling changes from the replication slot with [pg_stat_replication](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION-MONITORING)
* Create a publication on the new Lightsail DB
* Create a subscription on the RDS DB to the new Lightsail DB (this will automatically create a repliaction slot)
* Verify that the RDS DB is pulling changes from the new Lightsail DB's replication slot
* Once the RDS DB is fully caught up, halt writes to the old Lightsail DB, update sequences on the RDS DB, and cutover traffic

Sequences need to be updated as they do not transfer through with logical subscriptions.
To be on the safe side, pick an increment with plenty of headroom over the last entry
for each table (e.g. `N+1000`), taking into account your typical writes/sec.
