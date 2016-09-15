**Summary:**

Disk based transactions with record/index key granularity.

**Goals:**

Create tranasaction processing engine which supports following features:

1. Log records are created during transaction processing should be logged 
on disk not into memory to reduce memory consumption and avoid risk of OOM and data corruption.
2. On transaction level all locks should have record/key level granularity not component granularity.
3. On level of single component operation (for example during insertion of (key, rid) pair into sb-tree) locks should have 
page granularity (in example above some pages of sb-tree will be locked till insertion of (key,rid) pair in progress), not component 
granularity.

**Non-Goals:**

Improve speed of component operations in single-thread mode.

**Success metrics:**

Improved scalability of massive write operations on performance benchmakrs both in transactional and non transactional modes.

**Motivation:**

1. Almost all modern databases (MySQL, PostgreSQL, etc) work with concurrent operations on record/key level granularity of locks.
Also our investigation of results of performance benchmarks shows that component level locks are not enough to achieve good scalability of writes.
2. Keeping of changes of all pages only inside of RAM causes noticable memory consumption and may lead to OOM and data corruption.
3. During normal transaction processing we do not apply changes directly but collect all changes inside of atomic operation
(so called deferred updates) and if we need to read changed pages we apply those changes back to pages. Such approach increase system fragility.  Any error indisde of WAL or atomic operation will lead to data corruption. 
4. Apllying of changes to the pages at every read operation decreases speed of read of affected pages at ten times as result even read  of single page which is changed inside of transaction will affect speed of whole transaction.

**Description:**

**High level design**

All changes are performed on pages are logged inside of WAL. Each log record consist of two parts: first part contains information 
which is needed to restore page content after crash (redo part), second part contains information which is needed to rollback page 
changes (undo part). Redo part is stored in form of binary diff between original and changed binary presentations of page.

Proposed transaction protocoll allows to use fine granularity locks during transaction processing by 
using very interesting feature of all our data structures. When we store/delete entry inside of any data structure
each data structure has page which satisfies following requirement - if all changes are made on data structure 
but this single page (domain page) is not updated then data structure still is in valid state but 
data are treated as absent inside of given data structure. 

Each data structure changes may be split on two parts "structure modification changes" and "logical changes". 

Lets consider for example tree based index when we insert (key, rid) pair inside of tree we make such changes as 
split of parent nodes and split of leaf page itself. All those changes are "structure modification changes". 
As final step we should add (key, rid) pair inside of the leaf page (which plays role of domain page).
Till this entry is add to the leaf page (key, rid) pair is still  absent in database but tree structure is completely valid.
Once we put (key, rid) pair inside leaf page (execute "logical" insertion of data inside tree) it will be treated as stored inside 
of database.

In every index which we use leaf page plays role of domain page. For clusters such domain pages are pages of
cluster transaction map.

To restore data after the system crash we replay all transaction changes from the log till the point of crash and then 
revert all uncompleted transactions.

When transaction rollack is performed we read WAL records from the last record loged into the WAL till the first record
and rollback changes are logged into those records using infromation is stored in undo part of WAL record.

During rollback it is possible to perform two types of rollbacks "page level" rollback when changes performed on the page 
reverted from this page during rollback and "logical" rollback when action which is opposite to the executed logical  operation (insertion of entry inside of index for example) will be performed. Logical rollbacks are performed only to revert logical changes 
and page level rollbacks are performed on structure modification changes. 

Every logical rollback again consist of two phases on first phase we rollback changes are perfromed on domain page.
Then we execute structure modification changes, if needed, for example we may merge two almost empty 
pages in sb tree into single one.

There are several cases of processing of rollbacks and data restore operations.

1. If rollback happens in the middle of structure modification changes 
we will rollback all changes applied on the pages on binary level and at the end of rollback 
data structure will be look like it was at the begging of transaction.
2. If we roolback transaction after logical change is done, we will perform action on data structure which compensates executed action  instead of roling back of structure modifciation changes, so at the end of rollback data structure will *logically* look like 
it was at the begging of the transaction.

Taken all above into account it becomes obvious that to implement "isolation" feature of transaction it is enough to acquire only 
key/record level locks during transacion and page level locks inside of structure modification changes.

So what about restore of system after the crash. How to keep data in correct state even if system crashed during transactinol rollback.
To make it possible we introduce new type of log record which is called compensation log record (CLR). 
This record keeps pointer to the log record which should be rolled back next after the log record which
was rolled back by operation wich generates current CLR. 
Every time we peform rollback of page changes are logged inside of log record we put related CLR record.
Such CLR record contains only redo information, this information is binary diff of content which is changed inside of page druing 
applying of rollback operation. 

1. When we peform rollback of structure modification changes CLR redo part equals to log record undo part. 
2. When we perform logical rollback CLR record contains only binary diff applied to the domain page. 
3. That is most interesting part, when we complete sturucture modificaiton changes but before applying of changes to the domain page 
we also add CLR record with empty redo part which points to the last record logged before 
the first structure modification changes record.
The presence of such CLR record forces the rollback procedure to skip structure modification
changes during rollback of transaction once they are completed and changes of domain page are logged.

There are several variants of restoring of database content after system crash:

1. System is crashed inside of execution of structure modification changes. 
There are no any CLRs so we merely restore data structure till the point when system was crashed. 
And then rollback all page changes and restore initial state of data structure before transaction.
2. System is crashed during rollback of structure modification changes. 
In such case we restore all structure modification changes, then by applying of CLR records we will repeat partiall rollback and by 
examinating of content of last CLR record we will find which pages still should be rolledback 
and will restore initial state of data structure before transaction.
3. Structure modification changes are completed and CLR record is put at the end of this changes. 
In such case we restore all structure modification changes and 
by processing of CLR record will rollback all changes which exists *after* structure modfication changes but will not rollback structure 
modification changes itself. There is good reason why we can not rollback structure modification changes. 
Page locks are released once we apply structure modification changes and those pages may be changed by other succesesfully 
commited transactions. 
4. Structure modification changes and changes on domain page are completed in such case we will rollback only changes of domain page
and skip structure modification changes because of presence of CLR record. We do not perform logical remove of data but only revert content of domain page because of concurrent access to the data strucuture it may be in invalid state till complete restore of state 
of all tranactions will be completed.
At the end of data restore procedure data structure will be logically at the same state as it was before transaction is started.

As you can see above if we restore system after crash some space may be lost at the end of data restore procedure but amount of 
such space should be so minor that may be not taken into account.

What is interesting that even if we implement only new transaction processing protocol but will not change 
lock model of already existing compenents we still will increase system speed and scalability.
Lets suppose we have two ongoing transacitons on the system and component lock model is used.
First transaction changes components A and B and second transaction changes components A and C. 
In current implementation those transactions will be serialized but in new implementation transactions will go in parallel 
once one of them will complete changes on component A. 

So at first stage we may implement protocol itself and then change lock model of components from compoenent level to page level in next releases (such models will be described soon in separate OEPs) .

Lets consider two left operations on data structures which includes updates and deletes.

During delete order of operations are following:

1. Delete entry from domain page.
2. Put request on the сlean up queue to to clean up consumed space after tx commit (is needed only for cluster). 

So during transaction rollback or restore we revert only domain page content and record delete or index entry delete and as result data are automatically restored.

Consider in details second item of delete alghorithm - cleanup queue. 
When we delete record in cluster it is reasonable to claim space consumed by auxilary data back to storage manager.

To make that possible we create cleanup queue which is filled by operations peformed during transaction (delete/update) which contains ,in case of cluster, position to the record to be cleared 
(this position is always the same even during internal page defragmentation and will not be changed after the record delete).
If transaction is rolled back then clean up queue is cleared and no changes will be performed. If transaction is commited then 
changes are applied in background thread. This clean up queue consumes very few memory, each queue entry consist of only two field 
clusterId and position of record inside of data file, but it also may become bounded for example we may allow to  add no more then
1 000 000 of such entries at any time for single transaction and 10 000 000 entries in totall may be contained into the queue, the same bound may be applied to the amount of locks which may be acquired by the transacion to avoid any risk of OOM.

Clean up thread which process this queue will pull each entry in process it in separate transaction.
So even if system will be crashed then transaciton will be rolled back and very small amount of space will be lost.

The logic of update of data is similar because update is mix of deletion of previous data and addition of new data.

1. Peform structure modification operations to add new data if needed (not needed for indexes only value inside of leaf page is updated).
2. Update domain page.
3. Put request on the сlean up queue to to clean up consumed space after tx commit.  

So during rollback:
1. If rollback happens in the middle of execution of structure modification operations then we revert all changes on binary level.
2. If rollback happens after domain page update we remove new record data from cluster and revert record pointer to old record content.

Restore logic is same as logic during rollback with only exception that we do not remove already added record but revert domain page content. We do not perform logical remove of data but only revert content of domain page because of concurrent access to the data strucuture it may be in invalid state till complete restore of state of all tranactions will be completed.

Once again it is worth to note that all those procedures will work even if we will use component locks not page locks, providing much better level of scalability without of changing of component implementation.

**Low level design**


**Alternatives:**


**Risks and assumptions:**


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
