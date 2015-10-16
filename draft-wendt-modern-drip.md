# Distributed Registry Protocol

**draft-wendt-modern-drip-00**

## Abstract

This document describes a protocol for allowing a distributed set of nodes to synchronize a set of information in real-time with minimal amount of delay.  This is useful for registry types of information like identity and telephone numbers with associated routing and ownership information, although could be extended to support other distributed real-time information updates as well.

## 1. Introduction

This document describes the Distributed Registry Protocol (DRiP).  DRiP defines a set of peer protocols for how an arbitrary number of nodes arranged in a distributed mesh architecture can be used to synchronize data in real-time across a network.

## 2. Terminology

  - Initiator node - A node that initiates data propagation.
  - Non-Initiator node - A node that receives the propagated data.

## 3. DRiP Overview

DRiP uses a mix of Gossip protocol with the addition of a voting system.

### 3.1 Distributed MESH Architecture

The DRiP architecture is based on a peer-to-peer communication model where a given node associated with a data store is not necessarily aware of the total number of nodes in the entire network. Minimally, every node should reachable by at least one multi-node path from every other node. Each node in the DRiP network maintains a list of peer nodes from which it receives and transmits updates. Information is propogated by forwarding to it's peer nodes until the information received by a node has already been received.

       ___           ___                         ___           ___ 
      |DB |_________|DB |                       |DB |_________|DB |
      |___|         |___|                       |___|         |___|
        |   Data      |                           |   Data      |  
        |   Store     |                           |   Store     |  
       _|_  Cluster  _|_                         _|_  Cluster  _|_ 
      |DB |_________|DB |                       |DB |_________|DB |
      |___|         |___|                       |___|         |___|
                        \                       /
                         \                     /
                         _\___     DRIP     _ /__
                        |Node |------------|Node |
                        |  A  |    HTTPS   |  C  |
                        |_____|            |_____|
                            \H           H/
                            D\T         T/D
                             R\T       P/R 
                              I\P     P/I  
                               P\S   S/P   
                               __\__ /   DRIP     _____  
                              |Node |------------|Node |
                              |  B  |   HTTPS    |  D  |
                              |_____|            |_____|
                             /                   /
          ___           ___ /                 __/_          ___ 
         |DB |_________|DB |                 |DB |_________|DB |
         |___|         |___|                 |___|         |___|
           |   Data      |                     |   Data      |  
           |   Store     |                     |   Store     |  
          _|_  Cluster  _|_                   _|_  Cluster  _|_ 
         |DB |_________|DB |                 |DB |_________|DB |
         |___|         |___|                 |___|         |___|

**Figure 1: Distributed Mesh Architecture**

## 4. DRiP procedures

### 4.1 Entry Entitlement Verification

When a node owner would like to create or modify a set of information, generally in the context of a registry, there MAY be a verification procedure that an entry can be created.  This could include validating whether an entry is entitled to be written or modified in the first place.  For example, identity or telephone number ownership or porting.  The exact mechanics of this are out of scope of this document and are generally application specific.

### 4.2 Custom HTTP header fields

Custom HTTP header fields will be used to carry Node specific information.

Field Name| Description |
-------------:   | :----------- |
**DRiP-Node-ID** | Each node in the mesh MUST have a **unique** identifier.  An Initiator node **MUST** set its own node ID as the field value. A Non-Initiator node **MUST NOT** change the **DRiP-Node-ID** field value in the HTTPS request as it is propagated to its peer nodes (nodes it is aware of).|
**DRiP-Node-Counter** | Every node maintains a count of the number of times it **initiates** data propagation. This counter **MUST** be an **unsigned** type, typically, a 64 bit integer. The Initiator node **MUST** set this count as the field value. The Non-Initiator node **MUST NOT** change the **DRiP-Node-Counter** field value in the HTTPS request as it is propagated to its peer nodes (nodes it is aware of).|
**DRiP-Node-Counter-reset** | A node can reset the count (to zero) of the number of times it **initiates** data propagation. If the counter value is reset, prior to initiating data propagation, then this field value **MUST** be set to **true**. Otherwise, it **MUST** be set to **false**, at all times. A typical use case to reset the counter value is when the counter (of **unsigned** type) value **wraps around**. The Initiator node **MUST** set this field value to either **true** or **false**. The Non-Initiator node **MUST NOT** change the **DRiP-Node-Counter-reset** field value in the HTTPS request as it is propagated to its peer nodes (nodes it is aware of).|
**DRiP-Transaction-Type** | The Initiator node **MUST** set this field value to be either **real-time update**, **periodic sync** or **sync on demand**. See section 3.4. The Non-Initiator node **MUST NOT** change the **DRiP-Transaction-Type** field value in the HTTPS request as it is propagated to its peer nodes (nodes it is aware of).
**DRiP-Sync-Complete** | During **periodic sync** or **sync on demand** transaction type, the Initiator node **MUST** set this field value to be **true**, if no more data is left to be propagated. Otherwise, this field value **MUST** be set to **false**. The Non-Initiator node **MUST NOT** change the **DRiP-Sync-Complete** field value in the HTTPS request as it is propagated to its peer nodes (nodes it is aware of).|


### 4.3 Data Propagation Rules

- A node propagates data to all its peer nodes except the the node from which it received data. For example, in Figure 1, when node B receives data from node A, it will propagate the data received to nodes C and D but not back to node A.
- For each transaction type (**real-time update**, **periodic sync** or **sync on demand**), the following action MUST take place when a node receives a HTTPS request with propagated data
  - If DRiP-Node-ID field value (in the HTTP header) contains Initiator node ID that has never been seen, both DRiP-Node-ID and DRiP-Node-Counter field values MUST be stored for future reference and the     data is propagated to all peer nodes.
  - If DRiP-Node-ID field value ((in the HTTP header) matches with a stored node ID and DRiP-Node-Counter-reset field value is **false**
    - The received data MUST be propagated to the peer nodes if DRiP-Node-Counter field value is greater than the saved counter value. The DRiP-Node-Counter field value MUST be saved as the new counter   for the stored node ID.
    - If DRiP-Node-Counter field value is less than or equal to saved counter value, then no action is necessary (data **MUST NOT** be propagated to peer nodes). This ensures that data propagation stops when all nodes have received data from the Initiator node.
  - If DRiP-Node-ID field value matches with a stored node ID and DRiP-Node-Counter-reset field value is **true**
    - The received data MUST be propagated to the peer nodes. The DRiP-Node-Counter field value MUST be saved as the new counter for the stored node ID.

### 4.4 Transaction Types

#### 4.4.1 Real-Time Updates

When the Initiator node has data to provision, the update is propagated to its peers immediately to provide timely updates of information. Section 3.5 details the steps involved in a real-time update.

##### 4.4.1.1 REST API

See section 4.5

##### 4.4.2.2 State Diagram

                                                            ---------------------------------------   
                                                           |                                       |
                                                           |          Real-Time Updates,           |
                                                           |          Synchronization              |
                                                   ________|_______   From Peer Nodes              |
            ________________                      |                |  (If data received is same or |
           |                |                     |                |   data already provisioned -  |
           |                |-------------------> | Waiting For    |   vote "NO".                  |
       --->| Waiting For    | (Real-Time Update,  | Response From  |   Otherwise, vote "YES".      |
      |    | Events         |   Start Timer)      | Peer Nodes     |<------------------------------ 
      |    |________________|                     |                |
      |                                           |                |
      |                                  ---------|                |---------
      |                                 |         |________________|         |
      |                         Timer   |                                    |Received Votes
      |                         Expired |                                    |From All Peer
      |                                 |                                    |Nodes
      |                                 |                                    |
      |                                 |          _______________           |
      |                                 |         |                |         |
      |                                  -------->|                |         |
      |                                           |                |         |
      |                                           |  Validating    |         |
      |                                           |  Votes         |         |
      |                                           |                |         |
      |     (If all Votes are "YES",              |                |<--------
      |      propagate commit)                    |                |
       -------------------------------------------|________________|
       
#### 4.4.2 Periodic Full Synchronization

For maximum reliability and validation of information contained in the distributed registry, periodically, a node SHOULD propagate all of the entries of information to its peers. In practice, this could be done in low traffic times or other convienient times that would have less chance to slow down any concurrent real-time updates being processed. The two phase commit described in Section 3.5 applies for full syncronization. **Conflict handling during full synchronization is yet to be done**.

##### 4.4.2.1 REST API

See section 4.5

#### 4.4.3 Full Synchronization On Demand

A node, either newly added to the mesh or re-activated after being out of service due to network issues or other anomalies, will inform its peer nodes to add this node to their list of peer nodes. The resulting action from the peer nodes is to start synchronizing data from their respective data stores. The **two phase commit does NOT apply here** as the contents of the nodes's data store is either outdated or empty. During this phase (HTTPS requests received will have DRiP-Sync-Complete field value set to **false**), this node **SHOULD NOT** become an Initiator node to provision data. While this transaction is going on, the peer nodes **MUST NOT** propagate **real-time updates** or "periodic full synchronization** transaction types. The next cycle of periodic syncronization will resolve discrepancy, if any,  in data contained in this node's data store.

##### 4.4.3.1 REST API

**PUT /addnode/node/:nodeid**


### 4.5 Two Phase Commit

#### 4.5.1 Voting Phase

The Initiator node sets a timeout period to get response from its peer nodes, for data propagated. This response (see section 4.5.1.2) will indicate a "vote" for whether the transaction can be completed based on any conflicting updates to the same entry. If all peer nodes vote "yes", then a commit of the information to the local node is initiated. If any one of the peer nodes votes "no" or if there is no response from one or more peer nodes, then the commit of the information MUST not be completed. No action is taken for responses (see section 4.5.1.2) received after the timeout period.

       ___           ___      ___           ___      ___           ___ 
      |DB |_________|DB |    |DB |_________|DB |    |DB |_________|DB |
      |___|         |___|    |___|         |___|    |___|         |___|
        |   Data      |        |   Data      |        |   Data      |  
        |   Store     |        |   Store     |        |   Store     |  
       _|_  Cluster  _|_      _|_  Cluster  _|_      _|_  Cluster  _|_ 
      |DB |_________|DB |    |DB |_________|DB |    |DB |_________|DB |
      |___|         |___|    |___|         |___|    |___|         |___|
          \                     \                      |
           \                     \                     |
           _\___    Vote(HTTPS)  _\___    Vote(HTTPS)  |____
          |Node |  <----------  |Node |  ---------->  |Node |
          |  B  |---------------|  A  |---------------|  C  |
          |_____|  ---------->  |_____|  <----------  |_____|
                     Yes/No                Yes/No     
                     
**Figure 3: Voting Phase**  

The peer nodes known to the originating node will continue propagate the information to their peer nodes and so on. However, these peer nodes will no longer need to keep track of the time interval for responses. A node will stop continuing to propagate information when it determines it has received the same information again. This can be determined by keeping track of the counter and originating node id.

##### 4.5.1.1 REST API

###### 4.5.1.1 POST /voting

**Example (using cURL)**
Request
```sh
$ curl -i  -H "Content-Type: application/json" -H "DRiP-Node-ID: nodeA" -H "DRiP-Node-Counter: 1234" -H "DRiP-Node-Counter-reset: false" -X POST -d '{"publicID":"+12155551212","service":"pstn","routingID":+12155551212@pstn.comcast.com, "timestamp":"1422980496943" }' https:// nodebregistry.com/voting
```
Response
```sh
HTTP/1.1 200 OK
```

###### 4.5.1.2 POST /votingphase/node/:nodeid/response/:response

**Example (using cURL)**
Request
```sh
$ curl -i -X POST http://nodearegistry.com/node/nodeA/response/yes
```
Response
```sh
HTTP/1.1 200 OK
```

#### 4.5.2 Commit Phase

The Initiator node upon receiving a "yes" from all the peer registries, the registry that originated the gossip will now commit the data in the HTTPS request payload to its data store. Subsequently, this information is propagated to all the nodes so that each node in the mesh will commit the same information in to their respective data stores.

Figure 4: Commit Phase 

       ___           ___                         ___           ___ 
      |DB |_________|DB |                       |DB |_________|DB |
      |___|         |___|                       |___|         |___|
        |   Data      |                           |   Data      |  
        |   Store     |                           |   Store     |  
       _|_  Cluster  _|_                         _|_  Cluster  _|_ 
      |DB |_________|DB |                       |DB |_________|DB |
      |___|         |___|                       |___|         |___|
                        \                       /
                         \COMMIT               /COMMIT
                         _\___     COMMIT   _ /__
                        |Node |------------|Node |
                        |  A  |    HTTPS   |  C  |
                        |_____|            |_____|
                            \H           H/
                             \T         T/
                        COMMIT\T       P/COMMIT 
                               \P     P/  
                                \S   S/   
                               __\__ /  COMMIT    _____  
                              |Node |------------|Node |
                              |  B  |   HTTPS    |  D  |
                              |_____|            |_____|
                             /                   /
                            /COMMIT             /COMMIT
          ___           ___/                  _/__          ___ 
         |DB |_________|DB |                 |DB |_________|DB |
         |___|         |___|                 |___|         |___|
           |   Data      |                     |   Data      |  
           |   Store     |                     |   Store     |  
          _|_  Cluster  _|_                   _|_  Cluster  _|_ 
         |DB |_________|DB |                 |DB |_________|DB |
         |___|         |___|                 |___|         |___|

      
Figure 4: Commit Phase

##### 4.5.2.1 REST API

###### 4.5.1.1 POST /commit

**Example (using cURL)**
Request
$ curl -i  -H "Content-Type: application/json" -H "DRiP-Node-ID: nodeA" -H "DRiP-Node-Counter: 1234" -H "DRiP-Node-Counter-reset: false" -X POST -d '{publicID:"+12155551212","service":"pstn","routingID":+12155551212@pstn.comcast.com, "timestamp":"1422980496943", public_key: ,signature: ZYNBbHC00VMZr2kZt6VmCvPonWJMGvQTBDqghoWeLxJfzB2a1pxAr3VgrB0SsSAaifsRdiOPoQZYOy2wrVghuhcsMbHWUSFxI6p6q5TOQXHMmz6uEo3svJsSH49thyGnFVcnyaZ++yRlBYYQTLqWzJ+KVhPKbfU/pryhVn9Yc6U=  }' https:// nodebregistry.com/commit
Response
HTTP/1.1 200 OK

## 5. Transport-Specific Guidelines

### 5.1 HTTPS

The specifics of secure communication is beyond the scope of this document. However, at a minimum, all nodes communicate via HTTPS and must contain CA signed certificates installed on them.

### 5.2 Authentication

The specifics of authentication is beyond the scope of this document.

## 6. Acknowledgements

## 7. References

**Author's Address**

Harsha Bellur

Comcast

One Comcast Center
   
Philadelphia, PA 19103
   
US

Email: harsha_bellur@cable.comcast.com


Chris Wendt (editor)

Comcast

One Comcast Center
   
Philadelphia, PA 19103
   
US

Email: chris-ietf@chriswendt.net