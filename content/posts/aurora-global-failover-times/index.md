---
title: "Aurora Global Failovers"
date: 2022-12-02T08:17:57-05:00
draft: false
tags: ['aurora', 'failover', 'disaster recovery']
---

As part of an overwhelming stampede to migrate to the cloud, we are looking at using _AWS RDS Aurora MySQL_ as a platform for some of our database clusters.  Lots of people have lots of opinions about Aurora, some of them are probably justified and some probably not.  

I was interested in testing the high availability and disaster recovery capabilities of Aurora.  To be specific, I am testing Aurora v2 (though I expect v3 to work the same).  I am also specifically testing [Aurora Global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)

## Aurora Global Clusters

Aurora global databases are relatively simple to understand.  Perhaps the best way is to describe it in the order in which you would construct it:

1. Create your first regional cluster
2. Create a global cluster from the first cluster
3. Create replica clusters in other regions

{{<figure src="https://docs.aws.amazon.com/images/AmazonRDS/latest/AuroraUserGuide/images/aurora-global-databases-conceptual-illo.png" attr="Using Aurora global databases, User Guide for Aurora, AWS Documentation" attrlink="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html">}}

Only one cluster can take writes at a time, and it is designated as the _Primary cluster_.  The other cluster(s) are considered _Secondary_.  Replication across region is done by the Storage layer and _does not_ use standard MySQL replication.  

## Aurora Endpoints

Another important concept to understand is [Aurora endpoints](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Endpoints.html).  Each cluster (in each region) has two default endpoints, a writer and a reader.  The writer points to the currently primary writer instance and the reader points to all the reader instance(s) as you might expect.  These endpoints are how your client would connect to the Aurora cluster and they automatically follow when instance roles changes, for example if Writer instance failed over to another in-region.   

Each instance in the cluster also have their own direct endpoints, but use of these is not recommended since instance roles can change automatically.  

In global clusters, endpoints still exist for each regional cluster.  There is no concept of a global endpoint.  Whichever cluster is the Primary has it's Writer endpoint active.  Secondary cluster's have inactive Writer endpoints.  To be specific, the endpoint on a secondary cluster has the _Inactive_ status and the name cannot be resolved in DNS.

{{<figure src="endpoints-in-a-secondary-cluster.png" caption="The writer endpoint is inactive in this secondary cluster">}}

Ergo, the application must be aware of the endpoints in it's deployment region.  If the application is active in the non-primary Aurora region, it must somehow be aware of the endpoint in the Primary region or else not allow writes to the Aurora cluster.  To be fair, there is an Aurora feature for [Global Database Write forwarding](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-write-forwarding.html) that I have not experimented much with yet.  


## Planned Failover (aka Switchover)

Now that we have our shiny global cluster, let's test failover!  AWS calls this _planned failover_, though hopefully there isn't much failure involved. Our goal is to safely change which cluster is primary without any dataloss and (hopefully) not a lot of application downtime.  

This action can be taken from the AWS console with the 'Fail over global database' action on the Global database object, or else via the AWS rds CLI.  Anyone who has done mysql primary switchovers ever should be familiar with the basic steps here:

1. Set the old primary read-only
2. Wait for replication to catch up
3. Set the new primary read-write

Aurora is no exception, but there are a few additional steps.  Because the new primary cluster's writer endpoint is Inactive, it must be made active.

To test this, I connect a `sysbench` client to my primary cluster's endpoint.  Because there is no global endpoint, I have to pay attention to when the secondary cluster's endpoint becomes available in DNS so I can reconnect my client to it instead.  However, I have discovered that

Here is a representative sample of the timings of my operations:

| Time | Action / Event |
|------|----------------|
| 00:00 | Failover started |
| 02:19 | Writes cease in primary |
| 02:38 | Reads case on primary endpoint | 
| 05:05 | Writes start on new primary cluster's _instance_ endpoint |
| 14:19 | The writer endpoint resolves in DNS |

From these timings, we can see clearly that the global failover operation is really supposed to finish around the 5 minute mark (after approximately 3 minutes of write downtime).  Around this time, I can see the AWS console indicating that the new primary cluster's writer endpoint indicates it is in the _Available_ state, yet it does not resolve in DNS.  

I have tested this many times, and to be fair it is not always this slow.  I have observed the writer endpoint resolving in DNS much sooner than it did in my example here.  I believe this is an AWS _bug_ and AWS claims they will be fixing it soon.  

{{< admonition type=bug >}}
In my testing (as of December 2022) write downtime can currently take up to **15 minutes**.  
{{< /admonition >}}

{{< admonition type=warning >}}
Even when the above DNS propogation bug is fixed (assuming that's what it is), I expect that the best that can be expected for writer downtime will be approximately **3-5 minutes**.
{{< /admonition >}}

{{< admonition type=tip >}}
Anyone who needs to run routine DR testing on Aurora Global clusters will want to strongly consider doing so off-peak hours
{{< /admonition >}}

## Unplanned Failover

[Amazon's unplanned failover documentation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-failover) is pretty interesting in the sense that they explicitly tell you _not to use the failover button_ that you use for Planned failovers.  Instead the steps can be summarized as such:

1. Stop writing to your primary db (assuming their primary region is available enough that you can reach your application).
2. Pick your target cluster to failover to
3. Remove the target cluster from the global database
4. Start writing to the new target cluster.

Step 3 effectively splits the target cluster such that you now have two database clusters.  Depending on the severity of the AWS region outage, you may or may not be able to do any of the following:
* See if your primary cluster is up at all and taking writes
* Take some kind of action to ensure said primary cluster is not taking writes any more
* Execute any kind of reasonable STONITH/fencing procedures.  

Based on the fact that AWS recommends against using it, it also that the "Failover" action for global databases is not trustworthy enough to even attempt to execute in the event of a primary region failure and so you have to take an alternative process in the event of a true failure.  Personally, I'd prefer if something at least *tried* to execute the proper failover steps in case the outage is intermittent enough to occasionally be able to reach the region/cluster experiencing the problem.  

As far as the observed time to execute an Unplanned failover, detaching the secondary cluster is a relatively fast operation.  In that situation, nothing is setting the primary cluster read-only nor waiting for replication lag to catch up (at least in the sense that it can confirm all writes from the old primary have made it to the new).  However, the same DNS propogation issue exists in this scenario that will delay the availability of the writer endpoint.  Therefore, once this process is executed, I estimate it takes around 2-14 minutes or 2-4 minutes once the bug is fixed.  

## What AWS states about Failover

In short, the manual seems to acknowledge that "minutes" is to be expected:

{{< admonition type=quote >}}
For an Aurora global database, RTO can be in the order of minutes. 
[source](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-failover)
{{< /admonition >}}

While related marketing content seems to paint a rosier picture:
{{< admonition type=quote >}}
If your primary Region suffers a performance degradation or outage, you can promote one of the secondary Regions to take read/write responsibilities. An Aurora cluster can recover in less than 1 minute even in the event of a complete Regional outage. This provides your application with an effective Recovery Point Objective (RPO) of 1 second and a Recovery Time Objective (RTO) of less than 1 minute, providing a strong foundation for a global business continuity plan. 
[source](https://aws.amazon.com/rds/aurora/global-database/)
{{< /admonition >}}

If you find any other public statements, I'd be happy to hear about them!
