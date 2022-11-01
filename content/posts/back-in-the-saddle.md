---
title: "Back in the Saddle"
date: 2022-11-01T07:48:28-04:00
draft: false
toc: true
---

## Where I've been
After a long haiatus, I am back in the world of MySQL and infrastructure.  I spent over 10 years at Percona, first as a _MySQL Consultant_, then as a manager, then as an IT _doer of things_, finally as the Director of IT.  Earlier this year I made the decision to return to an individual contributor status at a new company, [Block](https://block.xyz), or more specifically, [Square](https://squareup.com).

{{< image src="https://images.ctfassets.net/2d5q1td6cyxq/2SqLXL2zJmcUUI2QSkUCy6/71701594cb1fdf6f2e60d34297262d6b/square.01.jpg" width="50%" class="center">}}

## What I'm doing now
No doubt some old-timers will remember my [blogs at Percona, mostly about PXC and Galera](https://www.percona.com/blog/author/jay-janssen/).  I'm still interested in cluster tech, but now I'm working with [MySQL Innodb Cluster](https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster-introduction.html).  This is pretty similar to Galera architecturally.  It's a (mostly) synchronous, shared-nothing, Innodb-based cluster.  An extra fun dimension here is deploying this on AWS.  

Because I've been removed from the MySQL ecosystem for so long (despite working at Percona), it's been fun to catch up on all that's new from MySQL community, including:
* MySQL 8.0 (I last used 5.6 seriously)
* MySQL shell and the AdminAPI
* MySQL router
* Innodb Cluster itself

On the infrastructure side, I've been delving deeper into:
* AWS
* Autoscaling groups
* Terraform
* Golang
* Lambdas

I'm hoping to restart my technical blogging as I'm learning more about all these topics and to even make it back to a conference or two in the future.  