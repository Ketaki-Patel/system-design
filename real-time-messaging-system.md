# End-to-End Real Time Messaging System (example.com)

We are implementing a distributed and scalable real-time notification system with multiple microservice instances and a distributed session store. The       system efficiently handles both online and offline users. 

## Goal
Lets define what we are going to do user4 is going to interact with our system with intention to send
notification to user1, user2 and user3, user1 and user2 are online so they should get real time notificatin that is our goal 

## Details (system work flow with architecture details)

### User Message/Notification Retrieval (WebSocket Service)

- Users (e.g., User1 and User2) connect to the WebSocket Service using a WebSocket client.

- The WebSocket Service has three instances, deployed behind a load balancer.

- Each WebSocket instance stores connection metadata in distributed Redis, keyed by userId, so online user sessions are discoverable across all instances.

- When Users are online and log in to system following endpoint is called and it establish connectios to websocket service using websocket  protocol(persistence connection)

```bash
 $ GET /ws?userId=<userId>
```

### Example:

- User1 and User2 are online and connected.

- User3 is currently offline.

### 2️⃣ Notification Publishing Flow

- Message Creation:

  - Notification Generator sevice creates meessaage and decides whom to send that message, A message is sent for multiple users (User1, User2, User3).

  - SNS / SQS Integration:
    
    - The message is published to an SNS topic.

    -  Notification Service instances (three instances) subscribe to the SNS topic via SQS.
 
- Notification Processing:

  -  Notification Persistence:

     - Notification Service consumes the message from SQS.

     - It persists the notification in a central Notification DB.

  -  Real-Time Delivery via Redis Pub/Sub:

     - The Notification Service publishes the message to a Redis channel, e.g., "alerts".

     -  WebSocket Service instances subscribe to the Redis "alerts" channel.

 - WebSocketService Push to Online Users using websocket protocol:

   - Redis listeners in each WebSocket Service instance receive the message.

   - Each WebSocket instance checks distributed Redis session store to determine which users are online on that instance.

     - For online users (User1, User2), the message is pushed in real-time via the WebSocket protocol.

     - Offline users (User3) will have their messages stored in the Notification DB for retrieval when they reconnect.
    
 ### Sotware System Design
 
   <img width="890" height="762" alt="image" src="https://github.com/user-attachments/assets/16c62b9b-dfc5-42ec-97aa-beea76b05e11" />












