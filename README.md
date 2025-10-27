# WebClient 非同步 HTTP 客戶端 Demo ⚡

[![Java](https://img.shields.io/badge/Java-21-orange.svg)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.5-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 專案介紹

本專案示範如何在 Spring Boot 環境下，使用 WebClient 以 Reactive（反應式）方式進行 HTTP 請求，實現非同步、非阻塞式的 RESTful API 呼叫。適合需要高效能、現代化 Java HTTP 客戶端的團隊。

- **核心功能**：
  - 以 WebClient 進行 GET/POST 請求，串接遠端 REST API。
  - Reactive 程式設計，支援高併發與非同步流程。
  - Money 型別序列化/反序列化範例。
- **解決問題**：
  - 傳統 RestTemplate 已逐步淘汰，WebClient 提供更現代且彈性的 HTTP 客戶端解決方案。
- **主要特色**：
  - 完整註解，方便團隊理解與維護。
  - 適用於台灣軟體開發團隊，專業術語本地化。
  - 可直接擴充應用於微服務、API Gateway 等場景。
- **目標使用者**：
  - Java/Spring 團隊、後端工程師、架構師。

> 💡 **為什麼選擇此專案？**
> - 採用最新 Spring Boot 3.x 與 Java 21，技術現代化。
> - Reactive 非同步設計，效能佳、可擴充性高。
> - 程式碼註解清楚，團隊溝通無障礙。

### 🎯 專案特色
- Reactive 非同步 HTTP 請求實作
- Money 型別序列化/反序列化
- 完整註解與台灣常用專業用語

## 技術棧

### 核心框架
- **Spring Boot 3.4.5** － 主體應用框架，快速建構微服務/REST API。
- **Spring WebFlux (WebClient)** － Reactive HTTP 客戶端，支援非同步與高併發。

### 開發工具與輔助
- **Lombok** － 精簡 Java POJO 程式碼。
- **Joda-Money** － 處理金額型別。
- **Netty** － 高效能網路通訊底層。

## 專案結構

```
webclient-demo/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── tw/fengqing/spring/reactor/webclient/
│   │   │       ├── model/         # 資料模型（如 Coffee）
│   │   │       ├── support/       # Money 序列化/反序列化
│   │   │       └── WebclientDemoApplication.java # 主程式（包含WebClient配置）
│   │   └── resources/
│   │       └── application.properties
│   └── test/
├── pom.xml
└── README.md
```

## 快速開始

### 前置需求
- JDK 21（建議使用 OpenJDK 21 或 Oracle JDK 21）
- Maven 3.8+
- 可連線的 REST API 伺服器（本專案預設連線 http://localhost:8080/coffee）

### 安裝與執行

1. **克隆此倉庫：**
```bash
git clone https://github.com/your-org/webclient-demo.git
```

2. **進入專案目錄：**
```bash
cd webclient-demo
```

3. **編譯專案：**
```bash
mvn clean package -Dmaven.test.skip=true
```

4. **執行應用程式：**
```bash
java -jar target/webclient-demo-0.0.1-SNAPSHOT.jar
```

## 進階說明

### 環境變數
（本專案預設無需資料庫連線，僅需 REST API 端點可用即可）

### 設定檔說明
```properties
# application.properties 主要設定
# 預設無特殊設定，如需自訂 WebClient baseUrl 請於 WebClientConfig.java 調整
```

## 參考資源
- [Spring WebFlux 官方文件](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html)
- [Spring Boot 官方文件](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)
- [Project Reactor](https://projectreactor.io/)
- [Netty 官方網站](https://netty.io/)

## 注意事項與最佳實踐

### ⚠️ 重要提醒

| 項目   | 說明             | 建議做法         |
|--------|------------------|-----------------|
| 註解   | 重要程式區塊註解 | 團隊維護更容易   |
| 專業用語 | 台灣常用術語      | 文件/註解皆本地化 |
| 效能   | Reactive 設計     | 適合高併發場景   |
| 安全性 | API 金鑰/密碼管理 | 使用環境變數     |

### 🔒 最佳實踐指南
- 重要邏輯務必加上註解，方便團隊成員理解與維護。
- 盡量使用台灣常用專業用語，確保團隊溝通順暢。
- Reactive 程式設計適合高併發、非同步需求。
- 外部 API 端點建議以環境變數或設定檔管理。
- 測試時可搭配 mock server 或本地 REST API。

## 授權說明

本專案採用 MIT 授權條款，詳見 LICENSE 檔案。

## 關於我們

我們專注於敏捷專案管理、物聯網（IoT）應用開發與領域驅動設計（DDD），致力於將先進技術與實務經驗結合，打造好用又靈活的軟體解決方案。

## 聯繫我們

- **FB 粉絲頁**：[風清雲談 | Facebook](https://www.facebook.com/profile.php?id=61576838896062)
- **LinkedIn**：[linkedin.com/in/chu-kuo-lung](https://www.linkedin.com/in/chu-kuo-lung)
- **YouTube 頻道**：[雲談風清 - YouTube](https://www.youtube.com/channel/UCXDqLTdCMiCJ1j8xGRfwEig)
- **風清雲談 部落格**：[風清雲談](https://blog.fengqing.tw/)
- **電子郵件**：[fengqing.tw@gmail.com](mailto:fengqing.tw@gmail.com)

---

**📅 最後更新：[2025-07-25]**  
**👨‍💻 維護者：[風清雲談團隊]** 