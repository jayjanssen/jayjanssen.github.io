---
title: "Is Xtrabackup's --safe-slave-backup still relevant?"
date: 2023-04-21T08:43:50-04:00
draft: true
tags: ['xtrabackup', 'mysql8']
---

Like a lot of people, we're going through MySQL 8 upgrades and finding the sorts of things that change from major version to major version.  One of these things that changed behavior in MySQL 8 is in Xtrabackup.

Specifically, and for time immemorial, we have used the `--safe-slave-backup` in Xtrabackup[^1].  We take Xtrabackups from our replicas, so it sounds like a setting everyone can agree with, right?

{{< admonition title="TL;DR" >}}
* If you use Xtrabackup with `--safe-slave-backup` on a MySQL 8 replica, it will stop replication for the entire backup when it did not do so on 5.7
* If you don't know what `--safe-slave-backup` really does and you use it, you should check it out.  Hint: You probably don't need it.
{{< /admonition >}}

## Differences between 5.7 and 8.0 behavior 

Take a close look at the differences in the Xtrabackup documentation between 8.0 and 2.3 (the Xtrabackup version that supported MySQL 5.7):
* [2.3 --safe-slave-backup](https://docs.percona.com/percona-xtrabackup/2.4/innobackupex/replication_ibk.html?h=replication+envir#innobackupex-safe-slave-backup)
* [8.0 --safe-slave-backup](https://docs.percona.com/percona-xtrabackup/8.0/xtrabackup_bin/replication.html#the-safe-slave-backup-option)

As one might expect from an option called `--safe-slave-backup`[^1], both pages state:
{{< admonition type=quote >}}
Using this option is always recommended when taking backups from a replica server.
{{< /admonition >}}

However, looking closely at the differences, Xtrabackup 8.0 doc states:

{{< admonition type=quote >}}
As of Percona XtraBackup 8.0.22-15.0, using a safe-slave-backup option stops the SQL replica thread before copying the InnoDB files.
{{< /admonition >}}

{{< admonition type=warning >}}
What may not be clear here is that copying InnoDB files is essentially the *entirety* of the Xtrabackup.  In 5.7, this was only done *after* the Innodb files were copied when other non-transaction tables were copied.
{{< /admonition >}}

We did not notice this behavior change until we started getting replication stopped alerts on our backup replicas.  

## Which temporary tables?

It all comes down to temporary tables.  First, we need to distinguish between [*internal* temporary tables](https://dev.mysql.com/doc/refman/8.0/en/internal-temporary-tables.html) and (what I like to call) *explicit* temporary tables, or those created explicity by clients with the [`CREATE TEMPORARY TABLES` statement](https://dev.mysql.com/doc/refman/8.0/en/create-temporary-table.html).

## Why do we care about explicit temporary tables and replication?

Why do we care about explicit temporary tables on replicas?  It's complicated, so I'll let the 5.7 MySQL manual explain: https://dev.mysql.com/doc/refman/5.7/en/replication-features-temptables.html.  Got all that?

**TL;DR:** `STATEMENT` based replication

## Why is Xtrabackup 8 stopping replication for the entire backup?

Ok, but why the change iin Xtrabackup 8?  I'm actually not sure.  At first I thought it was because explicit temporary tables were still defaulting MyISAM in 5.7, but that isn't true:

```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.41-log MySQL Community Server (GPL)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create schema test;
Query OK, 1 row affected (0.02 sec)

mysql> use test;
Database changed
mysql> create temporary table test(i int primary key);
Query OK, 0 rows affected (0.03 sec)

mysql> show create table test\G
*************************** 1. row ***************************
       Table: test
Create Table: CREATE TEMPORARY TABLE `test` (
  `i` int(11) NOT NULL,
  PRIMARY KEY (`i`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.01 sec)
```

though you can still tell it to use MyISAM:
```
mysql> create temporary table testmyisam(i int primary key) engine=myisam;
Query OK, 0 rows affected (0.02 sec)

mysql> show create table testmyisam\G
*************************** 1. row ***************************
       Table: testmyisam
Create Table: CREATE TEMPORARY TABLE `testmyisam` (
  `i` int(11) NOT NULL,
  PRIMARY KEY (`i`)
) ENGINE=MyISAM DEFAULT CHARSET=latin1
1 row in set (0.01 sec)
```

That behavior hasn't changed on 8.0.  So is the `--safe-slave-backup` even working as intended on 5.7?  I'm not sure.

## Estoeric use cases and binlog formats

But is this problem even still relevant?

Raise your hands if you use `CREATE TEMPORARY TABLES`.  Now raise your hand if you knew explicit temporary tables even exist.  I bet the majority of people writing queries for MySQL are *not* aware of them, much less use them.

Raise your hands if you still use `STATEMENT` or `MIXED` replication.  I'm sure some do, but the default has been `ROW` for a very long time.  Now raise your hands if you have application code that explicitly sets the `SESSION.binlog_format`.  I'm sure it's out there, but it can't be very common.  

This is not a fool-proof argument.  Some client somewhere can still explicitly set the session `binlog_format=STATEMENT` and use temporary tables, right?

Reading the upgrading to MySQL 8 page:
{{< admonition type=quote >}}
SET @@SESSION.binlog_format cannot be used if the session has any open temporary tables.  [source](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-failover)
{{< /admonition >}}

This is helpful to close the gap, but it only seems to do so partially:
```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10474
Server version: 8.0.32 MySQL Community Server - GPL

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

mysql> create temporary table temp(i int primary key);
Query OK, 0 rows affected (0.00 sec)

mysql> set @@session.binlog_format=statement;
ERROR 3745 (HY000): Changing @@session.binlog_format is disallowed when the session has open temporary table(s). You could wait until these temporary table(s) are dropped and try again.
```

That's great, but the opposite is not true:
```
mysql> set @@session.binlog_format=statement;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@binlog_format;
+-----------------+
| @@binlog_format |
+-----------------+
| STATEMENT       |
+-----------------+
1 row in set (0.00 sec)

mysql> create temporary table temp(i int primary key);
Query OK, 0 rows affected (0.00 sec)
```

I'd honestly just rather have a server variable that totally disallows changing session `binlog_format`.

All that said, clients can still do bad things.  Are there clients out there that are doing BOTH:
* setting session `binlog_format` away from `ROW`?
**AND**
* using an explicit temporary table to stage data loading into permanent tables?
  
I'm sure there is, but how likely is it and it is worth Xtrabackup stopping your replica to take the backup?

## Conclusion

I don't fully understand why replication is stopped for the entire backup in `--safe-slave-backup` in in Xtrabackup 8 and not in 2.3, but I'm not sure I need to know, I'm turning it off.  

It'd be nice if the argument was better named (maybe: `--safe-explicit-replicated-temp-tables`) because I expect a lot of people setting up Xtrabackup are using it just based on the name alone and without any real understanding of what it is protecting against.  

The gap that this solves is esoteric and an increasingly improbable case.  Is it possible by not using it I'll get some inconsistent backups, but stopping replication is just not an acceptable tradeoff at this point, at least for my purposes.  

Interestingly, the Xtrabackup manual, in another spot, does call out that this setting is not needed for `ROW` replication:
{{< admonition type=quote >}}
This option is implemented in order to deal with replicating temporary tables and isn’t necessary with Row-Based-Replication.  [source](https://docs.percona.com/percona-xtrabackup/8.0/xtrabackup_bin/xbk_option_reference.html?h=replica+backup#-safe-slave-backup)
{{< /admonition >}}
This is partially just the Xtrabackup documentation being messy.  I've heard they [take PRs](https://github.com/percona/pxb-docs) if you're interested in fixing it.


[^1]: Note that this how the option is still in Xtrabackup, whereas I'll be using the newer term 'replica' in this post.  I'm not interested in any political debates, but I do find the term 'replica' to be a clearer technical term.    
