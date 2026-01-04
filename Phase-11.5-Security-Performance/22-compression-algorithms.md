# Compression Algorithms

## 0️⃣ Prerequisites

- Understanding of HTTP basics (headers, request/response)
- Basic knowledge of data structures (strings, binary data)
- Understanding of performance optimization concepts
- Basic familiarity with web servers and CDNs

**Quick refresher**: HTTP compression reduces the size of data transferred over the network by encoding data more efficiently. This reduces bandwidth usage and improves transfer speeds, especially for text-based content like HTML, CSS, JavaScript, and JSON.

## 1️⃣ What problem does this exist to solve?

### The Pain Point

Without compression:

1. **Large file sizes**: HTML, CSS, JavaScript, JSON files are often large (hundreds of KB to MB)
2. **Slow transfers**: Large files take time to transfer, especially on slow connections
3. **Bandwidth costs**: More data transferred means higher bandwidth costs
4. **Poor mobile experience**: Slow connections (3G, 4G) make sites feel sluggish
5. **Wasted bandwidth**: Much of web content is redundant or repetitive

### What Systems Look Like Without Compression

Without compression:

- **1MB HTML file takes 8 seconds** on 1 Mbps connection
- **Mobile users wait 20+ seconds** for page to load
- **Bandwidth bills are 5-10x higher** than necessary
- **CDN costs scale with traffic** unnecessarily
- **API responses are slow** even for small payloads

### Real Examples

**Example 1**: E-commerce site product page. HTML + CSS + JavaScript = 2MB uncompressed. With gzip compression = 400KB. Result: Page loads 5x faster, especially on mobile.

**Example 2**: API returning JSON. Response is 500KB uncompressed, 50KB with gzip. Result: API responds 10x faster, reduces bandwidth costs by 90%.

**Example 3**: Single Page Application (SPA). JavaScript bundle is 3MB uncompressed, 600KB with gzip. Result: Initial page load 5x faster, better user experience.

## 2️⃣ Intuition and Mental Model

**Think of compression like packing a suitcase:**

Without compression, you put everything in as-is - takes up lots of space. With compression:
- **Remove air**: Eliminate empty space (remove redundancy)
- **Fold clothes**: Represent data more efficiently (encode patterns)
- **Use packing cubes**: Group related items (dictionary-based compression)
- **Result**: Same contents, much smaller space

**The mental model**: Compression finds patterns and redundancy in data, then represents it more efficiently. Text has lots of redundancy (repeated words, common patterns), so it compresses well. Already-compressed data (images, videos) or random data compresses poorly.

**Another analogy**: Think of compression like shorthand writing. Instead of writing "the quick brown fox jumps over the lazy dog" every time, you write it once and reference it. Compression does the same - finds repeated patterns and references them.

## 3️⃣ How it works internally

### Compression Basics

#### 1. Lossless vs Lossy Compression

**Lossless compression**: Original data can be perfectly reconstructed
- **Use case**: Text, code, JSON, configuration files
- **Algorithms**: gzip, deflate, brotli
- **Tradeoff**: Compression ratio vs CPU time

**Lossy compression**: Some data is lost, original can't be perfectly reconstructed
- **Use case**: Images (JPEG), audio (MP3), video (MP4)
- **Algorithms**: JPEG, MP3, H.264
- **Tradeoff**: Quality vs file size

#### 2. How Compression Works

**Step 1: Pattern Recognition**
Compressor finds repeated patterns in data:
```
Original: "the quick brown fox jumps over the lazy dog. the quick brown fox jumps over the lazy dog."
Patterns: "the quick brown fox jumps over the lazy dog." appears twice
```

**Step 2: Dictionary Building**
Build dictionary of patterns:
```
Dictionary:
1: "the quick brown fox jumps over the lazy dog."
2: "the "
3: " quick "

Compressed: [1][1]  (reference dictionary entries)
```

**Step 3: Encoding**
Encode using dictionary references (much shorter):
```
Original: 100 characters
Compressed: 2 dictionary references + dictionary = ~30 characters
Compression ratio: ~70% reduction
```

### Compression Algorithms

#### 1. gzip (GNU zip)

**Based on**: DEFLATE algorithm (LZ77 + Huffman coding)
**How it works**:
1. **LZ77**: Find repeated strings, replace with references to previous occurrences
2. **Huffman coding**: Encode frequent characters with shorter codes

**Example**:
```
Original: "the quick brown fox jumps over the lazy dog"
          "the quick brown fox jumps over the lazy dog"

LZ77: "the quick brown fox jumps over the lazy dog"
      [reference to previous 43 chars]  (back reference)

Huffman: Frequent characters (space, 'e', 'o') get short codes
         Less frequent characters get longer codes
```

**Characteristics**:
- Good compression ratio for text (60-80%)
- Fast compression and decompression
- Widely supported (all browsers, servers)
- CPU usage: Moderate

#### 2. deflate

**Relationship to gzip**: gzip = deflate + headers + checksum
**Use case**: HTTP compression, PNG images
**Characteristics**: Same algorithm as gzip, just different container

#### 3. Brotli

**Based on**: LZ77 + Huffman coding + context modeling
**How it works**: 
1. Similar to gzip but with better algorithms
2. Uses second-order context modeling (considers previous characters)
3. Better dictionary management

**Example**:
```
Original: "compression" appears 100 times
gzip: References it each time (good)
Brotli: Better context modeling, even better references (better)
```

**Characteristics**:
- Better compression ratio than gzip (15-25% better)
- Slower compression (2-5x slower)
- Fast decompression (similar to gzip)
- Good browser support (modern browsers)

#### 4. Compression Levels

**Tradeoff**: Compression ratio vs CPU time

**Levels** (0-9, higher = better compression, slower):
- **Level 1-3**: Fast compression, lower ratio
- **Level 4-6**: Balanced (default)
- **Level 7-9**: Slow compression, higher ratio

**Example**: 1MB file
```
Level 1: 500KB, 10ms to compress
Level 6: 400KB, 50ms to compress
Level 9: 350KB, 200ms to compress
```

**Best practice**: Level 6 for most cases (good balance)

## 4️⃣ Simulation-first explanation

### Scenario: Compressing HTML Response

**Start**: Server needs to send 100KB HTML file to client

**Without compression**:
```
Server: 100KB HTML
Network: 100KB transferred
Client: Receives 100KB, displays page
Time: 800ms on 1 Mbps connection
```

**With gzip compression**:
```
Server: 100KB HTML
Server: Compress with gzip (5ms CPU time)
Server: 20KB compressed HTML
Network: 20KB transferred
Client: Receives 20KB
Client: Decompress (2ms CPU time)
Client: 100KB HTML, displays page
Time: 160ms transfer + 7ms CPU = 167ms total

Improvement: 800ms → 167ms (4.8x faster)
```

**Breakdown**:
- Compression time: 5ms (server CPU)
- Network transfer: 160ms (80ms saved)
- Decompression time: 2ms (client CPU)
- **Net benefit**: 633ms saved (79% faster)

### Scenario: API JSON Response

**JSON response**: 500KB of product data

**Without compression**:
```
Response: 500KB JSON
Transfer time: 4 seconds (1 Mbps)
```

**With gzip**:
```
Response: 50KB compressed JSON
Transfer time: 400ms
Decompression: 3ms
Total: 403ms

Improvement: 4 seconds → 403ms (10x faster)
```

**Why JSON compresses well**: Lots of repetition
```json
{
  "products": [
    {"id": 1, "name": "Product", "price": 10},
    {"id": 2, "name": "Product", "price": 20},
    ...
  ]
}
```
Repeated patterns: `"id":`, `"name":`, `"price":`, `"Product"` → Compresses to ~10% of original

### Scenario: When Compression Doesn't Help

**Already-compressed image**: JPEG file (already compressed)
```
Original: 500KB JPEG
gzip attempt: 495KB (minimal compression)
Result: Not worth CPU time for 1% reduction
```

**Random data**: No patterns to compress
```
Original: 100KB random binary
gzip attempt: 99KB (minimal compression)
Result: Compression overhead > benefit
```

## 5️⃣ How engineers actually use this in production

### Real-World Practices

#### 1. Web Server Configuration

**Nginx**: Enable gzip for text content
```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000;
gzip_comp_level 6;
```

**Apache**: Enable mod_deflate
```apache
<Location />
    SetOutputFilter DEFLATE
    SetEnvIfNoCase Request_URI \.(?:gif|jpe?g|png)$ no-gzip
</Location>
```

#### 2. CDN Compression

**CloudFlare, AWS CloudFront**: Automatic compression
- Compresses at edge
- Caches compressed versions
- Reduces origin load

#### 3. Application-Level Compression

**Spring Boot**: Automatic compression
```yaml
server:
  compression:
    enabled: true
    mime-types: text/html,text/xml,text/plain,text/css,text/javascript,application/javascript,application/json
    min-response-size: 1024
```

#### 4. API Compression

**REST APIs**: Compress JSON/XML responses
- Automatic based on Accept-Encoding header
- Client indicates support: `Accept-Encoding: gzip, br`
- Server compresses if supported

### Production Workflow

**Step 1: Enable compression**
- Configure web server or application
- Choose compression algorithm (gzip or brotli)
- Set compression level (usually 6)

**Step 2: Verify**
- Check response headers: `Content-Encoding: gzip`
- Verify file size reduction
- Test browser compatibility

**Step 3: Monitor**
- Track compression ratios
- Monitor CPU usage
- Measure performance improvements

## 6️⃣ How to implement or apply it

### Java/Spring Boot Implementation

#### Example 1: Spring Boot Compression

**Configuration**:
```yaml
# application.yml
server:
  compression:
    enabled: true
    mime-types: 
      - text/html
      - text/xml
      - text/plain
      - text/css
      - text/javascript
      - application/javascript
      - application/json
      - application/xml
    min-response-size: 1024  # Only compress responses > 1KB
```

**Custom compression filter**:
```java
@Configuration
public class CompressionConfig {
    
    @Bean
    public FilterRegistrationBean<GzipFilter> gzipFilter() {
        FilterRegistrationBean<GzipFilter> registration = 
            new FilterRegistrationBean<>();
        registration.setFilter(new GzipFilter());
        registration.addUrlPatterns("/*");
        registration.setName("gzipFilter");
        registration.setOrder(1);
        return registration;
    }
}

public class GzipFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String acceptEncoding = httpRequest.getHeader("Accept-Encoding");
        
        if (acceptEncoding != null && acceptEncoding.contains("gzip")) {
            GzipResponseWrapper wrappedResponse = new GzipResponseWrapper(httpResponse);
            chain.doFilter(request, wrappedResponse);
            wrappedResponse.finish();
        } else {
            chain.doFilter(request, response);
        }
    }
}
```

#### Example 2: Response Compression Interceptor

**Compression interceptor**:
```java
@Component
public class CompressionInterceptor implements HandlerInterceptor {
    
    @Override
    public void postHandle(HttpServletRequest request, 
                          HttpServletResponse response, 
                          Object handler, 
                          ModelAndView modelAndView) {
        String acceptEncoding = request.getHeader("Accept-Encoding");
        String contentType = response.getContentType();
        
        // Only compress text-based content
        if (shouldCompress(contentType) && 
            acceptEncoding != null && 
            acceptEncoding.contains("gzip")) {
            
            response.setHeader("Content-Encoding", "gzip");
            response.setHeader("Vary", "Accept-Encoding");
        }
    }
    
    private boolean shouldCompress(String contentType) {
        if (contentType == null) {
            return false;
        }
        return contentType.startsWith("text/") ||
               contentType.contains("json") ||
               contentType.contains("xml") ||
               contentType.contains("javascript");
    }
}
```

#### Example 3: Manual Compression

**Compress response manually**:
```java
@RestController
public class ApiController {
    
    @GetMapping(value = "/api/data", produces = "application/json")
    public ResponseEntity<byte[]> getData(
            @RequestHeader(value = "Accept-Encoding", required = false) 
            String acceptEncoding) {
        
        // Generate response
        String jsonData = generateJsonData();
        byte[] data = jsonData.getBytes(StandardCharsets.UTF_8);
        
        // Compress if client supports it
        if (acceptEncoding != null && acceptEncoding.contains("gzip")) {
            byte[] compressed = compressGzip(data);
            return ResponseEntity.ok()
                .header("Content-Encoding", "gzip")
                .header("Content-Type", "application/json")
                .body(compressed);
        }
        
        return ResponseEntity.ok()
            .header("Content-Type", "application/json")
            .body(data);
    }
    
    private byte[] compressGzip(byte[] data) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try (GZIPOutputStream gzipOut = new GZIPOutputStream(baos)) {
            gzipOut.write(data);
        }
        return baos.toByteArray();
    }
}
```

#### Example 4: Brotli Compression

**Maven dependency**:
```xml
<dependency>
    <groupId>com.aayushatharva.brotli4j</groupId>
    <artifactId>brotli4j</artifactId>
    <version>1.11.0</version>
</dependency>
```

**Brotli compression**:
```java
@Service
public class BrotliCompressionService {
    
    public byte[] compress(byte[] data) throws IOException {
        BrotliOutputStream brotliOut = new BrotliOutputStream(
            new ByteArrayOutputStream(),
            new Brotli.Parameters.Builder().build()
        );
        brotliOut.write(data);
        brotliOut.close();
        return ((ByteArrayOutputStream) brotliOut.getOutputStream()).toByteArray();
    }
    
    public byte[] decompress(byte[] compressed) throws IOException {
        ByteArrayInputStream bais = new ByteArrayInputStream(compressed);
        BrotliInputStream brotliIn = new BrotliInputStream(bais);
        return brotliIn.readAllBytes();
    }
}
```

**Response with Brotli**:
```java
@GetMapping("/api/data")
public ResponseEntity<byte[]> getData(
        @RequestHeader(value = "Accept-Encoding", required = false) 
        String acceptEncoding) {
    
    byte[] data = generateData();
    
    // Prefer Brotli over gzip
    if (acceptEncoding != null && acceptEncoding.contains("br")) {
        byte[] compressed = brotliService.compress(data);
        return ResponseEntity.ok()
            .header("Content-Encoding", "br")
            .body(compressed);
    } else if (acceptEncoding != null && acceptEncoding.contains("gzip")) {
        byte[] compressed = compressGzip(data);
        return ResponseEntity.ok()
            .header("Content-Encoding", "gzip")
            .body(compressed);
    }
    
    return ResponseEntity.ok().body(data);
}
```

#### Example 5: Compression with Caching

**Cache compressed responses**:
```java
@Service
public class CompressedCacheService {
    
    @Autowired
    private CacheManager cacheManager;
    
    public byte[] getOrCompress(String key, Supplier<byte[]> dataSupplier, 
                                String acceptEncoding) throws IOException {
        String cacheKey = key + ":" + acceptEncoding;
        
        Cache cache = cacheManager.getCache("compressed");
        Cache.ValueWrapper wrapper = cache.get(cacheKey);
        
        if (wrapper != null) {
            return (byte[]) wrapper.get();
        }
        
        byte[] data = dataSupplier.get();
        byte[] compressed;
        String encoding;
        
        if (acceptEncoding != null && acceptEncoding.contains("br")) {
            compressed = compressBrotli(data);
            encoding = "br";
        } else if (acceptEncoding != null && acceptEncoding.contains("gzip")) {
            compressed = compressGzip(data);
            encoding = "gzip";
        } else {
            cache.put(cacheKey, data);
            return data;
        }
        
        cache.put(cacheKey, compressed);
        return compressed;
    }
}
```

## 7️⃣ Tradeoffs, pitfalls, and common mistakes

### Tradeoffs

#### 1. Compression Ratio vs CPU Time
- **Higher compression**: Smaller files, but more CPU time
- **Lower compression**: Larger files, but less CPU time
- **Sweet spot**: Level 6 for most cases

#### 2. Compression vs Caching
- **Compress on-the-fly**: No storage, but CPU cost per request
- **Pre-compress and cache**: Storage cost, but no CPU cost per request
- **Best practice**: Pre-compress and cache for static content

#### 3. Algorithm Choice: gzip vs Brotli
- **gzip**: Fast, widely supported, good compression
- **Brotli**: Better compression, slower, modern browsers only
- **Best practice**: Use Brotli when supported, fallback to gzip

### Common Pitfalls

#### Pitfall 1: Compressing Already-Compressed Data
**Problem**: Compressing images, videos, PDFs wastes CPU
```java
// Bad: Compressing JPEG
gzip.compress(jpegImage); // JPEG is already compressed!
```

**Solution**: Only compress text-based content
```java
// Good: Check content type
if (contentType.startsWith("text/") || contentType.contains("json")) {
    compress(response);
}
```

#### Pitfall 2: Double Compression
**Problem**: Compressing already-compressed data
```java
// Bad: Compressing gzipped data again
byte[] compressed = gzip.compress(data);
byte[] doubleCompressed = gzip.compress(compressed); // Wastes CPU, minimal benefit
```

**Solution**: Check if already compressed
```java
// Good: Check headers
if (!response.containsHeader("Content-Encoding")) {
    compress(response);
}
```

#### Pitfall 3: Not Setting Vary Header
**Problem**: Cached compressed response served to client that doesn't support compression
```java
// Bad: Missing Vary header
response.setHeader("Content-Encoding", "gzip");
// Cache serves gzip to client without Accept-Encoding: gzip
```

**Solution**: Set Vary header
```java
// Good: Proper caching
response.setHeader("Content-Encoding", "gzip");
response.setHeader("Vary", "Accept-Encoding");
```

#### Pitfall 4: Compressing Small Files
**Problem**: Compression overhead > benefit for small files
```java
// Bad: Compressing 100-byte response
if (data.length < 1000) {
    compress(data); // Overhead > benefit
}
```

**Solution**: Only compress files above threshold
```java
// Good: Minimum size threshold
if (data.length > 1024) { // Only compress > 1KB
    compress(data);
}
```

#### Pitfall 5: High Compression Level in Production
**Problem**: Level 9 compression is very slow
```java
// Bad: Maximum compression in production
gzip.setLevel(9); // Very slow, minimal benefit over level 6
```

**Solution**: Use balanced level
```java
// Good: Balanced compression
gzip.setLevel(6); // Good balance of speed and ratio
```

### Performance Gotchas

#### Gotcha 1: CPU Overhead
**Problem**: Compression consumes CPU, can become bottleneck
**Solution**: Pre-compress static content, use appropriate compression level

#### Gotcha 2: Memory Usage
**Problem**: Compression buffers consume memory
**Solution**: Stream compression for large files, limit buffer sizes

## 8️⃣ When NOT to use this

### When Compression Isn't Needed

1. **Already-compressed content**: Images (JPEG, PNG), videos, PDFs
2. **Very small files**: Compression overhead > benefit (< 1KB)
3. **Random/binary data**: Doesn't compress well
4. **Real-time streaming**: Compression adds latency

### Anti-Patterns

1. **Compressing everything**: Wastes CPU on non-compressible content
2. **Maximum compression level**: Slow, minimal benefit
3. **No Vary header**: Causes caching issues

## 9️⃣ Comparison with Alternatives

### gzip vs Brotli

**gzip**: DEFLATE algorithm
- Pros: Fast, widely supported, good compression
- Cons: Lower compression ratio than Brotli

**Brotli**: Advanced algorithm
- Pros: Better compression (15-25% better), good browser support
- Cons: Slower compression, newer (less universal support)

**Best practice**: Use Brotli when supported, fallback to gzip

### Compression vs Minification

**Compression**: Reduces file size by encoding efficiently
- Works on: Any content
- Reduces: Transfer size
- Reversible: Yes (decompression)

**Minification**: Removes whitespace, shortens names
- Works on: Code (JavaScript, CSS)
- Reduces: Source file size
- Reversible: No (information lost)

**Best practice**: Use both - minify then compress

### Server-Side vs Client-Side Compression

**Server-side**: Compress before sending
- Pros: Reduces bandwidth, faster transfer
- Cons: CPU cost on server

**Client-side**: Compress on client
- Pros: No server CPU cost
- Cons: More complex, limited use cases

**Best practice**: Server-side compression (standard approach)

## 🔟 Interview follow-up questions WITH answers

### Question 1: When should you compress responses and when shouldn't you?

**Answer**:
**Should compress**:
1. **Text-based content**: HTML, CSS, JavaScript, JSON, XML
   - High redundancy, compresses well (60-80% reduction)

2. **Large files**: Files > 1KB (compression overhead worth it)

3. **Static content**: Can pre-compress and cache

4. **API responses**: JSON/XML responses benefit significantly

**Shouldn't compress**:
1. **Already-compressed**: Images (JPEG, PNG), videos, PDFs
   - Already optimized, won't compress further
   - Wastes CPU

2. **Very small files**: < 1KB
   - Compression overhead > benefit
   - May actually increase size (headers)

3. **Random/binary data**: No patterns to compress
   - Won't compress well
   - Wastes CPU

4. **Real-time streaming**: Compression adds latency
   - Not suitable for low-latency requirements

### Question 2: How does HTTP compression work end-to-end?

**Answer**:
**Client request**:
```
GET /api/data HTTP/1.1
Host: api.example.com
Accept-Encoding: gzip, br
```

**Server response**:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Encoding: gzip
Vary: Accept-Encoding

[gzipped JSON data]
```

**Process**:
1. **Client**: Sends `Accept-Encoding` header indicating support
2. **Server**: Checks if compression supported and content type is compressible
3. **Server**: Compresses response body
4. **Server**: Sets `Content-Encoding: gzip` header
5. **Server**: Sets `Vary: Accept-Encoding` (important for caching)
6. **Network**: Transfers compressed data
7. **Client**: Receives compressed data
8. **Client**: Decompresses based on `Content-Encoding` header
9. **Application**: Uses decompressed data

**Key points**:
- Compression is transparent to application logic
- Client must indicate support via `Accept-Encoding`
- Server sets `Content-Encoding` to indicate compression used
- `Vary` header ensures correct cached responses

### Question 3: What's the difference between gzip and Brotli compression?

**Answer**:
**Algorithm differences**:

**gzip**:
- Based on DEFLATE (LZ77 + Huffman coding)
- Older, widely supported
- Good compression ratio (60-80% for text)
- Fast compression and decompression
- CPU usage: Moderate

**Brotli**:
- Based on LZ77 + Huffman + context modeling
- Newer algorithm (2015)
- Better compression ratio (15-25% better than gzip)
- Slower compression (2-5x slower)
- Fast decompression (similar to gzip)
- Modern browser support

**When to use**:
- **gzip**: Universal support, fast compression needed
- **Brotli**: Modern clients, better compression ratio desired, can pre-compress

**Best practice**: Use Brotli when supported, fallback to gzip

### Question 4: How do you handle compression in a microservices architecture?

**Answer**:
**Strategies**:

1. **API Gateway**: Compress at gateway level
   - Single point of configuration
   - Consistent compression across services
   - Reduces service-level configuration

2. **Service mesh**: Compression at mesh level
   - Transparent to services
   - Consistent across services

3. **Individual services**: Each service compresses
   - More control per service
   - Service-specific optimization
   - More configuration overhead

**Best practice**: Compress at API gateway or service mesh for consistency

**Considerations**:
- Content types to compress (configure centrally)
- Compression levels (service-specific if needed)
- Caching (gateway can cache compressed responses)
- Monitoring (track compression ratios, CPU usage)

### Question 5: How would you implement compression for a high-traffic API?

**Answer**:
**Optimizations**:

1. **Pre-compress static content**: Compress at build time, serve pre-compressed files
   ```java
   // Build time
   compress("api-docs.json", "api-docs.json.gz");
   
   // Runtime
   if (acceptEncoding.contains("gzip")) {
       return preCompressedFile;
   }
   ```

2. **Cache compressed responses**: Cache compressed versions
   ```java
   String cacheKey = data + ":gzip";
   byte[] compressed = cache.get(cacheKey);
   if (compressed == null) {
       compressed = compress(data);
       cache.put(cacheKey, compressed);
   }
   ```

3. **Async compression**: Compress asynchronously for large responses
   ```java
   CompletableFuture<byte[]> compressedFuture = 
       CompletableFuture.supplyAsync(() -> compress(largeData));
   ```

4. **CDN compression**: Let CDN handle compression
   - CloudFlare, AWS CloudFront compress automatically
   - Reduces origin server load

5. **Monitor and tune**:
   - Track compression ratios
   - Monitor CPU usage
   - Adjust compression levels based on metrics

### Question 6 (L5/L6): How would you design a compression system that handles different content types optimally?

**Answer**:
**Design considerations**:

1. **Content type analysis**: Different algorithms for different content types
   - **Text (HTML, CSS, JS)**: Brotli or gzip (high compression)
   - **JSON/XML**: Brotli or gzip (very high compression)
   - **Binary data**: Skip compression or use specialized algorithms
   - **Images**: Skip (already compressed)

2. **Adaptive compression levels**:
   - **Static content**: Higher level (9) - compress once, serve many times
   - **Dynamic content**: Lower level (6) - balance speed and ratio

3. **Multi-algorithm support**:
   ```java
   if (acceptEncoding.contains("br") && contentType.isText()) {
       return compressBrotli(data, level);
   } else if (acceptEncoding.contains("gzip")) {
       return compressGzip(data, level);
   }
   ```

4. **Content-aware optimization**:
   - **Large files**: Stream compression (don't load entire file in memory)
   - **Small files**: Skip compression (overhead > benefit)
   - **Repeated content**: Cache compressed version

5. **Monitoring and adaptation**:
   - Track compression ratios per content type
   - Monitor CPU usage
   - Adjust algorithms/levels based on metrics
   - A/B test different strategies

**Architecture**:
```
Request → Content Type Analyzer → Algorithm Selector
                                      ↓
                          Compression Engine (Brotli/gzip/None)
                                      ↓
                          Cache (if applicable)
                                      ↓
                          Response
```

## 1️⃣1️⃣ One clean mental summary

Compression algorithms reduce data size by finding patterns and redundancy, then encoding them more efficiently. HTTP compression (gzip, Brotli) dramatically reduces transfer sizes for text-based content (HTML, CSS, JavaScript, JSON), improving load times and reducing bandwidth costs. Key algorithms include gzip (fast, widely supported, good compression) and Brotli (better compression, slower, modern browsers). Compression works by the client indicating support via `Accept-Encoding` header, server compressing the response, setting `Content-Encoding` header, and client decompressing. Critical considerations include when to compress (text content, large files) vs when not to (already-compressed images, very small files), compression level tradeoffs (ratio vs CPU time), and proper HTTP headers (`Vary: Accept-Encoding` for caching). Always compress text-based content, use appropriate compression levels (typically 6), and consider pre-compressing and caching static content for optimal performance.

