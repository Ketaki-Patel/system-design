# End-to-End Real Time Messaging System 

Why I chose this design to deep dive?
- This is such a versatile design requirement(i.e real time messaging) which makes backbone for very popular system design architectures like
   - real time chat service (e.g whatsapp, signal, telegram etc)
   - broad casting platform(e.g twitter)
   - notifications platform for social media network(i.e facebook, instagram), we can extend this design for SMS, Email and push notifications

- Goal is to design a distributed and scalable real-time notification system with multiple microservice instances and a distributed session store. The system efficiently handles both online and offline users.
- My intention is to build gradual understanding of how exactly websocket, Redis data store plus Redis Pub/Sub works in our architecture. So I explained it in two steps first at high level and then in detail once you get high level understanding. Hope you all enjoy reading it!


Lets first start with something simple for understading i.e only one server instance for each service and then in second diagram we will move to scalable and distributed archiecture i.e multi server instance architecture

<img width="761" height="895" alt="image" src="https://github.com/user-attachments/assets/48613434-e084-4e30-bb96-8e06260572e4" />

## Flow summary

- Notification created → published to SNS.

- SQS delivers the event to  Notification Service instances.

- The consuming instance stores it in the Notification DB, then publishes to Redis Pub/Sub channel "alerts".

- All WebSocket service instances subscribe to the `"alerts"` channel in Redis Pub/Sub. When a payload is published, each instance receives it due to the Pub/Sub fanout
  behavior. Each service instance then checks the centralized `live_sessions` keys in the Redis datastore to determine the user’s status:

   - **Online:** If the key exists in Redis, the user is online.
   - **Offline:** If the key does not exist, the user is offline.
   - **Push message:** If the WebSocket service instance hosts the user’s `WebSocketSession` locally (i.e., the user has an active WebSocket connection with this
     instance), the service immediately pushes the message through the open WebSocket connection.


- Online users (e.g., User1, User2) receive messagee instantly; offline users retrieve messages/notificaction from the DB when they reconnect.

## Our Goal - scalable architecture
Lets define what our scalable system will do, we will undestand with example, lets say user4 is going to interact with our system with intention to send
notification to user1, user2 and user3; user1 and user2 are online so they should get real time notification
we want to build distributed and scalable system.

 ### Sotware System Design
 
   <img width="890" height="762" alt="image" src="https://github.com/user-attachments/assets/16c62b9b-dfc5-42ec-97aa-beea76b05e11" />

## High level system work flow with architecture details

### 1️⃣ User Message/Notification Retrieval (WebSocket Service)

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

  - User4 interacts with system, Notification Generator sevice creates meessaage and decides whom to send that message, A message is sent for multiple users      (User1, User2, User3).

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
    



 ## Detailed work flow 
 
 - **Work flow part 1**:  websocket connection establishment and user session details (redis + in memory local storage) 

 <img width="688" height="658" alt="image" src="https://github.com/user-attachments/assets/465d5d13-5f12-4045-a3f4-ac65a2ac7e63" />

 Each WebSocket server instance keeps a thread-safe map of userId → WebSocketSession inside its own memory, because only that live object can write bytes back to the 
 connected client.

 - step 1. Client connects (first time)
   - Client (User1 or User2) opens a WebSocket to your service:
   ```bash
   GET ws://example.com/ws?userId=user1
   GET ws://example.com/ws?userId=user2
   ```
   - The load balancer routes the request to one of the WebSocket Service instances (1/2/3).
     Server accepts handshake; Spring creates WebSocketSession with session.getId() (e.g. sess-abc123).

   - That instance stores the mapping in distributed Redis:
    ```bash
   live_sessions:user1 → sess-abc123:WS1
   live_sessions:user2 → sess-def456:WS2  
    ```



 - **Work flow part 2** : Message Creation (message creation → delivery)
 This diagram starts with a new message being created and shows how it travels through SNS → SQS → Notification Service → Redis Pub/Sub → WebSocket instances → connected
 users.

   <img width="550" height="878" alt="image" src="https://github.com/user-attachments/assets/3d428387-a7c2-4403-a1bf-71fe966affbe" />

- step 2. Message Creation & Queueing

   - User4 triggers a notification destined for User1, User2, User3.
   - Notification Generator publishes to SNS, which fans out to SQS.

- step 3. Notification Processing

  - Any of the three Notification Service instances consumes the SQS message.
  - It saves the payload to Notification DB for persistence.

- step 4. Real-time Fan-out
  - The same Notification Service instance publishes the message JSON to the Redis channel “alerts”.

- step 5. WebSocket Delivery

  - All three WebSocket Service instances are subscribed to "alerts".
  - Each instance:
    - Reads the user list from the message.
    - Checks Redis if user key i.e live_sessions:<userId>.
    - If it finds a session with its own instance name (e.g., sess-abc123:Instance1) and holds that WebSocketSession locally, it calls:
      session.sendMessage(new TextMessage(payloadJson));
    - User1 and User2 (online) receive the message immediately.
    - User3 is offline, but the message is already stored in Notification DB for retrieval when they reconnect.
      
## Extension to this architecture - Notification Service Platform
  We can easily extend this real time message processing and make it notification service platform by supporting notification delivery channel like
  SMS, Email and Push Notificaton as well

<img width="941" height="806" alt="image" src="https://github.com/user-attachments/assets/def4caa0-368e-4f9f-94e2-bbcac6838ba8" />



   
      








 









