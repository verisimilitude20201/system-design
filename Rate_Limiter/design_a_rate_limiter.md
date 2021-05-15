# Design a distributed rate-limiter

# Sincerest Credits: 
- System Design - https://www.youtube.com/watch?v=FU4WlwfS3G0 (16:22)
- Detailed explanation of Rate Limiters - https://dzone.com/articles/detailed-explanation-of-guava-ratelimiters-throttl

## Introduction
All public APIs have rate limiters in them. May be Facebook, Twitter, Google, Amazon

## The Problem Definition
1. A popular web service experiences a traffic spike. 
2. Malicious client that tries to DDoS our service. One client uses exhorbitant amount of resources of our service due to which other clients start facing a high latency or higher rate of failed requests

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

## The Solution at a very brief
Use throttling. Throttling helps to limit the requests a client can send in a given amount of time.

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

## Actual Algorithm to rate-limit
We will use a Token Bucket algorithm here. 
1. It's based on the analogy of a bucket filled with tokens
2. Bucket has 3 characteristics: 
    - Maximum amount of tokens it can hold
    - Amount of tokens currently available
    - Refill rate: Rate at which tokens are added to a bucket.
3. Every time a request comes, we take the token from the bucket. If there are no more tokens available, the next request is rejected. 
4. Bucket is refilled at a constant rate.


```
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
    this.maxBucketSize = min(this.maxBucketSize, this.maxBucketSize + tokensToAdd);
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
### Example execution

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

## Object-Oriented Design
### Interfaces
1. JobScheduler: Interface that schedules any job periodically and retrives rules from RuleService
2. RuleCache: Interface that stores the rules in memory.
3. Client identifier interface: Builds a unique ID that identifies every client.
4. RateLimiter: Interface is responsible for decision making.

### Classes
1. RuleJobScheduler: Implements JobScheduler interface and retrives rules from rule service. We can use ScheduledExecutor service for this. Calls RuleRetriever
2. TokenBucketCache: Implements RuleCache. Stores TokenBucket objects. Can uses a ConcurrentHashMap, Google Guava Cache.
3. ClientRequestBuilder: Maps a client to an identifier such as login. Can even be an IP address
4. RuleRetriever: Retrives rule from database, creates token buckets and loads them into cache.  
