# Chapter 5 - Replication
[Back to index](index.md)

## Distributed Data
Motivations for distributing data across multiple machines include **scalability** (data volume or read/write load scales beyond what a single machine can handle), **fault tolerance/high availability** and **latency** (keeping copies of the data geographically closer to users).

Distribution is achieved via **replication** (keeping a copy of the same data on multiple nodes) and **partitioning** (splitting a big dataset into smaller chunks that may be stored on different nodes). Although separate mechanisms, they often go hand in hand.

## Single Leader Replication
Difficulties with replication arise from the need to handle changes to the replicated data. On solution is **single leader replication** in which one node is designated as the leader. Writes can only be processed by the leader, which first updates its own local copy of the data before propagating changes to its read-only follower nodes as part of a replication log. Note that, for partitioned data, each partition may use a different node as its leader.

This mode of replication is used by many relational (PostgreSQL, MySQL) and non-relational (MongoDB, RethinkDB) databases as well as by message brokers such as Kafka.

Single leader replication is best suited for workloads consisting mostly of reads and very few writes. Capacity for serving read-only requests can be scaled up and down by increasing and decreasing the number of followers. However this is only feasible if replication is (at least partially) asynchronous. With fully synchronous replication, degradation of a single follower's response time (the likelihood of which increases with the number of followers) would make the whole system unavailable for writing.

### Follower Setup
New followers can be setup as follows:
  1. Take a snapshot of the leader's database at a specific point in time (corresponding to a known position in the replication log).
  2. Copy the snapshot to the new follower.
  3. The follower requests all data changes that have happened since the snapshot was taken.
  4. The follower is ready when it has caught up with the backlog of changes since the snapshot. 

### Follower Recovery
Each follower keeps track of the position in the replication log of the last transaction it processed. When a follower restarts after a crash, it requests from the leader and processes all the changes that occurred during the time is was disconnected until it has caught up with the leader.

### Failover After Leader Failure
Most systems use a timeout to determine whether the leader node has failed. A new leader must then be selected among the remaining nodes. While the best candidate is the node with most up-to-date changes (which minimizes data loss), getting the remaining nodes to agree on which one of them has the most up-to-date changes leads to a consensus problem. Once a new leader is selected, the system must be reconfigured so that writes are now sent to the new leader.

Many things can go wrong with failover, in particular:
  * The new leader may not have received all changes from the old leader before the latter went down. If the old leader gets comes back online, there may be conflicts between not yet replicated changes on the old leader and new changes on the new leader.
  * The above scenario becomes especially tricky if other storage systems are to be kept in sync with the database. In such case, it may not be possible to simply discard unreplicated changes from the old leader. See Github MySQL/Redis primary key sync issue page 157.
  
### Dealing with Replication Lag

#### Read Your Own Writes
As the name suggests, _read your own writes_ involves finding alternative mechanisms to serve users any data they may have submitted to the system rather than attempt to read said data from possibly stale replicas.

One way to implement _read your own writes_ is to ensure some data is always read from the leader. Users on social media platforms for instance can normally only modify their own profile. Hence, data from a user's own profile could be read from the leader while data from other profiles they browse could be read from followers. Unfortunately this is difficult to put in place for a geographically distributed user base, as some users will inevitably find themselves far from the leader node and experience a high latency.

#### Monotonic Reads
Monotonic reads are a guarantee that if a user makes multiple read queries in sequence, they will never be served older data after having previously been served newer data.

This can be implemented by ensuring that each user always reads from the same replica, for example by selecting replicas based on a hash of the user ID rather than randomly.

#### Consistent Prefix Reads
Consistent prefix reads guarantees that ordering will be preserved between a sequence of writes and the subsequent reading of those writes. In other words, writes will always be read in the order in which they were submitted.

This can be implemented by ensuring that causally related data is read from the same partition.

## Multi-Leader Replication
An alternative way to handle replicate data is multi-leader replication, in which multiple leaders may coexist, each simultaneously acting as a follower to the other leaders. While rarely used within a single data centre, multi-leader replication tends to be well-suited to systems in which data must be distributed across multiple data centres.
