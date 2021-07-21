# Video: 

Tushar Roy https://www.youtube.com/watch?v=us0qySiUsGU&list=PLrmLmBdmIlps7GJJWW9I7N0P0rB0C3eY2&index=4(17:03)

# Google Auto-Complete System for searching text

## What the design does'nt cover
1. No spelling checks
2. No locale info
3. No Personal information.

## The main flows
Autocomplete design involves two flows - 
1. Request Flow: You get a request and look in your distributed trie to populate the auto-complete options. 
2. Data-collection flow: Append the query strings in your distributed trie.


## The Request Flow
1. Maintain a distributed trie as follows

                b 
            a         i
        t       l     t    l 
                l  s       l

2. We store top K (=2) words for every prefix in the tree. 
   For example: for prefix b, we store bat and bit. For ba - bat and bath. For bi - bit and bill
   By doing this we can quickly return the terms associated with a prefix.

3. Storing all words in a trie is neither durable nor scalable. Let's suppose it's 1998 and Google has just started. So it does'nt have that much data. So what we do is
    - Have a set of distributed Redis servers functioning as a cache
    - Set of servers - N1, N2, N3 that accept requests 
    - Set of database servers actually storing a Trie  - T1, T2, T3.
    - Zookeeper servers storing a mapping of the character range (viz. a-z: T1, T2, T3) with the mapping of databases who have the Trie's data.

4. Request Flow 
  - Request comes via the load balancer to a server N1. Let's say it's for the prefix 'ba'. 
  - Load balancer re-routes the request to Redis. There is no terms associated with the prefix 'ba' in Redis. 
  - N1 consults Zookeeper which returns the database server T1 that has the terms mapping to this prefix. 
  - Here, T1-T3 contain the same trie and Zookeeper maps a-z on T1, T2 and T3.
  - N2 then first writes these terms to Redis and then returns the response - bat and bath

5. Splitting the Trie across multiple nodes.
   - We introduce 3 more database servers T4, T5 and T6 
   - The range is now split as a-k (excluding k) is stored in T1 to T3 and k-$ is stored in T4 to T6 
   - If the range a-k becomes skewed, we can even split it further as a-bc (T1-T3) and bc-k(T8-T10)
   - You can go on splitting the ranges till the data can be stored properly in Zookeeper.

6. Zookeeper is good for reads and writing data (especially configuration information) of a low size. 

## Data collection flow
1. Our data collection flow is getting a stream of terms and their weights. How the weights are computed is'nt important.
2. There are 3-4 aggregator services which basically do the job of aggregating the terms (may be the terms are passed to a hash function and terms with same hash go to same aggregator). The aggregation happens over a certain time period (say hourly data) and dumps data to a table in Cassandra. The hourly data can be reduced to a daily data to give rise to the below table and it's values.
        
        Phrase  Time Period     Sum of Weights
        bat      Nov 1 2000          11015
        bat      Nov 2 2000          10000
        bat      Nov 3 2000          19000

3. Phrase having less weights are not going to show in auto-complete.
4. We are storing data with the time attached to show more recent search terms in auto-complete.
5. Appliers applying data to a Trie