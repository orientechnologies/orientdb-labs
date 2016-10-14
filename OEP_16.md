**Summary:**
A complete review of the binary protocol, introducing streaming, server side transaction, and specifications for any other data send on the binary channel.

**Goals:**
Stabilize the binary protocol introducing all features needed for all the 3.x version and future, avoiding the need of review.

**Non-Goals:**

Define a replacement of the binary protocol.

**Success metrics:**

No change of the specification for all the 3.x version.
Support of this specifications from third party.

**Motivation:**

Stabilization of the binary protocol.
Add support for the functionality requirements for 3.0

**Description:**

For version 3.0 will be a general refactor on the binary protocol, it will guarantee the compatibility with 2.x protocol.

The changes are split in 6 main area

1. general review of protocol flow, handshaking and low basic data transfer

2. Redesign of push request based on subscribe.

3. Introduction of server side transaction

4. Introduction of new result set handling

5. Unification under the same protocol specification of all client required informations

6. Review of error format and specification of error by code


### 1) Genera review of protocol 

 - Introduction of a new phase of handshaking as soon as a connection happen where client and server share protocol versions and specifications.
 - remove all the protocol specific information from connect and open
 - evaluate if introduce varint in the serialization of network messages.


### 2)  Subscribe based Push Request 

The push request flow will change to the following steps:

 - redefine the push request flow with event subscribe, and client reception ack.
 - redefine the push messages for distributed configuration, schema structure, database structure(like list of clusters), live query


### 3) Server Side transaction

 - Introduction of a new begin transaction, and "rebegin transaction" with updated data
 - add support of transaction download and upload after a not idempotent query
 - Introduce of commit of a current running transaction


### 4) Introduction of new result set handling

 - Define a new pagined query result set, in a stateful way.
 - Define serialization of query result with support of  projection, nested projection and record with vertex and edge specialization.
 - Define message for close the query, interrupt it before end.


### 5) Unification under the same protocol specification of all client required informations

 - Integrate the record serialization format in the protocol, making it to follow the protocol versions.
 - Define specification of storage metadata.
 - Define specification of distributed configuration information for clients.
 - Define specification of database metadata (Schema, Index, Functions, Security,Scheduler, Sequences) information for clients.

### 6)  Review of error format and specification of error by code

 - Introduce of different error serialization format as a replacement of current java exception serialization
 - Specify error codes for all the major kind of errors, and user interaction errors.


**Alternatives:**

Define a new protocol, that cover all the case that we cover now with all the new cases.

**Risks and assumptions:**

Is going to require an big effort from contributors.

**Impact matrix**

- [ ] Storage engine
- [ ] SQL
- [x] Protocols
- [ ] Indexes
- [ ] Console
- [ ] Java API
- [ ] Geospatial
- [ ] Lucene
- [x] Security
- [ ] Hooks
- [ ] EE


*Details*

### 1) Genera review of protocol 

New handshaking messages, as soon as a connection happen:
The server will send:
 - current server protocol version
 - minimal supported protocol version by the server
 - server instance identifier (nodeID/Node name)
 - server software version 
 
 The client will send an handshaking operation with:
 - corrent protocol version
 - client name
 - client version
 - client id
 - error details format (txt or java)
 
 The new connect and open message will look have:
 
 Connect Request :
 - server user username
 - server user password
 Connect Response:
 - sessionId
 - token
 
 Open Request:
  - database name
  - user name
  - user password
 Open Response:
  - sessionId
  - token
 
 - evaluate if introduce varint in the serialization of network messages
  TODO


### 2)  Subscribe based Push Request 

the flow of the push request will be changed to:

 - client do a subscribe to a push request
 - server return the ok to the subscribe with inside the current valid information for the request push
 - the client waiting in read for a push request
 - at some point in time the server send a push message and wait for the client to ack
 - the client elaborate the message and give an ok to the server
 - In case of network failure is the client responsibility to resubscribe


Base messages are *Subscribe*, *resubscribe*, *push*, *usubscribe*

Subscribe Request:
 - valid token  
 - Id push message to subscribe
Subscribe Response:
 - Id push Request
 - First content of the specific push request
 
 

