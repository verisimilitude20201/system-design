# Design a distributed rate-limiter

# Sincerest Credits: 
- System Design - https://www.youtube.com/watch?v=FU4WlwfS3G0 (5:33)

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
     - Each application server may become slow due to software failures or overheating. So application servers need to communicate with each other and maintain information about how many requests each one has processed so far.  

## The Solution at a very brief
Use throttling. Throttling helps to limit the requests a client can send in a given amount of time