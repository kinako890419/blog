+++
title = "[課程筆記] Spring Boot 與 Spring Security 實作 RESTful API"
date = "2025-07-04T00:00:00+08:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
tags = ["課程筆記", "SpringAcademy", "SpringBoot"]
keywords = ["Spring Boot 課程筆記", "Spring Academy 筆記"]
showFullContent = false
readingTime = true
hideComments = false
+++

> 課程連結：https://spring.academy/courses/building-a-rest-api-with-spring-boot


# Part 0: 大綱

## 專案目標

建立一個家庭零用錢管理應用：

- 利用 Spring Boot 創建 **RESTful API** 與網頁應用
- Spring Data, Spring Security 初探
- 專案導向的課程
- Test-First 開發方法: 通常會在實作之前編寫測試

## 內容總覽

1. RESTful API 基礎設計原則
2. 測試導向開發（TDD）
3. 使用 Spring Boot 開發 RESTfulAPI
4. Spring Security 應用
5. 安全性機制與風險防護

<!--more-->

## 檔案架構

```
📦 src
├── 📁 main
│   ├── 📁 java
│   │   └── 📁 com.springacademy.cashcard
│   │       ├── 📄 CashCard.java             // 資料 Entity：代表一筆 CashCards，包含 id、amount、owner
│   │       ├── 📄 CashcardApplication.java  // 應用啟動入口（Spring Boot main class）
│   │       ├── 📄 CashCardController.java   // 負責接收 REST API 請求，回傳 HTTP 回應（GET/POST/PUT/DELETE）
│   │       ├── 📄 CashCardRepository.java   // 資料存取介面：繼承自 `CrudRepository` 和 `PagingAndSortingRepository`，提供資料庫 CRUD 功能
│   │       └── 📄 SecurityConfig.java       // Spring Security 安全性設定：角色權限、開啟/關閉 CSRF、HTTP Basic 認證
│   │
│   └── 📁 resources
│       ├── 📁 static
│       ├── 📁 templates
│       ├── 📄 application.properties
│       └── 📄 schema.sql                    // 專案啟動時會自動執行，用來建立 `CASH_CARD` 資料表
│
├── 📁 test
│   ├── 📁 java
│   │   └── 📁 com.springacademy.cashcard
│   │       ├── 📄 CashCardApplicationTests.java // 驗證 API 行為正確性
│   │       └── 📄 CashCardJsonTest.java         // JSON 測試：確認序列化與反序列化是否正確
│   │
│   └── 📁 resources
│       ├── 📁 com.springacademy.cashcard
│       │   ├── 📄 expected.json             // 單筆資料的預期 JSON 格式（for JSON 測試）
│       │   └── 📄 list.json                 // 多筆資料的預期 JSON 格式（for JSON 測試）
│       └── 📄 data.sql                      // 測試資料 SQL：在測試執行前匯入資料


```

---

# Part1: Spring Boot 開發 Restful API 與 TDD 初探

## Idempotence（冪等性） and HTTP

- Idempotence代表不管執行多少次，都可以得到相同的結果，不會有副作用、不會改變狀態
- HTTP Methods當中，`GET`、`PUT`、`DELETE`是冪等的，而`POST`和`PATCH`不是
- 例如新增一筆交易(POST)，應用如果使用自動生成的ID來辨識交易，每一筆交易的ID都會不同，因此不是冪等的

| 方法         | 用途        | 是否 Idempotent| 常見用途說明             |
| ---------- | --------- | ----------------- | ------------------ |
| **GET**    | 取得資源      | ✅ 是               | 查詢資料、讀取清單或單筆資料     |
| **POST**   | 建立資源／提交資料 | ❌ 否               | 新增資料（如建立帳戶、下訂單）    |
| **DELETE** | 刪除資源      | ✅ 是               | 刪除特定資源（如刪除帳號）      |
| **PUT**    | 整份更新／建立資源 | ✅ 是               | 更新*整個資源*內容，或直接設定資源 |
| **PATCH**  | 部分更新資源    | ❌ 否（不能保證）         | 修改部分欄位（如改密碼、加分）    |

- 冪等性影響到是否可重試：如果這個請求失敗了（例如網路斷線、伺服器沒回應），能不能放心地再發一次同樣的請求，而不用擔心會造成錯誤或重複副作用？
- PATCH方法無法保證每次都可以符合冪等性

## URI, URL, URI 的區別

### URI (Uniform Resource Identifiers)

- URI的功能是用於識別web上的資源，可能是文件、照片...等
- 發送HTTP請求時，請求的目標通常是一個URI
- 最常見的 URI 類型是 URL
- URI 除了用來取得資源（例如打開網頁、下載圖片）之外，還可以觸發不同的行為。例如開啟email程式 `<a href="mailto:someone@example.com">寄信給我們</a>` 
	- 這些 URI 通常用在 HTML 的 `<a>` 標籤中的 `href` 屬性，透過使用者點擊連結時觸發特定行為

#### URI 的組成格式

##### Schemes

- 位於URI的最開頭，指定瀏覽器必須使用哪種協議來獲取資源

##### Authority

- 位於Schemes的後面，Path之前的部分
- 包含用戶資訊、host和port

##### Path

- 位於Authority之後，資料通常是階層式組成(hierarchical form)
- 目的是在 URI 所屬的範圍內，識別出特定的資源

##### Query

- 位於Path之後，對資源進一步提供查詢條件或參數

##### Fragment

- URI 的最後一部分，以 `#` 開頭
- 用於標識資源中的特定部分，例如文檔的其中一部分

### URL (Uniform Resource Locator)

- URI的子集，用於網路資源的定位
- URL和URI可以視為是一樣的東西，捨棄嚴格的劃分方式，整合URI、URL及URN ([source](https://www.w3.org/TR/uri-clarification/))

### URN

- 指定資源的**名稱**

---

- [What is the difference between URI, URL and URN?](https://stackoverflow.com/questions/4913343/what-is-the-difference-between-uri-url-and-urn)
- [URIs - MDN Web Docs - Mozilla](https://developer.mozilla.org/en-US/docs/Web/URI)
- [What is a URL? - Learn web development | MDN](https://developer.mozilla.org/en-US/docs/Learn_web_development/Howto/Web_mechanics/What_is_a_URL)
- [URI、URL 與URN - HackMD](https://hackmd.io/@CynthiaChuang/URI-URL-and-URN)
- [URI 跟URL差異 - HackMD](https://hackmd.io/@BrentKao/BJMfdJ2O2)

---

## Test Driven Development (TDD)

> 撰寫測試程式 (提供API開發一個指引) ⭢ 實作API以符合測試

1. 測試序列化與反序列化
2. 測試API的正確性

### `@JsonTest`測試序列化與反序列化

```java {linenos=inline}
package com.springacademy.cashcard;

import org.assertj.core.util.Arrays;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.json.JsonTest;
import org.springframework.boot.test.json.JacksonTester;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThat;

@JsonTest
public class CashCardJsonTest {

    @Autowired
    private JacksonTester<CashCard> json;

    @Autowired
    private JacksonTester<CashCard[]> jsonList;

    @Test
    void cashCardSerializationTest() throws IOException {
        CashCard cashCard = new CashCard(99L, 123.45);
        assertThat(json.write(cashCard)).isStrictlyEqualToJson("expected.json");
        assertThat(json.write(cashCard)).hasJsonPathNumberValue("@.id");
        assertThat(json.write(cashCard)).extractingJsonPathNumberValue("@.id")
                .isEqualTo(99);
        assertThat(json.write(cashCard)).hasJsonPathNumberValue("@.amount");
        assertThat(json.write(cashCard)).extractingJsonPathNumberValue("@.amount")
                .isEqualTo(123.45);
    }

    @Test
    void cashCardDeserializationTest() throws IOException {
        String expected = """
           {
               "id":99,
               "amount":123.45
           }
           """;
        assertThat(json.parse(expected))
                .isEqualTo(new CashCard(99L, 123.45));
        assertThat(json.parseObject(expected).id()).isEqualTo(99);
        assertThat(json.parseObject(expected).amount()).isEqualTo(123.45);
    }
}
```

expected.json:

```json
{  
  "id": 99,  
  "amount": 123.45  
}
```

## GET, POST 單筆資料 API

- 透過id獲取單筆資訊
- post新增一筆新的資料

### 測試程式

```java {linenos=inline}
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class CashCardApplicationTests {

	@Autowired
	TestRestTemplate restTemplate;
	
	@Test
	void shouldReturnACashCardWhenDataIsSaved() {
		ResponseEntity<String> response = restTemplate.getForEntity("/cashcards/1", String.class);

		assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

		DocumentContext documentContext = JsonPath.parse(response.getBody());
		Number id = documentContext.read("$.id");
//		assertThat(id).isNotNull();
		assertThat(id).isEqualTo(1);

		Double amount = documentContext.read("$.amount");
		assertThat(amount).isEqualTo(123.45);
	}

	@Test
	void shouldNotReturnACashCardWithAnUnknownId() {
		ResponseEntity<String> response = restTemplate.getForEntity("/cashcards/1000", String.class);

		assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
		assertThat(response.getBody()).isBlank();
	}

	// 測試 POST 方法
	@Test
	void shouldCreateANewCashCard() {
		// DB 自動生成ID，因此這裡不需要提供
		CashCard newCashCard = new CashCard(null, 250.00);
		// POST 之後不用 return
		ResponseEntity<Void> createResponse = restTemplate.postForEntity("/cashcards", newCashCard, Void.class);
		assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
	}
}
```

### 實作 API -- Spring `UriComponentsBuilder` 介紹

- 在 Web 應用程式中動態產生、修改與擴展網址，減少手動字串拼接帶來的錯誤與繁瑣

#### 範例

- post API -- 成功後不回傳 response body

```java {linenos=inline}
@PostMapping  
private ResponseEntity<Void> createCashCard(@RequestBody CashCard newCashCardRequest, UriComponentsBuilder ucb) {  
    // inject UriComponentsBuilder  
    CashCard savedCashCard = cashCardRepository.save(newCashCardRequest); // save() returns the saved entity with an ID  
    URI locationOfNewCashCard = ucb  
            .path("cashcards/{id}")  
            .buildAndExpand(savedCashCard.id())  // 在URI當中插入變數
            .toUri();  
    System.out.println("New CashCard location: " + locationOfNewCashCard);  
    return ResponseEntity.created(locationOfNewCashCard).build();  // 組裝並產生一個 HTTP 回應物件 (ResponseEntity)，回傳 HTTP 201 Created 回應
}
```

- post API 成功後回傳 response body

```java
return ResponseEntity.created(locationOfNewCashCard).body(...); 
```


### 實作 API -- 程式碼

```java {linenos=inline}
package com.springacademy.cashcard;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.util.UriComponentsBuilder;

import java.net.URI;
import java.util.Optional;

@RestController
@RequestMapping("/cashcards")
class CashCardController {

    private final CashCardRepository cashCardRepository;

    private CashCardController(CashCardRepository cashCardRepository) {
        this.cashCardRepository = cashCardRepository;
    }

    @GetMapping("/{requestedId}")
    private ResponseEntity<CashCard> findById(@PathVariable Long requestedId) {
        Optional<CashCard> cashCardOptional = cashCardRepository.findById(requestedId);
        if (cashCardOptional.isPresent()) {
            return ResponseEntity.ok(cashCardOptional.get());
        } else {
            return ResponseEntity.notFound().build();
        }
    }

//    @PostMapping
//    private ResponseEntity<Void> createCashCard() {
//        return null; // 如果return null，會自動生成 200 OK status，測試通過
//    }

    @PostMapping
    private ResponseEntity<Void> createCashCard(@RequestBody CashCard newCashCardRequest, UriComponentsBuilder ucb) {
        // inject UriComponentsBuilder
        CashCard savedCashCard = cashCardRepository.save(newCashCardRequest); // save() returns the saved entity with an ID
        URI locationOfNewCashCard = ucb
                .path("cashcards/{id}")
                .buildAndExpand(savedCashCard.id())
                .toUri();
        System.out.println("New CashCard location: " + locationOfNewCashCard);
        return ResponseEntity.created(locationOfNewCashCard).build();
    }}

```

## GET 列出所有 CashCards 資料

### PagingAndSortingRepository 分頁

- 優化查詢效能，縮短使用者的等待時間
- 明確設定排序方式，不建議使用預設排序，以免造成資料難以維護與測試

```java {linenos=inline}
@GetMapping
private ResponseEntity<List<CashCard>> findAll(Pageable pageable) {
   Page<CashCard> page = cashCardRepository.findAll(
           PageRequest.of(
                   pageable.getPageNumber(),
                   pageable.getPageSize(),
                   pageable.getSortOr(Sort.by(Sort.Direction.DESC, "amount"))));
   return ResponseEntity.ok(page.getContent());
}
```

對應的 API URL 例如: `/cashcards?page=1&size=3&sort=amount,desc`
- Pageable 物件會自動解析網址中的 page, size, sort 參數
- 用`.getContent()` 來只取出資料內容，不回傳分頁資訊

#### 更新 Repository class 來應用 paging 與 sorting

```java {linenos=inline}
package com.springacademy.cashcard;  
  
import org.springframework.data.repository.CrudRepository;  
import org.springframework.data.repository.PagingAndSortingRepository;  
  
public interface CashCardRepository extends CrudRepository<CashCard, Long>, PagingAndSortingRepository<CashCard, Long> {  
}
```

#### Page 物件格式範例

- content 當中包含實際資料

```JSON
{
  "content": [
    {
      "id": 1,
      "amount": 10.0
    },
    {
      "id": 2,
      "amount": 0.19
    }
  ],
  "pageable": {
    "pageNumber": 1,
    "pageSize": 3
  },
  "totalElements": 5,
  "totalPages": 2,
  "first": false,
  "last": true
}
```

---

# Part 2: Spring Boot Restful API 結合 Spring Security

## Security 觀念介紹

### Same Origin Policy

- 如果兩個 URL 的protocol 、port 和 host 相同，則這兩個 URL 就是相同的origin
- SOP是一種安全機制，限制只有來自相同來源的腳本程式才能向該來源發送請求
- 有助於減少可能的攻擊媒介。例如：防止惡意網站在使用者的瀏覽器中偷偷執行 JavaScript，去讀取使用者登入的 Webmail 服務資料，或是讀取一些只能在公司內部使用、沒有對外開放的內部網站內容，然後再把這些資料偷偷傳給攻擊者

---

- [Same-origin policy - Security | MDN](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)

---

### Cross-Origin Resource Sharing (CORS)

- 現在系統經常由多個微服務構成，各自部署在不同的 URI 下。這樣就需要透過 CORS（跨來源資源共用） 來讓不同來源之間的請求被接受
- Spring Security 提供 `@CrossOrigin` 註解用來設定被允許的來源；但是如果使用了 `@CrossOrigin` 而未指定任何來源，則預設為開放所有來源，會造成安全風險

### Authentication (認證) & Authorization (授權)

> 在 API 的使用者中，可能是人，也可能是另一個程式，因此常使用"**Principal**"這個詞來代替"使用者"

- 認證是讓主體向系統證明自己的身分，當主體提供了正確的憑證，就可以稱該主體已驗證（Authenticated）
- 認證的接下來是授權，授權的目的是**區分不同使用者的權限**，決定哪些操作是被允許的

#### Authentication Session

- 由於 HTTP 是 stateless protocol，每一個請求都必須自行攜帶能證明使用者身分的資訊，效率較低，因此這裡採用Authentication Session (又可稱作 Auth Session, Session) 的方式
- 作法是生成一組隨機字串作為**Session Token**，並將其儲存在用戶端的 **Cookie** 中

##### 什麼是 Cookie

- Cookie 是儲存在瀏覽器端的一組資料，會自動隨著對應的 URI 一起傳送到伺服器
- Cookie 的兩大優點：
    1. **自動隨每次請求送出**，不需額外撰寫程式碼；
    2. **具有持久性**，即使關閉網頁後再次造訪仍可維持登入狀態，提升使用者體驗

### Role-Based Access Control（RBAC）

- Spring Security 採用 RBAC 來進行授權控制
- 每個主體（Principal）都擁有一組角色（Roles）
- 每項資源或操作會定義哪些角色可以存取
- RBAC 可應用在整體架構（全域）或單一方法層級上
- 例如：
	- 擁有「管理員角色」的使用者可執行更多操作
	- 「卡片擁有者角色」則限制於個人卡片管理

### 常見的網頁攻擊（Web Exploits）

#### 跨站請求偽造（Cross-Site Request Forgery, CSRF）

> Spring Security 預設啟用 CSRF 保護，開發者無須額外設定即可使用


- 使用者在登入一網站之後造訪其他惡意網站，讓惡意網站有機會假冒使用者的身份來發送非法的請求給其他網站
- 受害網站無法辨別請求是否為合法操作

#### 跨站腳本攻擊（Cross-Site Scripting, XSS）

- 將惡意腳本注入到網站內容中，使受害者在瀏覽網頁時自動執行該腳本

## Gradle.build 加入 Spring Security dependency

```gradle
dependencies {  
    implementation 'org.springframework.boot:spring-boot-starter-web'  
    implementation 'org.springframework.boot:spring-boot-starter-security'  
    testImplementation 'org.springframework.boot:spring-boot-starter-test'  
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'  
    implementation 'org.springframework.data:spring-data-jdbc'  
    implementation 'com.h2database:h2'  
}
```

![alt](/blog/images/gradle_image.png)

⭢ Spring Boot 預設會開啟安全機制保護所有 HTTP 路徑（`/**`），因此所有 API 都需要身份驗證（比如帳號密碼）才能存取，所有受保護的 endpoint 都會回傳 `401 Unauthorized`

## 設置 Spring Security

```java {linenos=inline}
package com.springacademy.cashcard;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### @Configuration 註解

- 加上註解的class代表該class是一個用來設定 Bean 的類別
- @Configuration class 當中定義的 bean 都會被 spring container註冊起來，在後續專案中使用該設定，不然會使用預設的設定

### @Bean 註解

- 用來**標記方法**，表示這個方法會**回傳一個物件**

## 設置基礎的 Authentication

```java {linenos=inline}
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
	http
			.authorizeHttpRequests(request -> request
					.requestMatchers("/cashcards/**")
					.authenticated())
			.httpBasic(Customizer.withDefaults())
			.csrf(csrf -> csrf.disable());
	return http.build();
}

@Bean
UserDetailsService testOnlyUsers(PasswordEncoder passwordEncoder) {
	User.UserBuilder users = User.builder();
	UserDetails sarah = users
			.username("sarah1")
			.password(passwordEncoder.encode("abc123"))
			.roles() // No roles for now
			.build();
	return new InMemoryUserDetailsManager(sarah);
}
```

- 如果 API 是給*非瀏覽器客戶端（non-browser clients*）使用的，就可以關閉 CSRF 保護
- 此範例中是實作 Restful API，並不涉及 cookie/session 式身份驗證，因此可以選擇關閉 CSRF 保護機制

### 測試認證功能

#### test data (sql)

main/resources/schema.sql

```sql {linenos=inline}
CREATE TABLE cash_card  
(  
   ID     BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,  
   AMOUNT   NUMBER NOT NULL DEFAULT 0,  
   OWNER    VARCHAR(256) NOT NULL  
);
```

test/resources/data.sql

```sql 
INSERT INTO CASH_CARD(ID, AMOUNT, OWNER) VALUES (100, 1.00, 'sarah1');
```

#### 單元測試

```java {linenos=inline}
@Test  
void shouldReturnACashCardWhenDataIsSaved() {  
    ResponseEntity<String> response = restTemplate  
          .withBasicAuth("sarah1", "abc123") // update  
          .getForEntity("/cashcards/100", String.class);  
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);  
  
    DocumentContext documentContext = JsonPath.parse(response.getBody());  
    Number id = documentContext.read("$.id");  
    assertThat(id).isEqualTo(100);  
  
    Double amount = documentContext.read("$.amount");  
    assertThat(amount).isEqualTo(1.0);  
}
```

## 實作 RBAC

- 測試ROLES認證功能

```java {linenos=inline}
@Test
void shouldRejectUsersWhoAreNotCardOwners() {
    ResponseEntity<String> response = restTemplate
      .withBasicAuth("hank-owns-no-cards", "qrs456")
      .getForEntity("/cashcards/99", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.FORBIDDEN);
}
```

- 增加更多不同ROLES

```java {linenos=inline}
@Bean
UserDetailsService testOnlyUsers(PasswordEncoder passwordEncoder) {
  User.UserBuilder users = User.builder();
  UserDetails sarah = users
    .username("sarah1")
    .password(passwordEncoder.encode("abc123"))
    .roles("CARD-OWNER") // new role
    .build();
  UserDetails hankOwnsNoCards = users
    .username("hank-owns-no-cards")
    .password(passwordEncoder.encode("qrs456"))
    .roles("NON-OWNER") // new role
    .build();
  return new InMemoryUserDetailsManager(sarah, hankOwnsNoCards);
}
```

- 更新 FilterChain

```java {linenos=inline}
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
     http
             .authorizeHttpRequests(request -> request
                     .requestMatchers("/cashcards/**")
                     .hasRole("CARD-OWNER")) // enable RBAC: Replace the .authenticated() call with the hasRole(...) call.
             .httpBasic(Customizer.withDefaults())
             .csrf(csrf -> csrf.disable());
     return http.build();
}
```

⭢ 問題：任何已驗證並擁有 `CARD-OWNER` 角色的使用者，都可以查看其他人的 CashCards

### 更新 `CashCardRepository` 以限制查詢資料範圍

> 不應依賴框架去做所有資料保護

#### 新增一名使用者

```sql
INSERT INTO CASH_CARD(ID, AMOUNT, OWNER) VALUES (102, 200.00, 'kumar2');
```

#### 測試*不能*存取別人的資料

```java {linenos=inline}
@Test
void shouldNotAllowAccessToCashCardsTheyDoNotOwn() {
    ResponseEntity<String> response = restTemplate
      .withBasicAuth("sarah1", "abc123")
      .getForEntity("/cashcards/102", String.class); // kumar2 的資料
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

error:

```
Expected :404 NOT_FOUND
Actual   :200 OK
<Click to see difference>

org.opentest4j.AssertionFailedError: 
expected: 404 NOT_FOUND
 but was: 200 OK
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:77)
	at java.base/jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.base/java.lang.reflect.Constructor.newInstanceWithCaller(Constructor.java:500)
	at com.springacademy.cashcard.CashCardApplicationTests.shouldNotAllowAccessToCashCardsTheyDoNotOwn(CashcardApplicationTests.java:178)
	at java.base/java.lang.reflect.Method.invoke(Method.java:569)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
```

透過以下測試可以發現，**資料查詢未過濾 OWNER，看到不該看的資料**

```java {linenos=inline}
@Test
	void shouldReturnAllCashCardsWhenListIsRequested() {

		ResponseEntity<String> response = restTemplate
				.withBasicAuth("sarah1", "abc123")
				.getForEntity("/cashcards", String.class);
		assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

		DocumentContext documentContext = JsonPath.parse(response.getBody());
		int cashCardCount = documentContext.read("$.length()");
		assertThat(cashCardCount).isEqualTo(3);
	}
```

#### 更新 `CashCardRepository` 來過濾 owner

```java {linenos=inline}
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.PagingAndSortingRepository;

interface CashCardRepository extends CrudRepository<CashCard, Long>, PagingAndSortingRepository<CashCard, Long> {
    CashCard findByIdAndOwner(Long id, String owner);
    Page<CashCard> findByOwner(String owner, PageRequest pageRequest);
}
```

### 更新 Controller 來應用過濾 owner 的功能

在 Spring Security 裡，當使用者通過身份驗證後，系統會提供一個 `Principal` 物件來代表這位使用者
- `Principal` 包含使用者的登入資訊
- 可以使用 `principal.getName()` 取得該使用者的帳號名稱（Basic Auth 傳進來的 username）

⭢ 把 `Principal` 當作參數加入方法中，使用 findByIdAndOwner(...) 來確保只查詢該使用者自己的CashCards

```java {linenos=inline}
@GetMapping("/{requestedId}")
private ResponseEntity<CashCard> findById(@PathVariable Long requestedId, Principal principal) {
	Optional<CashCard> cashCardOptional = Optional.ofNullable(cashCardRepository.findByIdAndOwner(requestedId, principal.getName()));
	if (cashCardOptional.isPresent()) {
		return ResponseEntity.ok(cashCardOptional.get());
	} else {
		return ResponseEntity.notFound().build();
	}
}

@GetMapping  
private ResponseEntity<List<CashCard>> findAll(Pageable pageable, Principal principal) {  
    Page<CashCard> page = cashCardRepository.findByOwner(principal.getName(),  
            PageRequest.of(  
                    pageable.getPageNumber(),  
                    pageable.getPageSize(),  
                    pageable.getSortOr(Sort.by(Sort.Direction.ASC, "amount"))  
            ));    return ResponseEntity.ok(page.getContent());  
}
```

### 測試 API

```java {linenos=inline}
@Test
void shouldReturnAllCashCardsWhenListIsRequested() {

//		ResponseEntity<String> response = restTemplate.getForEntity("/cashcards", String.class);
	ResponseEntity<String> response = restTemplate
			.withBasicAuth("sarah1", "abc123")
			.getForEntity("/cashcards", String.class);
	assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

	DocumentContext documentContext = JsonPath.parse(response.getBody());
	int cashCardCount = documentContext.read("$.length()");
	assertThat(cashCardCount).isEqualTo(3);

	JSONArray ids = documentContext.read("$..id");
	assertThat(ids).containsExactlyInAnyOrder(99, 100, 101);

	JSONArray amounts = documentContext.read("$..amount");
	assertThat(amounts).containsExactlyInAnyOrder(123.45, 1.0, 150.00);
}
```

![alt](/blog/images/pstmn_image.png)

### 新增功能的認證

- 測試 post API，使用一個有登入但沒有權限的帳號

```java {linenos=inline}
@Test
@DirtiesContext
void shouldCreateANewCashCard() {
	CashCard newCashCard = new CashCard(null, 250.00, null);
//		ResponseEntity<Void> createResponse = restTemplate.postForEntity("/cashcards", newCashCard, Void.class);
	ResponseEntity<Void> createResponse = restTemplate.withBasicAuth("sarah1", "abc123").postForEntity("/cashcards", newCashCard, Void.class);
	assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);

	URI locationOfNewCashCard = createResponse.getHeaders().getLocation();
	ResponseEntity<String> getResponse = restTemplate.getForEntity(locationOfNewCashCard, String.class);
	assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);

	DocumentContext documentContext = JsonPath.parse(getResponse.getBody());
	Number id = documentContext.read("$.id");
	Double amount = documentContext.read("$.amount");

	assertThat(id).isNotNull();
	assertThat(amount).isEqualTo(250.00);
}
```

error:

```
org.opentest4j.AssertionFailedError: 
expected: 201 CREATED
 but was: 403 FORBIDDEN
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:77)
	at java.base/jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.base/java.lang.reflect.Constructor.newInstanceWithCaller(Constructor.java:500)
	at com.springacademy.cashcard.CashCardApplicationTests.shouldCreateANewCashCard(CashcardApplicationTests.java:88)
	at java.base/java.lang.reflect.Method.invoke(Method.java:569)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
```

⮕ 更新 Controller

```java {linenos=inline}
@PostMapping
private ResponseEntity<Void> createCashCard(@RequestBody CashCard newCashCardRequest, UriComponentsBuilder ucb, Principal principal) {
	CashCard cashCardWithOwner = new CashCard(null, newCashCardRequest.amount(), principal.getName());
	CashCard savedCashCard = cashCardRepository.save(cashCardWithOwner);
	URI locationOfNewCashCard = ucb
			.path("cashcards/{id}")
			.buildAndExpand(savedCashCard.id())
			.toUri();
	System.out.println("New CashCard location: " + locationOfNewCashCard);
	return ResponseEntity.created(locationOfNewCashCard).build();
}
```

---

- [Spring Security Reference docs.spring.io](https://docs.spring.io/spring-security/site/docs/5.2.0.RELEASE/reference/html/index.html)
- [Spring Security :: Spring Security](https://docs.spring.io/spring-security/reference/index.html)

---

## 更新資料的方法: POST, PUT 與 PATCH 的差異

| HTTP 方法 | 操作  | URI              | 功能說明             | 回傳狀態碼          |
| ------- | --- | ---------------- | ---------------- | -------------- |
| POST    | 建立  | **由伺服器產生URI**並回傳 | 建立一個資源           | 201 CREATED    |
| PUT     | 建立  | 客戶端提供            | 建立一個資源           | 201 CREATED    |
| PUT     | 更新  | 客戶端提供            | **完整取代現有資料**     | 204 NO CONTENT |
| PATCH   | 更新  | 客戶端提供            | **部分更新：只改變傳入欄位** | 200 OK         |

- POST 例如呼叫 `POST /cashcards`，會建立一個 `/cashcards/101` 資源
- PUT 例如 `PUT /invoice/1234-567` 會直接放一個資源到這個 URI 上


## PUT 更新 CashCards 的 amounts 資料

### 測試程式

```java {linenos=inline}
import org.springframework.http.HttpEntity;  
import org.springframework.http.HttpMethod;

// 測試 PUT 方法  
@Test  
@DirtiesContext  
void shouldUpdateAnExistingCashCard() {  
    CashCard cashCardUpdate = new CashCard(null, 19.99, null);  
    //把 CashCard 包裝成一個 HTTP 請求實體（包含在 HTTP body 中）  
    HttpEntity<CashCard> request = new HttpEntity<>(cashCardUpdate);  
    ResponseEntity<Void> response = restTemplate  
          .withBasicAuth("sarah1", "abc123")  
          .exchange("/cashcards/99", HttpMethod.PUT, request, Void.class);  
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NO_CONTENT);  
  
    ResponseEntity<String> getResponse = restTemplate  
          .withBasicAuth("sarah1", "abc123")  
          .getForEntity("/cashcards/99", String.class);  
    assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);  
    
    DocumentContext documentContext = JsonPath.parse(getResponse.getBody());  
    Number id = documentContext.read("$.id");  
    Double amount = documentContext.read("$.amount");  
    assertThat(id).isEqualTo(99);  
    assertThat(amount).isEqualTo(19.99);  
}
```

- RestTemplate 一般的 PUT (`.put()`方法) 成功時會回傳 201或是200 response，因此沒有`putForEntity()`這個方法 => 使用 `exchange()` 方法替代
- `exhange(URL, HTTP方法, request HTTPEntity, response類型, ...URI變數)` 可以執行指定的HTTP方法並回傳 response entity

### 測試資料

```sql
INSERT INTO CASH_CARD(ID, AMOUNT, OWNER) VALUES (99, 100.00, 'sarah1');
```

### 實作 API

```java {linenos=inline}
@PutMapping("/{requestedId}")  
private ResponseEntity<Void> putCashCard(@PathVariable Long requestedId, @RequestBody CashCard cashCardUpdate, Principal principal) {  
    CashCard cashCard = cashCardRepository.findByIdAndOwner(requestedId, principal.getName());  
    CashCard updatedCashCard = new CashCard(cashCard.id(), cashCardUpdate.amount(), principal.getName());  
    cashCardRepository.save(updatedCashCard);  
    return ResponseEntity.noContent().build();  
}
```

⭢ 回傳 HTTP 狀態碼 204 NO CONTENT，表示更新成功，但不附帶任何內容

## 測試更新不存在的CashCards資料

```java {linenos=inline}
@Test
void shouldNotUpdateACashCardThatDoesNotExist() {
    CashCard unknownCard = new CashCard(null, 19.99, null);
    HttpEntity<CashCard> request = new HttpEntity<>(unknownCard);
    ResponseEntity<Void> response = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/99999", HttpMethod.PUT, request, Void.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

測試 Failed，錯誤如下:

```
ava.lang.NullPointerException: Cannot invoke "com.springacademy.cashcard.CashCard.id()" because "cashCard" is null
...
org.opentest4j.AssertionFailedError: 
expected: 404 NOT_FOUND
 but was: 403 FORBIDDEN
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:77)
	at java.base/jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.base/java.lang.reflect.Constructor.newInstanceWithCaller(Constructor.java:500)
	at com.springacademy.cashcard.CashCardApplicationTests.shouldNotUpdateACashCardThatDoesNotExist(CashcardApplicationTests.java:238)
	at java.base/java.lang.reflect.Method.invoke(Method.java:569)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
```

## 處理 `NullPointerException`

### 更新 Controller 層的程式

```java {linenos=inline}
@PutMapping("/{requestedId}")
private ResponseEntity<Void> putCashCard(@PathVariable Long requestedId, @RequestBody CashCard cashCardUpdate, Principal principal) {
    CashCard cashCard = cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
    if (cashCard != null) {
        CashCard updatedCashCard = new CashCard(cashCard.id(), cashCardUpdate.amount(), principal.getName());
        cashCardRepository.save(updatedCashCard);
        return ResponseEntity.noContent().build();
    }
    return ResponseEntity.notFound().build();
}
```

## 測試更新他人的 CashCards 資料 (未授權的情況)

```java {linenos=inline}
@Test
void shouldNotUpdateACashCardThatIsOwnedBySomeoneElse() {
    CashCard kumarsCard = new CashCard(null, 333.33, null);
    HttpEntity<CashCard> request = new HttpEntity<>(kumarsCard);
    ResponseEntity<Void> response = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/102", HttpMethod.PUT, request, Void.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

## 執行 DELETE 方法後可能的 response

| 回應碼              | 使用情境                                           |
| ---------------- | ---------------------------------------------- |
| `204 NO CONTENT` | 該紀錄存在｜使用者（Principal）有授權｜紀錄成功被刪除 |
| `404 NOT FOUND`  | 該紀錄不存在（也就是送了一個不存在的 ID）。                      |
| `404 NOT FOUND`  | 紀錄存在，但使用者並不是該紀錄的擁有者，沒有刪除權限。                  |

- 沒有權限刪除會顯示404error的原因是為了避免洩漏資訊，如果記錄簿存在和無權限刪除兩個情況回應碼不同，惡意的使用者可以藉此猜出哪些資料存在，進而猜測資料庫結構或資料內容

## Hard Delete v.s. Soft Delete

### Hard Delete

- 直接刪除DB當中的資料，無法復原
- 日後需要那筆資料的話就會找不到

⭢ 刪除成功後的response 可能是 `204 NO CONTENT`

### Soft Delete

- 並沒有真正的刪除資料，而是加上一個「已刪除」的標記
- 例如加上 IS_DELETE欄位存放 boolean，或是加上 DELETED_DATE Timestamp

⭢ 刪除成功後的response 可能是 `200 OK`

## 稽核紀錄（Audit Trail）與歸檔（Archiving）

在實際使用資料庫時，經常會有需求要保留資料的變更紀錄。例如：
- 客服可能需要知道使用者什麼時候刪除了某張 Cash Card。
- 有些法律規範要求，刪除的資料必須保留一段時間。
如果採用硬刪除，那我們就需要額外儲存這些資訊。以下是常見做法：
1. **歸檔（Archive）**：  
    將刪除的資料搬到另一個位置（資料表或資料庫）。
2. **加入稽核欄位（Audit Fields）**，例如：  
    - `DELETED_DATE`：刪除的時間
    - `DELETED_BY_USER`：誰刪的
    - *也可以用在新增和更新上*
3. **稽核紀錄（Audit Trail）**：保存每一筆操作的完整記錄（不只是最近一次）。這些紀錄可以存在另一個資料表，或是或日誌檔案（log file）

## DELETE 刪除單筆 CashCards 資料

### 測試程式

```java {linenos=inline}
@Test
@DirtiesContext
void shouldDeleteAnExistingCashCard() {
    ResponseEntity<Void> response = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/99", HttpMethod.DELETE, null, Void.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NO_CONTENT);

	ResponseEntity<String> getResponse = restTemplate
	            .withBasicAuth("sarah1", "abc123")
	            .getForEntity("/cashcards/99", String.class);
    assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

### 實作 API

```java {linenos=inline}
@DeleteMapping("/{id}")
private ResponseEntity<Void> deleteCashCard(@PathVariable Long id) {
    cashCardRepository.deleteById(id); // Add this line
    return ResponseEntity.noContent().build();
}
```

## 測試刪除不存在的 CashCards 資料

```java {linenos=inline}
@Test
void shouldNotDeleteACashCardThatDoesNotExist() {
    ResponseEntity<Void> deleteResponse = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/99999", HttpMethod.DELETE, null, Void.class);
    assertThat(deleteResponse.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

測試 Failed，錯誤如下:

```
Expected :404 NOT_FOUND
Actual   :204 NO_CONTENT
<Click to see difference>

org.opentest4j.AssertionFailedError: 
expected: 404 NOT_FOUND
 but was: 204 NO_CONTENT
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:77)
	at java.base/jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.base/java.lang.reflect.Constructor.newInstanceWithCaller(Constructor.java:500)
	at com.springacademy.cashcard.CashCardApplicationTests.shouldNotDeleteACashCardThatDoesNotExist(CashcardApplicationTests.java:271)
	at java.base/java.lang.reflect.Method.invoke(Method.java:569)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
```

⮕ 沒有處理「資料不存在」的情況
⮕ 呼叫 Spring Data JPA 的 deleteById() 方法，如果ID不存在，不會丟出例外，也不會拋錯

### 更新 Controller 層程式

```java {linenos=inline}
@DeleteMapping("/{id}")
    private ResponseEntity<Void> deleteCashCard(@PathVariable Long id, Principal principal) {
//        cashCardRepository.deleteById(id); // Add this line
        if (!cashCardRepository.existsByIdAndOwner(id, principal.getName())) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.noContent().build();
    }
```

### 更新 Repository 層程式

```java {linenos=inline}
package com.springacademy.cashcard;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.PagingAndSortingRepository;

interface CashCardRepository extends CrudRepository<CashCard, Long>, PagingAndSortingRepository<CashCard, Long> {
    CashCard findByIdAndOwner(Long id, String owner);
    Page<CashCard> findByOwner(String owner, PageRequest pageRequest);
    boolean existsByIdAndOwner(Long id, String owner);
}
```

## 測試刪除未授權存取的 CashCards 資料

```java {linenos=inline}
@Test
void shouldNotAllowDeletionOfCashCardsTheyDoNotOwn() {
    ResponseEntity<Void> deleteResponse = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/102", HttpMethod.DELETE, null, Void.class);
    assertThat(deleteResponse.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```