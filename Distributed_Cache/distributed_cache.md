Video: https://www.youtube.com/watch?v=DUbEgNw-F9c(24:00)

# Designing a Distributed Cache

## Introduction
1. Caching is high-speed data storage system saving transient data. Further requests won't hit main memory or won't redo computations.
2. Data is stored in faster access hardware such as RAM.

## Best practises for caching
1. Data validity: How long will a data be stored.
2. High Hitrate: Data was cached in the cache system and served from cache.
3. Cache Miss
4. TTL: Proper TTL needed.

## Features/Estimation
1. Terabytes of data.
2. Expectation 50K to 1M QPS
3. 1ms latency to get or set data
4. 100% availability.
5. Scalable.

## Cache Access Patterns
1. Write through Cache: Firstly the data is written in cache and then from there it goes to the DB. The process writing the data returns successfully whenever the data gets written successfully both to cache and DB.
2. Write Around cache: Write request goes first to the DB and once it gets returned sucessfully the application returns. On read, first the cache is checked and then if not found in cache, the data is fetched from DB, stored in cache and then returned from cache.
3. Write Back: Data first gets written to cache and process returns success if it gets written. There will be one more service that syncs writes from cache to DB asyncrhonously. 

## Designing a Cache - What data structure to use
1. We need a hash function, a list of buckets (10-buckets) and a hashing function
For example to store THOR, applying a hash function gives value 53, mod it by the size of the bucket array gives 5.
To save ANTMAN, applying a hash function gives value 93, mod that by side of the bucket array gives 3. This is a collision
2. To avoid collisions there are a couple of ways
  - Separate chaining: Have a linked list and maintain a list of colliding entries in a linked fashion. We would need to traverse the linkedlist to find the value.

### Hash Table Schematic

      ThOR --> 53 ---> 3
      Hulk --> 51 ---> 1
      Flash --> 56  --> 6


      0  
      1  -> Hulk
      2
      3  -> Thpr
      4
      5
      6  -> Flash


## Cache eviction: 
1. Basically means to evict the key-value entry from the cache.
2. Best strategy is least recently used. Delete all entries which have not been used recently.

### Approaches for Cache eviction

#### Least Recently Used: 
The least recently used objects are removed. It does'nt count the frequency of access of the key. There is a count-min-sketch data structure approach that can be used for this.

1. We can store an object of last accessed times for each value 

      ThOR --> 53 ---> 3
      Hulk --> 51 ---> 1
      Flash --> 56  --> 6


      0  
      1  -> (value=Hulk, last_accessed_time=12/02/2021 09:00 PM) 
      2
      3  -> Thor
      4
      5
      6  -> Flash

This is an O(n) strategy.

2. We will use a doubly-linked list to store actual value and the address of the value containing the node will be in the bucket array

      ThOR --> 53 ---> 3
      Hulk --> 51 ---> 1
      Flash --> 56  --> 6

      100       101        102
      Thor <->  Hulk  <-> Flash 

      0  
      1  -> 101
      2
      3  -> 100
      4
      5
      6  -> 102

  The nodes at the front of the list was least recently accessed. Append all accessed nodes at the end of the linked_list. If it's present leftwards towards the start of list, delete it and append it to the back.