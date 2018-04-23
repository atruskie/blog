---
title: "Investigating Excessive Memory Usage in Redis"
date: 2016-09-04T18:00:00+10:00
draft: false
tags: [Redis, Resque, RAM, ecosounds]
categories: [Software, Diagnostics]
---

At [www.ecosounds.org](http://www.ecosounds.org) we use Resque as our asynchronous job processing
system. Resque was created, used, and open sourced by GitHub. It is a queue based implementation
built on top of Redis. Redis is an in memory, key-value store (a database) that is exceptionally fast.

The system has its limitations but has generally worked pretty well for us. We’ve run in excess of 
a million jobs through Resque and we consider that to be a decent amount for a small site like ours.

Now to the problem: Redis is using an excessive amount of RAM. The job system was idle, no jobs were 
in queue and yet Redis was using 4.62GB of RAM. To compound the problem:

-   I did not write the code for our job system and have since inherited the code base
-   I did not fully understand Redis
-   I have previously interacted with Redis almost entirely through the Resque API 
    (and had essentially learnt nothing about Redis – great to get started, not so much now)
-   I knew that jobs should have been deleted after they were run
-   That much RAM usage meant we needed to maintain a more expensive server
-   And that much RAM usage meant that there was no way we would be able to enqueue another large batch of work

To further complicate matters, the [method Redis uses](http://redis.io/topics/faq#https://redis.io/topics/faq#background-saving-fails-with-a-fork-error-under-linux-even-if-i-have-a-lot-of-free-ram) to save periodic backups of the database, coupled with strange behaviour from our hypervisor, meant that we needed twice of Redis’ current RAM usage to make such backups. Not having another 4GB of RAM lying around this now meant our Redis instance was truly in memory – any outage would destroy the job queue.

The contents of the database were almost entirely opaque to me. I started trying to fix the problem by extracting invariant parts of our payloads out into separate keys. For example, the payloads to run a job would normally look like this:

{{< gist atruskie 5cc3f8cc519da268f846228ffc677624 "payload1_full.json" >}}

{{< gist atruskie 5cc3f8cc519da268f846228ffc677624 "payload2_full.json" >}}

In those payloads, you’ll note that 92% of them are identical. These identical payloads have twice before been replicated 350 000 times for large job sets. My idea was to add some extra logic so that the payloads would be reduced to:

{{< gist atruskie 5cc3f8cc519da268f846228ffc677624 "payload1_variant.json" >}}

{{< gist atruskie 5cc3f8cc519da268f846228ffc677624 "payload2_variant.json" >}}

And the rest of the payload could be resolved using some extra logic in the worker. The result 
should have been a saving of $1.62\text{GB}\;-\;132.86\text{MB}\;=\;1.48\text{GB}$ -- a 91.3% saving of RAM.

This is a fine solution but I realised--perhaps too late--that I wasn’t actually solving the real problem. If our jobs were correctly disposed of, then we could probably handle the temporary RAM spike for large jobs sets, even with the massively inefficient payloads.

To fix this, I had to learn about the system I was using. I had to find some way to inspect, analyse, and understand the data in Redis.

I started by SSHing directly into our Redis server and attempted to use the `redis-cli` tool to inspect the database. The tool is in many regards excellent – powerful, simple, scriptable. However, without expert knowledge of Redis commands I found I made little progress understanding the data. The `INFO` command provided some data on memory usage but it didn’t tell me how much or where the data was.

A quick search for a better solution led me to this fantastic package on GitHub: Redis Memory Analyzer (`rma`) by GameNet. This tool would apparently provide a breakdown of memory usage by datatypes and key for the database. Not wanting to run untrusted code on a production server (or install unnecessary dev dependencies), I first setup a stunnel client connection (TLS encrypted) from my dev machine to the production server. As we assume an unsecure public network between servers, we already had an stunnel server set up, that only accepted our certificate and connections from trusted IPs, to provide secure remote access to the Redis database.

I needed to add this section to the default stunnel config on my dev machine:

```toml
[ecosounds-redis]
accept = 56379
connect = db.ecosounds.org:*****
verify = 3
client = yes
CAfile = ..\redis.crt
```

<aside>An aside: my security precautions were a balance of risks; not perfect but good enough for the task. Even though I decided against installing untrusted software (and more importantly, all of *its* dev dependencies), I still gave untrusted code access to a production database. This risk was balanced by the fact that we store no sensitive information in Redis and that destruction of that database would have had minimal impact as the job queue was empty.
</aside>

Then I installed `rma` and executed the tool with:

```shell
C:\Users\Anthony> rma -s 127.0.0.1 -p 56379 -a *********************************************
```

The result was amazing. The tool did a particularly good job at revealing unobvious memory overheads that are a result of Redis’ penchant for speed. You see the full report [here](https://gist.github.com/atruskie/1597e5e4cd3e36c0f446c411c03ef812) and I’ll highlight some sections below:


-   99% of the keys stored are from the resque:status gem. That's 545 073 keys; one per resque job since February this year.
-   all of those stored are from completed (or failed) jobs!
-   the string values for the resque:status gem's keys are using 3.4GB or RAM, and 4.5GB in reality (1.33 ratio overhead)
-   For the special Resque failed jobs queue (which I originally thought may be the problem):
    -   The Resque `failed` queue is isolated and
    -   For Resque `failed` queue (it is a Redis `LIST`) for 7000 failed jobs uses only 111MB of RAM (stores full payload, and exception stack trace)
    -   Conclusion: the resque failed queue itself is not a problem

The resque:status gem was a plugin we added to resque that stores extra metadata for jobs that are processed. I now knew what was causing the problem but not why.

Now wanting to inspect the data, I found Redis Desktop <https://redisdesktop.com/>. I cannot overstate how useful it is to visualize data, relationships, and hierarchies. Sometimes GUIs are the best tool for the job. I was so impressed with Redis Desktop I donated to their bug bounty found.

After installing and connecting using the same stunnel connection from earlier I then loaded up the database.

{{< figure src="https://i.gyazo.com/615310456ce0b9135effa2a4cf690548.png" title="A screenshot of Redis Desktop in action" >}}

{{< aside >}}
An aside: Redis has almost no indexing or search capabilities. The commands it does have are usually
`O(n)` (linear) in terms of complexity/cost to run. It was evident from the command list that was
displayed at the bottom of the window how unoptimized Redis is for this sort of task. It will still
*very* quick, and only took 30 seconds to load the whole 500 000 keyspace.
{{< /aside >}}

Immediately I could see:

-   Which namespace had the large group of keys
-   I could simply point and click to load one of the resque:status keys
-   I could immediately see the `TTL` was set to `-1`
-   And I noticed another bug: the `name` field was a serialized version of the (very large) payload!

So, the status metadata was never deleted because the keys were set to never expire (`TTL=-1`) – every single job we ever run was being permanently stored. This problem was made twice as bad because for some reason every status had the excessively large payloads stored twice!

From there it was a relatively easy to fix the problems. For the expire problem I found where our application set the default expire options for the resque:status metadata when the application started, knew that setting had no effect, and moved it to a better spot.

For the duplicated payload in the `name` field problem, I searched through the source code of resque:status, found where the name method was defined, and saw how the default behaviour uses a native representation of the job’s class (which we had stuck a bunch of instance variables too) as the name for the status by default. Once I worked that out, I overrode the name method in the job class so that I could ensure an appropriate name of my choosing was used by resque:status for the `name` value.

I added unit tests for both bugs to ensure neither of these problems would happen again.

## Conclusion

This investigation took 5 hours to complete. I spent another 2 hours writing this blog post to document my work. I am ecstatic that I took the time to learn the intricacies of the system I inherited and glad I chose to properly understand the problem space before blindly pushing ahead with my original solution. Had I not taken the time those two bugs they would have surely vexed me in the future. I anticipate that those bug fixes will reduce the status metadata memory usage more than 50% and that Redis will delete expired status metadata regularly to keep overall memory usage down under 1GB even with a full ~300 000 size job set enqueued.

I am pleased that I found two useful, well maintained, open source tools that aided my exploration. Hats off to both of these tools and their maintainers for producing such excellent software.

While the problems I addressed were highly specific to our codebase my hope is that the tools, techniques, and the method I used are applicable to other situations – especially for developers that are new to using Redis.
