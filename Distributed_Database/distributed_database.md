Video: Tushar Roy - https://www.youtube.com/watch?v=rnZmdmlR-2M (9:33)
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