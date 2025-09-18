###End-to-End Notification System (example.com)

We are implementing a distributed and scalable real-time notification system with multiple microservice instances and a distributed session store. The system efficiently handles both online and offline users. 

###Goal
Lets define what we are going to do user4 is going to interact with our system with intention to send
notification to user1, user2 and user3, user1 and user2 are online so they should get real time notificatin that is our goal 

##Details (what we are trying to build or expected behaviour of the system)

1️⃣ User Notification Retrieval (WebSocket Service)

Users (e.g., User1 and User2) connect to the WebSocket Service using a WebSocket client.

The WebSocket Service has three instances, deployed behind a load balancer.

Each WebSocket instance stores connection metadata in distributed Redis, keyed by userId, so online user sessions are discoverable across all instances.

When Users are online and log in to system following endpoint is called and it establish connectios to websocket service using websocket protocol(persistence connection)

 $ GET /ws?userId=<userId>


Example:

User1 and User2 are online and connected.

User3 is currently offline.










