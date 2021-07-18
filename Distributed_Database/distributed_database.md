Video: Tushar Roy - https://www.youtube.com/watch?v=rnZmdmlR-2M
# Designing a distributed key-value database

## Characteristics

1. Durability: Cannot afford to loose customer data
2. Availability: Priotize this over performance.
3. Performance
4. Consistency Model: Strongly consistent. Read after most successful write will see the write. 
5. No ACID: No atomicity (Transactions) and no isolation (locking of tables )

## Operations
1. Create Table
2. Inserting/updating keys in a table
3. Getting the value of a key
4. Getting all keys in the table in a sorted order
5. Delete table (Option)
### Sequencer
The sequencer will be a unique number that will also be stored along with each key. It will be a combination of

    Nano_Timestamp (8 Bytes) + Unique Number(4 Bytes) + Unique Node ID(4 Bytes) = 16 bytes


### Create Table

            Stocks

        Name     Price      SeqId
        A        120         Seq_1
        B        130         Seq_2
        C        150         Seq_3

### Why sequencer
If we get two PUTs - PUT(D, 150) and PUT(D, 100) at the same time, the PUT which has the highest sequence number wins. 

## Architecture
1. Load balancers take in the request and forwards it to a Request Manager.
2. Request Manager consults a Meta-data manager to figure out which node owns the partition where this data will be ultimately sent.
3. Each node contains a replication group consisting of partitions of data (part of table.)
4. Replication group consists of 3 or 5 nodes for replication and one of them is the leader. The Request Manager communicates with the leader
5. Controller minutely observes all replication groups on each node for a skewed distribution of load. If there is, it splits the data again across multiple nodes.

### Meta-data manager
1. Holds a lot of data
2. Is highly consistent, has lots of reads but fewer writes.
3. Responsibilities
   - Leader election in each replication group.
   - Keep track of leaders.
   - Keep track of which part of table lies on which node.
4. We could use Zookeepker, etcd, Redis
5. We can even write our own Paxos or Raft consensus algorithms to ensure consensus
6. All data of Meta-data manager will be in-memory Request Manager. Changes to the meta-data manager will be propagated back to the Request manager for them to update their in-memory copies.
7. Extremely critical to back up it's data.

### Replication Group
1. Replication group is a collection of odd-number of nodes so as to achieve quorum. 
2. At max we can loose 2 replicas and still have the data
3. Inside every replication group, there is a leader and every request goes via that leader. So One leader and 2 followers.
4. On every PUT request, the leader in each replication group tries to ensure that the data gets updated on the maximum number of parts or partitions. Even if the leader goes down, at least one node will have the copy of data so that it can become the next leader.
5. Inside a node
    - Append only log where data gets appended because appends to a log are extremely cheap
    - For indexing, we use a B+ tree (Read heavy) or LSM tree (Write Heavy). 
    - Some data can be kept in memory. 
    - When a write comes, the leader appends it to the append only log and propagates it to both of its followers. When at least 2 (i.e. one leader and any one follower) write the data successfully, an ok is returned to client.

## Data request plane path
1. We have stocks table containing millions of records and we split it as 
        Stocks: A - L : Replication Group 1
        Stocks: L - Z: Replication Group 2
2. Request Put(Stocks, G, 100)
    - Load balancer routes it to one request manager
    - Request manager contains the above meta-data mapping. It sees that 'G' should go to Replication Group 1
    - It directs the request to the leader of RG1
    - The leader saves the data per the above mentioned process in point 5.
3. Similarly, PUT(Stocks, Y, 400) will go to RG2. 
4. GET(Stocks, Y) will again go to RG2
5. List(Stocks) will go to Request manager which knows that it should fetch data from 2 RGs. It starts scanning the first replication group and streams data to client, then it streams data from the next replication group. If the number of records are on the higher side, it can even page them in batches of 500s each.

## Controller Plane 

### Leader election
1. Based on the results of the leader election, Controller will elect a leader amongst the members of a replication group and update metadata manager.
2. Heartbeats are sent by the leader to Controller. If this heartbeats stops after a certain time, Controller initiates a new election. 
3. The present followers get caught up on the current state and participate in this election. Split brain problem can happen. 



## Error Scenarios

### Node unavailability 
4. Replication group can be unavailable for certain split seconds. In this database, we prefer consistency over availaibility
5. When follower starts lagging behind the leader, the controller realizes this. It takes another node from the pool of available nodes, assigns it to the leader, update meta-data manager and flags the former follower

### Splitting hot tables.
6. Splitting hot tables: Number of IOPs is very high. Controller finds out what table is responsible for the replication group to be huge in size. It also tries to figure out at what point should it split the table to balance. Controller first finds a replication group to hold say 30% of the data of the table and updates the data manager about the range of data that is assigned to the new replication group and what range is handled by the older replication group.

### Split-Brain - Two leaders
1. When a new leader gets elected by the controller, the old leader should stop assuming the role of leader. Certain times, a split brain does happen.
2. A new leader should confirm from all other nodes that it is a new leader. When all followers have accepted a node as their new leader, they should not accept PUT requests from any other node.
3. Consider a case where
    - Three nodes: A (leader) and two followers B and C.
    - Network partition happens and B and C cannot reach A. 
    - They initiate a new election and B becomes the new leader. C accepts that B is the new leader
    - Network partition resolves and A comes again and tries to sending writes to B and C.
    - B and C reject the writes. B inform A that it is the new leader. 
    - A now becomes a follower and accepts whatever writes that B sends it.

### No leader
1. A leader fails and for a split second, controller does not elect a new leader and the entire replication group is not available.
2. We can have the controller keep tab on the healthcheck returned by the leader and if it does'nt return healthcheck in 500ms, initiate new election.
3. Controller can also ensure that whatever stats are being captured and updated in meta-data manager.
4. A leader going down once in say 10-15 days is fine, but not fine if it goes down every few minutes or hours.

### Network split in meta-data managers / Outdated meta-data
1. This is also a set of services functioning as a meta-data manager in unison. 
2. Let's suppose that there are 5 nodes and 2 nodes are not reachable. Also if the leader is a part of the 3 nodes, these 2 nodes will not receive any meta-data updates and will have old meta-data.
3. If due to old meta-data, request gets redirected to the wrong replication group. It informs the meta-data manager that I'm not the owner of this partition, please check your meta-data. This meta-data manager node will try to connect with the leader and try to update it's meta-data. Assuming that by then the network partition gets resolved.

### Node goes down before applying changes to B-Tree/LSM-tree
1. We can replay the logs from the append-only logs to reapply the updates to the B-Tree/LSM tree after fixing the node.

### Incorrect meta-data in the meta-data manager
1. Replication group rejects incorrect data via consensus and so this point will be handled.

### Multiple node failures.
1. Have a 5-node replication group.
2. Controller can do rigorous checks on the system health of the nodes

## Storage Capacity
1. 5 nodes of 1 TB capacity each = 5 TB for 5 nodes.
2. Replication factor of 1000 = 5 PB of total data stored.
3. Limit on key value data size = 1 MB above which a blob storage should be used.
4. Overhead on storage of each key-value is 30 bytes of which 16 bytes are for the sequencer and 15 bytes for storing data in LSM tree/B+tree
5. IOPS per replication group = 2000
6. Metadata overhead per replication group = 30 bytes 
7. Metadata overhead per table = 30 bytes
8. RAM on request manager = 16 GB