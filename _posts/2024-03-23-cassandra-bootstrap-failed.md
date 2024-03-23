---
layout: post
title: Avoid Dropping Tables When Cassandra is Bootstrapping New Nodes
tags:
- cassandra
---

When you add a new node to a Cassandra cluster, the bootstrap process starts streaming data from other nodes. This process can fail in many situations, but it can be resumed with `nodetool bootstrap resume`.

However, when I added a new node to our cluster, the bootstrap process kept failing even when running it with `while nodetool status | grep UJ; do nodetool bootstrap resume; done`. I ran this command for 3 days, and it continued to fail during streaming. The `bootstrap resume` did not seem to pick up the work from the last failed position; instead, it started fresh streaming from 0%. The /data directory kept growing until a compaction process kicked in, causing very high load and disk I/O but no valid data.

I considered whether the node's disk was too large (3.5T SSD) and if the streaming throttle was too low (24MB/s), causing the streaming process to take a long time to finish. After 24 hours, it still failed, so I increased the throttle to 100MB/s and added more compaction threads. Despite these adjustments, the `nodetool status` continued to fail after reaching about 10-15% progress.

After days of debugging, I discovered that when our production Cassandra bulkload tasks finished, the streaming also failed. It's possible that the sstableloader streaming task caused the bootstrap streaming to fail because when the bulkload finished, it dropped the old table, resulting in a schema change. If the bootstrap streaming was working on that table, I wasn't sure if the bootstrap process could handle it.

To resolve this issue, I paused the bulkload tasks (thanks to Airflow DAG pause), and the bootstrap process finished successfully after some time.

In conclusion, avoid making schema changes like dropping tables and streaming bulk data to the cluster when bootstrapping a new node. Doing so may cause your bootstrapping process to fail. If your loading tasks occur frequently, they will repeatedly disrupt the bootstrapping process.

PS: After a failed bootstrap, remember to clean the data directory and then use `resume` to resume the bootstrap. It appears that the node that successfully joined has a higher disk usage compared to other nodes even after running `nodetool cleanup` (perhaps the bootstrap resume does not clean the data?).
