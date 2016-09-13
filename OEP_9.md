**Summary:**

Use 16k pages  for next versions of OrientDB by default

**Goals:**

Improve speed of read/writes operation by decreasing of default size of database page

**Non-Goals:**


**Success metrics:**

Improve numbers of performance benchmarks

**Motivation:**

If we decrease default size of cache page we decrease amplification of read and write operations.

**Description:**

According to our investigation during the database load:

1. Most of non-leaf index pages are placed into the disk cache and leaf pages are accessed using random IO pattern.
2. Difference between random IO and sequantiall IO much smaller on modern SSDs and in case of big pages like 64KB can not be neglected.
3. All writes which we perform on database are performed using sequential operations if possible.

All of this suggest do decrease page size from 64KB which is huge to 16 KB, which should increase speed of random IO operations 
and should not affect or improve speed of write operations.

Let's check all those speculations by real YCSB benchmark. If we run following YCSB loads (mixed load (50% writes, 50% reads), 
mostly read load (5% writes, 95% reads), read only load (100% of reads)) in case page sizes are 16 KB and 64 KB we get following 
results:

**Load of data**

**64k pages**

```
[INSERT], AverageLatency(us), 2659.181693165
[INSERT], MinLatency(us), 36.0
[INSERT], 95thPercentileLatency(us), 1935.0
[INSERT], 99thPercentileLatency(us), 3647.0
```

Total time: 18 hrs 32 mins 59.659 secs = 66 721 secs

**16k pages** 

```
[INSERT], AverageLatency(us), 1649.60618966
[INSERT], MinLatency(us), 35.0
[INSERT], 95thPercentileLatency(us), 1770.0
[INSERT], 99thPercentileLatency(us), 2481.0
```

Total time: 11 hrs 31 mins 58.591 secs = 41 461 secs

*Performance gain - 16k pages 1.61 times faster on load*

**Mixed load 50% of writes, 50% of reads**

**64k pages**

```
[READ], AverageLatency(us), 2289.592112263472
[READ], MinLatency(us), 17.0
[READ], 95thPercentileLatency(us), 1616.0
[READ], 99thPercentileLatency(us), 18895.0
```

Thoroughput = 3494 op/secs

```
[UPDATE], AverageLatency(us), 7351.444064681804
[UPDATE], MinLatency(us), 36.0
[UPDATE], 95thPercentileLatency(us), 3161.0
[UPDATE], 99thPercentileLatency(us), 72191.0
```

Thoroughput = 1 088 op/secs

**16k pages**

```
[READ], AverageLatency(us), 1199.684215890254
[READ], MinLatency(us), 17.0
[READ], 95thPercentileLatency(us), 787.0
[READ], 99thPercentileLatency(us), 1353.0
```

Thoroughput = 6668 op/secs

```
[UPDATE], AverageLatency(us), 5039.77029837364
[UPDATE], MinLatency(us), 32.0
[UPDATE], 95thPercentileLatency(us), 1308.0
[UPDATE], 99thPercentileLatency(us), 2781.0
```

Thoroughput = 1 587 op/secs

*Performance gain - read throughput is improved at 1.9 times, read latency is improved at 14 times !, 
write throughput is improved at 1.45 times, write latency is improved at 26 times !*


**Read mostly load 5% writes, 95% reads**

**64k pages**

```
[UPDATE], AverageLatency(us), 2338.448307545592
[UPDATE], MinLatency(us), 41.0
[UPDATE], 95thPercentileLatency(us), 2319.0
[UPDATE], 99thPercentileLatency(us), 7059.0
```

Thoroughput = 3 421 op/secs 

```
[READ], AverageLatency(us), 743.0695396303125
[READ], MinLatency(us), 16.0
[READ], 95thPercentileLatency(us), 1407.0
[READ], 99thPercentileLatency(us), 3575.0
```

Thoroughput = 10 767 op/secs

**16k pages**

```
[UPDATE], AverageLatency(us), 2492.1606831810336
[UPDATE], MinLatency(us), 44.0
[UPDATE], 95thPercentileLatency(us), 1049.0
[UPDATE], 99thPercentileLatency(us), 2615.0
```

Thoroughput = 3 210 op/secs

```
[READ], AverageLatency(us), 559.9084873968899
[READ], MinLatency(us), 18.0
[READ], 95thPercentileLatency(us), 668.0
[READ], 99thPercentileLatency(us), 1096.0
```

Thoroughput = 14 288 op/secs

*Performance gain: write throughput is almost the same, write latency is improved at 2.6 times.
Read throughput 1.32 times better, read latency is improved at 3.26 times.*

**Read only workload** 

**64k pages**

```
[READ], AverageLatency(us), 483.73006588
[READ], MinLatency(us), 16.0
[READ], 95thPercentileLatency(us), 1252.0
[READ], 99thPercentileLatency(us), 1825.0
```

Thoroughput = 16 538 op/secs

16k pages 

```
[READ], AverageLatency(us), 263.79829212
[READ], MinLatency(us), 16.0
[READ], 95thPercentileLatency(us), 583.0
[READ], 99thPercentileLatency(us), 823.0
```

Thoroughput = 30 327 op/secs

*Performance gain - read throughput 1.8 times better, read latency is improved at 2.2 times*


**Alternatives:**


**Risks and assumptions:**

Maximum key size will be decreased from 10240 to about 5120

**Impact matrix**

- [X] Storage engine
- [ ] SQL
- [ ] Protocols
- [ ] Indexes
- [ ] Console
- [ ] Java API
- [ ] Geospatial
- [ ] Lucene
- [ ] Security
- [ ] Hooks
- [ ] EE
