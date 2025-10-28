# Spring Boot WebClient 示範專案 ⚡

[![Java](https://img.shields.io/badge/Java-21-orange.svg)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.5-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Spring WebFlux](https://img.shields.io/badge/Spring%20WebFlux-6.2.2-blue.svg)](https://docs.spring.io/spring-framework/reference/web/webflux.html)
[![Netty](https://img.shields.io/badge/Netty-4.1.119-purple.svg)](https://netty.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 專案介紹

這是一個基於 Spring Boot 3.x 的 WebClient 示範專案，展示了如何使用 **Spring WebClient** 進行反應式（Reactive）、非阻塞（Non-blocking）的 HTTP 請求操作。專案主要用於學習 Spring WebFlux 中的 WebClient 使用方式，是現代化微服務架構中推薦的 HTTP 客戶端解決方案。

> 💡 **為什麼選擇此專案？**
> - 完整的 Spring WebClient 3.x 實作範例
> - 展示 Mono/Flux 反應式流處理
> - 示範非阻塞異步操作模式
> - 使用 Joda Money 處理貨幣類型
> - 基於 Netty 的高效 HTTP 客戶端

### 🎯 專案特色

- **WebClient 反應式客戶端**：取代傳統的 RestTemplate
- **非阻塞異步操作**：高並發場景下的最佳選擇
- **Mono/Flux 處理**：單一物件和多物件的反應式流
- **並發請求執行**：使用 CountDownLatch 控制執行順序
- **Reactor 執行緒池**：`reactor-http-nio-*` 執行緒模型
- **Money 類型序列化**：自訂 TWD 貨幣序列化器
- **WebApplicationType.NONE**：純客戶端模式（無 Web 伺服器）
- **Netty HTTP 客戶端**：針對 macOS ARM 的 DNS 解析優化

## 技術棧

### 核心框架
- **Spring Boot 3.4.5** - 微服務框架
- **Spring WebFlux** - 反應式 Web 框架（含 WebClient）
- **Project Reactor** - 反應式流處理（Mono/Flux）
- **Netty 4.1.119.Final** - 非阻塞 I/O 客戶端

### 開發工具與輔助
- **Joda Money 2.0.2** - 貨幣處理
- **Lombok** - 減少樣板程式碼
- **Maven 3.8+** - 建置工具
- **Java 21** - 開發環境

## 專案結構

```
webclient-demo/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── tw/fengqing/spring/reactor/webclient/
│   │   │       ├── WebclientDemoApplication.java  # 主程式（含 ApplicationRunner）
│   │   │       ├── model/
│   │   │       │   └── Coffee.java                # 咖啡實體類別
│   │   │       └── support/
│   │   │           ├── MoneySerializer.java       # Money JSON 序列化器
│   │   │           └── MoneyDeserializer.java     # Money JSON 反序列化器
│   │   └── resources/
│   │       └── application.properties             # 應用配置（空）
│   └── test/
└── pom.xml
```

## 快速開始

### 前置需求
- **Java 21** - 開發環境
- **Maven 3.8+** - 建置工具
- **REST API 伺服器** - 運行在 port 8080（例如 hateoas-waiter-service）

### 安裝與執行

**步驟一：啟動 REST API 伺服器**

```bash
# 啟動 hateoas-waiter-service（或任何相容的 REST API）
cd ../hateoas-waiter-service
mvn spring-boot:run

# 等待伺服器啟動完成
```

**步驟二：驗證 REST API**

```bash
# 測試咖啡端點
curl http://localhost:8080/coffee/1

# 預期輸出：Coffee JSON 回應
```

**步驟三：執行 webclient-demo**

```bash
# 使用 Maven 執行
mvn spring-boot:run

# 或使用編譯後的 JAR
mvn clean package
java -jar target/webclient-demo-0.0.1-SNAPSHOT.jar
```

## 配置說明

### 應用配置

WebClient 配置直接在 Java 程式碼中完成，無需額外的 properties 配置：

```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder.baseUrl("http://localhost:8080").build();
}
```

**配置細節：**
- `baseUrl`: 目標 REST API 端點
- 自動配置 Netty 客戶端
- 內建連線池管理
- 支援 HTTP/2

### WebClient Bean 配置

```java
@SpringBootApplication
@Slf4j
public class WebclientDemoApplication implements ApplicationRunner {
    
    public static void main(String[] args) {
        // 設定為純客戶端模式（不啟動 Web 伺服器）
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

## 核心程式碼解析

### 主程式執行流程

```java
@Override
public void run(ApplicationArguments args) throws Exception {
    CountDownLatch cdl = new CountDownLatch(2);

    // 1. 異步 GET 請求 - 單一咖啡
    webClient.get()
            .uri("/coffee/{id}", 1)
            .accept(MediaType.APPLICATION_JSON)
            .retrieve()
            .bodyToMono(Coffee.class)
            .doOnError(t -> log.error("錯誤: ", t))
            .doFinally(s -> cdl.countDown())
            .subscribeOn(Schedulers.single())
            .subscribe(c -> log.info("Coffee 1: {}", c));

    // 2. 異步 POST 請求 - 建立咖啡
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
            .subscribe(c -> log.info("建立的咖啡: {}", c));

    // 等待異步操作完成
    cdl.await();

    // 3. 同步 GET 請求 - 咖啡列表
    webClient.get()
            .uri("/coffee/")
            .retrieve()
            .bodyToFlux(Coffee.class)
            .toStream()
            .forEach(c -> log.info("列表中的咖啡: {}", c));
}
```

### Coffee 實體類別

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Coffee implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private Long id;
    private String name;
    private Money price;        // Joda Money 類型
    private Date createTime;
    private Date updateTime;
}
```

### Money 序列化器

```java
@JsonComponent
public class MoneySerializer extends StdSerializer<Money> {
    protected MoneySerializer() {
        super(Money.class);
    }
    
    @Override
    public void serialize(Money money, JsonGenerator jsonGenerator, 
                         SerializerProvider serializerProvider) throws IOException {
        // 直接輸出金額數字，而非 Money 物件的完整結構
        jsonGenerator.writeNumber(money.getAmount());
    }
}
```

**序列化流程：**
```
TWD 125.00 → getAmount() → BigDecimal 125.00
           → jsonGenerator.writeNumber() → JSON: 125.00
```

### Money 反序列化器

```java
@JsonComponent
public class MoneyDeserializer extends StdDeserializer<Money> {
    protected MoneyDeserializer() {
        super(Money.class);
    }
    
    @Override
    public Money deserialize(JsonParser p, DeserializationContext ctxt) 
            throws IOException, JsonProcessingException {
        // 從 JSON 數字建立 Money 物件（預設貨幣：TWD）
        return Money.of(CurrencyUnit.of("TWD"), p.getDecimalValue());
    }
}
```

**反序列化流程：**
```
JSON: 125.00 → getDecimalValue() → BigDecimal 125.00
             → Money.of(TWD, ...) → TWD 125.00
```

## 執行結果分析

### 實際執行輸出（2025-10-27）

**啟動階段：**
```
2025-10-27T21:17:42.708+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : Starting WebclientDemoApplication using Java 21.0.7 with PID 51746
2025-10-27T21:17:42.711+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : No active profile set, falling back to 1 default profile: "default"
2025-10-27T21:17:43.564+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : Started WebclientDemoApplication in 1.005 seconds (process running for 1.137)
```

**業務執行：**
```
# 1. 異步 GET 請求（執行緒：reactor-http-nio-2）
2025-10-27T21:17:43.832+08:00  INFO 51746 --- [ctor-http-nio-2] t.f.s.r.w.WebclientDemoApplication : Coffee 1: Coffee(id=1, name=espresso, price=TWD 100.00, createTime=Mon Oct 27 21:17:38 CST 2025, updateTime=Mon Oct 27 21:17:38 CST 2025)

# 2. 異步 POST 請求（執行緒：reactor-http-nio-3）
2025-10-27T21:17:43.832+08:00  INFO 51746 --- [ctor-http-nio-3] t.f.s.r.w.WebclientDemoApplication : 建立的咖啡: Coffee(id=6, name=americano, price=TWD 125.00, createTime=Mon Oct 27 21:17:43 CST 2025, updateTime=Mon Oct 27 21:17:43 CST 2025)

# 3. 同步 GET 請求 - 列表（執行緒：main）
2025-10-27T21:17:43.890+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : 列表中的咖啡: Coffee(id=1, name=espresso, price=TWD 100.00, ...)
2025-10-27T21:17:43.891+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : 列表中的咖啡: Coffee(id=2, name=latte, price=TWD 125.00, ...)
2025-10-27T21:17:43.891+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : 列表中的咖啡: Coffee(id=3, name=capuccino, price=TWD 125.00, ...)
2025-10-27T21:17:43.891+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : 列表中的咖啡: Coffee(id=4, name=mocha, price=TWD 150.00, ...)
2025-10-27T21:17:43.891+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : 列表中的咖啡: Coffee(id=5, name=macchiato, price=TWD 150.00, ...)
2025-10-27T21:17:43.891+08:00  INFO 51746 --- [main] t.f.s.r.w.WebclientDemoApplication : 列表中的咖啡: Coffee(id=6, name=americano, price=TWD 125.00, ...)
```

### 執行結果分析

**效能觀察：**
- **啟動時間**: 1.005 秒（非常快速，無 Web 伺服器啟動）
- **異步執行**: GET 和 POST 幾乎同時完成（21:17:43.832）
- **執行緒池**: `reactor-http-nio-2/3` 處理異步操作，`main` 處理同步
- **總執行時間**: 約 326ms 完成整個業務流程

**技術亮點：**

| 特性 | 實作方式 | 優勢 |
|------|---------|------|
| **異步執行** | Reactor 執行緒 | 非阻塞，高並發 |
| **CountDownLatch** | 流程控制 | 確保執行順序 |
| **Netty 執行緒池** | reactor-http-nio-* | 高效資源利用 |
| **反應式流** | Mono/Flux | 背壓（Backpressure）處理 |
| **快速啟動** | WebApplicationType.NONE | 無 Web 伺服器開銷 |

## 反應式程式設計

### Mono vs Flux

**Mono（0..1 個元素）：**
```java
// 單一物件回應
Mono<Coffee> coffeeMono = webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class);
```

**Flux（0..N 個元素）：**
```java
// 多個物件回應
Flux<Coffee> coffeeFlux = webClient.get()
    .uri("/coffee/")
    .retrieve()
    .bodyToFlux(Coffee.class);
```

### 反應式操作符

```java
webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class)
    .doOnError(t -> log.error("錯誤: ", t))         // 錯誤處理
    .doFinally(s -> cdl.countDown())               // 清理操作
    .subscribeOn(Schedulers.single())              // 執行緒選擇
    .subscribe(c -> log.info("咖啡: {}", c));       // 訂閱執行
```

**操作符說明：**
- `doOnError`: 處理錯誤而不中斷流
- `doFinally`: 完成或錯誤時執行清理
- `subscribeOn`: 指定執行執行緒池
- `subscribe`: 觸發異步執行

### 執行緒模型

```
主執行緒（Main Thread）
    ├─ 應用程式啟動
    ├─ WebClient 配置
    ├─ 異步請求分派
    └─ 同步列表查詢

Reactor 執行緒池
    ├─ reactor-http-nio-2 → GET /coffee/1
    ├─ reactor-http-nio-3 → POST /coffee/
    └─ reactor-http-nio-* → 其他異步操作
```

## WebClient 最佳實踐

### 1. 異步 vs 同步

**✅ 異步（非阻塞）：**
```java
// 推薦用於高並發場景
webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class)
    .subscribe(coffee -> {
        // 異步處理
    });
```

**⚠️ 同步（阻塞）：**
```java
// 避免在生產環境的高流量 API 中使用
Coffee coffee = webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class)
    .block();  // 阻塞當前執行緒！
```

### 2. 錯誤處理

```java
webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class)
    .onErrorResume(WebClientResponseException.class, ex -> {
        if (ex.getStatusCode().is4xxClientError()) {
            log.error("客戶端錯誤: {}", ex.getMessage());
            return Mono.empty();
        }
        return Mono.error(ex);
    })
    .doOnError(t -> log.error("意外錯誤: ", t))
    .subscribe(coffee -> log.info("咖啡: {}", coffee));
```

### 3. 超時配置

```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder
        .baseUrl("http://localhost:8080")
        .clientConnector(new ReactorClientHttpConnector(
            HttpClient.create()
                .responseTimeout(Duration.ofSeconds(5))  // 5 秒超時
                .doOnConnected(conn -> 
                    conn.addHandlerLast(new ReadTimeoutHandler(5))
                        .addHandlerLast(new WriteTimeoutHandler(5))
                )
        ))
        .build();
}
```

### 4. 連線池配置

```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    ConnectionProvider provider = ConnectionProvider.builder("custom")
        .maxConnections(100)              // 最大總連線數
        .pendingAcquireMaxCount(50)       // 最大等待請求數
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

| 特性 | RestTemplate | WebClient |
|-----|-------------|-----------|
| I/O 模型 | 阻塞（同步） | 非阻塞（異步） |
| 執行緒模型 | 每個請求一個執行緒 | 共享 Reactor 執行緒池 |
| 反應式支援 | ❌ 否 | ✅ 是（Mono/Flux） |
| 背壓處理 | ❌ 否 | ✅ 是 |
| Spring Boot 預設 | ❌ 已棄用 | ✅ 自 5.0 起推薦 |
| 使用場景 | 傳統應用 | 現代反應式應用 |

**遷移指引：**
- **傳統專案**：如無反應式需求，可保留 RestTemplate
- **新專案**：使用 WebClient 以獲得更好的效能和擴展性
- **本專案**：示範 WebClient 最佳實踐

## 常見問題

### 問題 1：連線被拒絕

**錯誤訊息：**
```
java.net.ConnectException: Connection refused: localhost/127.0.0.1:8080
```

**解決方案：**
```bash
# 確保 REST API 伺服器正在運行
cd ../hateoas-waiter-service
mvn spring-boot:run

# 驗證端點
curl http://localhost:8080/coffee/1
```

### 問題 2：超時

**錯誤訊息：**
```
java.util.concurrent.TimeoutException: Did not observe any item or terminal signal within 1000ms
```

**解決方案：**
```java
// 增加超時時間
webClient.get()
    .uri("/coffee/1")
    .retrieve()
    .bodyToMono(Coffee.class)
    .timeout(Duration.ofSeconds(5))  // 5 秒超時
    .block();
```

### 問題 3：JSON 解析錯誤

**錯誤訊息：**
```
com.fasterxml.jackson.databind.exc.MismatchedInputException: Cannot deserialize value
```

**解決方案：**
```java
// 確保 Money 序列化器已註冊
@JsonComponent
public class MoneyDeserializer extends StdDeserializer<Money> {
    // 正確的反序列化實作
}
```

## 測試

### 單元測試範例

```java
@SpringBootTest
class WebclientDemoApplicationTests {

    @Test
    void contextLoads() {
        // 驗證應用程式上下文載入成功
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

### 使用 MockWebServer 進行整合測試

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

## 監控

### 啟用 Actuator 指標

```properties
# 添加到 application.properties
management.endpoints.web.exposure.include=metrics,health
management.metrics.enable.webclient=true
```

### 請求日誌

```properties
# 啟用除錯日誌
logging.level.reactor.netty.http.client=DEBUG
logging.level.org.springframework.web.reactive.function.client=DEBUG
```

## 參考資源

- [Spring Boot 官方文件](https://spring.io/projects/spring-boot)
- [Spring WebFlux 文件](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [WebClient API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/client/WebClient.html)
- [Project Reactor 文件](https://projectreactor.io/docs)
- [Netty 文件](https://netty.io/wiki/)
- [Joda Money 文件](https://www.joda.org/joda-money/)

## 注意事項與最佳實踐

### ⚠️ 重要提醒

| 項目 | 說明 | 建議做法 |
|------|------|----------|
| 執行緒安全 | WebClient 是執行緒安全的 | 可共享單一實例 |
| 連線管理 | Netty 自動管理連線池 | 使用預設配置 |
| 錯誤處理 | 使用反應式錯誤處理 | 使用 onErrorResume |
| 超時設定 | 預設無超時 | 建議設定合理的超時時間 |

### 🔒 最佳實踐指南

- **WebClient 配置**：正確設定 baseUrl 和超時
- **反應式程式設計**：使用 Mono/Flux 進行異步操作
- **執行緒管理**：善用 Reactor 執行緒池
- **流程控制**：使用 CountDownLatch 控制執行順序
- **Money 序列化**：自訂序列化器處理 Money 類型
- **錯誤處理**：使用 doOnError 進行優雅的錯誤恢復
- **非阻塞 I/O**：高效的資源利用

### 🚀 效能優化建議

- 使用異步操作避免執行緒阻塞
- 適當配置連線池大小
- 設定合理的超時時間
- 考慮使用快取減少 HTTP 請求

## 授權說明

本專案採用 MIT 授權條款，詳見 LICENSE 檔案。

## 關於我們

我們主要專注在敏捷專案管理、物聯網（IoT）應用開發和領域驅動設計（DDD）。喜歡把先進技術和實務經驗結合，打造好用又靈活的軟體解決方案。近來也積極結合 AI 技術，推動自動化工作流，讓開發與運維更有效率、更智慧。持續學習與分享，希望能一起推動軟體開發的創新和進步。

## 聯繫我們

**風清雲談** - 專注於敏捷專案管理、物聯網（IoT）應用開發和領域驅動設計（DDD）。

- 🌐 官方網站：[風清雲談部落格](https://blog.fengqing.tw/)
- 📘 Facebook：[風清雲談粉絲頁](https://www.facebook.com/profile.php?id=61576838896062)
- 💼 LinkedIn：[Chu Kuo-Lung](https://www.linkedin.com/in/chu-kuo-lung)
- 📺 YouTube：[雲談風清頻道](https://www.youtube.com/channel/UCXDqLTdCMiCJ1j8xGRfwEig)
- 📧 Email：[fengqing.tw@gmail.com](mailto:fengqing.tw@gmail.com)

---

**📅 最後更新：2025-10-28**  
**👨‍💻 維護者：風清雲談團隊**
