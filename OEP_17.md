**Summary:**

LSM-trie based index

**Goals:**

Implement new index engine which is based on LSM-trie data structure. Given datastructure suffer from much less 
write amplification than LSM-tree and is ten times faster on benchmarks than LSM-tree implementation.

LSM-trie capable to handle till 10 TB of data with small memory footprint.

LSM-trie does not support range queries so it is similar to current hash index implementation.

**Non-Goals:**

LSM-trie is not supposed to be faster than current implementation of hash index on small and average data but its scalabity
on big data should be much better.

**Success metrics:**

Much lower degree of speed degradation of index speed on big data.

**Motivation:**

To minimize random IO overhead during insertion of data into index it is proposed to gather all data changes into memory in single 
batch operation and 

**Description:**

It is proposed to implement promising index engine which according to measurements of operations throughput
are done on YCSB benchmark using the same database implementation (LevelDB), but different index engines
gives ten times speedup on read and write operations.

First of all, let see results of testing of write throughput on LevelDB ( + LSM trie) vs. the original LevelDB implementation
and vs. RocksDB implementation.

![LSM-trie vs LevelDB](http://i.imgur.com/AIqu2ep.png?1)

Bellow are results of testing of read performance of the same implementations on two configurations:

1. In case of presence of 64 GB RAM on server
![Read throughput 64GB](http://i.imgur.com/9wpxmkK.png)

2. In the case of presence of 4 GB RAM on a server.
![Read throughput 4GB](http://i.imgur.com/Uji6CLi.png)

Please note that trie based indexes by its nature are hash indexes and do not support range queries.

In our current implementation of ODB,  we already use the advantage of trie based indexes over tree based indexes.
Our hash indexes obviously faster than sb-tree indexes because they ensure
at most one IO operation during read and at most three IO operations during write.

In reality, LSM-trie implementation uses the same advantage of trie based indexes over tree based indexes.

LSM -trie ensures that there are at most 2 IO operations on each level during read of data and if we have enough of RAM (5 GB for 10 TB of data I think it is quite a small requirement) 
at most one IO operation for all levels except the last one, in the last one,  we will have at most two operations (in the worst case).

Obviously, we found this paper not on a tweet )). Some LSM tree optimizations came into my mind, and I tried to google whether someone already uses it, and I found description of this 
algorithm.

But let's start from basics.
There are two types of implementation of LSM tree. Classical (honestly speaking only a few use it) and one proposed by LevelDB and used with some modifications for RocksDB (Facebook took LevelDB and applied several modifications) and Cassandra/SсyllaDB (they both have an identical binary format, but a different approach in the implementation of read indexes).

So I will start with the most popular version of LSM implemented in LevelDB.

LevelDB LSM tree is based on concepts of MemTable and SSTable.

SSTable stands for Sorted String Table, by "string" they mean not set of characters but a set of bytes so any string of bytes may be stored in SSTable.

SSTable is implemented as a simple sorted list which consumes fixed amount of space, about 16-32 MB (in LevelDB SSTable size was 2MB but it was early 2010 and most of the users used HDD). This size is not chosen by random, that is a size of SSD clustered block which allows using all potential of internal parallelism and if we write in SSD by data chunks equal in size to clustered block size speed of random writes and sequential writes will be equal.

MemTable is in memory presentation of SSTalbe.
It worth to note that SSTables are immutable (I know Luca likes "immutable" data structures) and as result durable by definition without any WAL support (good feature BTW).

So how this mix of SSTable and MemTable works.
When operations on data have performed any update and insert causes an addition of data items in MemTable (deletes are handled by addition special markers tombstones they added to indicate that this item is removed). For the sake of durability all operations on MemTable are logged into log segment. Once MemTable is full it flushed on disk with force sync and log segment is dropped. That is it, very fast and durable. 
There are downsides of course:

1. There are a lot of duplication of data on disk and big space consumption.
2. Read operations are slow 

To speed up read operations from SSTables:
Each SSTable is indexed and an index is stored in an immutable file along with SSTable. Very often those indexes are cached into memory.
To avoid to read index to find an item on all SSTables bloom filters are used , those bloom filters are placed in memory.

To improve space usage and partially improve the speed of read operations all SSTables are merged into a single sorted list. Merge operation is very fast O(N) and does not require a lot of RAM. This process is called compaction.

To understand how compaction works let split storage by levels on Level 0 we place all SSTables are created as result of flush of MemTables. 
On Level 1 we put all SSTables are created as result of merge of SSTables from level 0.

Merge or compaction is performed as following:

1. Several SSTables from Level 0 with overlapped keys are taken.
2. SSTables from Level 1 which overlaps with SSTables of level above are taken
3. All those tables are merged using merge sort (very fast and only few memory is needed in extreme case you need to hold in memory only currently processed items of all SSTables).
4. Old SSTables are removed and new SSTables are created and placed on Level 1.

Please note that ranges of keys of SSTables on Level 1 are not overlapped. So to read data with any key we may first find SSTable with given key range, check bloom filter and if bloom filter indicates that item with given key is present in SSTable then read index of SSTable (it may be B-Tree for example, it is really fast to create B-tree using sorted list of items) and then read key-value pair from SSTable.

So far so good. But we forget one thing - write amplification. Because an amount of files on Level 1 is growing we need to read and write more and more files from Level 1 to merge the same amount of SSTables from Level 0. Eventually, write amplification becomes so big that it blocks any other IO operations on disk and system is stuck.

To minimize influence of write amplification LevelDB uses leveled approach during merge of SSTables (which is reflected in its name).

The maximum amount of files on Level 1 is limited to 10, on Level 2 the maximum amount of files is limited to 100 and so on.
But algorithm stays the same. 

To merge SSTables from Level n to Level n+1 we:
Take one SSTable from Level n (with exception of Level 0 , on this level we take all SSTables with overlapped key ranges)
Read all SSTables which overlaps with SSTable from Level n and write all merged SSTables in Level n
The key difference between this case and case described above is that during any compaction in the worst case only 10 SSTables from Level N + 1 affected.
So far so good. We limited write amplification by introduction of levels.

But in reality we only delayed problem (if you google "RocksDB write amplification" for example you may find many references).
Yes during single compaction we merge only 11 SSTables (1 from Level N and 10 from Level N + 1) but lets take a look what is write amplification if we want eventually move a single key entry to the lowest level.

Write amplification of compaction of single level is 10 + 1, if we went to move key to down to K level write amplification will be (10 + 1) * K. So for big storages for example with K = 5 write amplification will be 55. Which blocks all disk IO operations so we will have the same problem as above but later. (for example, MongoRocks has this problem )) ) .

So how to solve this problem once and forever.
The solution is quite "simple". If we will be able to involve only upper-level SSTables in compaction write amplification will be minimal. So how this should work ?

We can treat each SSTable on Level 0 as sub-level. So Level 0 consist of sub levels, in sub level keys of SSTables are not overlapped (obviously), but between sub-levels keys are overlapped.
We may put this concept to other Levels. In each Level, we have sub levels, and keys of SSTables inside of sub-levels are not overlapped but between level are overlapped.
So to merge SSTables from Level N to Level N + 1:

We read one SSTable from each sub levels of Level N
Merge them together.
Put not overlapped SSTables in sub level of Leve N + 1
In such case to write single key from Level N to Level N + 1 we need to read one page and write one page (because we write the same amount of pages from as we read as opposite to the previous case when we read one page from Level N but write eleven pages in Level N + 1).

But there is one problem, all compactions are target to the same sub-level of Level N+1 should not have overlapped key ranges (otherwise we will fall to algorrithm is used by the first approach) which is highly unlikely.

We also can not create sub-level for each single merge because in such case we will still have enormous space usage and slow reads and big memory consumption (because many sub levels mean a lot of data duplication)

So how to solve given the problem ?

First of all, lets create a hash code for each key which is going to be added into an index using SHA - 1 function (that will guarantee uniformity of distribution of hash codes ). 
Then will use a prefix of a hash code to determine sub level of level.
Let's look at picture to get what is going on:

![Identification of level](http://i.imgur.com/OdadCVJ.png)

We create very small trie, which consists of table containers. 
All tables which are placed in the same position inside of table containers (for example we have 2 SSTables on position 2 of a first and second container) are constitute sub level.
All tables which are contained inside table containers of the same depth constitute a level in LevelDB terminology.

So what we do during compaction:
Take all nodes from single table container (it is the same like we take one SSTable for each sub-level in the previous algorithm).
Merge them into non-overlapped SSTables. There is an interesting difference in compaction of SSTables in LevelDB and LSM-trie. Items of SSTables are merged together and then placed in different containers according to hash codes. 

The result of compaction process is described in picture bellow:

![Compaction](http://i.imgur.com/DOv6y3n.png)

As a result, all key spaces (regarding LSM-trie) of SSTables will not be overlapped.

What is interesting despite the fact that LSM-trie minimizes write amplification, it also allows removing of SSTable indexes.

To remove SSTable indexes, SSTable is converted into HTable. HTable is SSTable, but instead of sorting them by items we put keys into buckets by their hash code.
Of course in such case, some buckets will be overflown, and some of them will do not have enough elements to be full.  

In this case, we do following:
We sort keys which are put into this bucket by their hash code.
Fill bucket to its capacity
In bucket reserve 8 bytes for the following information (source bucket id, maximum value of hash code after which items will be moved to another bucket, bucket destination id)
Move all items which hash code above recorded in step 3 to destination bucket.
Using such approach we solve the issue with overflown buckets and remove the need to have separate indexes for a bucket.
To minimize amount of operations we are going to cache bucket migration information into memory, but because size of bucket is 64 or 16 K (in new version) memory overhead of such data caching
will be minimal.

How do we read items in case of request?

First of all, we detect tables container by key hash code, then in this tables container, we read  Bloom filters to check which HTable's bucket contains the item and read data from a concrete bucket of HTable with single IO operation without any additional indexes. Each bloom filter requires about 16 bit per element or 2 GB for one billion items. This is not big memory consumption, but some of our users work with big disks and a small amount of RAM. To resolve this issue too )) we cluster all bloom filters for the same buckets of HTables in a single page (bloom filters are created for each bucket which equals to page in our implementation). 

As result, if we do not have enough memory to keep bloom filters in memory we read bloom filters with one disk IO and then check do we need to load specific bucket or not and then load given bucket.


**Alternatives:**


**Risks and assumptions:**


**Impact matrix**

- [ ] Storage engine
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
