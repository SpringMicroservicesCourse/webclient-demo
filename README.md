# webclient-demo

> Reactive HTTP client with WebClient and non-blocking asynchronous operations

[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.5-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Java](https://img.shields.io/badge/Java-21-orange.svg)](https://openjdk.org/)
[![Spring WebFlux](https://img.shields.io/badge/Spring%20WebFlux-6.2.2-blue.svg)](https://docs.spring.io/spring-framework/reference/web/webflux.html)
[![Netty](https://img.shields.io/badge/Netty-4.1.119-purple.svg)](https://netty.io/)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A comprehensive demonstration of using **Spring WebClient** for reactive, non-blocking HTTP requests, featuring Mono/Flux reactive streams, concurrent execution, Money type serialization, and Netty-based connection management.

## Features

- WebClient reactive HTTP client (NOT RestTemplate)
- Non-blocking asynchronous operations
- Mono (single object) and Flux (multiple objects) handling
- Concurrent request execution with CountDownLatch
- Reactor thread pool (`reactor-http-nio-*`)
- Money type serialization/deserialization (TWD currency)
- WebApplicationType.NONE (client-only mode)
- Netty HTTP client with DNS resolver for macOS ARM
- RESTful API integration (GET/POST operations)

## Tech Stack

- Spring Boot 3.4.5
- Spring WebFlux (WebClient)
- Project Reactor (Mono/Flux)
- Netty 4.1.119.Final
- Java 21
- Joda Money 2.0.2
- Lombok
- Maven 3.8+

## Getting Started

### Prerequisites

- JDK 21 or higher
- Maven 3.8+ (or use included Maven Wrapper)
- REST API server running on port 8080 (e.g., hateoas-waiter-service)

### Quick Start

**Step 1: Start REST API Server**

```bash
# Start hateoas-waiter-service (or any compatible REST API)
cd ../hateoas-waiter-service
mvn spring-boot:run
```

**Step 2: Verify REST API**

```bash
# Test coffee endpoint
curl http://localhost:8080/coffee/1

# Expected: Coffee JSON response
```

**Step 3: Run webclient-demo**

```bash
# Using Maven
./mvnw spring-boot:run

# Or using compiled JAR
./mvnw clean package
java -jar target/webclient-demo-0.0.1-SNAPSHOT.jar
```

## Configuration

### Application Properties

```properties
# WebClient base URL configuration (configured in Java code)
# baseUrl: http://localhost:8080

# No additional configuration required
# WebClient is configured via @Bean in WebclientDemoApplication.java
```

### WebClient Configuration

```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder.baseUrl("http://localhost:8080").build();
}
```

**Configuration Details:**
- `baseUrl`: Target REST API endpoint
- Automatic Netty client configuration
- Built-in connection pooling
- HTTP/2 support enabled

## Usage

### Application Flow

```
1. Spring Boot starts (WebApplicationType.NONE)
   ↓
2. WebClient configured with baseUrl
   ↓
3. ApplicationRunner executes:
   ├─ GET /coffee/1 (Mono<Coffee>) - Async on reactor-http-nio-2
   ├─ POST /coffee/ (Mono<Coffee>) - Async on reactor-http-nio-3
   └─ GET /coffee/ (Flux<Coffee>) - Sync on main thread
   ↓
4. CountDownLatch ensures ordering
   ↓
5. Application completes and exits
```

### Code Example

```java
@SpringBootApplication
@Slf4j
public class WebclientDemoApplication implements ApplicationRunner {

    private final WebClient webClient;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        CountDownLatch cdl = new CountDownLatch(2);

        // 1. Async GET request - Single coffee
        webClient.get()
                .uri("/coffee/{id}", 1)
                .accept(MediaType.APPLICATION_JSON)
                .retrieve()
                .bodyToMono(Coffee.class)
                .doOnError(t -> log.error("Error: ", t))
                .doFinally(s -> cdl.countDown())
                .subscribeOn(Schedulers.single())
                .subscribe(c -> log.info("Coffee 1: {}", c));

        // 2. Async POST request - Create coffee
        Mono<Coffee> americano = Mono.just(
                Coffee.builder()
                        .name("americano")
                        .price(Money.of(CurrencyUnit.of("TWD"), 125.00))
                        .build()
        );
        webClient.post()
                .uri("/coffee/")
                .body(americano, Coffee.class)
                .retrieve()
                .bodyToMono(Coffee.class)
                .doFinally(s -> cdl.countDown())
                .subscribeOn(Schedulers.single())
                .subscribe(c -> log.info("Coffee Created: {}", c));

        // Wait for async operations
        cdl.await();

        // 3. Sync GET request - Coffee list
        webClient.get()
                .uri("/coffee/")
                .retrieve()
                .bodyToFlux(Coffee.class)
                .toStream()
                .forEach(c -> log.info("Coffee in List: {}", c));
    }
}
```

### Sample Output (Actual Execution: 2025-10-27)

**Startup Phase:**
```
2025-10-27T21:17:42.708+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : Starting WebclientDemoApplication using Java 21.0.7 with PID 51746
2025-10-27T21:17:42.711+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : No active profile set, falling back to 1 default profile: "default"
2025-10-27T21:17:43.564+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : Started WebclientDemoApplication in 1.005 seconds (process running for 1.137)
```

**Business Execution:**
```
# 1. Async GET request (Thread: reactor-http-nio-2)
2025-10-27T21:17:43.832+08:00  INFO 51746 --- [ctor-http-nio-2] t.f.s.r.w.WebclientDemoApplication : Coffee 1: Coffee(id=1, name=espresso, price=TWD 100.00, createTime=Mon Oct 27 21:17:38 CST 2025, updateTime=Mon Oct 27 21:17:38 CST 2025)

# 2. Async POST request (Thread: reactor-http-nio-3)
2025-10-27T21:17:43.832+08:00  INFO 51746 --- [ctor-http-nio-3] t.f.s.r.w.WebclientDemoApplication : Coffee Created: Coffee(id=6, name=americano, price=TWD 125.00, createTime=Mon Oct 27 21:17:43 CST 2025, updateTime=Mon Oct 27 21:17:43 CST 2025)

# 3. Sync GET request - List (Thread: main)
2025-10-27T21:17:43.890+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : Coffee in List: Coffee(id=1, name=espresso, price=TWD 100.00, ...)
2025-10-27T21:17:43.891+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : Coffee in List: Coffee(id=2, name=latte, price=TWD 125.00, ...)
2025-10-27T21:17:43.891+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : Coffee in List: Coffee(id=3, name=capuccino, price=TWD 125.00, ...)
2025-10-27T21:17:43.891+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : Coffee in List: Coffee(id=4, name=mocha, price=TWD 150.00, ...)
2025-10-27T21:17:43.891+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : Coffee in List: Coffee(id=5, name=macchiato, price=TWD 150.00, ...)
2025-10-27T21:17:43.891+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : Coffee in List: Coffee(id=6, name=americano, price=TWD 125.00, ...)
```

**Output Analysis:**
- **Startup**: 1.005 seconds (very fast, no web server)
- **Async Execution**: GET and POST completed almost simultaneously (21:17:43.832)
- **Thread Pool**: `reactor-http-nio-2/3` for async operations, `main` for sync
- **Total Duration**: ~326ms for complete business flow
- **Data Consistency**: Successfully created americano (ID=6) and retrieved it in list

## Key Components

### WebClient Bean Configuration

```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder.baseUrl("http://localhost:8080").build();
}
```

**Configuration Details:**
- `WebClient.Builder` is provided by Spring Boot
- `baseUrl` sets default URL for all requests
- Netty HTTP client configured automatically
- Connection pooling enabled by default

### Money Serializer

```java
@JsonComponent
public class MoneySerializer extends StdSerializer<Money> {
    @Override
    public void serialize(Money money, JsonGenerator jsonGenerator, 
                         SerializerProvider serializerProvider) throws IOException {
        // Serialize Money to number (amount in currency minor units)
        jsonGenerator.writeNumber(money.getAmount());
    }
}
```

**Serialization Flow:**
```
TWD 125.00 → getAmount() → BigDecimal 125.00
           → jsonGenerator.writeNumber() → JSON: 125.00
```

### Money Deserializer

```java
@JsonComponent
public class MoneyDeserializer extends StdDeserializer<Money> {
    @Override
    public Money deserialize(JsonParser p, DeserializationContext ctxt) 
            throws IOException, JsonProcessingException {
        // Deserialize number to Money (default currency: TWD)
        return Money.of(CurrencyUnit.of("TWD"), p.getDecimalValue());
    }
}
```

**Deserialization Flow:**
```
JSON: 125.00 → getDecimalValue() → BigDecimal 125.00
             → Money.of(TWD, ...) → TWD 125.00
```

### Coffee Entity

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Coffee implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private Long id;
    private String name;
    private Money price;        // Joda Money type
    private Date createTime;
    private Date updateTime;
}
```

## Reactive Programming

### Mono vs Flux

**Mono (0..1 element):**
```java
// Single object response
Mono<Coffee> coffeeMono = webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class);
```

**Flux (0..N elements):**
```java
// Multiple objects response
Flux<Coffee> coffeeFlux = webClient.get()
    .uri("/coffee/")
    .retrieve()
    .bodyToFlux(Coffee.class);
```

### Reactive Operators

```java
webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class)
    .doOnError(t -> log.error("Error: ", t))           // Error handling
    .doFinally(s -> cdl.countDown())                   // Cleanup
    .subscribeOn(Schedulers.single())                  // Thread selection
    .subscribe(c -> log.info("Coffee: {}", c));        // Subscribe
```

**Operator Explanations:**
- `doOnError`: Handle errors without stopping stream
- `doFinally`: Execute cleanup on completion or error
- `subscribeOn`: Specify execution thread pool
- `subscribe`: Trigger async execution

### Thread Model

```
Main Thread
    ├─ Application startup
    ├─ WebClient configuration
    ├─ Async request dispatch
    └─ Sync list query

Reactor Thread Pool
    ├─ reactor-http-nio-2 → GET /coffee/1
    ├─ reactor-http-nio-3 → POST /coffee/
    └─ reactor-http-nio-* → Other async operations
```

## WebClient Best Practices

### 1. Async vs Sync

**✅ Async (Non-blocking):**
```java
// Recommended for high concurrency
webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class)
    .subscribe(coffee -> {
        // Process asynchronously
    });
```

**⚠️ Sync (Blocking):**
```java
// Avoid in production for high-volume APIs
Coffee coffee = webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class)
    .block();  // Blocks current thread!
```

### 2. Error Handling

```java
webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class)
    .onErrorResume(WebClientResponseException.class, ex -> {
        if (ex.getStatusCode().is4xxClientError()) {
            log.error("Client error: {}", ex.getMessage());
            return Mono.empty();
        }
        return Mono.error(ex);
    })
    .doOnError(t -> log.error("Unexpected error: ", t))
    .subscribe(coffee -> log.info("Coffee: {}", coffee));
```

### 3. Timeout Configuration

```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder
        .baseUrl("http://localhost:8080")
        .clientConnector(new ReactorClientHttpConnector(
            HttpClient.create()
                .responseTimeout(Duration.ofSeconds(5))  // 5s timeout
                .doOnConnected(conn -> 
                    conn.addHandlerLast(new ReadTimeoutHandler(5))
                        .addHandlerLast(new WriteTimeoutHandler(5))
                )
        ))
        .build();
}
```

### 4. Connection Pool Configuration

```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    ConnectionProvider provider = ConnectionProvider.builder("custom")
        .maxConnections(100)              // Max total connections
        .pendingAcquireMaxCount(50)       // Max waiting requests
        .pendingAcquireTimeout(Duration.ofSeconds(30))
        .build();
    
    return builder
        .baseUrl("http://localhost:8080")
        .clientConnector(new ReactorClientHttpConnector(
            HttpClient.create(provider)
        ))
        .build();
}
```

## Performance Analysis

### Execution Timeline

```
T0: Application started (1.005s startup)
    ↓
T1: Async requests dispatched (268ms later)
    ├─ reactor-http-nio-2: GET /coffee/1
    └─ reactor-http-nio-3: POST /coffee/
    ↓ (Both complete at same timestamp: 21:17:43.832)
T2: List query executed (58ms later)
    
Total business duration: ~326ms (High performance!)
```

### Key Observations

1. **Concurrent Execution** ⚡
   - GET and POST completed simultaneously
   - Different Reactor threads: `reactor-http-nio-2/3`
   - Demonstrates non-blocking I/O efficiency

2. **Thread Model** 🧵
   - `reactor-http-nio-2`: First GET request
   - `reactor-http-nio-3`: POST request
   - `main`: List query (waits for async completion)

3. **Execution Efficiency** 🚀
   - Application startup: **1.005 seconds**
   - Business flow: **~326ms**
   - Excellent performance for modern microservices

4. **Data Consistency** ✅
   - Successfully fetched espresso (ID=1)
   - Successfully created americano (ID=6)
   - List query includes all 6 coffees

### Technical Highlights

| Feature | Implementation | Benefit |
|---------|---------------|---------|
| **Async Execution** | Reactor threads | Non-blocking, high concurrency |
| **CountDownLatch** | Flow control | Ensures execution order |
| **Netty Thread Pool** | reactor-http-nio-* | Efficient resource utilization |
| **Reactive Streams** | Mono/Flux | Backpressure handling |
| **Fast Startup** | WebApplicationType.NONE | No web server overhead |

## Monitoring

### WebClient Metrics

```java
@Component
public class WebClientMetrics {
    
    @Autowired
    private WebClient webClient;
    
    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        // Monitor WebClient through Micrometer
        log.info("WebClient metrics available at /actuator/metrics");
    }
}
```

### Enable Actuator Metrics

```properties
# Add to application.properties
management.endpoints.web.exposure.include=metrics,health
management.metrics.enable.webclient=true
```

### Request Logging

```properties
# Enable debug logging
logging.level.reactor.netty.http.client=DEBUG
logging.level.org.springframework.web.reactive.function.client=DEBUG
```

## Testing

### Unit Test Example

```java
@SpringBootTest
class WebclientDemoApplicationTests {

    @Test
    void contextLoads() {
        // Verify application context loads successfully
    }
    
    @Test
    void testWebClientConfiguration() {
        WebClient webClient = WebClient.builder()
            .baseUrl("http://localhost:8080")
            .build();
        
        assertNotNull(webClient);
    }
}
```

### Integration Test with MockWebServer

```java
@Test
void testGetCoffee() {
    MockWebServer server = new MockWebServer();
    server.enqueue(new MockResponse()
        .setBody("{\"id\":1,\"name\":\"espresso\",\"price\":100.00}")
        .addHeader("Content-Type", "application/json"));
    
    WebClient testClient = WebClient.create(server.url("/").toString());
    
    StepVerifier.create(
        testClient.get()
            .uri("/coffee/1")
            .retrieve()
            .bodyToMono(Coffee.class)
    )
    .expectNextMatches(coffee -> coffee.getName().equals("espresso"))
    .verifyComplete();
}
```

## Common Issues

### Issue 1: Connection Refused

**Error:**
```
java.net.ConnectException: Connection refused: localhost/127.0.0.1:8080
```

**Solution:**
```bash
# Ensure REST API server is running
cd ../hateoas-waiter-service
mvn spring-boot:run

# Verify endpoint
curl http://localhost:8080/coffee/1
```

### Issue 2: Timeout

**Error:**
```
java.util.concurrent.TimeoutException: Did not observe any item or terminal signal within 1000ms
```

**Solution:**
```java
// Increase timeout
webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class)
    .timeout(Duration.ofSeconds(5))  // 5s timeout
    .block();
```

### Issue 3: JSON Parsing Error

**Error:**
```
com.fasterxml.jackson.databind.exc.MismatchedInputException: Cannot deserialize value
```

**Solution:**
```java
// Ensure Money serializers are registered
@JsonComponent
public class MoneyDeserializer extends StdDeserializer<Money> {
    // Proper deserialization implementation
}
```

## WebClient vs RestTemplate

| Feature | RestTemplate | WebClient |
|---------|-------------|-----------|
| I/O Model | Blocking (sync) | Non-blocking (async) |
| Thread Model | One thread per request | Shared Reactor thread pool |
| Reactive Support | ❌ No | ✅ Yes (Mono/Flux) |
| Backpressure | ❌ No | ✅ Yes |
| Spring Boot Default | ❌ Deprecated | ✅ Recommended since 5.0 |
| Use Case | Legacy apps | Modern reactive apps |

**Migration Guidance:**
- **Legacy Projects**: Keep RestTemplate if no reactive requirements
- **New Projects**: Use WebClient for better performance and scalability
- **This Project**: Demonstrates WebClient best practices

## Dependencies

```xml
<properties>
    <java.version>21</java.version>
    <joda-money.version>2.0.2</joda-money.version>
    <netty.version>4.1.119.Final</netty.version>
</properties>

<dependencies>
    <!-- Spring WebFlux (includes WebClient) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    
    <!-- Joda Money -->
    <dependency>
        <groupId>org.joda</groupId>
        <artifactId>joda-money</artifactId>
        <version>${joda-money.version}</version>
    </dependency>
    
    <!-- Netty DNS Resolver for macOS ARM -->
    <dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-resolver-dns-native-macos</artifactId>
        <classifier>osx-aarch_64</classifier>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

## Best Practices Demonstrated

1. **WebClient Configuration**: Proper bean setup with baseUrl
2. **Reactive Programming**: Mono/Flux for async operations
3. **Thread Management**: Reactor thread pool utilization
4. **Flow Control**: CountDownLatch for execution ordering
5. **Money Serialization**: Custom serializers for Money type
6. **Error Handling**: Graceful error recovery with doOnError
7. **Non-blocking I/O**: Efficient resource utilization

## References

- [Spring WebFlux Documentation](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [WebClient API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/client/WebClient.html)
- [Project Reactor Documentation](https://projectreactor.io/docs)
- [Netty Documentation](https://netty.io/wiki/)
- [Joda Money Documentation](https://www.joda.org/joda-money/)

## License

MIT License - see [LICENSE](LICENSE) file for details.

## About Us

我們主要專注在敏捷專案管理、物聯網（IoT）應用開發和領域驅動設計（DDD）。喜歡把先進技術和實務經驗結合，打造好用又靈活的軟體解決方案。近來也積極結合 AI 技術，推動自動化工作流，讓開發與運維更有效率、更智慧。持續學習與分享，希望能一起推動軟體開發的創新和進步。

## Contact

**風清雲談** - 專注於敏捷專案管理、物聯網（IoT）應用開發和領域驅動設計（DDD）。

- 🌐 官方網站：[風清雲談部落格](https://blog.fengqing.tw/)
- 📘 Facebook：[風清雲談粉絲頁](https://www.facebook.com/profile.php?id=61576838896062)
- 💼 LinkedIn：[Chu Kuo-Lung](https://www.linkedin.com/in/chu-kuo-lung)
- 📺 YouTube：[雲談風清頻道](https://www.youtube.com/channel/UCXDqLTdCMiCJ1j8xGRfwEig)
- 📧 Email：[fengqing.tw@gmail.com](mailto:fengqing.tw@gmail.com)

---

**⭐ If this project helps you, please give it a star!**
