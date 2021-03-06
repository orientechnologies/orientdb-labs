#Summary:

Disk-based transactions with record/index key granularity.

##Goals:

Create transaction processing engine which supports following features:

1. Log records are created during transaction processing should be logged on disk not into memory to reduce memory consumption and avoid a risk of OOM and data corruption.
2. On transaction level, all locks should have record/key level granularity, not component granularity.
3. On level of single component operation (for example during insertion of (key, rid) pair into sb-tree) locks should have page granularity (in the example above some pages of sb-tree will be locked till insertion of (key, rid) pair in progress), not component granularity.

##Non-Goals

Improve speed of component operations in single-thread mode.

##Success metrics:

Improved scalability of massive write operations on performance benchmarks both in transactional and nontransactional modes.

##Motivation:

1. Almost all modern databases (MySQL, PostgreSQL, etc.) work with concurrent operations on record/key level granularity of locks.
Also, our investigation of results of performance benchmarks shows that component level locks are not enough to achieve good scalability of writes.
2. Keeping of changes of all pages only inside of RAM causes conspicuous memory consumption and may lead to OOM and data corruption.
3. During normal transaction processing we do not apply changes directly but collect all changes inside of atomic operation
(so-called deferred updates), and if we need to read changed pages, we apply those changes back to pages. Such approach increase system fragility. Any error inside of WAL or atomic operation will lead to data corruption. 
4. Applying of changes to the pages at every read operation decreases the speed of read of affected pages at ten times as result even read a single page which is changed inside of transaction will affect the speed of the whole transaction.

##Description:

###High level design

All changes are performed on pages are logged inside of WAL. Each log record consist of two parts: first part contains information which is needed to restore page content after the crash (redo part), the second part contains information which is needed to rollback page changes (undo part). Redo part is stored in the form of binary diff between original and changed binary presentations of a page.

Proposed transaction protocol allows using fine granularity locks during transaction processing by using a fascinating feature of all our data structures.  When we store/delete entry inside of any data structure
it has a page which satisfies the following requirement - if all changes are made on data structure but this single page (domain page) is not updated then data structure still is in the valid state, but 
data are treated as absent inside of given data structure. 

Each data structure changes may be split on two parts "structure modification changes" and "logical changes." 

Let's consider for example tree based index. When we insert (key, rid) pair inside of a tree, we make such changes as a split of parent nodes and a split of leaf page itself. All those changes are "structure modification changes." 
As a final step, we should add (key, rid) pair inside of the leaf page (which plays a role of domain page).
Till this entry is add to the leaf page (key, rid) pair is still absent in database, but tree structure is completely valid.
Once we put (key, rid) pair inside leaf page (execute "logical" insertion of data inside tree) it will be treated as stored inside 
of database.

In every index which we use leaf, page plays a role of domain page. For clusters, such domain pages are pages of
cluster transaction map.

To restore data after the system crash, we replay all transaction changes from the log till the point of crash and then 
revert all uncompleted transactions.

When transaction rollback is performed, we read WAL records from the last record logged into the WAL till the first record
and rollback changes are logged into those records using information is stored in undo part of WAL record.

During rollback it is possible to perform two types of rollbacks "page level" rollback when changes performed on the page reverted from this page during rollback and "logical" rollback when the action which is opposite to the executed logical operation will be performed. Logical rollbacks are performed only to revert logical changes and page level rollbacks are performed on structure modification changes. 

Every logical rollback changes again logged into the WAL. It will allow restoring data in case of system crash.

There are several cases of processing of rollbacks and data restore operations.

1. If rollback happens in the middle of structure modification changes 
we will rollback all changes applied to the pages on binary level and at the end of rollback 
the data structure will look like it was at the begging of a transaction.
2. If we rollback transaction after logical change is done, we will perform action on data structure which compensates executed action  instead of rolling back of structure modification changes, so at the end of rollback data structure will *logically* look like 
it was at the begging of the transaction.

Taken all above into account it becomes obvious that to implement "isolation" feature of transaction it is enough to keep only 
key/record level locks during a transaction and keep only page level locks inside of structure modification changes.

So what about restore of a system after the crash. How to keep data in a correct state even if the system crashed during transactional rollback.
To make it possible, we introduce a new type of log record which is called compensation log record (CLR). 
This record keeps a pointer to the log record which should be rolled back next after the log record which
was rolled back by operation wich generates current CLR. 
Every time we perform a rollback of page changes are logged inside of log record we put related CLR record.

Such CLR record contains only redo information. 
Redo information may include following data:

1. During rollback of a structure modification changes CLR redo part contains log record undo part. 
2. If we perform a logical rollback, then redo part is empty. 
3. If we complete structure modification changes before applying of changes to the domain page we also add CLR record with empty redo part which points to the last record logged before the first structure modification changes to a record.
The presence of such CLR record forces the rollback procedure to skip structure modification
changes during rollback of the transaction once they are completed and changes of domain page are logged.

There are several variants of restoring of database content after system crash:

1. A system is crashed inside of execution of structure modification changes. 
There are no any CLRs, so we merely restore data structure till the point when a system was crashed. 
And then rollback all page changes and restore initial state of the data structure before a transaction.
2. A system is crashed during rollback of structure modification changes. 
In such case we restore all structure modification changes, then by applying of CLR records, we will repeat partial rollback and by examinating of a content of last CLR record, we will find which pages still should be rolled back and will restore initial state of the data structure before a transaction.
3. Structure modification changes are completed, and CLR record is put at the end of this changes. 
In such case, we restore all structure modification changes and by processing of CLR record will rollback all changes which exist *after* structure modification changes but will not rollback structure modification changes itself. There is a good reason why we can not rollback structure modification changes. 
Page locks are released once we apply structure modification changes, and those pages may be changed by other successfully committed transactions. At the end of data, restore procedure changed data structure will be logically in the same state as it was before a transaction is started.
4. Structure modification changes and changes on domain page are completed in such case we will rollback only changes of domain page
and skip structure modification changes because of a presence of CLR record. We do not perform logical remove of data but only revert content of domain page because of concurrent access to the data structure it may be in the invalid state till complete restore of a state 
of all transactions will be completed.
At the end of data restore procedure changed data structure will be logically in the same state as it was before a transaction is started.

As you can see above if we restore system after crash, some space may be lost at the end of data restore procedure but amount of 
such space should be so minor that may be not taken into account.

What is interesting that even if we implement only new transaction processing protocol but will not change 
lock model of already existing components we still will increase system speed and scalability.
Let's suppose we have two ongoing transactions on the system, and component lock model is used.
First transaction changes components A and B and second transaction changes components A and C. 
In the current implementation, those transactions will be serialized but in the new implementation, transactions will go in parallel once one of them completes changes in component A. 

So at a first stage, we may implement protocol itself and then change lock model of components from component level to page level in next releases (such models will be described soon in separate OEPs if this OEP will be approved).

Let's consider the rest two operations on data structures - update and delete.
 
During execution of delete operation it is executed by following steps:

1. Delete an entry from domain page.
2. Put a request on the сlean up a queue to clean up consumed space after tx commit (it is needed only for a cluster). 

So during transaction rollback or restore, we revert only domain page content and as a result, a record is automatically restored.

Consider in details second item of delete algorithm - cleanup queue. 
When we delete record in a cluster, it is reasonable to claim space consumed by data back to the storage manager.

To make that possible, we create a cleanup queue which is filled by operations performed during a transaction (delete/update) which contains position to the record to be cleared  (this position is always the same even during internal page defragmentation and will not be changed after the record delete).
If a transaction is rolled back then, the cleanup queue is cleared, and no changes will be performed. If a transaction is committed then, changes are applied in a background thread. This clean up queue consumes very few memory, each entry queue consists of only two fields  
clusterId and position of record inside of data file, but it also may be bounded, we may allow adding no more than
1 000 000 of such entries at any time for a single transaction and 10 000 000 entries in total may be contained in the queue. The same limit may be applied to the amount of locks which may be acquired by the transaction to avoid any risk of OOM.
One of the ideas is to use ThreadPoolExecutor with a bounded queue and a maximum number of threads equal to an amount of cores.

Cleanup threads which process this queue will pull each entry and process it in a separate transaction.
So even if a system is crashed then the transaction will be rolled back, and a tiny amount of space will be lost.

The logic of update of data is similar because an update is a mix of deletion of previous data and an addition of new data.

1. Perform structure modification operations to add new data if needed (not needed for indexes, in the index, the only value inside of leaf page is updated).
2. Update domain page.
3. Put a request on the сlean up a queue to clean up consumed space after tx commit.  

So during rollback:
1. If rollback happens in the middle of execution of structure modification operations, then we revert all changes on binary level.
2. If rollback happens after domain page update, we remove new record data from a cluster and revert record pointer to old record content.

Restore logic is same as logic during rollback with the only exception that we do not remove already added record but revert domain page content. We do not perform logical remove of data but only revert content of domain page because of concurrent access to the data structure it may be in the invalid state till complete restore of a state of all transactions will be performed.

Once again it is worth to note that all those procedures will work even if we will use component locks, not page locks, providing a much better level of scalability without of changing of component implementation.

The isolation of visibility of changes for single item requests for cluster and index is trivial. 
We lock record or item inside of transaction then try to find this item inside of index or lock.

Implementation of range queries or iteration over ridbag or cluster is a bit complex:

1.  We lock page which contains given record or key
2.  Try to lock record or index
3.  If lock attempt fails, we release page
4.  Lock the record or key
5.  Lock the page 
6.  Check presence of key or record 
7.  Continue to iterate over data structure

Let's estimate risks of a presence of deadlocks in proposed transaction protocol and approaches to avoid them.

In a current implementation, we track changes which are going to be applied both to records, and key indexes.
To prevent deadlocks between records and keys we need to sort all records and all keys and acquire locks for all of them before we start transaction inside of storage. 

Such approach will prevent any deadlocks between keys and existing records.
But there are still two variants of deadlocks:

1. RIDs of new records are unknown before insertion, so before acquiring a lock on the RID we need to acquire page locks to get a value of the RID, but during record read, we acquire locks in opposite direction which may lead to deadlock. Such cases are very rare. For most of the cases, we will not provide invalid RIDs to read record content, but such case is still possible.
2. We do not track changes of ridbags which are implemented as small trees and also there is an automatical conversion between embedded and tree version of presentation of ridbag.  

To solve the first problem, we will use "try locks."
Algorithm of insertion of records will look like following:

1. Make structure modification changes and remember the position of record content.
2. Lock domain page, but not change it, instead calculate RID value.
3. Try to acquire  record lock if attempt is failed release page locks
4. Acquire record lock with calculated RID.
5. Acquire cluster page locks.
6. Acquire domain page and if RID is still the same complete change, otherwise, calculate new RID and repeat algorithm.

To solve the second problem, it is proposed to use deadlock detection which is based on an algorithm of finding of cycles in "wait-for" graph. When a deadlock is detected, system rollbacks operation on one of the ridbags and will repeat it later.  Such log detector is implemented on 80%  in https://github.com/orientechnologies/orientdb/tree/dreadlock/core/src/main/java/com/orientechnologies/common/concur/dreadlock. 
The detector has O(N) complexity and as result time which is needed to detect deadlock matters of hundreds of nanoseconds at max. So we may execute deadlock detection every 0.1 usec without risk of performance loss in case of presence of deadlock.

###Low-level design

Some parts of the design have already implemented and described here for clarity.

Let's look at data structures are used in current transaction protocol.

**Log records**

Log records contain following fields:

1. LSN of the record. This value grows continuously and presents the logical address of the log record inside of WAL.
2. Transaction id. It has not to be UUID, usage of the simple counter will be enough. 
3. PrevLSN. LSN of the previous record written by the same transaction.
4. PageID. The id of the changed page.
5. FileID. The id of a file which contains changed record.
6. UndoNextLSN. Exists only in CLR records. Pointer to the next log record which should be rolled back.
7. Data. Undo and redo data. Either one or both of them may be empty.

**Pages**

Each page contains the pageLSN and CRC32 fields. Each page update looks like following:

1. Page lock is acquired.
2. The page is changed.
3. Page changes are logged into WAL.
4. LSN of WAL record is set to pageLSN field.
5. Page lock is released.

When buffer flushes page to the disk, it checks that all records of WAL are flushed at least till the LSN stored in pageLSN.
If that is not the true flush of WAL content is performed. During page flush, CRC32 code for page content including pageLSN is calculated and assigned to the CRC32 field. CRC32 field is used to detect whether a page is partially flushed to the disk because of the process crash.

**Transaction table**

This table is used to track the state of the active transactions during normal processing of the transaction and restore of a database after a crash. In our project, it will be implemented as a mix of two classes atomic operations manager and the atomic operation itself.

Transaction table contains following fields:

1. Transaction id.
2. Last LSN. LSN of last log record written by a transaction.
3. UndoNextLSN. LSN of next record processed during rollback. If the most recent log record written or seen for this transaction is an
undoable non-CLR log record, then this field’s value will be set to last LSN. If that most recent log record is a CLR, then this field’s value is set to the UndoNxtLSN value from that CLR.

**Dirty pages table**

This table contains all pages which are not written to the disk yet. Table consists of three fields (pageID, fileID, dirtyLSN). Dirty LSN value is calculated using the following approach. If a page is acquired from the disk cache for an update, then a value of latest LSN operation which is written to the log becomes a value of dirtyLSN. This table shows LSN of operations which are probably are not stored on the disk yet. Once a page is written to the disk its entry is removed from dirty pages table. 

**Rollbacks**

The protocol supports partial rollbacks till provided LSN. This feature is used for example to rollback ridbag operations after deadlock is detected. It is possible because of usage of CLRs which allow rewinding transaction until the desired point.

Pseudo code for rollback looks like following:

```java
rollback(saveLSN, transID) {
 undoNext = transTable.get(transID).undoNextLSN; // LSN of first record to undo
 while(saveLSN < undoNext) //loop through all records
 {
  logRecord = wal.read(undoNext);
  if(logRecord is update record) {
   if(logRecord.data.undo is not empty) {
    page = diskCache.acquire();
    undoPage(page, logRecord);//the real undo operation depends on undo information it may be logical rollback
    // or may be rollback of content of single page
    clrLSN = wal.log(new CLR(logRecord.transID, transTable.get(transID).lastLSN /*prevLSN*/, logRecord.fileID, logRecord.pageID, logRecord.prevLSN/*undoNextLSN*/, logRecord.data.undo)); //write CLR
    page.pageLSN = clrLSN; 
    diskCache.releasePage(page); //in case of logical undo the only CLR record is written because changes of content of the page 
    //are logged in other records
    
    transTable.get(transID).lastLSN = clrLSN;
   }
   
   undoNext = logRecord.prevLSN;
  } else if(logRecord is CLR) {
   undoNext = logRecord.undoNextLSN; 
  } else {
   undoNext = logRecord.prevLSN; //service record just skip it
  }
  
  transTable.get(transID).undoNextLSN = undoNext;
 }
}
```

**Checkpoints**

Obviously, it is impossible to write log forever. Also, it is very time consuming to restore the database from the first operation ever performed on the database. To solve those problems we periodically make checkpoints of database content.

During checkpoint following actions are performed:

1. Log record which indicates that check point is started is written to the WAL. LSN of this record is written to the WAL master
record once log record which indicates end checkpoint is flushed to the disk. Master record is a special place in WAL, writes to which always forced to the disk. Also, its content is protected by CRC32 code and by usage of Twin Blocks pattern.
2. Background flush of dirty pages is stoped. We can not write dirty pages table without force sync because of presence internal caches inside of disk drives and OS.
3. All the files are force synced to the disk (only files not a content of the disk cache).
4. Dirty pages table is written to the disk.
5. Background flush of dirty pages is started. 
5. Transaction table is written to the disk.
5. Log record which indicates that check point is stopped is written to the WAL.
6. WAL content is flushed to the disk.
7. At this point, we can cut WAL till the first log record which is smaller than minimum LSN in dirty pages table.


The checkpoint is treated to be valid only when "end checkpoint" record is stored to the disk.
During a checkpoint, transactions can be processed, and data can be written, the correct state of dirty pages and transaction tables will be restored during analysis pass of data restore process.

**Restore data after crash**

Data restore process consist of three passes:

1. Analysis pass. This pass accepts LSN of the checkpoint as input data and returns: dirty pages table, transaction table and redo LSN (LSN of record from which redo pass will restore data).
2. Redo pass. During this pass state of all transactions is restored.
3. Undo pass. During this pass, all unfinished operations are reverted.
4. At the end of the restore process, a checkpoint is made. It is not needed to flush buffer at this point.
5. Transaction id counter is set to the maximum value of transaction id in a transaction table.
 
Please note that WAL operations are mostly sequential (with exception of Undo pass), and several subsequent passes of WAL will not provide noticeable performance overhead. The main target of analysis pass is to minimize amount of random IO operations caused by 
page loads.

**Analysis Pass**

The only records which are written at this pass are the transaction end records which indicates that transaction is finished.

During this pass, if a log record is encountered for a page whose identity does not already appear in the dirty pages table, then an entry is made in the table with the current log record’s LSN as the page’s recLSN. The transaction table is modified to track the state changes of transactions and also to note the LSN of the most recent log record that would need to be
undone if it were determined that the transaction had to be rolled back.

Returned redo LSN is minimum LSN of the dirty pages table.

Pseudo code for analysis pass looks like following:

```java
restartAnalysis(checkpointLSN, transTable, dirtyPages) {
  logRec = wal.read(checkpointLSN);
  logRec = wal.readNext(logRec.lsn);//skip checkpoint record
  while(logRec != null) { // loop till the end of the log
   if(logRec is transaction related record && !transTable.contains(logRec.transID)) { //we put in the log service records too
     transTable.put(logRec.transID, logRec.lsn/*last LSN*/, logRec.prevLSN/*undoNextLSN*/); //insert entry in transaction table
   }
   
   if((logRec is update) || (logRec is CLR)) {
    transTable.get(logRec.transID).lastLSN = logRec.lsn;
    
    if(logRec is update) {
     if(logRec.data.undo != null) {
      transTable.get(logRec.transID).undoNextLSN = logRec.lsn;
     }
    } else {
     transTable.get(logRec.transID).undoNextLSN = logRec.undoNextLSN;
    }
    
    if(logRec.data.redo != null && !dirtyPages.contains(logRec.pageID)) {
     dirtyPages.put(logRec.pageID, logRec.lsn); 
    }
   } else if(logRec is end checkpoint record) {
    for(entry : logRec.transTable) {
     if(!transTable.contains(entry.transID)) {
      transTable.put(entry.transID, entry);
     }
    }
    
    for(entry : logRec.dirtyPages) {
     if(!dirtyPages.contains(entry.pageID)) {
      dirtyPages.put(entry.pageID, entry.recLSN);
     } else {
      dertyPages.get(entry.pageID).recLSN = entry.recLSN;
     }
    }
    
   } else if(logRec is transaction end record) {
    transTable.remove(logRec.transID);
   }
   
   logRec = wal.readNext(logRec.lsn); 
  }
  
  for(entry : transTable) {
   if(entry.undoNextLSN == null) { // end of transaction
    wal.log(new TxEND(entry.transID, entry.lastLSN /*prevLSN*/));
    transTable.remove(entry.transID);
   }
  }
  
  return min(dirtyPages.recLSN);
}
```

**Redo pass**

The redo pass starts scanning the log records from the RedoLSN point. When a redoable log record is encountered,  a check is made to see if the referenced page appears in the dirty pages table. If a page is not present in dirty pages table then it is already flushed on disk and result of an operation is not lost and not needed to be restored. If the log record’s LSN is greater than or equal to the recLSN for the page on the table, then it is suspected that the page state might be such that the log record’s update might have to be redone. To resolve this suspicion, the page is accessed. Then CRC32 code of page content is calculated. If CRC32 codes calculated and stored inside of page are not equal then the content of the page was written only at the half since the last flush and change on the page has to be redone. If CRC32 codes are equal, we compare LSNs of the page and log record.  If the page’s LSN is found to be less than the log record’s LSN, then the update is redone. Thus, the RecLSN information serves to limit the number of pages which have to be examined.  This routine reestablishes the database state as of the time of system failure.

Pseudo code for redo pass looks like following:

```java
restartRedo(redoLSN, dirtyPages) {
 logRecord = wal.read(redoLSN);
 while(logRecord != null) {
  if((logRecord is update || logRecord is CLR) && logRecord.data.redo != null && 
  dirtyPages.contains(logRecord.pageID) && logRecord.lsn >= dirtyPages.get(logRecord.pageID).recLSN) {
   page = diskCache.acquire();
   crc32 = calculateCRC32(page);
   
   if(crc32 != page.crc32) {    //record content is broken we update it anyway 
    redoUpdate(page, logRecord);
    page.lsn = logRecord.lsn;
   } else {
    if(page.lsn < logRecord.lsn) {
     redoUpdate(page, logRecord);
     page.lsn = logRecord.lsn;
    } else {
     dirtyPages.get(logRecord.pageID).recLSN =  page.lsn + 1; //dirty pages contain out of dated information, we update it to prevent //loading of allready stored operations
    }
   }
   
   diskCache.release(page);
  }
  
  logRecord = wal.readNext(logRecord.lsn);
 }
}
```

**Undo pass**

The input to this routine is the restart transaction table. The dirty pages table is not consulted during this undo pass. 
Also, since history is repeated before the undo pass is initiated, the LSN on the page is not consulted to determine whether an undo operation should be performed or not. The restart undo routine rolls back transactions, in reverse chronological
order, in a single sweep of the log. This is done by continually taking the maximum of the LSNs of the next log record to be processed for each of the yet-to-be-completely-undone transactions until no transaction remains to be undone. The next record to process for each transaction to be rolled back is determined by an entry in the transaction table for each of those transactions.
In the process of rolling back the transactions, this routine writes CLRs.

Pseudo code for the undo pass looks like following:

```java
restartUndo(transTable) {
 while(!transTable.isEmpty()) {
  undoNextLSN = max(transTable.undoNextLSN);
  logRecord = wal.read(undoNextLSN);
  
  if(logRecord is update) {
   if(logRecord.data.undo != null) {
    page = diskCache.acquire(logRecord.pageID);
    undoPage(page, logRecord);
    
    clrLSN = wal.log(new CLR(logRecord.transID, transTable.get(transID).lastLSN /*prevLSN*/, logRecord.fileID, logRecord.pageID, logRecord.prevLSN/*undoNextLSN*/, logRecord.data.undo)); //write CLR
    
    page.pageLSN = clrLSN;
    diskCache.releasePage(page);
    
    transTable.get(transID).lastLSN = clrLSN;
    
    diskCache.release(page);
   }
   
   transTable.get(transID).undoNextLSN = logRecord.prevLSN;
   
   if(logRecord.prevLSN == null) {
    wal.log(new EndTX(logRecord.transID, transTable.get(logRecord.transIS).lastLSN));
    transTable.remove(logRecord.transID);
   }
  } else if(logRecord is CLR) {
   transTable.get(logRecord.transID).undoNextLSN = logRecord.undoNextLSN;
  }
 }
}
```

**Parallelisataion of data restore after crash**

Time of data restore is critical for applications which rely on database high availability. 
There are two approaches how time of data restore can be decreased:

1. Minimize amount of pages which can be accessed during data restore.
2. Restore transaction in parallel threads.

In "redo pass" the only limitation is that all changes applied to the pages have to be ordered.
So during redo pass, we may read all log records in a single thread and partition redo operations between different threads.

In undo pass all CLR records from single transaction has to be chained so we may process undo operations in parallel too but all processing will be partitioned using entries of transaction table.

##Alternatives:

There are several proposed alternatives, the main differences between current proposal and the rest are following:

1. Lock based proposals are page based, not key/record based. Page based locks are too expencive for indexes because single non-leaf
page locks may block a lot of underlying pages.
2. Some of the lock based proposals do not take issue with ridbag and possible deadlocks into account.
3. CAS based approaches may lead to performance degradation in case of a high load.

##Risks and assumptions:

There is a risk of data corruption in case of incorrect implementation of data rollback or data restore.

##Impact matrix

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
