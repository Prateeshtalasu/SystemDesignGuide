# 🔄 Real-Time Communication: Long Polling vs WebSockets vs SSE

## 0️⃣ Prerequisites

Before diving into real-time communication patterns, you should understand:

- **HTTP Request-Response Model**: Traditional HTTP is a pull-based protocol. Client sends a request, server sends a response, connection closes (or stays idle with keep-alive). Server cannot initiate communication.

- **TCP Connections**: All these patterns run over TCP. Creating a TCP connection requires a 3-way handshake (1.5 RTT). Keeping connections open has memory and resource costs.

- **Stateless vs Stateful**: HTTP is designed to be stateless. Each request is independent. Real-time communication requires maintaining state (who is connected, what they're subscribed to).

- **Latency**: The time between an event occurring and the client receiving it. For real-time applications, we want this as low as possible (ideally under 100ms for interactive apps).

---

## 1️⃣ What Problem Does Real-Time Communication Solve?

### The Specific Pain Point

Traditional HTTP is **pull-based**: the client asks, the server responds. But many applications need **push-based** communication: the server needs to notify clients when something happens.

**The Problem**: How does a server tell a client "Hey, you have a new message!" without the client constantly asking "Do I have a new message? Do I have a new message? Do I have a new message?"

### What Systems Looked Like Before Real-Time Solutions

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    NAIVE POLLING (The Bad Old Days)                          │
└─────────────────────────────────────────────────────────────────────────────┘

Client                                                    Server
   │                                                         │
   │  GET /messages?since=100 ──────────────────────────────>│
   │  <────────────────────────────────────── 200 OK (empty) │
   │                                                         │
   │  [Wait 5 seconds]                                       │
   │                                                         │
   │  GET /messages?since=100 ──────────────────────────────>│
   │  <────────────────────────────────────── 200 OK (empty) │
   │                                                         │
   │  [Wait 5 seconds]                                       │
   │                                                         │
   │  GET /messages?since=100 ──────────────────────────────>│
   │  <────────────────────────────────────── 200 OK (empty) │
   │                                                         │
   │  ... 100 more empty responses ...                       │
   │                                                         │
   │  GET /messages?since=100 ──────────────────────────────>│
   │  <──────────────────────────── 200 OK (1 new message!)  │

Problem: 99% of requests return nothing. Massive waste of resources.
```

### What Breaks Without Real-Time Solutions

**Without Real-Time Communication**:
- Chat apps would feel laggy (polling every 5 seconds = 5 second delay)
- Stock tickers would show stale prices
- Collaborative editing would be impossible
- Notifications would be delayed
- Online games would be unplayable
- Live dashboards would require manual refresh

### Real Examples of the Problem

**Example 1: Facebook Messenger (2008)**
Before WebSockets, Facebook used long polling for chat. Each user maintained an open HTTP connection. With 500 million users, this was a massive infrastructure challenge.

**Example 2: Trading Platforms**
Stock prices change multiple times per second. Polling every second would still miss updates and waste bandwidth. Real-time push is essential.

**Example 3: Collaborative Tools (Google Docs)**
Multiple users editing the same document need instant updates. Even 1-second delay makes collaboration frustrating.

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Analogy

**Regular HTTP Polling** is like calling a restaurant every 5 minutes to ask "Is my table ready?"
- Annoying for both you and the restaurant
- You might miss the table if you call at the wrong time
- Wastes everyone's time

**Long Polling** is like calling and saying "I'll stay on the line until my table is ready."
- Restaurant keeps you on hold
- As soon as table is ready, they tell you
- You hang up, then call back to wait again
- Better, but still awkward

**Server-Sent Events (SSE)** is like the restaurant having a PA system.
- You give them your number once
- They announce "Table for Smith is ready!" over the PA
- One-way communication (restaurant → you)
- Simple and effective for notifications

**WebSockets** is like having a walkie-talkie with the restaurant.
- You establish a channel once
- Either side can talk anytime
- Two-way, instant communication
- Perfect for conversations (chat, gaming)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  COMMUNICATION PATTERN COMPARISON                            │
└─────────────────────────────────────────────────────────────────────────────┘

Regular Polling:     Client ──────> Server    (request)
                     Client <────── Server    (response, maybe empty)
                     [repeat every N seconds]

Long Polling:        Client ──────> Server    (request, held open)
                     [server waits for data...]
                     Client <────── Server    (response when data ready)
                     [client immediately reconnects]

SSE:                 Client ──────> Server    (subscribe once)
                     Client <══════ Server    (stream of events)
                     Client <══════ Server    (more events)
                     Client <══════ Server    (keeps flowing)

WebSocket:           Client <═════> Server    (bidirectional channel)
                     Client ══════> Server    (client can send)
                     Client <══════ Server    (server can send)
                     [either side, anytime]
```

---

## 3️⃣ How Each Pattern Works Internally

### Long Polling

**Concept**: Client makes a request, server holds it open until data is available or timeout occurs.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LONG POLLING FLOW                                    │
└─────────────────────────────────────────────────────────────────────────────┘

Client                                                    Server
   │                                                         │
   │  GET /events?timeout=30s ──────────────────────────────>│
   │                                                         │
   │  [Server holds connection open]                         │
   │  [Waiting for events...]                                │
   │                                                         │
   │         ... 15 seconds pass ...                         │
   │                                                         │
   │  [Event occurs on server!]                              │
   │                                                         │
   │  <────────────────────────────── 200 OK + Event Data    │
   │                                                         │
   │  [Client processes event]                               │
   │                                                         │
   │  GET /events?timeout=30s ──────────────────────────────>│
   │  [Immediately reconnects]                               │
   │                                                         │
   │  [Server holds connection open again...]                │
   │                                                         │
```

**How Server Holds Connection**:

```java
// Servlet 3.0+ Async Support
@WebServlet(urlPatterns = "/events", asyncSupported = true)
public class LongPollingServlet extends HttpServlet {
    
    // Queue of waiting clients
    private final Queue<AsyncContext> waitingClients = 
        new ConcurrentLinkedQueue<>();
    
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        // Start async processing (don't block servlet thread)
        AsyncContext asyncContext = req.startAsync();
        asyncContext.setTimeout(30000); // 30 second timeout
        
        // Add to waiting queue
        waitingClients.add(asyncContext);
        
        // Handle timeout
        asyncContext.addListener(new AsyncListener() {
            @Override
            public void onTimeout(AsyncEvent event) {
                // Send empty response on timeout
                sendResponse(asyncContext, "[]");
                waitingClients.remove(asyncContext);
            }
            // ... other listener methods
        });
    }
    
    // Called when event occurs
    public void broadcastEvent(String eventData) {
        AsyncContext client;
        while ((client = waitingClients.poll()) != null) {
            sendResponse(client, eventData);
        }
    }
}
```

**Pros**:
- Works with any HTTP infrastructure (proxies, load balancers)
- Simple to implement
- No special protocol needed

**Cons**:
- Connection overhead for each poll cycle
- Server must track waiting connections
- Not truly real-time (small delays on reconnection)
- Scaling challenge: many open connections

### Server-Sent Events (SSE)

**Concept**: Client opens a persistent HTTP connection. Server sends events as they occur. One-way: server to client only.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SSE CONNECTION FLOW                                  │
└─────────────────────────────────────────────────────────────────────────────┘

Client                                                    Server
   │                                                         │
   │  GET /events                                            │
   │  Accept: text/event-stream ────────────────────────────>│
   │                                                         │
   │  HTTP/1.1 200 OK                                        │
   │  Content-Type: text/event-stream                        │
   │  Cache-Control: no-cache                                │
   │  Connection: keep-alive                                 │
   │  <──────────────────────────────────────────────────────│
   │                                                         │
   │  data: {"type": "connected"}                            │
   │  <══════════════════════════════════════════════════════│
   │                                                         │
   │  [Connection stays open]                                │
   │                                                         │
   │  event: message                                         │
   │  data: {"user": "Alice", "text": "Hello!"}              │
   │  <══════════════════════════════════════════════════════│
   │                                                         │
   │  event: message                                         │
   │  data: {"user": "Bob", "text": "Hi there!"}             │
   │  <══════════════════════════════════════════════════════│
   │                                                         │
   │  [Events keep flowing as they occur...]                 │
```

**SSE Message Format**:

```
event: eventName
id: 12345
retry: 3000
data: {"key": "value"}
data: More data on multiple lines

```

- `event`: Optional event type (client can listen for specific types)
- `id`: Optional event ID (for resuming after disconnect)
- `retry`: Reconnection time in milliseconds
- `data`: The actual payload (can span multiple lines)
- Empty line marks end of event

**How SSE Works Internally**:

```java
// Spring WebFlux SSE
@RestController
public class SSEController {
    
    private final Sinks.Many<ServerSentEvent<String>> sink = 
        Sinks.many().multicast().onBackpressureBuffer();
    
    @GetMapping(path = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> streamEvents() {
        return sink.asFlux();
    }
    
    // Called when event occurs
    public void publishEvent(String eventType, String data) {
        ServerSentEvent<String> event = ServerSentEvent.<String>builder()
            .id(UUID.randomUUID().toString())
            .event(eventType)
            .data(data)
            .retry(Duration.ofSeconds(3))
            .build();
        
        sink.tryEmitNext(event);
    }
}
```

**Client-Side (JavaScript)**:

```javascript
const eventSource = new EventSource('/events');

// Listen for all events
eventSource.onmessage = (event) => {
    console.log('Received:', event.data);
};

// Listen for specific event types
eventSource.addEventListener('message', (event) => {
    const message = JSON.parse(event.data);
    console.log(`${message.user}: ${message.text}`);
});

// Handle connection events
eventSource.onopen = () => console.log('Connected');
eventSource.onerror = () => console.log('Error/Reconnecting...');
```

**Pros**:
- Simple protocol (just HTTP with streaming)
- Automatic reconnection (browser handles it)
- Event IDs allow resuming from last event
- Works with HTTP/2 (efficient multiplexing)

**Cons**:
- One-way only (server to client)
- Limited browser connections per domain (6 in HTTP/1.1)
- No binary data (text only, need base64 for binary)
- Some proxy issues (may buffer or timeout)

### WebSockets

**Concept**: Upgrade HTTP connection to a persistent, bidirectional channel. Either side can send messages anytime.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      WEBSOCKET HANDSHAKE                                     │
└─────────────────────────────────────────────────────────────────────────────┘

Client                                                    Server
   │                                                         │
   │  GET /chat HTTP/1.1                                     │
   │  Host: example.com                                      │
   │  Upgrade: websocket                                     │
   │  Connection: Upgrade                                    │
   │  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==           │
   │  Sec-WebSocket-Version: 13                              │
   │  ──────────────────────────────────────────────────────>│
   │                                                         │
   │  HTTP/1.1 101 Switching Protocols                       │
   │  Upgrade: websocket                                     │
   │  Connection: Upgrade                                    │
   │  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=    │
   │  <──────────────────────────────────────────────────────│
   │                                                         │
   │  ═══════════════════════════════════════════════════════│
   │            WebSocket Connection Established             │
   │  ═══════════════════════════════════════════════════════│
   │                                                         │
   │  [Binary Frame: "Hello Server!"] ══════════════════════>│
   │                                                         │
   │  <══════════════════════ [Binary Frame: "Hello Client!"]│
   │                                                         │
   │  [Either side can send anytime...]                      │
```

**WebSocket Frame Structure**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      WEBSOCKET FRAME FORMAT                                  │
└─────────────────────────────────────────────────────────────────────────────┘

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+

Opcodes:
- 0x0: Continuation frame
- 0x1: Text frame
- 0x2: Binary frame
- 0x8: Connection close
- 0x9: Ping
- 0xA: Pong
```

**How WebSocket Works Internally**:

```java
// Spring WebSocket Handler
@Component
public class ChatWebSocketHandler extends TextWebSocketHandler {
    
    private final Set<WebSocketSession> sessions = 
        ConcurrentHashMap.newKeySet();
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        sessions.add(session);
        System.out.println("Client connected: " + session.getId());
    }
    
    @Override
    protected void handleTextMessage(WebSocketSession session, 
                                     TextMessage message) throws Exception {
        String payload = message.getPayload();
        System.out.println("Received: " + payload);
        
        // Broadcast to all connected clients
        for (WebSocketSession s : sessions) {
            if (s.isOpen()) {
                s.sendMessage(new TextMessage("Echo: " + payload));
            }
        }
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, 
                                      CloseStatus status) {
        sessions.remove(session);
        System.out.println("Client disconnected: " + session.getId());
    }
    
    @Override
    public void handleTransportError(WebSocketSession session, 
                                     Throwable exception) {
        System.err.println("Error: " + exception.getMessage());
        sessions.remove(session);
    }
}
```

**WebSocket Configuration**:

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    
    @Autowired
    private ChatWebSocketHandler chatHandler;
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatHandler, "/chat")
            .setAllowedOrigins("*");  // Configure CORS
    }
}
```

**Client-Side (JavaScript)**:

```javascript
const socket = new WebSocket('ws://example.com/chat');

socket.onopen = () => {
    console.log('Connected');
    socket.send(JSON.stringify({ type: 'join', user: 'Alice' }));
};

socket.onmessage = (event) => {
    const message = JSON.parse(event.data);
    console.log('Received:', message);
};

socket.onclose = (event) => {
    console.log('Disconnected:', event.code, event.reason);
};

socket.onerror = (error) => {
    console.error('Error:', error);
};

// Send message
function sendMessage(text) {
    if (socket.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify({ type: 'message', text: text }));
    }
}
```

**Pros**:
- True bidirectional communication
- Low latency (no HTTP overhead after handshake)
- Binary and text support
- Full-duplex (both sides can send simultaneously)

**Cons**:
- More complex to implement
- Stateful (harder to scale)
- Some proxies/firewalls block WebSocket
- No automatic reconnection (must implement)
- No built-in message acknowledgment

---

## 4️⃣ Simulation: Building a Chat Application

Let's trace how each pattern would handle a chat message.

### Long Polling Chat

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LONG POLLING CHAT FLOW                                    │
└─────────────────────────────────────────────────────────────────────────────┘

Alice (Browser)          Server              Bob (Browser)
      │                     │                     │
      │ GET /messages ─────>│                     │
      │ (waiting...)        │<───── GET /messages │
      │                     │       (waiting...)  │
      │                     │                     │
      │ POST /send ────────>│                     │
      │ {"text":"Hello!"}   │                     │
      │                     │                     │
      │<── 200 OK ──────────│                     │
      │                     │                     │
      │                     │──── 200 OK ────────>│
      │                     │ [{"text":"Hello!"}] │
      │                     │                     │
      │                     │<───── GET /messages │
      │                     │       (reconnect)   │
      │                     │                     │

Timeline:
- Alice sends message: instant
- Bob receives: depends on when his poll was active
- Worst case delay: poll interval (e.g., 30 seconds)
- Best case: instant (if poll was waiting)
```

### SSE Chat (with separate POST for sending)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SSE CHAT FLOW                                        │
└─────────────────────────────────────────────────────────────────────────────┘

Alice (Browser)          Server              Bob (Browser)
      │                     │                     │
      │ GET /events ═══════>│<═══════ GET /events │
      │ (SSE stream open)   │    (SSE stream open)│
      │                     │                     │
      │ POST /send ────────>│                     │
      │ {"text":"Hello!"}   │                     │
      │                     │                     │
      │<── 200 OK ──────────│                     │
      │                     │                     │
      │<══ event: message ══│══ event: message ══>│
      │ data: {"text":"Hello!", "user":"Alice"}   │
      │                     │                     │

Timeline:
- Alice sends message: instant (POST)
- Bob receives: instant (SSE push)
- Both Alice and Bob see the message via SSE
- Latency: ~RTT (typically <100ms)
```

### WebSocket Chat

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      WEBSOCKET CHAT FLOW                                     │
└─────────────────────────────────────────────────────────────────────────────┘

Alice (Browser)          Server              Bob (Browser)
      │                     │                     │
      │ WS Connect ════════>│<════════ WS Connect │
      │ (bidirectional)     │     (bidirectional) │
      │                     │                     │
      │═══ {"type":"msg", ═>│                     │
      │     "text":"Hello!"}│                     │
      │                     │                     │
      │<══ {"type":"msg", ══│══ {"type":"msg", ══>│
      │     "text":"Hello!",│    "text":"Hello!", │
      │     "user":"Alice"} │    "user":"Alice"}  │
      │                     │                     │

Timeline:
- Alice sends message: instant (WebSocket frame)
- Server broadcasts: instant
- Bob receives: instant
- Latency: ~RTT/2 (no request needed, just push)
```

---

## 5️⃣ How Engineers Use These Patterns in Production

### Real-World Usage

**Slack (Chat Application)**
- Uses WebSockets for real-time messaging
- Falls back to long polling when WebSocket fails
- Maintains millions of concurrent connections
- Reference: [Slack Engineering Blog](https://slack.engineering/)

**Twitter (Live Feed)**
- Uses SSE for streaming API
- `GET /2/tweets/search/stream` returns SSE stream
- Efficient for one-way notification streams

**Uber (Driver Location)**
- Uses WebSockets for real-time location updates
- Bidirectional: driver sends location, receives ride requests
- Millions of concurrent connections globally

**Robinhood (Stock Prices)**
- Uses WebSockets for real-time price updates
- Prices change multiple times per second
- SSE would work but WebSocket allows user actions too

### Production Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PRODUCTION WEBSOCKET ARCHITECTURE                         │
└─────────────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
                         │   Load Balancer │
                         │  (L7, sticky)   │
                         └────────┬────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
            ▼                     ▼                     ▼
    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
    │  WS Server 1  │    │  WS Server 2  │    │  WS Server 3  │
    │  (10K conns)  │    │  (10K conns)  │    │  (10K conns)  │
    └───────┬───────┘    └───────┬───────┘    └───────┬───────┘
            │                     │                     │
            └─────────────────────┼─────────────────────┘
                                  │
                         ┌────────▼────────┐
                         │   Redis Pub/Sub │
                         │  (Message Bus)  │
                         └─────────────────┘

How it works:
1. User connects to one WS server (sticky session via load balancer)
2. User sends message to their connected server
3. Server publishes to Redis Pub/Sub
4. All servers receive message from Redis
5. Each server sends to their connected clients
```

### Scaling Considerations

```java
// Redis Pub/Sub for cross-server messaging
@Component
public class RedisPubSubBridge {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Autowired
    private ChatWebSocketHandler webSocketHandler;
    
    // Subscribe to Redis channel
    @PostConstruct
    public void subscribe() {
        redisTemplate.execute((RedisCallback<Void>) connection -> {
            connection.subscribe((message, pattern) -> {
                String payload = new String(message.getBody());
                webSocketHandler.broadcastToLocal(payload);
            }, "chat:messages".getBytes());
            return null;
        });
    }
    
    // Publish message to Redis (all servers will receive)
    public void publishMessage(String message) {
        redisTemplate.convertAndSend("chat:messages", message);
    }
}
```

---

## 6️⃣ Complete Implementation Examples

### Long Polling Server (Spring Boot)

```java
// pom.xml dependencies
/*
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
*/

import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.async.DeferredResult;
import java.util.*;
import java.util.concurrent.*;

@RestController
@RequestMapping("/api")
public class LongPollingController {
    
    // Queue of messages
    private final BlockingQueue<Message> messageQueue = 
        new LinkedBlockingQueue<>();
    
    // Waiting clients
    private final List<DeferredResult<List<Message>>> waitingClients = 
        Collections.synchronizedList(new ArrayList<>());
    
    /**
     * Long polling endpoint.
     * Client calls this and waits for messages.
     */
    @GetMapping("/messages")
    public DeferredResult<List<Message>> getMessages(
            @RequestParam(defaultValue = "30000") long timeout) {
        
        DeferredResult<List<Message>> result = new DeferredResult<>(timeout);
        
        // Handle timeout
        result.onTimeout(() -> {
            waitingClients.remove(result);
            result.setResult(Collections.emptyList());
        });
        
        // Handle completion
        result.onCompletion(() -> waitingClients.remove(result));
        
        // Check if messages already available
        List<Message> pending = drainMessages();
        if (!pending.isEmpty()) {
            result.setResult(pending);
        } else {
            // No messages, add to waiting list
            waitingClients.add(result);
        }
        
        return result;
    }
    
    /**
     * Send a message.
     * Notifies all waiting clients.
     */
    @PostMapping("/messages")
    public Message sendMessage(@RequestBody Message message) {
        message.setId(UUID.randomUUID().toString());
        message.setTimestamp(System.currentTimeMillis());
        
        // Add to queue
        messageQueue.offer(message);
        
        // Notify all waiting clients
        notifyWaitingClients();
        
        return message;
    }
    
    private List<Message> drainMessages() {
        List<Message> messages = new ArrayList<>();
        messageQueue.drainTo(messages);
        return messages;
    }
    
    private void notifyWaitingClients() {
        List<Message> messages = drainMessages();
        if (messages.isEmpty()) return;
        
        synchronized (waitingClients) {
            for (DeferredResult<List<Message>> client : waitingClients) {
                client.setResult(messages);
            }
            waitingClients.clear();
        }
    }
    
    // Message DTO
    public static class Message {
        private String id;
        private String user;
        private String text;
        private long timestamp;
        
        // Getters and setters
        public String getId() { return id; }
        public void setId(String id) { this.id = id; }
        public String getUser() { return user; }
        public void setUser(String user) { this.user = user; }
        public String getText() { return text; }
        public void setText(String text) { this.text = text; }
        public long getTimestamp() { return timestamp; }
        public void setTimestamp(long timestamp) { this.timestamp = timestamp; }
    }
}
```

### SSE Server (Spring WebFlux)

```java
// pom.xml dependencies
/*
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
*/

import org.springframework.http.MediaType;
import org.springframework.http.codec.ServerSentEvent;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Sinks;
import java.time.Duration;
import java.util.UUID;

@RestController
@RequestMapping("/api")
public class SSEController {
    
    // Sink for broadcasting events to all subscribers
    private final Sinks.Many<Message> messageSink = 
        Sinks.many().multicast().onBackpressureBuffer();
    
    /**
     * SSE endpoint.
     * Client subscribes once, receives stream of events.
     */
    @GetMapping(path = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Message>> streamEvents() {
        return messageSink.asFlux()
            .map(message -> ServerSentEvent.<Message>builder()
                .id(message.getId())
                .event("message")
                .data(message)
                .retry(Duration.ofSeconds(3))
                .build());
    }
    
    /**
     * SSE with heartbeat to keep connection alive.
     */
    @GetMapping(path = "/events-with-heartbeat", 
                produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Object>> streamWithHeartbeat() {
        // Message stream
        Flux<ServerSentEvent<Object>> messages = messageSink.asFlux()
            .map(message -> ServerSentEvent.builder()
                .id(message.getId())
                .event("message")
                .data((Object) message)
                .build());
        
        // Heartbeat every 15 seconds
        Flux<ServerSentEvent<Object>> heartbeat = Flux.interval(Duration.ofSeconds(15))
            .map(i -> ServerSentEvent.builder()
                .event("heartbeat")
                .data((Object) "ping")
                .build());
        
        // Merge both streams
        return Flux.merge(messages, heartbeat);
    }
    
    /**
     * Send a message (broadcasts to all SSE subscribers).
     */
    @PostMapping("/messages")
    public Message sendMessage(@RequestBody Message message) {
        message.setId(UUID.randomUUID().toString());
        message.setTimestamp(System.currentTimeMillis());
        
        // Emit to all subscribers
        messageSink.tryEmitNext(message);
        
        return message;
    }
    
    // Message DTO (same as before)
    public static class Message {
        private String id;
        private String user;
        private String text;
        private long timestamp;
        
        // Getters and setters...
        public String getId() { return id; }
        public void setId(String id) { this.id = id; }
        public String getUser() { return user; }
        public void setUser(String user) { this.user = user; }
        public String getText() { return text; }
        public void setText(String text) { this.text = text; }
        public long getTimestamp() { return timestamp; }
        public void setTimestamp(long timestamp) { this.timestamp = timestamp; }
    }
}
```

### WebSocket Server (Spring WebSocket + STOMP)

```java
// pom.xml dependencies
/*
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
*/

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.handler.annotation.*;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.stereotype.Controller;
import org.springframework.web.socket.config.annotation.*;

// Configuration
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // Enable simple broker for subscriptions
        config.enableSimpleBroker("/topic", "/queue");
        // Prefix for messages FROM client
        config.setApplicationDestinationPrefixes("/app");
        // Prefix for user-specific messages
        config.setUserDestinationPrefix("/user");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // WebSocket endpoint
        registry.addEndpoint("/ws")
            .setAllowedOrigins("*")
            .withSockJS();  // Fallback for browsers without WebSocket
    }
}

// Controller
@Controller
public class ChatController {
    
    @Autowired
    private SimpMessagingTemplate messagingTemplate;
    
    /**
     * Handle messages sent to /app/chat.send
     * Broadcasts to all subscribers of /topic/chat
     */
    @MessageMapping("/chat.send")
    @SendTo("/topic/chat")
    public ChatMessage sendMessage(ChatMessage message) {
        message.setId(UUID.randomUUID().toString());
        message.setTimestamp(System.currentTimeMillis());
        return message;
    }
    
    /**
     * Handle user joining
     */
    @MessageMapping("/chat.join")
    @SendTo("/topic/chat")
    public ChatMessage userJoin(ChatMessage message) {
        message.setType(MessageType.JOIN);
        message.setText(message.getUser() + " joined the chat");
        return message;
    }
    
    /**
     * Send private message to specific user
     */
    @MessageMapping("/chat.private")
    public void sendPrivateMessage(PrivateMessage message) {
        messagingTemplate.convertAndSendToUser(
            message.getRecipient(),
            "/queue/private",
            message
        );
    }
    
    // DTOs
    public static class ChatMessage {
        private String id;
        private String user;
        private String text;
        private MessageType type = MessageType.CHAT;
        private long timestamp;
        
        // Getters and setters...
    }
    
    public enum MessageType {
        CHAT, JOIN, LEAVE
    }
    
    public static class PrivateMessage {
        private String sender;
        private String recipient;
        private String text;
        
        // Getters and setters...
    }
}
```

**Client-Side (JavaScript with SockJS and STOMP)**:

```html
<!DOCTYPE html>
<html>
<head>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/stompjs@2/lib/stomp.min.js"></script>
</head>
<body>
    <div id="messages"></div>
    <input type="text" id="messageInput" placeholder="Type a message...">
    <button onclick="sendMessage()">Send</button>
    
    <script>
        let stompClient = null;
        const username = 'User' + Math.floor(Math.random() * 1000);
        
        function connect() {
            const socket = new SockJS('/ws');
            stompClient = Stomp.over(socket);
            
            stompClient.connect({}, function(frame) {
                console.log('Connected: ' + frame);
                
                // Subscribe to public chat
                stompClient.subscribe('/topic/chat', function(message) {
                    const msg = JSON.parse(message.body);
                    showMessage(msg);
                });
                
                // Subscribe to private messages
                stompClient.subscribe('/user/queue/private', function(message) {
                    const msg = JSON.parse(message.body);
                    showMessage(msg, true);
                });
                
                // Announce joining
                stompClient.send('/app/chat.join', {}, JSON.stringify({
                    user: username,
                    type: 'JOIN'
                }));
            });
        }
        
        function sendMessage() {
            const input = document.getElementById('messageInput');
            const text = input.value.trim();
            
            if (text && stompClient) {
                stompClient.send('/app/chat.send', {}, JSON.stringify({
                    user: username,
                    text: text
                }));
                input.value = '';
            }
        }
        
        function showMessage(msg, isPrivate = false) {
            const div = document.getElementById('messages');
            const prefix = isPrivate ? '[Private] ' : '';
            div.innerHTML += `<p>${prefix}<b>${msg.user}</b>: ${msg.text}</p>`;
        }
        
        // Connect on page load
        connect();
    </script>
</body>
</html>
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Pitfall 1: Not Implementing Heartbeats

**Scenario**: WebSocket connections silently die (network change, proxy timeout).

**Mistake**: Assuming connection is alive because no error was received.

**Problem**: TCP keepalive is too slow (typically 2 hours). Proxies may timeout idle connections in 30-60 seconds.

**Solution**: Implement application-level heartbeat:

```java
// Server-side ping
@Scheduled(fixedRate = 25000) // Every 25 seconds
public void sendHeartbeat() {
    for (WebSocketSession session : sessions) {
        try {
            session.sendMessage(new PingMessage());
        } catch (IOException e) {
            // Connection dead, remove it
            sessions.remove(session);
        }
    }
}

// Client-side (JavaScript)
setInterval(() => {
    if (socket.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify({ type: 'ping' }));
    }
}, 25000);
```

### Pitfall 2: Not Handling Reconnection

**Scenario**: Connection drops, user sees stale data.

**Mistake**: Not implementing automatic reconnection.

**Solution**:

```javascript
// Robust WebSocket client with reconnection
class ReconnectingWebSocket {
    constructor(url) {
        this.url = url;
        this.reconnectInterval = 1000;
        this.maxReconnectInterval = 30000;
        this.connect();
    }
    
    connect() {
        this.socket = new WebSocket(this.url);
        
        this.socket.onopen = () => {
            console.log('Connected');
            this.reconnectInterval = 1000; // Reset on success
        };
        
        this.socket.onclose = () => {
            console.log('Disconnected, reconnecting in', this.reconnectInterval);
            setTimeout(() => this.connect(), this.reconnectInterval);
            // Exponential backoff
            this.reconnectInterval = Math.min(
                this.reconnectInterval * 2,
                this.maxReconnectInterval
            );
        };
        
        this.socket.onerror = (error) => {
            console.error('WebSocket error:', error);
            this.socket.close();
        };
    }
    
    send(data) {
        if (this.socket.readyState === WebSocket.OPEN) {
            this.socket.send(data);
        }
    }
}
```

### Pitfall 3: Blocking on Message Handling

**Scenario**: Slow message processing blocks all clients.

**Mistake**: Processing messages synchronously in WebSocket handler.

**Solution**: Use async processing:

```java
@Override
protected void handleTextMessage(WebSocketSession session, TextMessage message) {
    // DON'T do this:
    // processMessageSynchronously(message); // Blocks!
    
    // DO this:
    CompletableFuture.runAsync(() -> {
        processMessage(message);
    }, executorService);
}
```

### Pitfall 4: Memory Leaks from Unclosed Connections

**Scenario**: Server memory grows over time.

**Mistake**: Not cleaning up closed/dead connections.

**Solution**:

```java
// Proper cleanup
private final Set<WebSocketSession> sessions = ConcurrentHashMap.newKeySet();

@Override
public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
    sessions.remove(session);
    cleanupUserState(session);
}

// Periodic cleanup of zombie connections
@Scheduled(fixedRate = 60000)
public void cleanupDeadConnections() {
    sessions.removeIf(session -> !session.isOpen());
}
```

### Pitfall 5: Not Considering Proxy/Load Balancer Timeouts

**Scenario**: Connections drop after 60 seconds.

**Mistake**: Not configuring infrastructure for long-lived connections.

**Solution**:

```nginx
# Nginx configuration for WebSocket
location /ws {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    
    # Increase timeouts
    proxy_connect_timeout 7d;
    proxy_send_timeout 7d;
    proxy_read_timeout 7d;
}
```

```yaml
# AWS ALB - increase idle timeout
resource "aws_lb" "main" {
  idle_timeout = 3600  # 1 hour (max is 4000 seconds)
}
```

---

## 8️⃣ When NOT to Use Each Pattern

### When NOT to Use Long Polling

| Scenario | Why | Alternative |
|----------|-----|-------------|
| High-frequency updates | Too much overhead | SSE or WebSocket |
| Bidirectional communication | Long polling is one-way | WebSocket |
| Many concurrent users | Connection overhead | SSE or WebSocket |
| Low-latency requirements | Reconnection delay | WebSocket |

### When NOT to Use SSE

| Scenario | Why | Alternative |
|----------|-----|-------------|
| Bidirectional communication | SSE is server-to-client only | WebSocket |
| Binary data | SSE is text-only | WebSocket |
| Many event streams | 6 connection limit per domain | WebSocket |
| Need message acknowledgment | No built-in ACK | WebSocket + custom protocol |

### When NOT to Use WebSocket

| Scenario | Why | Alternative |
|----------|-----|-------------|
| Simple notifications | Overkill, more complex | SSE |
| One-way server updates | Unnecessary bidirectional | SSE |
| Stateless architecture | WebSocket is stateful | Long polling or SSE |
| Behind restrictive firewall | May be blocked | Long polling |
| Need HTTP features | No cookies, caching, etc. | Long polling |

---

## 9️⃣ Comparison Table

| Feature | Long Polling | SSE | WebSocket |
|---------|--------------|-----|-----------|
| **Direction** | Client → Server (pull) | Server → Client (push) | Bidirectional |
| **Protocol** | HTTP | HTTP | WS (upgrade from HTTP) |
| **Connection** | New per poll | Persistent | Persistent |
| **Latency** | Medium (reconnection) | Low | Lowest |
| **Browser Support** | All | All modern | All modern |
| **Binary Data** | Yes (via encoding) | No (text only) | Yes |
| **Automatic Reconnect** | Manual | Built-in | Manual |
| **Proxy Friendly** | Yes | Mostly | Sometimes |
| **Scalability** | Medium | Good | Challenging |
| **Complexity** | Low | Low | Medium |

---

## 🔟 Interview Follow-Up Questions

### L4 (Junior/Mid) Level Questions

**Q1: What is the difference between WebSocket and HTTP?**

**A**: 
- **HTTP**: Request-response model. Client initiates, server responds, connection typically closes. Stateless.
- **WebSocket**: Persistent, bidirectional connection. Either side can send anytime. Stateful.

WebSocket starts as HTTP (upgrade handshake), then switches to WebSocket protocol. After upgrade, it's no longer HTTP.

**Q2: When would you use SSE over WebSocket?**

**A**: Use SSE when:
1. You only need server-to-client communication (notifications, live feeds)
2. You want simpler implementation (just HTTP streaming)
3. You need automatic reconnection (built into EventSource API)
4. You're behind proxies that might block WebSocket

Use WebSocket when you need bidirectional communication (chat, gaming) or binary data.

**Q3: How does long polling differ from regular polling?**

**A**: 
- **Regular polling**: Client asks server every N seconds. Most responses are empty. Wastes bandwidth.
- **Long polling**: Client asks server, server holds request until data available (or timeout). Reduces empty responses, more efficient, lower latency.

### L5 (Senior) Level Questions

**Q4: How would you scale a WebSocket-based chat application to 1 million concurrent users?**

**A**: 
1. **Horizontal scaling**: Multiple WebSocket servers, each handling ~50K connections
2. **Sticky sessions**: Load balancer routes user to same server (IP hash or cookie)
3. **Message bus**: Redis Pub/Sub or Kafka for cross-server communication
4. **Connection state**: Store in Redis for failover
5. **Graceful shutdown**: Drain connections before server restart
6. **Health checks**: Remove unhealthy servers from rotation

Architecture:
```
Users → Load Balancer (sticky) → WebSocket Servers → Redis Pub/Sub
                                                   → Database (persistence)
```

**Q5: A WebSocket service has high memory usage. How do you diagnose?**

**A**: 
1. **Check connection count**: Are connections being cleaned up?
2. **Check message queues**: Are outbound queues growing?
3. **Check for leaks**: Are event listeners being removed?
4. **Profile memory**: Use heap dump to find large objects
5. **Check buffer sizes**: Are receive/send buffers too large?

Common causes:
- Dead connections not removed
- Large messages queued for slow clients
- Event listener leaks
- Large per-connection state

### L6 (Staff+) Level Questions

**Q6: Design a real-time notification system for a social media platform with 100M users.**

**A**: 
1. **Connection tier**:
   - WebSocket servers in each region
   - ~2000 servers globally (50K connections each)
   - SSE fallback for restricted networks

2. **Message routing**:
   - User → Server mapping in Redis Cluster
   - Kafka for durable message delivery
   - Fan-out service for popular users (millions of followers)

3. **Delivery guarantees**:
   - At-least-once delivery
   - Client-side deduplication (message IDs)
   - Offline queue for disconnected users

4. **Optimization**:
   - Batch notifications (don't push every like)
   - Priority lanes (DMs > likes)
   - Rate limiting per user

5. **Monitoring**:
   - Connection count per server
   - Message latency percentiles
   - Delivery success rate

**Q7: How would you implement exactly-once delivery over WebSocket?**

**A**: WebSocket doesn't guarantee delivery. Implement at application level:

1. **Message IDs**: Unique ID for each message
2. **Acknowledgments**: Client sends ACK for each message
3. **Retry with timeout**: Server retries if no ACK
4. **Idempotency**: Client ignores duplicate message IDs
5. **Sequence numbers**: Detect gaps, request retransmission
6. **Persistent storage**: Store unacknowledged messages

```
Server                          Client
   │                               │
   │  msg(id=1, seq=1) ══════════> │
   │                               │ [Process, store seq=1]
   │  <══════════════════ ack(id=1)│
   │                               │
   │  msg(id=2, seq=2) ══════════> │
   │  [Timeout, no ACK]            │
   │  msg(id=2, seq=2) ══════════> │ [Retry]
   │                               │ [Dedupe by id, ignore]
   │  <══════════════════ ack(id=2)│
```

---

## 1️⃣1️⃣ Mental Summary

**Real-time communication** enables servers to push data to clients without polling. Three main patterns exist:

**Long Polling**: Client holds request open until server has data. Simple, works everywhere, but has reconnection overhead. Good for: compatibility, simple use cases.

**Server-Sent Events (SSE)**: Persistent HTTP connection for server-to-client streaming. Simple, automatic reconnection, but one-way only. Good for: notifications, live feeds, dashboards.

**WebSockets**: Persistent bidirectional connection. Lowest latency, most flexible, but more complex to scale. Good for: chat, gaming, collaborative editing.

**Choose based on your needs**: If you only need server-to-client, prefer SSE (simpler). If you need bidirectional, use WebSocket. If you need maximum compatibility, use long polling as fallback.

**For production**: Always implement heartbeats, reconnection logic, and proper cleanup. Use message bus (Redis) for horizontal scaling. Monitor connection counts and message latency.

---

## 📚 Further Reading

- [WebSocket RFC 6455](https://tools.ietf.org/html/rfc6455)
- [Server-Sent Events Spec](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [Socket.IO Documentation](https://socket.io/docs/)
- [Spring WebSocket Guide](https://spring.io/guides/gs/messaging-stomp-websocket/)
- [Scaling WebSockets](https://ably.com/topic/scaling-websockets)
- [Discord Engineering Blog](https://discord.com/blog/how-discord-handles-push-request-bursts-of-over-a-million-per-minute-with-elixirs-genstage)

