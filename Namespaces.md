# Namespaces #

The idea behind namespaces is to be able to segregate data to make updating/querying it more efficient and to isolate it while still being able to use classes and schema.

The big motivator for this is multi-tenancy. Many OrientDB users have 10s of 1000s of clients, and they need a way to isolate the clients' data.

We could isolate each client as a separate database, but having to open each one to access a client is not efficient, maintaining that many databases would not be fun, and there's no way to use the same class schema across all the databases.  

## Implementation ##
What if we created quasi sub-databases under the main directory for a database, each representing a namespace?

/databases/MyDatabase

    /American

    /Lufthansa

    /Southwest


Each sub-directory representing a namespace could contain all the clusters and indices, etc., for a specific namespace without polluting the top-level database files.

The cool thing is that you could still use the top-level database classes (especially with inheritance) but your CRUD ops and queries would be isolated to just the namespace content.

Clusters would never be shared between namespaces nor between a namespace and the top-level database files.  What I mean is that if cluster #20 is used in the Lufthansa namespace, #20 is not also used in the top-level database nor by any other namespace.

But, we could still link to records that reside between a namespace and the top-level database.

## Queries ##
I could do "select from Lufthansa:MyClass where Age > 50" (or use Lufthansa.MyClass or Lufthansa::MyClass or whatever).


I think that "select from MyClass..." would only return the top-level records if the namespace is not specified.


## Inheritance/Polymorphism
This design would give us the benefit of having multiple databases but without having to open potentially 1000s of different databases while still allowing class inheritance and polymorphism.


## Data Isolation ##

Data isolation could even be on a different drive if the sub-directory is linked to a different partition.

Thinking of distributed mode, we could even partition data using namespaces so that all client data resides on a single node (not including replication).

## Security ##
I haven't quite worked out how OUser and ORoles might be affected by this.  It would be nice if each namespace could have its own set of users and roles, especially when using ORestricted. 


