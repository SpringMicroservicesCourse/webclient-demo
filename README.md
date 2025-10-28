# webclient-demo

> Reactive HTTP client with WebClient and non-blocking asynchronous operations

[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.5-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Java](https://img.shields.io/badge/Java-21-orange.svg)](https://openjdk.org/)
[![Spring WebFlux](https://img.shields.io/badge/Spring%20WebFlux-6.2.2-blue.svg)](https://docs.spring.io/spring-framework/reference/web/webflux.html)
[![Netty](https://img.shields.io/badge/Netty-4.1.119-purple.svg)](https://netty.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

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

### Core Frameworks
- **Spring Boot 3.4.5** - Microservices framework
- **Spring WebFlux** - Reactive web framework (includes WebClient)
- **Project Reactor** - Reactive streams (Mono/Flux)
- **Netty 4.1.119.Final** - Non-blocking I/O client

### Development Tools & Libraries
- **Joda Money 2.0.2** - Money handling
- **Lombok** - Reduce boilerplate code
- **Maven 3.8+** - Build tool
- **Java 21** - Development environment

## Project Structure

```
webclient-demo/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── tw/fengqing/spring/reactor/webclient/
│   │   │       ├── WebclientDemoApplication.java  # Main application (with ApplicationRunner)
│   │   │       ├── model/
│   │   │       │   └── Coffee.java                # Coffee entity
│   │   │       └── support/
│   │   │           ├── MoneySerializer.java       # Money JSON serializer
│   │   │           └── MoneyDeserializer.java     # Money JSON deserializer
│   │   └── resources/
│   │       └── application.properties             # Application config (empty)
│   └── test/
└── pom.xml
```

## Getting Started

### Prerequisites

- **Java 21** - Development environment
- **Maven 3.8+** - Build tool
- **REST API Server** - Running on port 8080 (e.g., hateoas-waiter-service)

### Installation & Execution

**Step 1: Start REST API Server**

```bash
# Start hateoas-waiter-service (or any compatible REST API)
cd ../hateoas-waiter-service
mvn spring-boot:run

# Wait for server startup
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
mvn spring-boot:run

# Or using compiled JAR
mvn clean package
java -jar target/webclient-demo-0.0.1-SNAPSHOT.jar
```

## Configuration

### Application Configuration

WebClient configuration is done directly in Java code, no additional properties configuration required:

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

### WebClient Bean Configuration

```java
@SpringBootApplication
@Slf4j
public class WebclientDemoApplication implements ApplicationRunner {
    
    public static void main(String[] args) {
        // Set to client-only mode (no web server)
        SpringApplication application = new SpringApplication(WebclientDemoApplication.class);
        application.setWebApplicationType(WebApplicationType.NONE);
        application.run(args);
    }
    
    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder.baseUrl("http://localhost:8080").build();
    }
}
```

## Core Code Analysis

### Main Application Execution Flow

```java
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

### Money Serializer

```java
@JsonComponent
public class MoneySerializer extends StdSerializer<Money> {
    protected MoneySerializer() {
        super(Money.class);
    }
    
    @Override
    public void serialize(Money money, JsonGenerator jsonGenerator, 
                         SerializerProvider serializerProvider) throws IOException {
        // Directly output amount as number instead of full Money object
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
    protected MoneyDeserializer() {
        super(Money.class);
    }
    
    @Override
    public Money deserialize(JsonParser p, DeserializationContext ctxt) 
            throws IOException, JsonProcessingException {
        // Create Money object from JSON number (default currency: TWD)
        return Money.of(CurrencyUnit.of("TWD"), p.getDecimalValue());
    }
}
```

**Deserialization Flow:**
```
JSON: 125.00 → getDecimalValue() → BigDecimal 125.00
             → Money.of(TWD, ...) → TWD 125.00
```

## Execution Results Analysis

### Actual Execution Output (2025-10-27)

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

### Performance Analysis

**Performance Observations:**
- **Startup Time**: 1.005 seconds (very fast, no web server)
- **Async Execution**: GET and POST completed almost simultaneously (21:17:43.832)
- **Thread Pool**: `reactor-http-nio-2/3` for async operations, `main` for sync
- **Total Duration**: ~326ms for complete business flow

**Technical Highlights:**

| Feature | Implementation | Benefit |
|---------|---------------|---------|
| **Async Execution** | Reactor threads | Non-blocking, high concurrency |
| **CountDownLatch** | Flow control | Ensures execution order |
| **Netty Thread Pool** | reactor-http-nio-* | Efficient resource utilization |
| **Reactive Streams** | Mono/Flux | Backpressure handling |
| **Fast Startup** | WebApplicationType.NONE | No web server overhead |

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

## Monitoring

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

- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Spring WebFlux Documentation](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [WebClient API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/client/WebClient.html)
- [Project Reactor Documentation](https://projectreactor.io/docs)
- [Netty Documentation](https://netty.io/wiki/)
- [Joda Money Documentation](https://www.joda.org/joda-money/)

## Best Practices & Notes

### ⚠️ Important Reminders

| Item | Description | Recommendation |
|------|-------------|----------------|
| Thread Safety | WebClient is thread-safe | Can share single instance |
| Connection Management | Netty manages connection pool automatically | Use default configuration |
| Error Handling | Use reactive error handling | Use onErrorResume |
| Timeout Setting | No default timeout | Set reasonable timeout |

### 🔒 Best Practice Guidelines

- **WebClient Configuration**: Properly set baseUrl and timeout
- **Reactive Programming**: Use Mono/Flux for async operations
- **Thread Management**: Leverage Reactor thread pool
- **Flow Control**: Use CountDownLatch for execution ordering
- **Money Serialization**: Custom serializers for Money type
- **Error Handling**: Use doOnError for graceful recovery
- **Non-blocking I/O**: Efficient resource utilization

### 🚀 Performance Optimization

- Use async operations to avoid thread blocking
- Configure appropriate connection pool size
- Set reasonable timeout values
- Consider caching to reduce HTTP requests

## License

MIT License - see [LICENSE](LICENSE) file for details.

## About Us

We focus on Agile Project Management, IoT application development, and Domain-Driven Design (DDD). We enjoy combining advanced technologies with practical experience to create user-friendly and flexible software solutions. Recently, we've been actively integrating AI technology to drive automated workflows, making development and operations more efficient and intelligent. We continue to learn and share, hoping to promote innovation and progress in software development together.

## Contact

**風清雲談 (Fengqing Cloud Talk)** - Specialized in Agile Project Management, IoT application development, and Domain-Driven Design (DDD).

- 🌐 Official Website: [Fengqing Cloud Talk Blog](https://blog.fengqing.tw/)
- 📘 Facebook: [Fengqing Cloud Talk Fan Page](https://www.facebook.com/profile.php?id=61576838896062)
- 💼 LinkedIn: [Chu Kuo-Lung](https://www.linkedin.com/in/chu-kuo-lung)
- 📺 YouTube: [Cloud Talk Fengqing Channel](https://www.youtube.com/channel/UCXDqLTdCMiCJ1j8xGRfwEig)
- 📧 Email: [fengqing.tw@gmail.com](mailto:fengqing.tw@gmail.com)

---

**📅 Last Updated: 2025-10-28**  
**👨‍💻 Maintained by: Fengqing Cloud Talk Team**
