# system-design
New repository that contains solved system design problems and their resources. This will be from the various channels and Internet sources. Each question will be using the following template

# System_Design_template for interviews

**Sincerest courtesy:** https://towardsdatascience.com/the-complete-guide-to-the-system-design-interview-ba118f48bdfc

1. Requirements
    - Functional: What functionalities can the system or application provider to the user? Example: In Twitter, a user can follow another user
    - Non-functional: For any distributed system, the following are the fundamental concepts to consider
        - High availability: Most of the systems must be highly available.
        - Consistency: The systems with high availability will have eventual consistency. Any banking system favors consistency over availability as there cannot be discrepancies in data (account balance).
        - Reliability: No loss of user data
        - Latency: Response time of a user action such as loading a web page, liking a post, etc.

2. Data storage estimations based on back-of-the-envelope calculations:
    - Based on the data modality: A rough estimate of how much data must be stored — To know what type of database can be used and file storage for storing images/videos.
    - The number of requests to the service — To know how to scale the services. A service that is read-heavy can be scaled to handle high traffic of requests.
    - Read Write ratio — Determines whether the system is read-heavy or not.

3. Database Design: SQL Vs NoSQL and actual Schema

4. High-level system design
     - Very basic design: For showing showing just the clients, the Web server, the backend and the datastore as single components
     - Drill-down design using the following components
       - Replicate the services and databases- mention about a single point of failure.
	   - Load balancer — Application side & Database side if needed
	   - Message Queues — Tight coupling to loose coupling / Synchronous to Asynchronous communication
	   - Content Delivery Network - To avoid round trips to the main server (reduces latency)
	   - Distributed cache and client-side cache (For faster read access) 
	   - API Gateway
	   - Service discovery: to dynamically identify microservices
	   - Analytics Service:  For analyzing requests and user data
	   - ML Service: recommendation/ news feed ranking 
	   - Encryption