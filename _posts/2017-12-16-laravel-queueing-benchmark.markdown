---
layout: post
title:  "Laravel queueing benchmark"
date:   2017-12-16 22:10:39 +0200
categories: blog
excerpt_separator: <!-- more -->

---

Practical benchmark of classic Laravel queueing mechanisms: 
redis, beanstalkd, database (mysql, postgre) 

<!-- more -->

## Intro

[Laravel] provides several different mechanism for [job queueing]. 
I will cover: Redis, Beanstalkd, Database (MySQL, PostgreSQL)

For the benchmarking I created a new Laravel 5.5 [project]. 
The benchmarks were performed on 2 VPS:

#### Amazon EC2 [c4.large]
- Intel(R) Xeon(R) CPU E5-2666 v3 @ 2.90GHz
- 2x VCPU, 4 GB RAM
- EBS, SSD storage (with [burst credit] > 0)
  
#### [Forpsi] smart VPS, low budget server
- Intel(R) Xeon(R) CPU E5-2650L v4 @ 1.70GHz
- 1x VCPU, 1 GB RAM
- SSD storage, without IO throttling

Both servers are running the same software, latest Ubuntu 16.04.3 LTS.
MySQL version: 5.7.20, PHP 7.1.12-3 with opcache, 
PostgreSQL 9.5.10, Redis 3.0.6, Beanstalkd 1.10.  

The packages are in the default configuration. 

## Method

- There are always 10 workers (processing the jobs) started before the test.
- Jobs being processed are empty - doing nothing. This way I measure the job 
scheduling system overhead as the workers are competing over the jobs.
- 10 000 Jobs are pre-generated on the job queue. During the pre-generation the 
Laravel is in the *maintenance* mode - not processing the jobs.
- After the jobs are generated Laravel is switched to normal mode - jobs processing will start.
- Sample the size of the job queue each 0.5 seconds. Measure the time needed to process 10 000 jobs.
- The whole test is run 10 times.  

The overall test produces average jobs per second.  

Note the Database queue driver has been slightly modified to overcome 
the current limitation of running one job multiple times. The original
implementation fails to delete some of the finished job from the database under high load.
More on this problem in the next blog post.

## Results

The box plot below is a box chart visualizing the statistical distribution of the
jobs per second in 10 runs for each configuration.   

[![Benchmark results](/static/queue01/benchmark_base.png)](/static/queue01/benchmark_base.png)



### Forpsi: 

| Method | Jobs per second |
|:-------|:----------------|
| Beanstalkd | 1109.83
| Redis | 872.47
| DB-mysql | 190.47
| DB-pgsql | 206.39
{:.mbtablestyle3}


Amazon C4 offers 2 thread cores and the processor is a bit faster than forpsi:

### C4.large

| Method | Jobs per second |
|:-------|:----------------|
| Beanstalkd | 2600.96
| Redis | 1905.53
| DB-mysql | 452.04
| DB-pgsql | 273.13
{:.mbtablestyle3}

<br/>

## Burst credit

When testing on Amazon EC2 its important to realize the EBS storage drives
are subject to IO throttling. 

Each drive has so called [burst credit]. Intensive IO operations decreases
the credit over the time. Conversely during low IO usage the credit restores slowly.

When credit reaches 0 the IO throughtput decreases significantly (10% ish). 
I performed the benchmarks only with non-null burst credits to measure the best
possible performance.

 

<!-- refs -->
[Laravel]: https://laravel.com/docs/5.5/
[job queueing]: https://laravel.com/docs/5.5/queues
[project]: https://github.com/ph4r05/laravel-queueing-benchmark
[c4.large]: https://aws.amazon.com/ec2/instance-types/
[burst credit]: https://aws.amazon.com/blogs/aws/new-burst-balance-metric-for-ec2s-general-purpose-ssd-gp2-volumes/
[Forpsi]: https://www.forpsi.com/virtual/





