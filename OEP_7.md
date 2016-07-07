##Cluster with single page access in case of fetching record by rid

**Summary:**

Create cluster which allows accessing only single page for the most of the times when we need to fetch page by RID.

**Goals:**

Refactor cluster data structure which allows to:

1. Fetch single record from a cluster by loading only one disk cache page for the most of the cases.
2. A fetch batch of the records by RID using a theoretical minimum of pages are needed to be loaded from the disk cache.  
3. Reuse space which is used to store rid of delete record.


**Non-Goals:**

Create optimization on query layer wich allows using a new cluster to improve full scan queries.

**Success metrics:**

1. The improved speed of YCSB tests in the case when only 20% of data is cached.
2. Improved speed of "graphdb" tests such as finding the shortest path between vertexes, a traverse of vertexes, etc.

**Motivation:**

Most of the queries which are executed on database may be split into two types:

1. Make search by index in order to fetch rids of records which match query limits and then fetch those records from a cluster.
2. Filter initial vertices in graph database then traverse vertices relationships by some of the critereis.

All those queries load a massive amount of pages during loading of records by rids.
Instead of sorting of rids by indexes of pages which contains them and then loading of all records that belong to the single page  
at once we load two pages for each rid.

**Description:**

We already implemented data structure which may be used to overcome all of the issues listed above, that is the data structure which is used
inside of OPaginatedCluster itself, but we never used all power of this data structure. 

Inside of paginated cluster data structure, each page contains its rid-record_physical_position_map which
embedded inside of it. So when we create a new record we add record to the page and inside of the page we create a link to a real 
physical position of the record on disk. There will be three types of links:

1. An internal link, which will be used to refer to the record inside of the page.
2. External link, which will be used to refer to the record outside of page (more about this later).
3. Tombstone to indicate deleted record.

When update is performed following actions are done:

1. If there is enough free space on a current page to put record inside of the current page, 
then position inside of rid-postion_mab embedded inside of current page is updated.
2. If there is no enough free space to store the record to current page new free page is allocated and external reference is created
in embedded rid-position map.

Entry of embedded rid-position map consist of two items internal pointer which consist of 2 bytes to make references inside of 65k page and delversion (more details later) which consist of 8 bytes and is positive if a link is not a tombstone and negative in other cases.
So common space which is consumed by reference is 10 bytes. If we need to create external reference (which should be pretty rare case)
we create an internal reference to the record which serves as an external reference to another page where the real record is allocated.

In a case of creation of records, if there is a tombstone inside of rid_positions map we reuse this tombstone as rid for a new record. Such approach allows us to reuse space of removed records completely as opposite to now. But RIDs should be unique so to keep them unique new field which is called delversion and consist of 8 bytes is introduced when a record is deleted we increment this field and then make it negative, but when the tombstone is reused we make this field positive again.

It implies that we need to change the format of RID by addition new field which will be called delversion.
After this change, format of rid will be following:

1. Cluster id , 4 bytes (because we planned to increase an amount of bytes needed for cluster id anyway).
2. Cluster position will be split as following: first 2 bytes will be reserved for an index of a position of embedded rid map 
inside of the page , the last 6 bytes will be reserved for an index of a page inside of the cluster.
3. Delversion will contain 8 bytes.

This format of cluster gives following advantages:

1. If we fetch record by rid we for most of the cases will fetch it by loading an only single page from the disk cache.
2. If we need to load a set of rids (it includes cases when we traverse vertices by edges or when we load records from a query which includes usage of indexes), we will sort rids by its position and will be able to load several records at once by loading of the only single page.
3. We do not waste space of removed records anymore but reuse it and will resolve few requests made by users and customers.

*Record preallocation functionality*

Right now to avoid the cases when to get rid of record in graph database in case of recursion of serialization we need to store empty
a record we perform so-called record preallocation, it means that we allocate a slot in rid-position_map but do not write real data.
Then when the record will be serialized we store real record inside data file.

To keep the same functionality following is proposed because the only unknown value is rid value we may calculate size 
which is a need for a record before we will know real values of rids, then we may book this space in a data file and do not write real data.
But when we will know rids we will write real record content, as usual, which effectively will be the same record preallocation procedure.

**Alternatives:**

Keep original cluster design.

**Risks and assumptions:**

1. Because of bigger space needed to store record, initial load will be slower but it is not guaranteed because to create a new record we will 
need one page instead of two which will also give a good speed up in record creation.
2. There is a risk of compatibility of versions if some of the users rely on cluster format. 

**Impact matrix**

- [x] Storage engine
- [x] SQL
- [ ] Protocols
- [ ] Indexes
- [ ] Console
- [ ] Java API
- [ ] Geospatial
- [ ] Lucene
- [ ] Security
- [ ] Hooks
- [ ] EE

