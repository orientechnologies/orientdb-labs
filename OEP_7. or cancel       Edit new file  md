##Cluster with singe page access in case of fetching record by rid

**Summary:**

Create cluster which allows to access only single page for the most of the times when we need to fetch page by RID.

**Goals:**

Refactor cluster data structure which allows to:

1. Fetch single record from a cluster by loading only one disk cache page for most of the cases.
2. Fetch batch of the records by RID using theoretical minimum of pages are needed to be loaded from disk cache.  
3. Reuse space which is used to store rid of delete record.


**Non-Goals:**

Create optimization on qurey layer wich allows to use new cluster to improve full scan queries.

**Success metrics:**

1. Imporved speed of YCSB tests in case only 20% of data is cached.
2. Improved speed of graphdb tests such as finding of shortest path between vertexis, traverse of vertexes, etc.

**Motivation:**

Most of the queries which are executed on database may be split on 2 types:

1. Make search by index to get rids of records which match quiry limits and then fetch those records from cluster.
2. Filter initial vertices in graph database then traverce vertices relatioships by some of critereis.

All those quiries load very big amount of pages during loading of records by rids.
Instead of sorting of rids by indexes of pages which contains them and then loading of all records that belong to the single record 
at once we load two pages for each rid.

**Description:**

We already implemented data structure which may be used to overcome all of issues listed above, that is data strucuture which is used
inside of OPaginatedCluster itself, but we never used all power of this datastructure. 

Inside of paginated cluster data structure each page cotains its own rid-record_physical_position_map which
embedded inside of each page. So when we create new record we add record to the page and inside of page we create link to real 
physical position of the record on disk. There will be three types of links:

1. Internal link, which will be used to refer to the record inside of the page.
2. External link, which will be use to refert to the record outside of page (more about this later).
3. Tomstone to indicate deleted record.

When update is peformed following actions are done:

1. If there is enough free space on current page  to put record inside of current page, 
then position inside of rid-postion_mab embedded inside of current page is updated.
2. If there is no enough free space to store record to current page new free page is allocated and external refrence is created
in emmbedded rid-posision map.

Entry of embedded rid-position map consit of two items internal pointer which consist of 2 bytes to make references inside of 65k page 
and delversion (more details lates) which consit of 8 bytes and is positive if link is not tobmstoen and negative in other case.
so common space which is consumed by refencre is 10 bytes. If we need to create external reference (which should be pretty rare case)
we create internal reference to the record which sereves as external reference to other page where real record is allocated.

In case of creation if there is a tombstone inside of rid_positions map we reuse this tobmstone as a rid for new record. This allows us 
to completely reuse space of removed records as opposite to now. But RIDs shoulb be unique so to keep them unique new field which is 
called delversion and consist of 8 bytes is introduced, when record is deleted we increment this field and make it negative, but 
when tombstone is reused we make this field positive.

It implies that we need to change format of RID by addition new fiedl which will be called delversion.
After this change format of rid will be following:

1. Cluster id , 4 bytes (becasue we planed to increase amount of bytes I needed for clsuter id any way).
2. Cluster positition will be split as following first 2 bytes will be reserved for index of position of embedded rid map 
inside of page , the last 14 bytes will be reserved for index of page inside of cluster.
3.Delversion will contain 8 bytes.

This format of cluster gives follwoing advantatges:

1. If we fetch record by rid we for most of the cases will fetch it by loading only single page from disk cache.
2. If we need to load set of rids (it includes cases when we traverse verteices by edges or when we load records 
from queiry which includes usage of indexes) we will sort rids by it's position and will be able to load severeral records at once 
by loading of only sinlge page.
3. We do not waste space of removed records anymore but reuse it and will resolve few requests made by users and customers.

*Record preallocation functionalty*

Right now to avoid the cases when to get rid of record in graph database in case of recursion of serialization we need to store empty
record we perform so called record preallocation, it means that we allocate slot in rid-position_map but do not write real data.
Then when record will be serialzed we store real record inside data file.

To keep the same functionality following is proposed, because the only unknown value is rid value we may calculate size of
which is need for record before we will know real values of rids,then we may book this space in data file and do not write real data
after we will know rids we will write real record content as usual, whcich effectivelly will be the same record perallocation procedure.

**Alternatives:**

Keep origianl cluster design.

**Risks and assumptions:**

1. Because of bigger space needed to store record inital load will be slower but it is not gurantied because to create new recrod we will 
need one page insteade of two which will give also good speed up in record creation.
2. There is risk of compatibility of versions , if some of users rely on cluster format. 

**Impact matrix**

- [*] Storage engine
- [*] SQL
- [ ] Protocols
- [ ] Indexes
- [ ] Console
- [ ] Java API
- [ ] Geospatial
- [ ] Lucene
- [ ] Security
- [ ] Hooks
- [ ] EE

