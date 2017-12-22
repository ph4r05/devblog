---
layout: post
title:  "Laravel queueing benchmark"
date:   2017-10-04 22:10:39 +0200
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

Both servers are running the same software, latest Ubuntu 16.0.6.
MySQL version: 5.7.20, PHP 7.1 with opcache. The packages are in the default 
configuration. 

TODO: postgre version, php exact version, beanstalkd, redis versions.

## Method

- There are always 10 workers (processing the jobs) started before the test.
- Jobs being processed are empty - doing nothing. This way I meassure the job 
scheduling system overhead as the workers are competing over the jobs.
- 10 000 Jobs are pregenerated on the job queue. During the pregeneration the 
Laravel is in the *maintenance* mode - not processing the jobs.
- After the jobs are generated Laravel is switched to normal mode - jobs processing will start.
- Sample the size of the job queue each 0.5 seconds. Measure the time needed to process 10 000 jobs.
- The whole test is run 10 times.  

The overall test produces average jobs per second.  

## Results

The box plot below is a box chart visualizing the statistical distribution of the
jobs per second in 10 runs for each configuration.

[![Benchmark results](/static/queue01/benchmark_base.png)](/static/queue01/benchmark_base.png)



### Forpsi: 

| Method | Jobs per second |
|:-------|:----------------|
| Beanstalkd | 1109.82779118
| Redis | 872.469462269
| DB-mysql | 190.46838932
| DB-pgsql | 206.385861109
{:.mbtablestyle3}


Amazon C4 offers 2 thread cores and the processor is a bit faster than forpsi:

### C4.large

| Method | Jobs per second |
|:-------|:----------------|
| Beanstalkd | 2600.95996443
| Redis | 1905.52931478
| DB-mysql | 452.03984117
| DB-pgsql | 273.132717217
{:.mbtablestyle3}

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




