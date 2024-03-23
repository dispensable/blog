---
layout: post
title: Fix cassandra bootstrap keep failed
tags:
- cassandra
---

When you add a new node to cassandra cluster, the bootstrap process will be started to streaming data from other nodes. This process can failed in many situation. It can be resumed with `nodetool bootstrap resume`.

But when i add a new node to our cluster. The boostrap process keep failing even if you run them with `while nodetool status | grep UJ; do nodetool bootstrap resume; done`. I run this process for 3 days it still keep failing during streaming. And the `bootstrap resume` seems not geting their work continue from last failed postion. It just start with a fresh streaming from 0%. The /data dir keeps grows until a compaction process kicks in. With very high load and disk io but zero valide data.

I was thinking if our node's disk is too big(3.5T SSD). And the streaming throttle is too low(24MB/s). So it need a long time to finished the streaming. But after 24h struggle it still failing. So I incr the throttle to 100MB/s. And add more compaction threads. But still the `nodetool status` keep failed after about 10-15% progress which is very fruastrating.

After days of debug, I find that when our prod cassandra bulkload task finished the straming seems failed. Maybe the sstableloader streaming task makes the bootstrap streaming failed, which is possible cause when bulkload finished it will drop the old table. Which means the schema changed and if the bootstrap streaming is work on that table. I don't know if the bootstrap process can handle this.

So i pause the bulkload tasks(thank airflow again). And the bootstrap finished after a while! Finally.

Wrapping up: do not do schema change like droptable and streaming bulkdata to cluster when u bootstraping a new node. It may failed your bootstraping process.

Ps: You should clean the data dir after the bootstrap failed. Looks like the node successfully joined has a high disk usage comparing to other nodes(May be the bootstrap resume not cleaing the data?)
