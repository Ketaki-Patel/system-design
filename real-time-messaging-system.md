# End-to-End Real Time Messaging System (example.com)

We are implementing a distributed and scalable real-time notification system with multiple microservice instances and a distributed session store. The       system efficiently handles both online and offline users. 


Lets first start with something simple for understading i.e only one server instance for each service and then in second diagram we will move to scalable and distributed archiecture i.e multi server instance architecture

<img width="761" height="895" alt="image" src="https://github.com/user-attachments/assets/48613434-e084-4e30-bb96-8e06260572e4" />

## Flow summary

- Notification created → published to SNS.

- SQS delivers the event to  Notification Service instances.

- The consuming instance stores it in the Notification DB, then publishes to Redis Pub/Sub channel "alerts".

- Every WebSocket Service instance subscribed to "alerts" receives the payload, checks the centralized live_sessions keys to verify if user is online or offline if key is   found in datstore user is online if not found user is offline and if websocket service hosts the user’s WebSocketSession locally, it immediately pushes the message down the open WebSocket connection.

- Online users (e.g., User1, User2) receive messagee instantly; offline users retrieve from the DB when they reconnect.

## Utlimate Goal - saclable architecture
Lets define what our scalable systtem will do, we will undestand with example, lets say user4 is going to interact with our system with intention to send
notification to user1, user2 and user3; user1 and user2 are online so they should get real time notification
we want to build distributed and scalable system.

 ### Sotware System Design
 
   <img width="890" height="762" alt="image" src="https://github.com/user-attachments/assets/16c62b9b-dfc5-42ec-97aa-beea76b05e11" />

## High level system work flow with architecture details

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
    



 ### Detailed work flow 
 
 - **Work flow part 1**:  websocket connection establishment and user session details (redis plus in memory local storage) 

 <img width="688" height="658" alt="image" src="https://github.com/user-attachments/assets/465d5d13-5f12-4045-a3f4-ac65a2ac7e63" />

 Each WebSocket server instance keeps a thread-safe map of userId → WebSocketSession inside its own memory, because only that live object can write bytes back to the 
 connected client.

 - step 1. Client connects (first time)
   - Client (User1 or User2) opens a WebSocket to your service:
   ```bash
   GET ws://example.com/ws?userId=user1
   GET /ws://example.com/ws?userId=user2
   ```
   - The load balancer routes the request to one of the WebSocket Service instances (1/2/3).
     Server accepts handshake; Spring creates WebSocketSession with session.getId() (e.g. sess-abc123).

   - That instance stores the mapping in distributed Redis:
    ```bash
   live_sessions:user1 → sess-abc123:WS1
   live_sessions:user2 → sess-def456:WS2  
    ```



 - Work flow part 2 : Message Creation (message creation → delivery)
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








 









