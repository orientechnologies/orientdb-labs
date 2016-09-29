## Summary: ##
We need a way to isolate user data on a large scale while still being able to use class inheritance and polymorphism.  Think multi-tenancy.

A `namespace` is the current way to describe the concept, but we're open to better names/concepts.

## Goals: ##
- Isolate user data so that all related clusters, indices, etc. can be moved together, efficiently, and can reside in a different physical location (disk partition).
- Have a way to limit CRUD operations and queries to the records residing in the namespace only.
- Support class inheritance of classes residing within the namespace from classes in the top-level domain.
- Support separate indices created against classes residing within the namespace, storing them inside the namespace, to not pollute the top-level indices.
- Support users and roles per namespace.  

## Motivation: ##
The primary motivator is to support multi-tenancy efficiently.  Currently, we support multi-tenancy in three different ways:
1. Use a separate database for each client.
2. Use a separate cluster for each client's class.
3. Use an identifier within each record to mark it belonging to a specific client.

### Separate Database
There are multiple problems with using separate databases for this, though it does provide good data isolation as well as better security:
- Even a small number of clients (100) would make maintaining so many different databases a huge nuisance.  Many customers have 1000s of clients.
- Making a small schema change would require updating all databases.
- Accessing a client's data would require opening a new database connection for every client.  This is not efficient and prevents using a collection pool.

### Separate Cluster
Today, we can store each client's data in a separate named cluster.  The primary problems with this are:
- Querying via cluster name loses the advantage of using polymorphic queries on a base class.
- Limiting records to a single cluster prevents the storage engine from spreading the load across multiple clusters when running with multiple threads.
- Likewise, limiting records to a single cluster per class affects the distributed systems ability to partition data efficiently.
- It's currently not possible to place an index against an individual cluster.

### Unique Identifier
Using a unique identifier property to separate client records is a common pattern.  However, there are multiple problems with it as well:
-  If a common class is used by all clients and each client has millions of records of data, a single class could end up storing billions and billions of records.  This ultimately reduces efficiency.
-  That unique identifier must be indexed, and with billions of records, retrieving the records for a single client or adding new records to the index can take longer than it should.
-  If all client data is stored in the same class clusters, there is no way to isolate the records on a per-client basis.


## Description: ##
### Implementation ###
One possible solution is to create quasi sub-databases under the main directory for a database, each representing a namespace:

/databases/MyDatabase

    /American

    /Lufthansa

    /Southwest


Each sub-directory representing a namespace could contain all the clusters and indices, etc., for a specific namespace without polluting the top-level database files (making them grow extremely large).

You could still use the top-level database classes (especially with inheritance) but your CRUD ops and queries could be isolated to just the namespace content.

Clusters would never be shared between namespaces nor between a namespace and the top-level database files.  What this means is that if cluster #20 is used in the Lufthansa namespace, #20 is not also used in the top-level database nor by any other namespace.

But, you could still link to records that reside between a namespace and the top-level database.

### Queries ###
The syntax needs to be chosen (Lufthansa:MyClass, Lufthansa.MyClass, or Lufthansa::MyClass or whatever), but
```"select from Lufthansa:MyClass where Age > 50"```
would only search on the *MyClass* class contained within the 'Lufthansa' namespace directory. 


Using the normal `"select from MyClass..."` would only return the records contained in the top-level directory. 

### Inheritance ###
A class created in a namespace should be able to inherit from a class that's been created in the top-level database.  The advantage to this is being able to produce schema that's used by all clients.  Modifying the schema happens in only one place instead of in 1000 different databases.


### Data Isolation ###
Data isolation is easily supported since all data related to a namespace resides in its own directory.

The namespace could even be on a different drive if the sub-directory is linked to a different partition.

Thinking of the distributed mode, another advantage to using a namespace is that all client data could reside on a single node (not including replication).

### Security ###
It would be nice if each namespace could have its own set of users and roles, especially when using ORestricted.

## Alternatives: ##
We do nothing.

## Risks and assumptions: ##
Since implementing this concept properly will involve almost all modules, the risks are the time required to do it properly and then missing a piece and either introducing bugs as a side-effect or missing support in a critical area.

## Impact matrix ##
In one way or another, this will impact almost all systems.
