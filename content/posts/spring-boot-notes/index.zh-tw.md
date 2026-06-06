---
title: "Spring Boot 零基礎入門學習筆記"
date: 2022-12-24
draft: false
summary: "自學 Spring Boot 的整理筆記：IoC/DI、AOP、Spring MVC、RESTful API、參數驗證、Spring JDBC、三層式架構，到 JUnit 5 與 Mockito 單元測試。"
tags: ["Java", "Spring Boot", "學習筆記"]
---

這是我當初自學 Spring Boot 的整理筆記，內容對應 Hahow 課程
[《Java 工程師必備！Spring Boot 零基礎入門》](https://hahow.in/courses/5fe22e7fe810e10fc483dd78)。
放上來當作自己的速查表，也希望對入門的人有幫助。

## Spring IoC

IoC（Inversion of Control）：將物件的控制權交由外部的 Spring 容器管理。

優點：

1. Loose coupling（鬆耦合）
2. Lifecycle management（生命週期管理）
3. More testable（方便測試）

DI（Dependency Injection）：透過外部容器取得物件。

- `@Component`：加在 class 上，把 class 變成由 Spring 容器管理的 bean。
- `@Autowired`：加在變數上，從 Spring 容器取得 bean（依賴注入）。

### Bean 的注入

- `@Autowired`：根據變數的類型，在 Spring 容器中尋找符合類型的 bean。
- `@Qualifier`：當同時有兩個 class 實作相同介面時，用 `@Qualifier("{bean}")` 指定 bean。
  （bean 名稱為 class 首字大寫轉小寫。）

### Bean 的創建

1. `@Configuration` + `@Bean`
   - `@Configuration`：表示這個 class 用來設定 Spring。
   - `@Bean`：只能加在帶有 `@Configuration` 的 class 的方法上。
2. 前述 `@Component` 的方式創建 bean。

### Bean 的初始化

1. 使用 `@PostConstruct`。
2. 實作 `InitializingBean` 介面的 `afterPropertiesSet()` 方法。

`@PostConstruct` 的條件：方法必須是 `public`、回傳 `void`、且無參數傳入。

### Bean 的生命週期

1. 創建 → 初始化 → 拿來使用。
2. 創建時若依賴其他 bean，Spring 會回頭先創建並初始化被依賴的 bean。
3. 避免循環依賴。

### Spring Boot 設定檔

`application.properties`，搭配 `@Value("${key:default_value}")` 讀取設定值。

## Spring AOP

AOP（Aspect-Oriented Programming，切面導向程式設計）：

1. `pom.xml`：引入 `spring-boot-starter-aop`。
2. `@Aspect` + `@Component`：加在 class 上，宣告這個 class 是一個切面。
3. 常用切面註解：`@Before` / `@After` / `@Around`，加在宣告的方法上。

AOP 常見應用：

- 權限驗證：Spring Security
- 統一的 Exception 處理：`@ControllerAdvice`
- Log 記錄

## Spring MVC

### `@RequestMapping`

- 用法：加在 class 或方法上，小括號內填寫 URL 路徑。
- 用途：將 URL 路徑對應到方法上。
- 註記：class 上一定要加 `@Controller` / `@RestController`。

### `@Controller` / `@RestController`

- 用法：只加在 class 上。
- 用途：把該 class 變成 bean，並可使用 `@RequestMapping`（等於 `@Component` 加強版）。

## RESTful API

- 目的：簡化工程師之間的溝通成本。
- 設計的 API 符合 REST 風格，就是 RESTful API。
- 它**不是一個規範**，使用最適當的作法即可。

### 取得請求參數

**1. `@RequestParam`**

- 用法：只能加在方法的參數上。
- 用途：取得 URL 裡的參數（query parameter）。
- 設定：`name`（或 `value`）指定參數名、`required`（預設 true）、`defaultValue`（提供預設值）。
- URL 上多傳的參數會被 Spring Boot 忽略。

```java
@RequestMapping("/test1")
public String test1(@RequestParam Integer id,
                    @RequestParam(defaultValue = "Nick") String name) {
    System.out.println("id: " + id);
    System.out.println("name: " + name);
    return "Hello test1";
}
```

**2. `@RequestBody`**

- 用法：只能加在方法的參數上。
- 用途：取得 request body 裡的參數（將 JSON 轉為 Java Object）。

```java
@RequestMapping("/test2")
public String test2(@RequestBody Student student) {
    System.out.println(student);
    return "Hello test2";
}
```

**3. `@RequestHeader`**

- 用法：只能加在方法的參數上。
- 用途：取得 request header 裡的參數。
- 設定：`name`（或 `value`，常用，因為 header 常有橫線 `-`）、`required`、`defaultValue`。

常見的 request header：

| Request header | 意義 | 常見的值 |
| --- | --- | --- |
| Content-Type | request body 的格式 | `application/json`（最常用）、`application/octet-stream`（上傳檔案）、`multipart/form-data`（上傳圖片） |
| Authorization | 用於身份驗證 | |

```java
@RequestMapping("/test3")
public String test3(@RequestHeader String info) {
    System.out.println("info: " + info);
    return "Hello Test3";
}
```

**4. `@PathVariable`**

- 用法：只能加在方法的參數上。
- 用途：取得 URL 路徑中的值。

```java
@RequestMapping("/test4/{id}")
public String test4(@PathVariable Integer id) {
    System.out.println("id: " + id);
    return "Hello Test4";
}
```

> 上述四種可以混用。

### 回應狀態

Spring Boot 預設返回的 HTTP 狀態碼：正常執行完方法返回 200；噴出 exception 返回 500。

`ResponseEntity<?>`：作為方法的返回類型，可自定義回傳的 HTTP response 細節。

```java
@RequestMapping("/test")
public ResponseEntity<String> test() {
    return ResponseEntity.status(HttpStatus.ACCEPTED)
                         .body("Hello World");
}
```

`@ControllerAdvice`：加在 class 上，把它變成 bean，並可在內部使用 `@ExceptionHandler`（底層由 Spring AOP 實作）。

```java
@ControllerAdvice
public class MyExceptionHandler {
    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<String> handle(RuntimeException exception) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body("RuntimeException:" + exception.getMessage());
    }
}
```

### 攔截器（Interceptor）

用途：決定要不要允許這次的 HTTP request 通過、進到 Controller 執行對應方法。

```java
@Configuration
public class MyConfig implements WebMvcConfigurer {
    @Autowired
    private MyInterceptor myInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(myInterceptor).addPathPatterns("/**");
    }
}
```

```java
@Component
public class MyInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        System.out.println("執行 MyInterceptor 的 preHandle 方法");
        response.setStatus(401);
        return false;
    }
}
```

### REST 風格

用 HTTP method 表示動作：

| HTTP method | 對應的資料庫操作 | 說明 |
| --- | --- | --- |
| POST | **C**reate（新增） | 新增一個資源 |
| GET | **R**ead（查詢） | 取得一個資源 |
| PUT | **U**pdate（修改） | 更新一個已存在的資源 |
| DELETE | **D**elete（刪除） | 刪除一個資源 |

用 URL 路徑描述資源之間的階層（`/`）關係：

| HTTP method + URL | 說明 |
| --- | --- |
| `GET /users` | 取得所有 user |
| `GET /users/123` | 取得 user id 為 123 的 user |
| `GET /users/123/articles` | 取得該 user 寫的所有文章 |
| `GET /users/123/articles/456` | 取得該 user 寫的、article id 為 456 的文章 |
| `GET /users/123/videos` | 取得該 user 錄的所有影片 |

response body 返回 JSON 或 XML 格式。

## 驗證請求參數：`@Valid` / `@Validated` / `@NotNull`…

- 使用 `@RequestBody`：要在該參數上加 **`@Valid`**，class 裡的驗證註解才會生效。
- 使用 `@RequestParam` / `@RequestHeader` / `@PathVariable`：要在 controller 上加 **`@Validated`**，驗證註解才會生效。

| 註解 | 詳細資訊 |
| --- | --- |
| `@NotNull` | 不能為 null |
| `@NotBlank` | 不能為 null 且不能為空白字串（用於 String） |
| `@NotEmpty` | 不能為 null 且 size > 0（用於 List、Set、Map 等集合） |
| `@Min(value)` | 值必須 >= value（用於數字） |
| `@Max(value)` | 值必須 <= value（用於數字） |

## RestTemplate

用途：發起一個 REST 風格的 HTTP 請求（GET、POST、PUT、DELETE），並可將收到的 response body JSON 字串轉換成 Java Object。

## Spring JDBC

`NamedParameterJdbcTemplate` 依 SQL 語法分兩類：

**1. `update` 方法（INSERT / UPDATE / DELETE）**

```java
@Autowired
private NamedParameterJdbcTemplate namedParameterJdbcTemplate;

@DeleteMapping("/students/{studentId}")
public String delete(@PathVariable Integer studentId) {
    String sql = "DELETE FROM student WHERE id = :studentId";

    Map<String, Object> map = new HashMap<>();
    map.put("studentId", studentId);

    namedParameterJdbcTemplate.update(sql, map);
    return "執行 Delete sql";
}
```

**2. `query` 方法（SELECT）**

- 用 `SELECT *` 的缺點：花費額外網路流量、無法提升查詢速度。
- `query` 永遠回傳一個 List：取數據前記得先判斷是否有資料。
- `RowMapper`：把查出來的數據轉成 Java Object，用 `resultSet.getXXX(column)` 取值。
- `ResultSetExtractor`：和 RowMapper 用途類似，可組合不同 row 之間的數據。

## Controller-Service-Dao 三層式架構

1. **Controller**：接收前端的 HTTP request，並驗證請求參數。
2. **Service**：負責業務邏輯。
3. **Dao**：負責和資料庫溝通。

注意：

- Class 命名以 `Controller` / `Service` / `Dao` 結尾，表示所屬層級。
- 三層的 class 都變成 bean，用 `@Autowired` 注入。
- Controller 不能直接呼叫 Dao，只能 call Service，再由 Service 呼叫 Dao。
- Dao 只執行 SQL、存取資料庫，不能加任何業務邏輯。

## 單元測試（Unit Testing）

目的：**自動化**測試程式的正確性。

特性：一次只測一個功能點（一個 method 或一個 API）；可自動化；各測試互相獨立；結果穩定、不受外部服務影響。

注意：測試程式放在 `test` 資料夾內；測試 class 以原 class 名加 `Test` 結尾；package 結構與原 class 一致。

### JUnit 5 用法

- 在方法上加 `@Test` 即可生成一個單元測試（只能在 `test` 資料夾下使用）。
- 方法名稱可隨意取，用來表達這個 test case 要測的功能點——這是精華所在。

常用 Assert：

| 用法 | 用途 |
| --- | --- |
| `assertNull(A)` | 斷言 A 為 null |
| `assertNotNull(A)` | 斷言 A 不為 null |
| `assertEquals(expected, actual)` | 斷言相等（用 `equals()` 判斷） |
| `assertTrue(A)` | 斷言 A 為 true |
| `assertFalse(A)` | 斷言 A 為 false |
| `assertThrows(exception, method)` | 斷言執行 method 時會噴出 exception |

其他常用註解：

- `@BeforeEach` / `@AfterEach`：每次 `@Test` 前/後各執行一次。
- `@BeforeAll` / `@AfterAll`：所有 `@Test` 前/後各執行一次（方法須為 `public static void`）。
- `@Disabled`：忽略該 `@Test`。
- `@DisplayName`：自定義顯示名稱。

### 用 JUnit 5 測試 Spring Boot 程式

在測試 class 上加 `@SpringBootTest`，運行時 Spring Boot 會啟動容器、創建所有 bean（所有 `@Configuration` 也都會執行，效果等同運行整個程式）。

`@Transactional`：

1. 在 `test` 資料夾：可加在方法或 class 上，單元測試結束後 rollback 所有資料庫操作，恢復原狀。
2. 在 `main` 資料夾：程式運行中途發生錯誤時，rollback 已執行的資料庫操作。

#### Controller 層測試：MockMvc

模擬真實的 API 呼叫：在測試 class 加 `@AutoConfigureMockMvc`、注入 `MockMvc`。

```java
@Test
public void getById() throws Exception {
    // 設定發起的 Request 與相關參數（URL、請求參數、header）
    RequestBuilder requestBuilder = MockMvcRequestBuilders.get("/students/3");

    // mockMvc.perform 發起 Http Request
    // .andDo / .andExpect / .andReturn 處理並驗證 response
    // jsonPath 參考：https://jsonpath.com/
    mockMvc.perform(requestBuilder)
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id", equalTo(3)));
}
```

#### Mock 測試（Mockito）

- 目的：避免為了測一個單元，而去建構整個 bean 的依賴。
- 作法：創造一個假的 bean，替換掉 Spring 容器中原有的 bean。
- `@MockBean`：產生假的 bean，未定義的方法預設返回 null。

```java
// 模擬方法的返回值
Mockito.when(...).thenReturn(...);
Mockito.doReturn(...).when(...);

// 模擬拋出 exception
Mockito.when(...).thenThrow(...);   // 方法不返回 void
Mockito.doThrow(...).when(...);     // 方法返回 void
```

- 也可記錄方法的使用次數與順序。
- 限制：不能 mock static 方法、private 方法、final class。
- `@SpyBean`：只替換其中幾個方法，其餘仍是正常的 bean。

其他心得：

- Run test with coverage 可看覆蓋範圍，但**不要為了提升覆蓋率而寫測試**，要思考哪些場景沒考慮到，別被數字迷惑。
- 若測試中 `@SpyBean` 過多，表示功能切分得不夠好。
- 進一步可以走測試驅動開發（TDD）的流程。

## IntelliJ 實用技巧（Mac）

| 功能 | 快捷鍵 |
| --- | --- |
| 萬用鍵（顯示意圖動作） | `⌥ + Enter` |
| 跳轉到定義 | `⌘ + 滑鼠左鍵` |
| 全域搜尋 | `⌘ + ⇧ + F` |
| 回到前一次游標位置 | `⌘ + ⌥ + ←` |
| 註解／取消註解 | `⌘ + /` |
| 移除多餘的 import | `⌃ + ⌥ + O` |
| 調整排版（格式化） | `⌘ + ⌥ + L` |
| 快速查詢（Search Everywhere） | 連按兩下 `⇧` |
| 列出近期開過的檔案 | `⌘ + E` |
| 自動產生（Generate） | `⌘ + N` |
| 選取多行（column selection） | `⌥ + 拖曳滑鼠` |

其他：Endpoints 工具視窗可找到所有 url-mapping 方法；連點視窗兩下可放大；split to right 切割分頁；`File → New → Module from Existing Sources → 選 pom.xml` 可同時載入多個 Spring 專案。

## Maven 生命週期

1. `clean`：刪除 `target` 資料夾。
2. `compile`：編譯程式。
3. `test`：運行單元測試。
4. `package`：打包成 `.jar`（放在 `target/`）。
5. `install`：把 `.jar` 放到 Local repository（`~/.m2/repository`）。
6. `deploy`：把 `.jar` 上傳到 remote repository。
