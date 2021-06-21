# Design Whatsapp

# Sincerest Credits
  - Gaurav Sen https://www.youtube.com/watch?v=vvhC64hQZMk(14:49)
  - Scaling Whatsapp to Millions of simulaneous connections https://vimeo.com/44312354
  - Whatsapp - The Architecture that Facebook bought http://highscalability.com/blog/2014/2/26/the-whatsapp-architecture-facebook-bought-for-19-billion.html
  - Code Karle https://www.youtube.com/watch?v=RjQjbJ2UJDg(15:26)

## Functional Requirements
1. Group Messaging
2. Sent / Delivered / Read receipts
3. Online / Last seen
4. Image Sharing
5. Chats are temporary / permanent.
6. One to one chat.

## Non-functional Requirements
1. Low latency / real-time
2. High availaibility
3. No Lags
4. Scale
   - 2 Billion users 
   - 1.6 billion users access it on a monthly basis
   - 65 Billion messages per day.


## How one-to-one messaging works
1. Your mobile devices connect to whatsapp cloud. 
2. They first connect to the gateway server, because whatsapp back-end microservices may be using a different protocol. Main reason being, you don't need those big-big HTTP headers and all and security.
3. Assume user A wants to send message to user B. If the gateways itself stores a mapping of which user connects to which gateway like 

        User    Gateway
        A         1
        B         2
 It will be easier to route the messages in this case.
4. It's a bit expensive to store this information, we want gateway servers to serve TCP connection requests only from mobiles. Secondly, this information is duplicated on all the 3 Gateway servers. Bit of high coupling. The connection should be dumb, it takes information and gives information.
5. We can have a sessions microservice store information about which user is connected to whom.
6. A calls a send_message(56) and passes the user ID of B i.e. 56. The dumb gateway routes this to the Session's micro-service which figures out that B is connected to Gateway 2 and sends the message there.
7. But you cannot do this, you cannot send a message from server to client, the server can only give responses. HTTP is a client-server protocol, the client gives requests, the server sends responses. There are ways to work-around this
    - Long polling: Client polls the server periodically say every 10-15 mins and server responds if there are any new messages for the client.
8. Long-polling won't be realtime and it will be expensive. We need some other protocol atop TCP i.e Web sockets. Web sockets allow peer-to-peer chat communication without any server in between.
9. Sent receipt is gotten by A when the message for B is successively sent to the Sessions service via Gateway 1. 
10. Delivery receipt is an acknowledgement by B's phone to the Gateway 2 once it receives A's message.
11. Read receipt is similar to delivery. So when B opens the chat app on his/her phone opens the chat tab, it sends another read receipt to the server via Gateway 2 so A knows it's message was read by B.
12. If a User A intends to send a message to user B, it sends it via the usual route and the Sessions micro-services saves that message. Consider user B's device is offline. So Gateway services cannot locate user B's device message does not get delivered there. When user B's device comes online, it connects to the gateway and periodically polls the sessions service to find out if there are any sent messages for it. The sent messages are delivered to it and the delivered/read acknowledgements are sent via the same channel.

## When A was online the last time B wants to know.
1. Whenever A does any activity, that timestamp needs to be persisted in a table tracking the timestamp and user A.
2. We can have a specific threshold say if A did an activity less than 30 seconds ago, the tag should say "online". Otherwise, there should be a user-friendly mapping such as "Last seen 5 mins ago, 1 hour ago and so on".
3. Client needs to differentiate between user activities and system-generated activities, may be via a flag in the request.
4. All user-generated activities will be tracked via  Last seen microservice which tracks the last time when the user was seen. in a table.


## Group messaging
1. We can have a separate Group service decoupled from the sessions service to 
  - Store the details of which all groups a given user is a part of.
  - User IDs of the other members of a group to which a user wants to send a message to.
2. Once the session service knows the user_ids for a group, it can then help relay messages to them. It can do this by using Websockets to relay these messages.
3. We basically will maintain a limit of users that can be a part of a group to reduce the fan-out of messages that need to be relayed to that many users.
4. The memory footprint on the Gateway servers should be as less as possible. So, the gateways will forward an un-parsed message to the Sessions service.
5. We can have a parser micro-service between the Gateway and the sessions service. This message cna be converted to an internal format (say Thrift may be) that may be used for internal consumption by the other services. 
6. Group service stores a mapping of group_id with the user_id. We can use consistent hashing to smartly route requests to fetch user_ids for a group from sessions service. 
7. We could use message queues to ensure that even if the group service fails, it can safely respond with user_ids when it's back up and the request is retried.
8. Group receipts (sent/delivered/read) are pretty expensive because every one on the group needs to do this action
9. Let's say user U1 wants to send a group message to group G1. The socket handler service routes it the message to Cassandra's messages table and writes it to a Kafka group topic. 
10. A Group message handler listens to this topic and queries Group service for the list of users belonging to this group. Once it knows which users are there, the remaining process is similar to as described in one-to-one messaging.

## Image uploading
1. Let's say user u1 wants to send an image to user u2. 
2. Via the socket handler service, it gets re-routed to an asset service. 
3. Before routing it, the socket handler service compresses the image. 
4. Asset service uploads the image to S3. Depending on the number of hits to the image, it can even store it in a CDN.
5. Via the same pipeline for one-to-one messaing, u2 is passed a message about that image. U2's device can then download that image. Once he sees that image, that image gets deleted from the server. 
6. We can make one optimization where the same image gets uploaded multiple times. The socket handler servers can send a hash of the image to asset service to check if its already available, if yes, don't upload it again. 


## Few tips and tricks
1. Facebook messager deprioritizes unimportant messages when there are special events like new year, christmas. 
2. Actually sending the message and the intended recipient is more important than showing Last seen or whether that user saw that message

## System Architecture

WebSockets are bidirectional connections built on top of TCP.

## Main components
1. Client devices: A set of one or more client devices that are used to message. 
2. Web Socket Handler servers: Keeps open connections with all active users which are connected. Each machine has 65534 active connections. Rougly one machine can handle 60,000 machines. These would be distributed across the globe because users in certain region would talk to web socket handlers closer to that region. Each Web socket handler will locally cache the number of users connected to it. It will also cache the conversations it has had recently. For example: if User 2 was connected to Web socket handler 2, Web socket handler 1 will cache it. This should have a very low TTL.
3. Web socket managers: Web socket manager servers interface with Redis that stores the details of which user is mapped to which socket handler. Any connection updates say a client disconnects from a Web socket server and reconnects to a different server 
4. Redis: Stores a mapping of Web_Socket_1 along with a set of which user_ids it is connected to and a mapping a which user_id is connected to which web socket handler.
5. Message Service: Repository of messages in the system. Fetch messages by message_id, fetch all messages by user_id and so on. It interfaces with Cassandra.
6. Cassandra: We use Cassandra over here because it's has good performance for writes. We can either store messages permanently to Cassandra like Facebook or delete them once they are successfully delivered like Whatsapp. User receipts will be stored in Cassandra against messages for a short time.

## Main services
1. User services: Owns user data - User profile, user id and user name. Redis cluster may cache this information.
2. Group Service: Which group has what all users. May be cached in Redis.
3. Analytics service: User's actions drive the analytics service. All events encompassing all user actions done by a user are sent to Analytics service which re-routes them to a Kafka topic. A Spark consumer could be listening to this analytics service and we could do message classification, tagging based on topics and so on. We can also infer top trending topics that the users are talking about. It's not possible if the users are end-to-end encrypted though. Spark consumer dump the data in HDFS
4. Last-Seen Service: Sits on the top of a Cassandra table storing the user_id and the timestamp the user's last activity was done. Cassandra has a high write throughput so using cassandra makes more sense here.

## Monitoring
Need to monitor
1. CPU utilization
2. Disk utilization of databases
3. Throughput of services
4. Kafka lags

Alerting can also be added if the metrics go beyond a certain threshold.

