Video: https://www.youtube.com/watch?v=DUbEgNw-F9c(05:29)

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