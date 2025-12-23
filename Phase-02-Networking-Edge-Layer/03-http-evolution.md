# 🌐 HTTP Evolution: From HTTP/1.0 to HTTP/3

## 0️⃣ Prerequisites

Before diving into HTTP evolution, you should understand:

- **TCP (Transmission Control Protocol)**: A reliable, connection-oriented protocol that guarantees ordered delivery of data. HTTP/1.x and HTTP/2 run on top of TCP. See `02-tcp-vs-udp.md` for details.

- **UDP (User Datagram Protocol)**: A fast, connectionless protocol with no delivery guarantees. HTTP/3 runs on top of UDP (via QUIC).

- **TLS (Transport Layer Security)**: Encryption protocol that secures data in transit. HTTPS = HTTP + TLS. TLS adds 1-2 round trips for handshake.

- **Client-Server Model**: Browser (client) sends requests to web server, server sends responses back. Each request-response pair is independent in HTTP.

- **Latency and RTT**: Round-Trip Time (RTT) is the time for a packet to travel from client to server and back. Typical RTT: 10-50ms local, 100-300ms cross-continent.

---

## 1️⃣ What Problem Does HTTP Evolution Solve?

### The Specific Pain Point

The original HTTP was designed in 1991 for simple document retrieval. Tim Berners-Lee needed a way to fetch hypertext documents. It worked great for that.

**The Problem**: Modern web pages are not simple documents. A typical page today loads:
- 1 HTML document
- 20-50 CSS/JS files
- 50-100 images
- Multiple fonts
- API calls for dynamic data

With HTTP/1.0, each resource required a new TCP connection. With HTTP/1.1, connections could be reused, but requests were still sequential. This created massive bottlenecks.

### What Systems Looked Like Before HTTP/2

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HTTP/1.1 PAGE LOAD (Simplified)                           │
└─────────────────────────────────────────────────────────────────────────────┘

Browser                                                    Server
   │                                                          │
   │  GET /index.html ───────────────────────────────────────>│
   │  <──────────────────────────────────────── 200 OK + HTML │
   │                                                          │
   │  Parse HTML, discover CSS needed                         │
   │                                                          │
   │  GET /style.css ────────────────────────────────────────>│
   │  <──────────────────────────────────────── 200 OK + CSS  │
   │                                                          │
   │  Parse CSS, discover font needed                         │
   │                                                          │
   │  GET /font.woff ────────────────────────────────────────>│
   │  <─────────────────────────────────────── 200 OK + Font  │
   │                                                          │
   │  ... 50 more requests, one at a time ...                 │

Total time: 50 resources × 100ms RTT = 5000ms minimum
(Even with 6 parallel connections: 5000ms / 6 ≈ 833ms)
```

### What Breaks Without HTTP Evolution

**Without HTTP/2**:
- Pages load slowly (sequential requests)
- Developers use hacks (sprite sheets, CSS concatenation, domain sharding)
- Mobile users suffer most (high latency networks)
- Server resources wasted on connection overhead

**Without HTTP/3**:
- Packet loss causes head-of-line blocking (one lost packet blocks all streams)
- Connection migration fails (switching WiFi to cellular drops connections)
- TLS handshake adds latency

### Real Examples of the Problem

**Example 1: Amazon found that every 100ms of latency cost them 1% in sales.**

**Example 2: Google found that a 500ms delay in search results caused 20% drop in traffic.**

**Example 3: Mobile users on 3G networks experienced 3-5 second page loads with HTTP/1.1.**

---

## 2️⃣ Intuition and Mental Model

### The Highway Analogy

**HTTP/1.0 is like a single-lane road with a toll booth**:
- One car goes through at a time
- After each car, the toll booth closes and reopens (new connection)
- Very slow for rush hour traffic

**HTTP/1.1 is like a single-lane road with a fast pass**:
- Cars can go through one after another (keep-alive)
- But still single lane, cars must wait their turn
- If one car breaks down, everyone behind waits

**HTTP/2 is like a multi-lane highway**:
- Many cars travel simultaneously (multiplexing)
- Fast cars don't wait for slow cars (stream prioritization)
- But if the road itself has a pothole, all lanes stop (TCP head-of-line blocking)

**HTTP/3 is like multiple independent roads**:
- Each stream is its own road (QUIC streams over UDP)
- Pothole on one road doesn't affect others
- Roads can be rerouted mid-journey (connection migration)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     HTTP EVOLUTION VISUALIZATION                             │
└─────────────────────────────────────────────────────────────────────────────┘

HTTP/1.0:  [Request1]────────[Response1]
           [Request2]────────[Response2]
           [Request3]────────[Response3]
           (New connection each time)

HTTP/1.1:  [Request1]────────[Response1][Request2]────────[Response2][Req3]...
           (Same connection, but sequential)

HTTP/2:    [Req1][Req2][Req3] ════════════════> [Resp1][Resp2][Resp3]
           (Multiplexed on single connection)

HTTP/3:    [Req1]═══> [Resp1]
           [Req2]═══> [Resp2]    (Independent streams, no blocking)
           [Req3]═══> [Resp3]
```

---

## 3️⃣ How Each HTTP Version Works Internally

### HTTP/1.0 (1996)

**Key Characteristics**:
- One request per TCP connection
- Connection closed after each response
- No persistent connections by default

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HTTP/1.0 REQUEST FLOW                                │
└─────────────────────────────────────────────────────────────────────────────┘

Client                                                    Server
   │                                                         │
   │  [TCP SYN] ─────────────────────────────────────────────>
   │  <───────────────────────────────────────── [TCP SYN-ACK]
   │  [TCP ACK] ─────────────────────────────────────────────>
   │                                                         │
   │  GET /page.html HTTP/1.0                                │
   │  Host: example.com                                      │
   │  ──────────────────────────────────────────────────────>│
   │                                                         │
   │  HTTP/1.0 200 OK                                        │
   │  Content-Type: text/html                                │
   │  Content-Length: 1234                                   │
   │  <html>...</html>                                       │
   │  <──────────────────────────────────────────────────────│
   │                                                         │
   │  [TCP FIN] ─────────────────────────────────────────────>
   │  <───────────────────────────────────────────── [TCP FIN]
   │                                                         │
   │  CONNECTION CLOSED                                      │
   │                                                         │
   │  Next request? Start over with new TCP handshake...     │
```

**Problems**:
- 3 RTTs minimum per resource (TCP handshake + request/response + close)
- No way to pipeline requests
- Massive overhead for pages with many resources

### HTTP/1.1 (1997)

**Key Improvements**:
- **Persistent Connections (Keep-Alive)**: Reuse TCP connection for multiple requests
- **Pipelining**: Send multiple requests without waiting for responses (rarely used)
- **Chunked Transfer**: Stream responses without knowing size upfront
- **Host Header**: Multiple websites on same IP address (virtual hosting)
- **Cache Control**: Better caching directives

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HTTP/1.1 WITH KEEP-ALIVE                                  │
└─────────────────────────────────────────────────────────────────────────────┘

Client                                                    Server
   │                                                         │
   │  [TCP Handshake - once]                                 │
   │  ═══════════════════════════════════════════════════════│
   │                                                         │
   │  GET /page.html HTTP/1.1                                │
   │  Host: example.com                                      │
   │  Connection: keep-alive                                 │
   │  ──────────────────────────────────────────────────────>│
   │                                                         │
   │  HTTP/1.1 200 OK                                        │
   │  Connection: keep-alive                                 │
   │  <html>...</html>                                       │
   │  <──────────────────────────────────────────────────────│
   │                                                         │
   │  GET /style.css HTTP/1.1      (Same connection!)        │
   │  Host: example.com                                      │
   │  ──────────────────────────────────────────────────────>│
   │                                                         │
   │  HTTP/1.1 200 OK                                        │
   │  .body { ... }                                          │
   │  <──────────────────────────────────────────────────────│
   │                                                         │
   │  ... more requests on same connection ...               │
```

**HTTP/1.1 Pipelining** (Theoretical):

```
Client                                                    Server
   │                                                         │
   │  GET /a.js ────────────────────────────────────────────>│
   │  GET /b.js ────────────────────────────────────────────>│
   │  GET /c.js ────────────────────────────────────────────>│
   │  (Send all requests without waiting)                    │
   │                                                         │
   │  <─────────────────────────────────────── Response /a.js│
   │  <─────────────────────────────────────── Response /b.js│
   │  <─────────────────────────────────────── Response /c.js│
   │  (Responses MUST be in order)                           │
```

**Why Pipelining Failed**:
- Responses must be in order (head-of-line blocking)
- If /a.js is slow, /b.js and /c.js wait even if ready
- Many proxies and servers don't support it correctly
- Browsers disabled it by default

**HTTP/1.1 Workarounds** (Still used today):

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HTTP/1.1 OPTIMIZATION HACKS                               │
└─────────────────────────────────────────────────────────────────────────────┘

1. Domain Sharding:
   - Browsers limit connections per domain (6 in Chrome)
   - Use multiple domains: img1.example.com, img2.example.com
   - 6 connections × 4 domains = 24 parallel requests

2. Resource Bundling:
   - Concatenate all JS into one file
   - Concatenate all CSS into one file
   - Fewer requests = faster load

3. Sprite Sheets:
   - Combine many small images into one large image
   - Use CSS to show only the needed portion
   - One request instead of 50

4. Inlining:
   - Embed small CSS/JS directly in HTML
   - Embed small images as base64 data URIs
   - Zero additional requests
```

### HTTP/2 (2015)

**Key Improvements**:
- **Binary Protocol**: More efficient parsing than text-based HTTP/1.1
- **Multiplexing**: Multiple requests/responses on single connection simultaneously
- **Stream Prioritization**: Important resources load first
- **Header Compression (HPACK)**: Reduce redundant header data
- **Server Push**: Server sends resources before client asks

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HTTP/2 FRAME STRUCTURE                               │
└─────────────────────────────────────────────────────────────────────────────┘

HTTP/2 breaks everything into frames:

┌─────────────────────────────────────────┐
│ Length (24 bits)                        │  How big is this frame?
├─────────────────────────────────────────┤
│ Type (8 bits)                           │  DATA, HEADERS, SETTINGS, etc.
├─────────────────────────────────────────┤
│ Flags (8 bits)                          │  END_STREAM, END_HEADERS, etc.
├─────────────────────────────────────────┤
│ Stream Identifier (31 bits)             │  Which stream does this belong to?
├─────────────────────────────────────────┤
│ Frame Payload (variable)                │  The actual data
└─────────────────────────────────────────┘

Frame Types:
- DATA: Request/response body
- HEADERS: HTTP headers (compressed)
- PRIORITY: Stream priority
- RST_STREAM: Cancel a stream
- SETTINGS: Connection configuration
- PUSH_PROMISE: Server push announcement
- PING: Keep-alive and RTT measurement
- GOAWAY: Graceful connection shutdown
- WINDOW_UPDATE: Flow control
- CONTINUATION: Large header continuation
```

**HTTP/2 Multiplexing**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      HTTP/2 MULTIPLEXED STREAMS                              │
└─────────────────────────────────────────────────────────────────────────────┘

Single TCP Connection, Multiple Streams:

Time ──────────────────────────────────────────────────────────────────────>

Stream 1 (HTML):   [HEADERS]─────────────────[DATA]─────────[DATA]─[END]
Stream 3 (CSS):    ────[HEADERS]──[DATA]──[DATA]──[END]
Stream 5 (JS):     ──────────[HEADERS]────────────────[DATA]──[DATA]──[END]
Stream 7 (Image):  ────────────────[HEADERS]──[DATA]──[DATA]──[DATA]──[END]

All streams interleaved on same connection!
No head-of-line blocking at HTTP level.
```

**HTTP/2 Header Compression (HPACK)**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HPACK COMPRESSION                                    │
└─────────────────────────────────────────────────────────────────────────────┘

HTTP/1.1 Headers (repeated every request):
  GET /page1.html HTTP/1.1
  Host: example.com
  User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...
  Accept: text/html,application/xhtml+xml,...
  Accept-Language: en-US,en;q=0.9
  Accept-Encoding: gzip, deflate, br
  Cookie: session=abc123; user=john; preferences=...
  
  (500-800 bytes per request, mostly identical!)

HTTP/2 HPACK:
  - Static table: 61 common header name-value pairs (pre-defined)
  - Dynamic table: Headers from previous requests (per-connection)
  - Huffman encoding: Compress header values
  
  First request: Full headers, populate dynamic table
  Subsequent requests: Reference table indices
  
  Request 1: [Full headers] → 500 bytes
  Request 2: [Index 1, Index 5, Index 12, "new-value"] → 50 bytes
  Request 3: [Index 1, Index 5, Index 12, Index 62] → 20 bytes
```

**HTTP/2 Server Push**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HTTP/2 SERVER PUSH                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Without Server Push:
  Client: GET /index.html
  Server: Here's the HTML
  Client: (parses HTML) Oh, I need style.css
  Client: GET /style.css
  Server: Here's the CSS
  
  2 round trips before CSS arrives

With Server Push:
  Client: GET /index.html
  Server: Here's the HTML
  Server: PUSH_PROMISE for /style.css
  Server: Here's /style.css (pushed)
  Client: (parses HTML) Oh, I need style.css... wait, I already have it!
  
  1 round trip, CSS arrives with HTML
```

**HTTP/2 Limitations**:

Despite improvements, HTTP/2 has a fundamental problem: **TCP head-of-line blocking**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TCP HEAD-OF-LINE BLOCKING                                 │
└─────────────────────────────────────────────────────────────────────────────┘

HTTP/2 multiplexes at the application layer, but TCP sees one byte stream:

TCP View:     [Stream1][Stream3][Stream1][Stream5][Stream3][Stream1]...
                  │        │        │        │        │        │
                  ▼        ▼        ▼        ▼        ▼        ▼
TCP Packets:  [Pkt1]   [Pkt2]   [Pkt3]   [Pkt4]   [Pkt5]   [Pkt6]

If Pkt2 is lost:
  - TCP waits for retransmission
  - Pkt3, Pkt4, Pkt5, Pkt6 are buffered but NOT delivered to application
  - ALL streams blocked, even though only Stream3 data was lost!

This is worse on lossy networks (mobile, WiFi)
```

### HTTP/3 (2022)

**Key Innovation**: Replace TCP with QUIC (Quick UDP Internet Connections).

QUIC provides:
- **UDP-based**: No TCP head-of-line blocking
- **Built-in TLS 1.3**: Encryption is mandatory and integrated
- **Independent Streams**: Packet loss affects only that stream
- **Connection Migration**: Connections survive IP changes
- **0-RTT Connection Establishment**: Resume connections instantly

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HTTP/3 ARCHITECTURE                                  │
└─────────────────────────────────────────────────────────────────────────────┘

HTTP/1.1 & HTTP/2 Stack:          HTTP/3 Stack:
┌─────────────────────┐           ┌─────────────────────┐
│     HTTP/1.1 or     │           │       HTTP/3        │
│      HTTP/2         │           │                     │
├─────────────────────┤           ├─────────────────────┤
│        TLS          │           │        QUIC         │
├─────────────────────┤           │  (includes TLS 1.3) │
│        TCP          │           ├─────────────────────┤
├─────────────────────┤           │        UDP          │
│        IP           │           ├─────────────────────┤
└─────────────────────┘           │        IP           │
                                  └─────────────────────┘
```

**QUIC Connection Establishment**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONNECTION ESTABLISHMENT COMPARISON                       │
└─────────────────────────────────────────────────────────────────────────────┘

TCP + TLS 1.2 (HTTP/1.1, HTTP/2):
  RTT 1: TCP SYN → SYN-ACK → ACK
  RTT 2: TLS ClientHello → ServerHello
  RTT 3: TLS Certificate → Finished
  Total: 3 RTT before first HTTP request

TCP + TLS 1.3 (HTTP/2 with TLS 1.3):
  RTT 1: TCP SYN → SYN-ACK → ACK
  RTT 2: TLS ClientHello → ServerHello + Finished
  Total: 2 RTT before first HTTP request

QUIC (HTTP/3) - First Connection:
  RTT 1: QUIC Initial + TLS ClientHello → ServerHello + Finished
  Total: 1 RTT before first HTTP request

QUIC (HTTP/3) - Resumed Connection (0-RTT):
  RTT 0: Send request with 0-RTT data
  Total: 0 RTT before first HTTP request!
```

**QUIC Stream Independence**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    QUIC STREAM INDEPENDENCE                                  │
└─────────────────────────────────────────────────────────────────────────────┘

QUIC treats each stream as independent:

Stream 1: [Pkt1]──────[Pkt3]──────[Pkt5]────────> Delivered immediately
Stream 2: [Pkt2]──X───[Pkt4]──────[Pkt6]────────> Waits for Pkt2 retransmit
Stream 3: [Pkt7]──────[Pkt8]──────[Pkt9]────────> Delivered immediately

Packet loss in Stream 2 does NOT block Streams 1 and 3!
```

**Connection Migration**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    QUIC CONNECTION MIGRATION                                 │
└─────────────────────────────────────────────────────────────────────────────┘

Scenario: User walks from WiFi to cellular

TCP (HTTP/1.1, HTTP/2):
  WiFi IP: 192.168.1.100 ──── Connection to server
  [User moves]
  Cellular IP: 10.0.0.50
  Connection DEAD (different IP = different connection)
  Must establish new TCP + TLS connection (2-3 RTT)

QUIC (HTTP/3):
  WiFi IP: 192.168.1.100 ──── Connection ID: ABC123
  [User moves]
  Cellular IP: 10.0.0.50 ──── Connection ID: ABC123 (same!)
  Connection CONTINUES (identified by Connection ID, not IP)
  Zero interruption
```

---

## 4️⃣ Simulation: Loading a Web Page

Let's trace loading a page with 10 resources across HTTP versions.

### HTTP/1.1 (6 connections, no pipelining)

```
Time (ms)   Connection 1    Connection 2    Connection 3    Connection 4    Connection 5    Connection 6
─────────────────────────────────────────────────────────────────────────────────────────────────────────
0-100       TCP+TLS         TCP+TLS         TCP+TLS         TCP+TLS         TCP+TLS         TCP+TLS
100-200     GET html        GET css         GET js1         GET js2         GET img1        GET img2
200-300     Response        Response        Response        Response        Response        Response
300-400     GET img3        GET img4        GET img5        GET img6        idle            idle
400-500     Response        Response        Response        Response        idle            idle

Total: ~500ms for 10 resources
```

### HTTP/2 (1 connection, multiplexed)

```
Time (ms)   Single Connection
────────────────────────────────────────────────────────────────────────────────
0-100       TCP + TLS handshake
100-150     HEADERS(html) HEADERS(css) HEADERS(js1) HEADERS(js2) ...
150-250     DATA(html) DATA(css) DATA(js1) DATA(js2) DATA(img1) DATA(img2) ...
250-300     DATA(img3) DATA(img4) DATA(img5) DATA(img6) END_STREAM(all)

Total: ~300ms for 10 resources (40% faster!)
```

### HTTP/3 (1 connection, 0-RTT)

```
Time (ms)   Single QUIC Connection
────────────────────────────────────────────────────────────────────────────────
0-50        QUIC handshake (1 RTT, or 0-RTT if resumed)
50-100      HEADERS(all) + DATA(html, css, js) interleaved
100-150     DATA(images) interleaved
150-200     END_STREAM(all)

Total: ~200ms for 10 resources (60% faster than HTTP/1.1!)
```

---

## 5️⃣ How Engineers Use HTTP Versions in Production

### Real-World Adoption

**Google**:
- Invented SPDY (precursor to HTTP/2) in 2009
- Invented QUIC in 2012
- All Google services support HTTP/3
- Chrome has native HTTP/3 support

**Cloudflare**:
- Enabled HTTP/2 by default in 2015
- Enabled HTTP/3 by default in 2019
- Reports 12% faster page loads with HTTP/3
- Reference: [Cloudflare HTTP/3 Blog](https://blog.cloudflare.com/http3-the-past-present-and-future/)

**Facebook**:
- Uses HTTP/3 for mobile apps
- Reports 6% latency improvement, 15% fewer errors
- Critical for users on poor mobile networks

### Production Configuration

**Nginx HTTP/2**:

```nginx
# /etc/nginx/nginx.conf

http {
    server {
        listen 443 ssl http2;  # Enable HTTP/2
        server_name example.com;
        
        ssl_certificate /etc/ssl/certs/example.com.crt;
        ssl_certificate_key /etc/ssl/private/example.com.key;
        
        # TLS 1.3 for best performance
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
        
        # HTTP/2 specific settings
        http2_max_concurrent_streams 128;
        http2_idle_timeout 3m;
        
        location / {
            root /var/www/html;
            
            # Server Push (use sparingly)
            http2_push /css/style.css;
            http2_push /js/app.js;
        }
    }
}
```

**Nginx HTTP/3 (with quiche)**:

```nginx
# Requires nginx compiled with quiche or similar QUIC library

http {
    server {
        listen 443 ssl http2;
        listen 443 quic reuseport;  # HTTP/3 on UDP
        
        server_name example.com;
        
        ssl_certificate /etc/ssl/certs/example.com.crt;
        ssl_certificate_key /etc/ssl/private/example.com.key;
        
        # Advertise HTTP/3 support
        add_header Alt-Svc 'h3=":443"; ma=86400';
        
        # QUIC specific
        ssl_early_data on;  # 0-RTT
        
        location / {
            root /var/www/html;
        }
    }
}
```

**Spring Boot with HTTP/2**:

```yaml
# application.yml
server:
  port: 8443
  http2:
    enabled: true
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
```

```java
// For embedded Tomcat with HTTP/2
@Configuration
public class Http2Config {
    
    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> 
            tomcatCustomizer() {
        return factory -> {
            factory.addConnectorCustomizers(connector -> {
                connector.addUpgradeProtocol(new Http2Protocol());
            });
        };
    }
}
```

### Checking HTTP Version

```bash
# Check HTTP version with curl
curl -I --http2 https://example.com
# Look for: HTTP/2 200

curl -I --http3 https://cloudflare.com
# Look for: HTTP/3 200

# Check with Chrome DevTools
# Network tab → Right-click headers → Protocol column
# Shows h2 (HTTP/2) or h3 (HTTP/3)

# Check Alt-Svc header (advertises HTTP/3)
curl -I https://google.com | grep -i alt-svc
# alt-svc: h3=":443"; ma=2592000
```

---

## 6️⃣ How to Implement HTTP Clients in Java

### HTTP/1.1 Client (java.net.HttpURLConnection)

```java
import java.io.*;
import java.net.*;

public class Http11Client {
    
    public String get(String urlString) throws IOException {
        URL url = new URL(urlString);
        HttpURLConnection connection = (HttpURLConnection) url.openConnection();
        
        try {
            // Configure request
            connection.setRequestMethod("GET");
            connection.setConnectTimeout(5000);
            connection.setReadTimeout(10000);
            connection.setRequestProperty("User-Agent", "Java HTTP/1.1 Client");
            
            // Get response
            int responseCode = connection.getResponseCode();
            System.out.println("Response Code: " + responseCode);
            System.out.println("Protocol: " + connection.getHeaderField(null)); // HTTP/1.1 200 OK
            
            // Read body
            try (BufferedReader reader = new BufferedReader(
                    new InputStreamReader(connection.getInputStream()))) {
                StringBuilder response = new StringBuilder();
                String line;
                while ((line = reader.readLine()) != null) {
                    response.append(line).append("\n");
                }
                return response.toString();
            }
        } finally {
            connection.disconnect();
        }
    }
    
    public static void main(String[] args) throws IOException {
        Http11Client client = new Http11Client();
        String response = client.get("https://httpbin.org/get");
        System.out.println(response);
    }
}
```

### HTTP/2 Client (java.net.http.HttpClient - Java 11+)

```java
import java.net.URI;
import java.net.http.*;
import java.time.Duration;
import java.util.concurrent.CompletableFuture;

public class Http2Client {
    
    private final HttpClient client;
    
    public Http2Client() {
        this.client = HttpClient.newBuilder()
            .version(HttpClient.Version.HTTP_2)  // Prefer HTTP/2
            .connectTimeout(Duration.ofSeconds(10))
            .followRedirects(HttpClient.Redirect.NORMAL)
            .build();
    }
    
    /**
     * Synchronous HTTP/2 GET request
     */
    public HttpResponse<String> get(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .header("User-Agent", "Java HTTP/2 Client")
            .GET()
            .build();
        
        HttpResponse<String> response = client.send(
            request, 
            HttpResponse.BodyHandlers.ofString()
        );
        
        System.out.println("Protocol: " + response.version()); // HTTP_2
        System.out.println("Status: " + response.statusCode());
        
        return response;
    }
    
    /**
     * Asynchronous HTTP/2 GET request
     */
    public CompletableFuture<HttpResponse<String>> getAsync(String url) {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .GET()
            .build();
        
        return client.sendAsync(request, HttpResponse.BodyHandlers.ofString());
    }
    
    /**
     * Parallel requests using HTTP/2 multiplexing
     */
    public void parallelRequests(String... urls) {
        // All requests go over single HTTP/2 connection (multiplexed)
        var futures = java.util.Arrays.stream(urls)
            .map(this::getAsync)
            .toList();
        
        // Wait for all to complete
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenRun(() -> {
                futures.forEach(f -> {
                    try {
                        HttpResponse<String> resp = f.get();
                        System.out.println(resp.uri() + " → " + resp.statusCode());
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                });
            })
            .join();
    }
    
    /**
     * POST request with JSON body
     */
    public HttpResponse<String> postJson(String url, String json) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .header("Content-Type", "application/json")
            .POST(HttpRequest.BodyPublishers.ofString(json))
            .build();
        
        return client.send(request, HttpResponse.BodyHandlers.ofString());
    }
    
    public static void main(String[] args) throws Exception {
        Http2Client client = new Http2Client();
        
        // Single request
        HttpResponse<String> response = client.get("https://nghttp2.org/httpbin/get");
        System.out.println("Body: " + response.body().substring(0, 100) + "...");
        
        // Parallel requests (multiplexed over single connection)
        System.out.println("\nParallel requests:");
        client.parallelRequests(
            "https://nghttp2.org/httpbin/get",
            "https://nghttp2.org/httpbin/headers",
            "https://nghttp2.org/httpbin/ip"
        );
    }
}
```

### HTTP/3 Client (using Jetty)

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.eclipse.jetty.http3</groupId>
        <artifactId>http3-client</artifactId>
        <version>12.0.3</version>
    </dependency>
    <dependency>
        <groupId>org.eclipse.jetty</groupId>
        <artifactId>jetty-client</artifactId>
        <version>12.0.3</version>
    </dependency>
</dependencies>
```

```java
import org.eclipse.jetty.client.HttpClient;
import org.eclipse.jetty.client.ContentResponse;
import org.eclipse.jetty.http3.client.HTTP3Client;
import org.eclipse.jetty.http3.client.transport.HttpClientTransportOverHTTP3;

public class Http3Client {
    
    public void makeRequest() throws Exception {
        // Create HTTP/3 transport
        HTTP3Client h3Client = new HTTP3Client();
        HttpClientTransportOverHTTP3 transport = 
            new HttpClientTransportOverHTTP3(h3Client);
        
        // Create HTTP client with HTTP/3 transport
        HttpClient client = new HttpClient(transport);
        client.start();
        
        try {
            // Make request (will use QUIC/HTTP3)
            ContentResponse response = client.GET("https://cloudflare.com");
            
            System.out.println("Status: " + response.getStatus());
            System.out.println("Content: " + response.getContentAsString()
                .substring(0, 200) + "...");
        } finally {
            client.stop();
        }
    }
    
    public static void main(String[] args) throws Exception {
        new Http3Client().makeRequest();
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Pitfall 1: Assuming HTTP/2 is Always Faster

**Scenario**: Migrating from HTTP/1.1 to HTTP/2.

**Mistake**: Expecting automatic performance improvement.

**Reality**: HTTP/2 can be slower if:
- TLS is not already in use (HTTP/2 requires HTTPS in browsers)
- Server is far away (single connection = single TCP slow start)
- Resources are already optimized for HTTP/1.1 (bundled, sprited)

**Solution**: 
- Unbundle resources (HTTP/2 handles many small files well)
- Use Server Push carefully (can waste bandwidth)
- Test actual performance, don't assume

### Pitfall 2: Over-using Server Push

**Scenario**: Pushing all resources to speed up page load.

**Mistake**: 
```nginx
http2_push /css/style.css;
http2_push /js/app.js;
http2_push /js/vendor.js;
http2_push /images/logo.png;
# ... pushing everything
```

**Problem**: 
- Browser may already have resources cached
- Pushed resources can't be cancelled once started
- Wastes bandwidth on repeat visits

**Solution**:
- Use Push sparingly for critical resources
- Implement cache digests (experimental)
- Consider `<link rel="preload">` instead

### Pitfall 3: Not Handling Protocol Negotiation

**Scenario**: Client expects HTTP/2, server only supports HTTP/1.1.

**Mistake**: Hard-coding HTTP/2 expectations.

**Solution**: Always handle fallback gracefully.

```java
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)  // Prefer HTTP/2
    .build();

HttpResponse<String> response = client.send(request, handler);

// Check actual protocol used
if (response.version() == HttpClient.Version.HTTP_1_1) {
    System.out.println("Server doesn't support HTTP/2, fell back to HTTP/1.1");
}
```

### Pitfall 4: Ignoring HTTP/3 Firewall Issues

**Scenario**: Deploying HTTP/3 in enterprise environment.

**Mistake**: Assuming UDP port 443 is open.

**Problem**: Many corporate firewalls block UDP or only allow TCP 443.

**Solution**:
- Always have HTTP/2 fallback
- Use Alt-Svc header to advertise HTTP/3
- Browser will try HTTP/3, fall back to HTTP/2 automatically

```nginx
# Advertise HTTP/3, but serve HTTP/2 as fallback
add_header Alt-Svc 'h3=":443"; ma=86400, h2=":443"; ma=86400';
```

### Pitfall 5: TCP Tuning for HTTP/2

**Scenario**: HTTP/2 performance is worse than expected.

**Mistake**: Using default TCP settings.

**Problem**: HTTP/2 uses single connection, so TCP settings matter more:
- Small initial congestion window = slow start hurts more
- Small receive buffer = can't utilize full bandwidth

**Solution**: Tune TCP for HTTP/2:
```bash
# Increase initial congestion window
ip route change default via <gateway> initcwnd 10 initrwnd 10

# Increase buffer sizes
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
```

---

## 8️⃣ When NOT to Use Each Version

### When NOT to Use HTTP/2

| Scenario | Why | Alternative |
|----------|-----|-------------|
| No HTTPS | Browsers require TLS for HTTP/2 | HTTP/1.1 or add TLS |
| Very small payloads | Header overhead not worth it | HTTP/1.1 |
| Highly lossy networks | TCP HOL blocking hurts | HTTP/3 |
| Legacy proxy in path | May not support HTTP/2 | HTTP/1.1 |

### When NOT to Use HTTP/3

| Scenario | Why | Alternative |
|----------|-----|-------------|
| UDP blocked | Firewalls block UDP 443 | HTTP/2 |
| Server doesn't support | Limited server support | HTTP/2 |
| Debugging needed | QUIC harder to inspect | HTTP/2 |
| Low-latency LAN | Benefits minimal | HTTP/2 |

### When to Stick with HTTP/1.1

| Scenario | Why |
|----------|-----|
| Simple APIs | Overhead not justified |
| Internal services | Complexity not needed |
| Legacy integration | Compatibility required |
| Debugging/learning | Easier to understand |

---

## 9️⃣ Comparison Table

| Feature | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|----------|--------|--------|
| **Year** | 1996 | 1997 | 2015 | 2022 |
| **Transport** | TCP | TCP | TCP | QUIC (UDP) |
| **Connections** | New per request | Persistent | Single multiplexed | Single multiplexed |
| **Multiplexing** | No | No (pipelining failed) | Yes | Yes |
| **Header Compression** | No | No | HPACK | QPACK |
| **Server Push** | No | No | Yes | Yes |
| **TLS Required** | No | No | Effectively yes | Yes (built-in) |
| **HOL Blocking** | Per connection | Per connection | TCP level | None |
| **Connection Migration** | No | No | No | Yes |
| **0-RTT** | No | No | No | Yes |

---

## 🔟 Interview Follow-Up Questions

### L4 (Junior/Mid) Level Questions

**Q1: What is the difference between HTTP/1.1 and HTTP/2?**

**A**: 
- **HTTP/1.1**: Text-based, one request at a time per connection (or pipelining which failed), separate connections needed for parallelism
- **HTTP/2**: Binary protocol, multiplexes many requests over single connection, header compression (HPACK), optional server push

Key benefit: HTTP/2 eliminates head-of-line blocking at HTTP level, reduces connection overhead, and compresses repetitive headers.

**Q2: What is multiplexing in HTTP/2?**

**A**: Multiplexing allows multiple HTTP requests and responses to be sent simultaneously over a single TCP connection. Each request/response is assigned a stream ID, and frames from different streams can be interleaved. This eliminates the need for multiple connections and removes HTTP-level head-of-line blocking.

**Q3: Why does HTTP/2 require HTTPS?**

**A**: Technically, HTTP/2 spec allows unencrypted connections (h2c). However, all major browsers only implement HTTP/2 over TLS. Reasons:
1. Security by default
2. Easier deployment (TLS already handles protocol negotiation via ALPN)
3. Avoids middlebox interference (proxies that don't understand HTTP/2)

### L5 (Senior) Level Questions

**Q4: Explain head-of-line blocking in HTTP/2 and how HTTP/3 solves it.**

**A**: HTTP/2 multiplexes streams over a single TCP connection. TCP sees this as one byte stream. If a packet is lost, TCP buffers all subsequent packets until retransmission completes, even if those packets belong to different HTTP streams. This is TCP-level head-of-line blocking.

HTTP/3 uses QUIC over UDP. QUIC implements its own reliability per-stream. If a packet for Stream A is lost, only Stream A waits for retransmission. Streams B and C continue unaffected. This is especially beneficial on lossy networks (mobile, WiFi).

**Q5: How would you decide between HTTP/1.1, HTTP/2, and HTTP/3 for a new service?**

**A**: Decision framework:

1. **Start with HTTP/2** as default for web services:
   - Widely supported
   - Significant performance benefits
   - Easy to deploy (just enable in web server)

2. **Consider HTTP/3** if:
   - Mobile users are significant portion
   - Users on lossy/high-latency networks
   - Connection migration matters (users switching networks)
   - You control both client and server

3. **Stick with HTTP/1.1** if:
   - Internal service-to-service (simpler, debugging easier)
   - Legacy integration required
   - Very simple API with few requests

4. **Always have fallback**: HTTP/3 → HTTP/2 → HTTP/1.1

### L6 (Staff+) Level Questions

**Q6: Design the HTTP layer for a global CDN.**

**A**: Key considerations:

1. **Protocol Support**:
   - HTTP/3 primary (lowest latency, best for mobile)
   - HTTP/2 fallback (when UDP blocked)
   - HTTP/1.1 fallback (legacy clients)
   - Use Alt-Svc to advertise HTTP/3

2. **Connection Management**:
   - Long-lived connections to origin (connection pooling)
   - HTTP/2 multiplexing to origin servers
   - QUIC between edge locations for faster internal communication

3. **Optimization**:
   - Early hints (103) to preload critical resources
   - Server Push for known dependencies
   - Brotli compression
   - Edge-side includes for partial caching

4. **Security**:
   - TLS 1.3 everywhere
   - 0-RTT with replay protection
   - Certificate management at scale (Let's Encrypt automation)

5. **Observability**:
   - Protocol version metrics
   - QUIC vs TCP performance comparison
   - Connection migration success rates

**Q7: A service experiences intermittent slowdowns only for HTTP/2 clients. How do you diagnose?**

**A**: Systematic approach:

1. **Check for TCP issues**:
   - Single connection means TCP problems affect all streams
   - Check for packet loss (triggers retransmissions)
   - Check congestion window (slow start on new connections)

2. **Check for stream prioritization issues**:
   - Low-priority streams blocking high-priority?
   - Server respecting priority hints?

3. **Check for flow control issues**:
   - WINDOW_UPDATE frames being sent?
   - Connection or stream window exhausted?

4. **Check for HPACK issues**:
   - Dynamic table size appropriate?
   - Header compression ratio healthy?

5. **Compare with HTTP/1.1**:
   - If HTTP/1.1 is faster, likely TCP HOL blocking
   - Consider HTTP/3 or multiple HTTP/2 connections

6. **Check server resources**:
   - HTTP/2 more CPU intensive (binary parsing, HPACK)
   - Memory for connection state

---

## 1️⃣1️⃣ Mental Summary

**HTTP has evolved to match the modern web's needs**. HTTP/1.0 was simple but inefficient. HTTP/1.1 added persistent connections but still suffered from head-of-line blocking. HTTP/2 introduced multiplexing and header compression but is limited by TCP. HTTP/3 moves to QUIC over UDP, eliminating TCP's limitations.

**Key insight**: Each version solves problems of the previous while introducing new tradeoffs. HTTP/2 is the sweet spot for most applications today. HTTP/3 is the future, especially for mobile and lossy networks.

**For production**: Enable HTTP/2 by default, add HTTP/3 for mobile-heavy traffic, always have fallbacks. Monitor protocol distribution and performance metrics.

**For interviews**: Understand the progression and WHY each version exists. Be able to explain multiplexing, head-of-line blocking, and when to choose each version.

---

## 📚 Further Reading

- [HTTP/2 RFC 7540](https://tools.ietf.org/html/rfc7540)
- [HTTP/3 RFC 9114](https://www.rfc-editor.org/rfc/rfc9114.html)
- [High Performance Browser Networking](https://hpbn.co/) - Ilya Grigorik
- [Cloudflare HTTP/3 Blog](https://blog.cloudflare.com/http3-the-past-present-and-future/)
- [Google QUIC Protocol](https://www.chromium.org/quic/)
- [HTTP/2 Explained](https://http2-explained.haxx.se/) - Daniel Stenberg (curl author)

