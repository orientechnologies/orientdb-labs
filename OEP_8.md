**Summary:**
 
Asynchronous read cache with state machine

**Goals:**

1. Remove exclusive locks on read-only benchmarks.
2. Increase scalability of read cache in general.

**Non-Goals:**

1. Improve speed in single thread case.

**Success metrics:**

1. Improved throughput of read only YCSB workloads.
2. Improved performance of all YCSB workloads.

**Motivation:**

To implement thread safety guarantees inside of read cache we use implementation of group locks which are close to Google Striped.
But this approach has couple of disadvantages:

1. Even in a case of read-only benchmarks, we need to acquire an exclusive lock on the cache page to move page between queues of 2Q cache. 
There are https://screencloud.net/v/3PZd states of threads in case of YCSB "read only" benchmark. All "wait" states of threads are caused by exclusive page locks inside of 2Q cache.
2. Very often to evict pages from cache in case of cache overflow we need to apply an exclusive lock on all cache which leads to high degradation of system scalability.

An alternative approach is proposed to overcome these disadvantages.
It is proposed to:

1. Gather statistics about usage of pages in an asynchronous manner which decrease the precision of cache but it will allow avoiding locking of pages during page load from cache.
2. It is proposed to create a state machine to keep thread safety guarantees.

The proposed design is an adoption of the design of Caffeine framework which has excellent scalability characteristics.

**Description:**

Current workflow of 2Q cache looks like following:

1. System requests page from cache.
2. "Striped like" lock is applied on a page.
3. The page either loaded from cache or disk.
4. The page is moved between queues of 2Q cache.
5. The page is unlocked and returned to the requestor.

As alternative following design is proposed:

1. All pages are stored not inside of several queues but inside of the single concurrent hash map. So we split cache content and its state. The last one, as usual, is stored inside of queues.
2. When a page is requested it is loaded from a hash map but locks are not applied on the page. Instead, operations are logged into the lock-free 
buffer. 
3. State of the page is changed according to state machine rules, so page either returned by requestor which prevents removal of the page from cache during an eviction or will be reloaded again from cache (very rare case) and will be again returned to a user.
4. If a threshold of operations buffer is reached it if flushed by one of the threads and a state of 2Q eviction policy will be updated accordingly. 
5. During phase 4 if the cache is overflowed pages will be removed from the cache by one of the threads if the state of the page allows removing the page (page is not used).

Lock-free operations buffer.

To gather statics, we will use ring buffer which will be implemented using a plain array.  But this buffer will be presented as not a single instance of the array but as an array of arrays. Each of the arrays will be used by a subset of threads to minimize contention between threads.
If threshold on one of those arrays will be reached, all those arrays will be emptied by one of the threads by applying tryLock operation which prevents contention between threads. Lock mentioned above is used only during buffer flush and is not used during logging of a statistic inside of buffers. Pointers inside of buffer will be implemented without 
CAS operations, as a result, few of operations will be lost, but it will not cause significant changes in overall statistics.
There is limit on amount of records which will be flushed at once per single thread. Eache thread flushes no more than 2 * threshold amount of elements from buffer.

State machine.

To achieve thread safety guarantees a state machine with following states will be introduced:

1. Acquired - when a page in "acquired" state, it is impossible to remove a page from cache. 
2. Released - a page is free to acquire it by cache user.
3. Removed - this state is set by eviction procedure before the page will be withdrawn from the cache. If requested page has the given state, it should be deleted from the cache (in such way we help eviction process to remove it), but the content of this page is copied and inserted back to cache with "released" state unless the page is not inserted in other thread. This trick is needed to keep thread safety guarantees during page eviction. 

Eviction process

The removal process is performed during operations buffer flush (when state of eviction policy is updated).
During this process:
1. State of a page which is going to be removed from the cache is changed to "removed" state.
2. If the state is not allowed to be changed then, next eviction candidate is taken.
3. If the state is changed, a cache entry is removed from the concurrent hash map. 
To prevent case when we remove the reloaded page (see "state machine" description item 3) concurrent hash map item is removed using method map.remove(key, value) https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html#remove-java.lang.Object-java.lang.Object- .

**Alternatives:**

There is no any other proposal for cache lock models at the moment.

**Risks and assumptions:**

In the case of incorrect or poorly tested implementation, there is a risk of data corruption.

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

