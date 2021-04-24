# Design a URL shortener like bit.ly

# Credits: https://www.youtube.com/watch?v=JQDHz72OA3c (9:08)

## Introduction
All-in-all, this appears to have an 

1.  **App service:** Contains an API/code that will convert a URL to its shortened version and whenever the shortened URL is used, it converts it into its original form by looking up the key-value data

2. Key-Value Database: A key-value database that will save the shortened version of the URL. Key will be the shortened version of the URL and value will be its original version

However, the scale, the traffic everything is important.

## Few questions that could be asked

1. What is the length of the URL to be shortened?
2. What would be the traffic?
3. Should a single service be used or should we scale it? This would depend on the traffic

## Assumptions

Some assumptions about the traffic/users
1. Twitter has 300 million users/month. Assume we get 10% of that, that is 30 million users per month.
2. Lets keep the length of shortened URL to 7 characters.
3. After creation, the shortenend version of the URL will expire after 72 hours.

## Data capacity model

### Fields
We will have the following fields stored for a single URL

1. Long URL: 2 KB (2048 chars)
2. short URL: 17 bytes (17 chars. http://www.ly.com/_______  7 shortenend chars + 10 chars of website name) 
3. created_at: Epoch Timestamp 7 Bytes
4. expired at: Epoch Timestamp: 7 Bytes.

---------------------------------------
Total: 2.031 KB

### Storage required for 1 month, 1 year and 5 years
Let's assume that 30 M users each have a single URL saved in a single month. So the storage capacity would be

30 M users ---> 

    - 60.031 GBs per month
    - 0.7 TBs per year
    - 3.5 TBs per 5 years

The choice of whether to use an RDBMS/NoSQL depends on data access patterns.

## Core logic to generate a shortened URL

We have to generate a 7 char random ID for each URL viz. http://www.ly.com/XXXXXXX, where XXXXXXX is the unique ID. We have below choices of algorithms to be used here.

1. MD5: 
    - Give the long URL as input to MD5 generator.  Take a 7-character substring of the hash as the shortened version of the URL. 
    - MD5 may lead to collisions wherein two different long URLs have the same hash. The theoratical probablity of collision is if you generate 6 billion hashes per second, a collision can happen in 100 years. However, now its artificially possible to generate MD5 collisions (https://www.avira.com/en/blog/md5-the-broken-algorithm#:~:text=MD5%3A%20The%20fastest%20and%20shortest,%3A%201.47*10%2D29.&text=SHA256%3A%20The%20slowest%2C%20usually%2060,generated%20hash%20(32%20bytes). All we need is time, hardware and proper software.
    - Even if we don't get the entire hash collision, the first 7 characters of any generated hash can be same.

2. Base62: Base62 uses 62 characters (A-Z, a-z and 0-9). We get 62^7 ~ 3.5 trillion different combinations. If we generate 1000 URLs per second, the system will last to 110 years 

3. Base10: Base10 uses just 10 characters. we get just 62^10 ~ 10 million different combinations

We choose Base62 hashing scheme on the basis of the above arguments.

## Choice of database