Video: Tushar Roy - https://www.youtube.com/watch?v=rnZmdmlR-2M (20:22)
# Designing a distributed database

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