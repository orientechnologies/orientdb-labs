**Summary:**

Reallocate hot records to the same pages with the help of Stream-Summary algorithm

**Goals:**

Minimize read and write amplification to keep all hot records together inside of disk pages

**Non-Goals:**

None

**Success metrics:**

Improved speed of benchmarks which are based on skewed data distributions similar to Zephian distribution. 

**Motivation:**

During profiling of YCSB tests we found that:

1. Index pages are cached.
2. Cluster position map pages are cached.
3. But data cluster pages are not cached.

Data cluster pages are not cached because cold and hot pages are spread randomly between pages as result disk cache efficiency hurts.

We also may look at this from another point of view.
When hot records are mixed with cold records on the same page, we write and read much more data for
the same time interval but a speed of the disk is limited.

**Description:**

We may use the Stream-Summary algorithm to calculate top s clusters are used in storage and for those clusters keep top k records accessed by users during read/update operations.

Stream-Summary has an interesting property. This algorithm not only calculates top k items but also provides a level of error in such calculations.

Once we calculate top s clusters with applicable error, we start to calculate top records accessed in those clusters. Items which associated with each record will contain the following information:

1. Record position.
2. Record page index.

Also, we are going to support information about a set of pages which contain top-k records. 

Beside of pages indexes, the following information is supposed to be held in each item of such set:

1.  Amount of top records in page
2.  Amount of cold records in page.

Data structures mentioned above will be updated during read/create/update/delete operations.

We will update following information:

1. Top k records.
2. If a record is added to top k list for the first time, then we check if page which contains this record is absent in set of pages.
If that is true, we add a new entry to the set of pages and set amount of hot records to one and amount of cold records to amount of records in page-1.
3. If a record is added to top k list for the first time, but the page which contains records already is presented inside of page set we merely update quantity of hot records.
4. When we create/delete records, we update information about an amount of cold and hot records accordingly.
5. When we record is removed from top k list, we update information about hot and cold pages using an approach similar to described above.

Once we identify top k records with applicable error, we may relocate them in such manner that all of them 
will be put together in disk pages.

There are following approaches which are proposed to use for records reallocation.

When record is updated/read:
1. It is identified whether it belongs to top K records.
2. If it belongs to top K records, we check whether page which contains records hit cold pages limits 
(I propose to set this limit to 20% by default).
3. If page hit a limit, we schedule relocation procedure which will be run in background. 
4. In this procedure, we remove all hot records from detected page to any other page from the set of hot pages in most of the cases increasing the percent of hot records on this page to 100%.

Because the distribution of hot records may be changed dynamically, there is a risk that we only introduce write overhead in such case.
To mitigate such risks following limits will be applied:

1. Record relocation procedure will be applied every 30 minutes.
2. All entries with metadata about pages which hold top k records will have a timestamp which indicates when record reallocation procedure was scheduled. And only pages reallocation of which were scheduled hour ago will be processed. 

In such case, all loads with stable or slow changing distribution of data will get a noticeable increase in latency and throughput numbers from one side. 
And from another side, all loads which change distribution of their data very quickly will not suffer from often record reallocations.

About real values of top K records and top S cluster. 
I propose to track top 100 clusters and top 10 000 records.

What it gives to us. Obviously, all benchmarks (they use a static law of distribution) will benefit from such strategy.
But despite marketing reasons I suppose there are many applications when a law of data distribution is quite stable which also may benefit from proposed approach. 

**Alternatives:**

None

**Risks and assumptions:**

1. Reallocation procedure should fulfill ACID guarantees.
2. There will be only a few cases with a stable or quite stable law of data distribution so to prevent memory overhead we need to gather statistic how many times records were reallocated and how many of pages (in percents ) hit cold records limit. If we see that there are many pages with cold records, limit we should switch off this feature on storage configuration level.

**Impact matrix**

- [x] Storage engine
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

