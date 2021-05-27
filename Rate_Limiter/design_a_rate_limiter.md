# Design a distributed rate-limiter

# Sincerest Credits: 
- System Design - https://www.youtube.com/watch?v=FU4WlwfS3G0
- Detailed explanation of Rate Limiters - https://dzone.com/articles/detailed-explanation-of-guava-ratelimiters-throttl
- Rate Limiting System Design: https://www.youtube.com/watch?v=mhUQe4BKZXs(25:47)
- https://www.figma.com/blog/an-alternative-approach-to-rate-limiting/

## Introduction
All public APIs have rate limiters in them. May be Facebook, Twitter, Google, Amazon

## The Problem Definition
1. A popular web service experiences a traffic spike. 
2. Malicious client that tries to DDoS our service. One client uses exhorbitant amount of resources of our service due to which other clients start facing a high latency or higher rate of failed requests
3. Imagine you have a service that gives weather forecasts, machine learning algorithms or stock prices and you need to give access to developers on test them out on a trial basis before they purchase a full subscription
4. Few more scenarious
  - User experience: Avoid few users from bombarding your service with requests affecting other user's experience
  - Security: Avoid DDoS
  - Reduce operational cost

## Clarification Questions
1. Can we auto-scale to handle the high load?
   - Auto-scaling takes time and by the time it happens, the service already may have crashed
2. What about max connection on the load balancer and max threads count on the service end-point, do we still need rate limiting?

   - Load balancer will reject requests over the limit of max connection or forward them to a queue to be processed later. 
   - This solution is indiscriminate. 
   - Load balancer does not have knowledge of the cost of any operation so throttling may be required at the application level

3. Is this still a distributed system problem? What if we throttle on each host individually?
     - Load balancer will reject requests over the limit of max connection or forward them to a queue to be processed later. This becomes a single instance problem and no need for any distributed solution
     - Application servers throttle requests independently. Load balancers cannot distribute requests in an evenly manner. Different operatons cost differently. Each application server may become slow due to software failures or overheating
     - So application servers need to communicate with each other and maintain information about how many requests each one has processed so far.  

4. Can I assume that we have some publicly exposed REST APIs from the Web service for which we have to throttle their calls?
     - Of course, since it's a back-end web service you can definitely assume this. 
     - Admins can add rules through a management API
     - Each rule can have the API path for which it is supposed to be applied. 
     - We can let the load-balancer rate limit unauthenticated requests. 

## The Solution at a very brief
Use throttling. Throttling helps to limit the requests a client can send in a given amount of time. The purpose of throttling is to protect the system by restricting concurrent access or requests or restricting requests of a specified time window. 

### Different types of Rate Limiting
1. User based rate limiting: Allow a user to send only X-amount of requests in a given time
2. Concurrent rate limit: For a user, how many parallel sessions/connections are allowed. Limits DDoS attacks
3. Location based rate limiting: Useful whenever you're running a campaign for a location and you want to rate limit for all except this given location.
4. Server-based rate limiting: Rate limit different services that a server can offer


## Requirements

### Functional Requirements
Only one requirement. It will support just one kind of operation. This operation will return true if the request is not throttled or false other wise.

      boolean allowRequest(request)

### Non-functional requirments

1. Latency: Make the decision as quickly as possible on every request to the service.
2. Accurate: No throttling unless it's absolutely required
3. Scalable: If we should add more hosts to the rate limiter it should not be a problem.
4. If we don't know whether we should rate-limit or not, we should'nt
5. The rate-limiter should offer a seamless integration with other systems in the company.

## System design for a single host
### Components
1. Client request builder: Intercepts a client request and passes it to a Rate Limiter component. Client identification is an important part of the whole process
2. Rate Limiter Service: Checks the cache to see if there are any rules that can be applied to this request. If yes, the request is denied by using a 429 (Too many requests) status code. If not, the request is forwarded to the Request Processor
3. Rule Service: Two purposes 
   - If there are no rules in the cache, the Rate Limiter component issues a REST call to a rule service to populate the cache with rules and then serve the rules from cache only
   - It also has a PUT and POST API to modify and add new rules to the rules database.
4. Request processor handles all client requests those are not rate-limited

## Actual Algorithms to rate-limit

### Token Bucket algorithm With Periodic Refill
We will use a Token Bucket algorithm here. 
1. It's based on the analogy of a bucket filled with tokens
2. Bucket has 3 characteristics: 
    - Maximum amount of tokens it can hold
    - Amount of tokens currently available
    - Refill rate: Rate at which tokens are added to a bucket.
3. Every time a request comes, we take the token from the bucket. If there are no more tokens available, the next request is rejected. 
4. Bucket is refilled at a constant rate.

#### Sample code
```
# TokenBucket.java
# TokenBucket.java
package parser;

import Math;

public class TokenBucket {
  private final long maxBucketSize;
  private final long refillRate;

  private double currentBucketSize;
  private long lastRefillTimestamp;

  private TokenBucket(long maxBucketSize, long refillRate) {
    this.maxBucketSize = maxBucketSize;
    this.refillRate = refillRate;
    lastRefillTimestamp = System.nanoTime();
  }
  private void refill() {
    long now = System.nanoTime();
    double tokensToAdd = (now - lastRefillTimestamp) * refillRate / 1e9;
    this.currentBucketSize = min(this.maxBucketSize, this.currentBucketSize + tokensToAdd);
    this.lastRefillTimestamp = System.nanoTime();
  }

  public synchronized boolean allowRequest(int tokens) {
    if (currentBucketSize < this.maxBucketSize) {
      refill();
    }
    if (currentBucketSize > tokens) {
      currentBucketSize -= tokens;
      return true;
    }

    return false;
  }

}

```
#### Example execution

      0)    Time t0 = 100,  bucket was created.
            Max capacity = 10
            Refill capacity = 10 tokens/second


      1) Time t1 = t0 + 300 ms later allow request method was called as allowRequest(4)
         Since the bucket is full, we don't add any more tokens.
         Remaining tokens in bucket = 10 - 4 = 6 tokens


      2)  Time t2 = t1 + 200 ms later, allowRequest(5)
         i) Refill method adds = (500 ms - 100) * 10 / 1000 = 4000 / 1000 = 4 tokens
         ii) Remaining tokens in bucket = 6 + 4 - 5 = 5
         iii) lastRefillTimestamp = 500 ms

### Token bucket algorithm without refill

1. For each user, track the last time which request was made and the available tokens.
2. Every time request comes in, we fetch the token for a user (usually from a Redis key)
3. Update token, decrement the token count and update the last time when request was made.

#### Example execution
Only 5 tokens per minute are allowed

1. Request comes first at 11:01:00
    U1:     11:01:00       5

2. Request comes at 11:01:30
    U1:   11:01:30          4

3. Request comes at 11:01:30
    U1:   11:01:45          3

4. Request comes at 11:01:30
    U1:   11:01:46          2

5. Request comes at 11:01:30
    U1:   11:01:55          1

6. Request comes at 11:01:30 (BLOCK - Return 429)
    U1:   11:01:59          0

7. Request comes at 11:01:30 
    U1:   11:02:03          5

#### Disadvantages
1. May lead to a race condition in Distributed systems where in two instances of an application may try modify the record.


### Leaky bucket algorithm
1. This is equivalent to having a bucket with a fixed size N-requests which it can cater, consider N is 3.
2. So it can serve only 3 requests. If the 4th, 5th, 6th, 7th requests now come, they'll spill over.
3. Till the time 8th request comes, the request processor at the other end finished processing 1 so 8 can now get inserted into the bucket.
4. This is equivalent to a distributed queue of size N and it can smoothen out the burst traffic.

### Fixed window counter
1. Increment a counter on the basis of the incoming requests. This is same as the token bucket algorithm without refill only thing is we increment counter from 0 (initially)
2. One disadvantage is we can have more number of requests at the nearing end of the limit

### Sliding logs
1. Keep a key in redis as the username
2. For every request, add an entry to a list keyed at that array. Each request will also have a timestamp. 
3. Filter out all older entries less than the last minute.
4. If the count of the remaining entries is less than the allowed number of tokens per minute, then the system can take in additional requests. IF not, we reject any additional
5. Here the number of entries = Number of requests. Also if there are a million users in the system, we would need to store a million keys for each user containing a similar list of requests.

### Sliding Window counter.
1. This is a memory optimized solution over sliding logs.
2. Maintain a similar list to sliding logs.
3. In that list, maintain a count of how many requests arrive at that second along with the timestamp. Let's take an example.

#### Sliding Window counter
Imagine that the request rate is 10 requests per minute for U1

1. 1st request 11:01:01

    U1 = [(11:01:01, 01)]

2. 2nd request 11:01:01
U1 = [(11:01:01, 02)]

3. 4 requests come in at 11:01:15
 U1 = [(11:01:01, 02), (11:01:15, 4)]

 After receiving each request, we need to update this list and compute the last minute count of requests from this array. Delete any additional requests from this array before the last one minute. For example: In this case delete all requests <= 10:59:59 


#### Example execution
Only 5 tokens per minute are allowed

1. Request comes first at 11:01:00
    U1:     11:01:00       0

2. Request comes at 11:01:30
    U1:   11:01:30          1

3. Request comes at 11:01:30
    U1:   11:01:45          2

4. Request comes at 11:01:30
    U1:   11:01:46          3

5. Request comes at 11:01:30
    U1:   11:01:55          4

6. Request comes at 11:01:30 (BLOCK - Return 429)
    U1:   11:01:59          5

7. Request comes at 11:01:30 
    U1:   11:02:03          0

## Object-Oriented Design
### Interfaces
1. JobScheduler: Interface that schedules any job periodically and retrives rules from RuleService
2. RuleCache: Interface that stores the rules in memory.
3. Client identifier interface: Builds a unique ID that identifies every client.
4. RateLimiter: Interface is responsible for decision making.

### Classes
1. RuleJobScheduler: Implements JobScheduler interface and retrives rules from rule service. We can use ScheduledExecutor service for this. Calls RuleRetriever
2. TokenBucketCache: Implements RuleCache. Stores TokenBucket objects. Can uses a ConcurrentHashMap, Google Guava Cache.
3. ClientRequestBuilder: Maps a client to an identifier such as login username. Can even be an IP address
4. RuleRetriever: Retrives rule from database, creates token buckets and loads them into cache.

### How these interact with each other
1. RetrieveJobScheduler runs RetrieveRulesTask which makes a remote call to the RulesService. Creates token bucket and puts them in the cache.
2. When client request comes, RateLimiter makes a call to the ClientIdentifier builder to build a client identity, passes this key to the cache and retrives the bucket.
3. Last step is calling the allowRequest function on the Token Bucket.


### Stepping into the Distributed World
0. There are 3 hosts - Host A, Host B and Host C. And we need to allow 4 requests per second per client.
1. Each bucket should have 4 tokens initially. The reason being all buckets for the same bucket may in theory land on the same host. Load balancers try to distribute requests evenly, but they may not have an idea about keys and requests for same key may not be evenly distributed.
2. We add a load balancer.
    - First request goes to Host A. One token is used. 3 remain
    - Second requests go to Host C. One token is used. 3 remain.
    - Two other requests within the same second go to Host B. 2 tokens remain
    - All 4 requests hit the cluster. We must now throttle the requests within this second. 
    - Hosts should be allowed to communicate with each other to figure out how many tokens used.
        - Host A realizes that the other two hosts spent 3 (Host B: 2 tokens and Host C: 1 token). It removes 3 tokens available to it. It has 0 tokens 
        - Host B removes it's available tokens because it realizes that Host A and C consumed 1-1 token each
        - Host C removes it's all tokens. It realizes that A consumes 1 token and B consumes 2 tokens. It too removes all it's 3 tokens
        - All hosts now have 0 tokens available. 4 requests have been processed and no more requests allowed.

#### Some disadvantages
1. If more than 4 requests hit the cluster in a second, they may go through.
2. Because the communication between the two hosts may take time to agree on the number of tokens to be removed, additional requests > 4 may slip into the system.
    - At times our system processes more requests than it expects and we need to scale our systems accordingly.
    - Token bucket algorithm handles this use-case well if we extend the algorithm to allow negative number of tokens.


## How does each host share the number of tokens with each other? 

There are several different ways

### Network Mesh - Every host contacts every other host. 
1. Every host queries every other host for the information. This is similar to a Network Mesh topology
2. Host Discovery: 
    - We may use a 3rd party service that listens in to heartbeats coming from all registered hosts. Each host can contact this registry to find out with whom all it can talk with.
    - User provides a configuration file containing of IPs/hostnames of the hosts that it needs to contact with. This file will be deployed on all hosts.
3. Relatively simple to implement but does not scale. Number of messages grows quadratically w.r.t number of hosts in the cluster. We can support smaller clusters but not larger ones

### Gossip Protocol  (Used as Yahooo)
1. Protocol works in a way like epidemic spreads.
2. Implement this with a form of random peer selection.
3. With a frequency, each machine selects other machine at random and shares data

### Distributed Cache
1. Redis can be used here.
2. Our service cluster can scale out independently since the cache cluster is a separate cluster.
3. Can be shared amongst many different teams in the organization

### Coordination Service (Paxos and Raft)
1. Coordination service such as Zookeeper helps to choose the leader.
2. This helps to reduce number of messages broadcast in the network
3. Leaders asks each host the data, computes and calculates the final result and sends it.
4. Coordination service is a sophisticated service and it requires that there be only one leader
5. In case there are multiple leaders elected, each leader will compute the same result eventually. But this will have a messaging overhead.

### Host commmunication of protocols
We have two options TCP and UDP.
1. If we want rate limiting solution to be more accurate but having little bit of performance overhead, we can use TCP.
2. Less accurate solution that is more fast - UDP

## How to integrate all this?
Rate limiter can be run as a part of the service process or as a separate process (or even a micro-service) entirely
1. Part of service process: A common library integrated with the serivce code. Faster, no inter-process communication
2. We have two libraries - the daemon itself and the client that is responsible for inter-process communication between the service process and daemon. Programming language agnostic. Rate-limiter can be used written in an entirely different language. Rate limiter uses it's own memory space. Service memory does'nt need to allocate buckets for the rate limiter process. Service developers often come up with a lot of questions whenever they need to integrate with any other library. In this case, the service will not be impacted by any bugs in the Rate limiter library

## Some other questions
1. My service is insanely popular, does that mean I should keep millions of buckets in memory
  - Buckets will only be stored in memory as long as a client continues to send requests in an interval less than a second. 
  - If there are no requests coming for a client, we may remove this bucket. 
  - Bucket will be re-created when client makes a new request.

2. Failure modes
  - Rate-limiter daemon can fail. Less requests will be throttled in total.
  - Network partition can happen when several hosts in may not be able to broadcast requests to the rest of the group. Less requests throttled in total. If hosts talk to each other only 4 requests allowed across them. In this case, each host will accept 4 requests.
  - Service teams needs to have a tool to update, add and delete rules.

3. Synchronization needed at various places
  - Token Bucket class needs to have thread-safety. Atomic references can be used
  - Token bucket cache: Too many buckets and we want to deleted unused buckets and delete them when needed, we end up with synchronization. Concurrent Hash Map may be used.

4. What can clients do with throttled requests?
  - They can eithe retry them later or queue them.
  - To smartly retry, we use Exponential backoff and jitter. Exponential backoff retries requests several times but we wait longer with every retry. Jitter adds randomness to retry intervals. If we don't add jitter, exponential backoff retries requests at multiples of the same intervals of time.

5. The solution depends on number of hosts in the cluster, the request rate and the number of rules in the system. This can scale upto tens of thousands of hosts and Gossip protocol with UDP can give a consistent performance. For 10,000 hosts and above, Gossip will be expensive and so we can rely on a distributed caching system for our solution.