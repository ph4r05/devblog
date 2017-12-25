---
layout: post
title:  "Laravel queueing optimization"
date:   2017-12-23 12:10:39 +0200
categories: blog
excerpt_separator: <!-- more -->

---

Analysis of the database queueing mechanism with few optimizations proposed.
Deadlocks, pessimistic vs. Optimistic locking.

<!-- more -->

## Intro

In the [previous article] I performed a simple benchmark of the basic 
[Laravel] queueing mechanisms. For the benchmarking, I created a new Laravel 5.5 [project].

During the development of another Laravel based project, I came across a 
deadlock problem of the Database worker. The issue is described
in the Laravel issue [#7046].

In the following section, I will describe the pitfalls of the database 
queueing in the Laravel, deadlocks and the solution to handle and 
avoid them.

### Job claim mechanism

The jobs are stored in the `jobs` table in the database.

Available workers are querying the available jobs for processing by the following query:

```sql
BEGIN TRANSACTION;
SELECT * FROM `jobs` WHERE `queue` = ? AND ((`reserved_at` IS NULL and `available_at` <= NOW()) OR (`reserved_at` <= ?)) ORDER BY `id` ASC limit 1 FOR UPDATE; 
UPDATE `jobs` SET `reserved_at` = NOW(), `attempts` = `attempts` + 1 WHERE `id` = ?;
COMMIT;
```

The first `select` essentially selects the first available job in the FIFO manner.
Job is available if `available_at <= NOW()` or the job is *expired* (will get to that later).
The important part of the select query is `FOR UPDATE`. It asks for an *exclusive lock* for the 
selected record meaning "we are going to update the record in the transaction". 

The another `update` query is claiming the particular job by the worker.
The worker claims the job for itself by setting `reserved_at` field to the current time.
If the `reserved_at < NOW() + 90` then the job is not eligible for running and
won't be selected by other workers by the `select` query.

If the job is present in the `jobs` table after some configured time (e.g., 90 seconds)
the job is considered *expired* and is again eligible to run (if attempt counter is not too high). 
This is quite a robust mechanism - if some error interrupts the job execution (e.g., a runtime exception, server reboot)
the job is not deleted from the table and is re-run after the 90 seconds.

[![Jobs queue](/static/queue02/jobs.svg)](/static/queue02/jobs.svg)

Job 1 is expired - it will be picked by the select query. Jobs 2 and 3 are reserved - they are being 
processed by another workers. Select query ignores those. Jobs 4-8 are available to run. Job 9 will be 
available to run in 100 seconds. 

The `reserved_at` criteria guarantees that the job will be run *at least once*
before removing from the database (unless the attempt counter is too high).

After the job is processed the job is deleted from the database:

```sql
BEGIN TRANSACTION;
SELECT * from `jobs` WHERE `id` = ? FOR UPDATE;
DELETE FROM `jobs` WHERE `id` = ?;
COMMIT;
``` 

Note: You may notice the `select` query is a bit redundant. 
The code performing the deletion looks like this:

```php
$this->database->transaction(function () use ($queue, $id) {
    if ($this->database->table($this->table)->lockForUpdate()->find($id)) {
        $this->database->table($this->table)->where('id', $id)->delete();
    }
});
```

The reason why the first select is performed is not quite clear.
My hypothesis is it was some workaround for deadlocks or some refactoring artifact.
I will show the select is not needed, job can be removed immediately without
changing the properties of the system.

The `jobs` database in Laravel 5.5:

```sql
CREATE TABLE `jobs` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `queue` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
  `payload` longtext COLLATE utf8mb4_unicode_ci NOT NULL,
  `attempts` tinyint(3) unsigned NOT NULL,
  `reserved_at` int(10) unsigned DEFAULT NULL,
  `available_at` int(10) unsigned NOT NULL,
  `created_at` int(10) unsigned NOT NULL,
  PRIMARY KEY (`id`),
  KEY `jobs_queue_index` (`queue`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
```

### Deadlocks

There is a possible deadlock when multiple workers interfere on the `jobs`
table with the two queries shown above.

The example of a deadlock:

```
_____________________________________________________________________________________________ Deadlock Transactions _____________________________________________________________________________________________
ID  Timestring           User      Host       Victim  Time   Undo  LStrcts  Query Text                                                                                                                           
 4  2017-12-09 09:54:02  keychest  localhost  Yes     00:00     0        3  select * from `jobs` where `queue` = ? and ((`reserved_at` is null and `available_at` <= ?) or (`reserved_at` <= ?)) order by `id` as
 5  2017-12-09 09:54:02  keychest  localhost  No      00:00     1        3  delete from `jobs` where `id` = ?                                                                                                    

______________________________________ Deadlock Locks _______________________________________
ID  Waiting  Mode  DB        Table  Index                         Special          Ins Intent
 4        1  X     keychest  jobs   PRIMARY                       rec but not gap           0
 5        0  X     keychest  jobs   PRIMARY                       rec but not gap           0
 5        1  X     keychest  jobs   jobs_queue_at_index           rec but not gap           0
```

The worker 4 is selecting the next available job. The select tries to lock
the primary key index because of the `FOR UPDATE`.

The worker 5 processed the job and it is trying to issue delete query to remove the job
from the table. The delete lock is already holding primary key lock. As the deletion 
affects also `jobs_queue_at_index` index the query asks for that lock.

There is also a third select query of another worker, not displayed in this listing (produced by *innotop*) which 
holds lock on `jobs_queue_at_index`.

This in global picture causes a deadlock - each transaction is waiting for a lock that is held by another transaction. 
In the modern MySQL database there is `innodb_deadlock_detect` set to 1 by default. This means the deadlock
is detected immediately and one transaction is rolled back to remove the deadlock condition and progress. 

With older databases or `innodb_deadlock_detect=0` the deadlock will cause 
queries to stall. Each query is waiting for locks that cannot be locked for them.
Until `innodb_lock_wait_timeout` is reached and the rollback is issued. 

Deadlocking is a serious problem if your database does not support `innodb_deadlock_detect`
because it paralyzes the whole job queueing mechanism.

### Rollback consequence

MySQL usually picks the delete query for the rollback.

Note this is the worse option of the two. If select fails it does no harm,
the exception is thrown but worker swallows it and tries to load a new job again - issue the select again.  

On the other hand, the delete rollback causes a little trouble.
As the job remains in the `jobs` table due to deletion fail, it became expired
and processed again by another worker. If the next worker again fails with the deletion 
the process repeats. 

Due to this problem, it can happen the job is processed more than once before deleting from the queue.
This may be a serious problem for some jobs. It is good to keep in mind this situation may occur
when implementing the jobs logic. 

### Laravel ≤ 5.4 deadlock

Before Laravel 5.4 the index was made of `(queue, reserved_at)` which caused another deadlock.
In the Laravel 5.5 the index is just `(queue)` but the upgrade does not change the existing job table
thus you may experience the following deadlock also in Laravel 5.5 project:

```
______________________________________________________________________________________________________ Deadlock Transactions ______________________________________________________________________________________________________
ID    Timestring           User      Host       Victim  Time   Undo  LStrcts  Query Text                                                                                                                                           
8767  2017-12-11 18:08:03  keychest  localhost  No      00:00     1        5  update `jobs` set `reserved_at` = ?, `attempts` = ? where `id` = ?                                                                                   
8768  2017-12-11 18:08:03  keychest  localhost  Yes     00:00     0        4  select * from `jobs` where `queue` = ? and ((`reserved_at` is null and `available_at` <= ?) or (`reserved_at` <= ?)) order by `id` asc limit 1 for up

_______________________________________ Deadlock Locks ________________________________________
ID    Waiting  Mode  DB        Table  Index                         Special          Ins Intent
8767        0  X     keychest  jobs   PRIMARY                                                 0
8767        1  X     keychest  jobs   jobs_queue_reserved_at_index  gap before rec            1
8768        1  X     keychest  jobs   PRIMARY                       rec but not gap           0
```

Here we see 2 workers deadlocking on the first query.
Worker 8767 is trying to claim the job by issuing the `UPDATE` query.
It locked the record *for update* by the previous select and now holds `PRIMARY` key index.
As the `jobs_queue_reserved_at_index` index is affected by the update query it has to be locked too.

Worker 8768 is performing the select with the *for update* locking - asking for a primary key lock.

There is another select query of another worker, not displayed in the listing holding the `jobs_queue_reserved_at_index` index.

This again causes a deadlock.

## Solutions

In the sections below I propose few fixes to the mentioned problem. At first I will focus on the current queueing mechanism
and try it improve a bit. Later is a new locking mechanism proposed.

### Deadlock handling - retry

There is one simple workaround to handle multiple runs of the job: 

```php
$this->database->transaction(function () use ($queue, $id) {
    if ($this->database->table($this->table)->lockForUpdate()->find($id)) {
        $this->database->table($this->table)->where('id', $id)->delete();
    }
}, 5);
```

The number of attempts for performing the delete operation is set to 5. 
It is kind of magic constant approach but works pretty well in practice - shown below.

The probability of the job won't be removed after 5 attempts is very low thus
with this workaround, job is run exactly once with very high probability.

However, the deadlock problem is still present which is a problem for older database servers.

### Deadlock handling - delete mark 

Another way of fixing the multiple job run problem is adding a new column to the jobs table:

```sql
ALTER TABLE jobs ADD COLUMN delete_mark TINYINT(1) NOT NULL DEFAULT 0;
```

Instead of deleting the job the `delete_mark` is set to 1. Since this particular update
cannot cause a deadlock (not requiring index lock) it will always commit so the job 
is not executed more than once.

When the worker asking for next available job gets a job with `delete_mark`
the job is removed from the database. Even if the delete fails this converges
to a state the finished job is removed, eventually. 

The deadlock still happens, just the occurrence is changed. 

### Deadlock handling - removing the queue index

In order to avoid deadlock an elimination of a required condition is needed. 
In this case, the obvious move is to remove `jobs_queue_at_index` index.

This method works and no deadlock will occur anymore but the performance is
degraded significantly (see results below).


### Summary

The proposed methods can improve the system properties by assuring the job is run exactly 
once. If the database supports deadlock detection the system recovers quickly. However, the older database versions will stall the queues significantly. 

## Optimistic locking

The previous approach uses so-called *pessimistic locking*. Pessimistic in a sense the caller
assumes there will be a conflict on rows so it locks the relevant records for further update by the exclusive lock. 
Other concurrent transactions cannot read or modify the record due to the exclusive lock.

Optimistic locking does not use explicit locking mechanism by the database. 
The integrity is maintained via conditioning update by load-time information. 
Let's demonstrate it on the workers:

```sql
ALTER TABLE jobs ADD COLUMN version int(10) unsigned NOT NULL DEFAULT 0;
```

The select query with job claim:

```sql
SELECT * FROM `jobs` WHERE `queue` = ? AND ((`reserved_at` IS NULL and `available_at` <= NOW()) OR (`reserved_at` <= ?)) ORDER BY `id` ASC limit 1; 
UPDATE `jobs` SET `reserved_at` = NOW(), `attempts` = `attempts` + 1, `version` = `version` + 1 WHERE `id` = ? AND version=?;
```

The difference is there is no transaction required to wrap the logic of select + claim, select query is almost the same but
`for update` is missing.
We also added a new column `version` for the optimistic locking. The subsequent update is conditioned on the 
select-time value of the version column. If the update succeeds it increments the version column.  

If the record was modified between select and update by another worker the version value will 
change and the update will fail.

Multiple workers try to load next available job but the only one will succeed to update the version column 
due to the query atomicity. The key here is `affected rows` variable returned by the
SQL server. If the return value is 1 then the worker succeeded in claiming the job and can proceed with working
on it. Other workers will get 0 and try to load next available job again - got preempted by another worker, the
version column changed since they performed their own select.

[![Optimistic locking](/static/queue02/jobs_optimistic_h.svg)](/static/queue02/jobs_optimistic_h.svg)

In the example above only the worker 2 succeeded in claiming the job 1, other workers failed and will 
load a next available job in the next round.

Benefits of the optimistic locking:

- No DB lock so no deadlocks.
- No explicit transactions required. Single auto-commit transactions are enough.
- No need to maintain the connection over the whole transaction.

The disadvantage is visible if jobs are rather short and workers compete for the next available jobs.
The preemption is quite often which increases the overhead of the queueing system.

### Optimization

In the current scenario, the competition over the next available job is quite high if the job duration is
small or if there is a high number of workers. Let's say there are N workers.

Now each one of N workers tries to load one next available job and only one will succeed in claiming the job.
N-1 workers will have to load next available job again. This creates quite an overhead.

But we can load more than 1 next available job from the queue in one worker. The optimized version loads N next available jobs
from the queue and picks one job to process *randomly* (uniform choice). 

Using this strategy the collision of all workers on one job is highly improbable \\(N^{-N}\\). 
This significantly increases the throughput of the queueing system as only a few workers will have to query for 
available worker again due to a collision on jobs.

[![Optimized optimistic locking](/static/queue02/optimized_optimistic.svg)](/static/queue02/optimized_optimistic.svg)

### Optimization price - ordering

Using the original pessimistic locking preserves the job ordering (FIFO). All workers are competing for the first 
available job.

However, with multiple workers the concurrency, OS scheduling and other factor cause the job ordering 
is not guaranteed anymore. N workers pick jobs in order but worker thread scheduling causes job with ID 2 
can be processed sooner than the job with ID 1. This is quite natural and expected property of the queueing system
with multiple workers. 

Optimistic locking optimization adds randomness to the job selection process thus the reordering 
can be higher than a window of N workers. We will see the reordering side effects are not that significant.

With a use of a little math its possible to estimate a round the job will get processed by some worker.
Let's say if a job is processed in round 1 it means it was processed in time - in the round the job became available. 

Let \\(X\\) be a [random variable] for the round in which job is being processed. Then \\(E[X]\\), an [expected value] of X, 
is the average case of the round processing the job. 

Then \\( E[X] = x_1p_1 + x_2p_2 + \cdots \\) where \\( x_i \\) is the round number and \\(p_i\\) is the probability 
of a job being processed in the round \\( x_i \\).

\\( E[X] = \sum_{i=1}^{\infty} i \cdot \left( \frac{N-1}{N} \right)^{N^{i-1}} \cdot \left(1 - \left(\frac{N-1}{N}\right)^N \right) \\)
where \\( \left(\frac{N-1}{N}\right)^N \\) is the probability a job won't be selected in the given round.

\\( E[X] \approx 1.6 \\) for N ≥ 2. Thus on average, the job is processed in 1.6 rounds. 

## Results

The benchmarking was performed with the same methodology as in the [previous article].
Databases used supported immediate deadlock detection and rollback. 

[![Queueing benchmark](/static/queue02/benchmark_optim.svg)](/static/queue02/benchmark_optim.svg)

Where:

- DBP-mysql-0-0-5 is a pessimistic strategy using MySQL database with 5 delete retry counts. 
This is the fastest pessimistic locking technique from all proposed above (ignoring the DB backend). 

- DBP-mysql-0-0-5 is a pessimistic strategy using MySQL database with delete mark.

- DBP-mysql-0-0-5 is a pessimistic strategy using MySQL database without queue index.

- DBO-mysql-0 is an optimistic strategy using MySQL and job select window of size 1 (original optimistic 
without optimization)

- DBO-mysql-1 is an optimistic strategy using MySQL and job select window of size N (10 workers in this benchmark).

Average jobs per seconds: 

Forpsi:

| Method | Jobs per second |
|:-------|:----------------|
| DBP-mysql-0-0-5 | 190.47|
| DBP-mysql-1-0-1 | 155.87|
| DBP-mysql-noindex | 92.29 |
| DBO-mysql-0 | 90.33|
| DBO-mysql-1 | 210.06|
| DBP-pgsql-0-0-5 | 206.39|
| DBP-pgsql-1-0-1 | 184.03|
| DBP-pgsql-noindex | 142.65|
| DBO-pgsql-0 | 54.1|
| DBO-pgsql-1 | 201.06|
{:.mbtablestyle3}

We see the original optimistic locking overhead is significant as the strategy is 50% slower than the pessimistic one.
Using the optimized version the performance is similar to the pessimistic one. MySQL and PostgreSQL perform very
similar to this configuration.

C4.large:

| Method | Jobs per second |
|:-------|:----------------|
| DBP-mysql-0-0-5 | 452.04|
| DBP-mysql-1-0-1 | 347.82|
| DBP-mysql-noindex | 171.61 |
| DBO-mysql-0 | 386.42|
| DBO-mysql-1 | 644.52|
| DBP-pgsql-0-0-5 | 273.13|
| DBP-pgsql-1-0-1 | 332.50|
| DBP-pgsql-noindex | 263.86|
| DBO-pgsql-0 | 180.41|
| DBO-pgsql-1 | 468.33|
{:.mbtablestyle3}

On C4.large the difference between MySQL and PostgreSQL is apparent. C4 has 2 vCPU and more RAM which may cause
this difference.

The optimized optimistic locking is faster than another technique.

### Job ordering analysis

Let's see how the job ordering behaves for multiple different techniques. 

The job ordering is measured in the following way:

- A new table "protocol" is created which stores `(id, timestamp, job_id)`.
- When a worker fetches the job it inserts a new entry to the protocol table corresponding to the job.
- After all 10 000 jobs are processed the protocol table is loaded for analysis, ordered by id.
- The sequence of Job IDs is analyzed: for each protocol record a diff sequence is computed.
- From the protocol we can also check if the job was executed exactly once.

Lets define \\(\text{Diff}_i = \lvert i - \text{job_idx}_i \rvert \\)

Example:

Job ID   | 1 | 2 | 4 | 7 | 3 | 6 | 5
Sequence | 1 | 2 | 3 | 4 | 5 | 6 | 7
Diff     | 0 | 0 | 1 | 3 | 2 | 0 | 2
{:.mbtablestyle3}

If there is no reordering the Diff sequence would be \\(0,0,0,0,...,0\\)

All plots below are C4.large benchmarks with MySQL. The plots are histograms of Diff. 

[![Beanstalkd job ordering](/static/queue02/counts_run_1513507259_mysql_conn2_dm1_dtsx0_dretry1_batch10000_cl0_window0_verify1.json.svg)](/static/queue02/counts_run_1513507259_mysql_conn2_dm1_dtsx0_dretry1_batch10000_cl0_window0_verify1.json.svg)
Beanstalkd preserves the job ordering quite well. The biggest job offset was 6, with very few cases.


[![Pessimistic job ordering](/static/queue02/counts_run_1513587645_mysql_conn0_dm0_dtsx1_dretry1_batch10000_cl0_window0_verify1.json.svg)](/static/queue02/counts_run_1513587645_mysql_conn0_dm0_dtsx1_dretry1_batch10000_cl0_window0_verify1.json.svg)
Pessimistic with 1 delete retry (the original database queue implementation).
Here is apparent that some jobs run multiple times - high reordering. 
The job expiration time was set to 4 seconds. This queueing technique executed 11 800 jobs in total instead 
of 10 000 jobs.


[![Pessimistic job ordering](/static/queue02/counts_run_1513585597_mysql_conn0_dm0_dtsx0_dretry5_batch10000_cl0_window0_verify1.json.svg)](/static/queue02/counts_run_1513585597_mysql_conn0_dm0_dtsx0_dretry5_batch10000_cl0_window0_verify1.json.svg)
Pessimistic with 5 delete retries. The reordering is quite small, very similar to the beanstalkd.


[![Optimistic job ordering](/static/queue02/counts_run_1513508173_mysql_conn1_dm1_dtsx0_dretry1_batch10000_cl0_window0_verify1.json.svg)](/static/queue02/counts_run_1513508173_mysql_conn1_dm1_dtsx0_dretry1_batch10000_cl0_window0_verify1.json.svg)
Optimistic with window size 1. The reordering frequencies are also very similar to pessimistic 
locking with slightly more jobs being shifted.


[![Optimized Optimistic job ordering](/static/queue02/counts_run_1513508909_mysql_conn1_dm1_dtsx0_dretry1_batch10000_cl0_window1_verify1.json.svg)](/static/queue02/counts_run_1513508909_mysql_conn1_dm1_dtsx0_dretry1_batch10000_cl0_window1_verify1.json.svg)
Optimistic with window size N.
The job reordering is slightly higher but in all 10 test runs with 10 000 jobs it is bounded by 70. If you can tolerate such small
job reordering the optimized optimistic queueing mechanism seems like a good choice because of its benefits.

### Job duplicates

All tested job queueing methods runs the job exactly once but one method - pessimistic with 1 retry count, the original implementation.

The graph below shows the number of job executions vs. number of events (1 execution is left over as it is a normal condition). 

[![Pessimistic duplicities](/static/queue02/dupl_run_1513587645_mysql_conn0_dm0_dtsx1_dretry1_batch10000_cl0_window0_verify1.json.svg)](/static/queue02/dupl_run_1513587645_mysql_conn0_dm0_dtsx1_dretry1_batch10000_cl0_window0_verify1.json.svg)

11 800 jobs were executed in total instead of 10 000.

### Fetch before delete

I also analyzed the influence of the fetch before deleting on the system as mentioned above.

- The deadlock occurred anyway.
- There were still some jobs run multiple times.
- Removing the fetch before delete does not change the processing logic or number of times the job is being processed.
- Removing the fetch increases the throughput of the job queue.

Performance on MySQL:
- Forpsi fetch vs. no fetch is 181.3 vs 190.5 (95%) 
- C4.large fetch vs. no fetch is 313.4 vs. 452 (69%)

I conclude the fetch part is redundant and job delete can be one single delete query.

## Optimistic Database Queue package

I implemented the optimistic locking package as a Laravel package:

[https://github.com/ph4r05/laravel-queue-database-ph4](https://github.com/ph4r05/laravel-queue-database-ph4)

```bash
composer require ph4r05/laravel-queue-database-ph4
```

Benefits:

 - No need for explicit transactions. Single query auto-commit transactions are OK.
 - No DB level locking, thus no deadlocks. Works also with databases without deadlock detection (older MySQL).
 - Job executed exactly once (as opposed to original pessimistic default DB locking)
 - High throughput.
 - Works with MySQL, PostgreSQL and Sqlite.
 
Cons:
 - Job ordering can be slightly shifted with multiple workers (reordering 0-70 in 10 000 jobs)

## Conclusion

- Pessimistic approach as implemented now in Laravel can execute some jobs multiple times due to the deadlock and job delete fail.
- Pessimistic approach is usable with a little tweak on a number of attempts for delete query.
- Pessimistic approach is usable only with databases supporting immediate deadlock detection.
- If database does not support the deadlock detection either a) use different queue backend b) remove queue index c) use only one worker, d) use optimistic locking
- If the slight job reordering is not a problem use optimistic locking as it gives better performance and requires no DB level locks.


<!-- refs -->
[previous article]: https://ph4r05.deadcode.me/blog/2017/10/04/laravel-queueing-benchmark.html
[Laravel]: https://laravel.com/docs/5.5/
[#7046]: https://github.com/laravel/framework/issues/7046
[optimistic locking]: https://vladmihalcea.com/2014/09/14/a-beginners-guide-to-database-locking-and-the-lost-update-phenomena/

[job queueing]: https://laravel.com/docs/5.5/queues
[project]: https://github.com/ph4r05/laravel-queueing-benchmark
[c4.large]: https://aws.amazon.com/ec2/instance-types/
[burst credit]: https://aws.amazon.com/blogs/aws/new-burst-balance-metric-for-ec2s-general-purpose-ssd-gp2-volumes/
[Forpsi]: https://www.forpsi.com/virtual/

[random variable]: https://en.wikipedia.org/wiki/Random_variable
[expected value]: https://en.wikipedia.org/wiki/Expected_value



