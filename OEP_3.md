## OrientDB Document Database Factory/Engine

**Summary:**
Create an unified API for manipulate database.

**Goals:**
- Provide an uniform API for any connection type (local,remote,distributed)
- Hide Implementation details from public API
- Have a centralized context that can be used for free all the used resources

**Non-Goals:**
- Provide integration with Object/Graph API, will be discussed in a different contex

**Motivation:**
- We often have issues on public API because of not clear split between public and private.
- Hard to free resources or portion of resources due to global contexts.
- Need to provide different implementation between remote/plocal databases

**Description:**
Define an OrientDBFactory which can be created for different environments:
- *embedded* based on a folder that can allow disc or in memory databases
- *remote* connect on a remote server allowing on disc or memory database on the server 
- *distributed* otionally provida an api to join a cluster of nodes with embedded distributed feature and possibility to have on disc or memory distributed databases

provide an as well an API to create the factory from a simple URL.

Each factory have methods for manipulate database instances:

- *create* create database providing  auth
- *drop* drop database providing  auth
- *open* open a database providing  auth
- *list* list databases providing auth

Example of the API:
https://github.com/orientechnologies/orientdb/blob/poc_oriendb_factory/core/src/main/java/com/orientechnologies/orient/core/db/OrientFactory.java
https://github.com/orientechnologies/orientdb/blob/poc_oriendb_factory/core/src/test/java/com/orientechnologies/orient/core/db/OrientFactoryTests.java  

**Alternatives:**
- leave as it is right now.
- provide different API for different contexts

**Risks and assumptions:**
The API break with the past and need a refactor of the user code


**Impact matrix**

- [ ] Storage engine
- [ ] SQL
- [ ] Protocols
- [ ] Indexes
- [ ] Console
- [x] Java API
- [ ] Geospatial
- [ ] Lucene
- [ ] Security
- [ ] Hooks
- [ ] EE

