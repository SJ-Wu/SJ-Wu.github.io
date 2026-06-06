---
title: "用共享模組統一 Feign API 契約：Spring Cloud 微服務介面共享實戰"
date: 2026-02-11
draft: false
summary: "把 Feign 介面、VO、DTO 集中在一個共享模組：服務端 implements 介面當 Controller、消費端直接注入呼叫，消除 legacy 寫法中介面與實作各寫一份、容易 drift 的重複。"
tags: ["Java", "Spring Cloud", "Feign", "微服務"]
---

在 Spring Cloud 微服務裡，跨服務呼叫常用 OpenFeign。傳統（legacy）寫法的問題是：**服務端 Controller 和消費端的 Feign Client 各寫一份**——HTTP 路徑、請求/回應物件分散在兩處，介面一改就容易兩邊 drift，DTO 也常被複製貼上。

這篇分享一個改善作法：**把 Feign API 介面、VO、DTO 全部集中在一個共享模組**，當作服務之間的單一契約。服務端 `implements` 這個介面作為 Controller，消費端直接 `@Autowired` 注入即可呼叫。介面只有一份，兩邊都依賴它，自然不會 drift。

以下以一個電商「訂單服務」為完整範本（`order` 領域）。

---

## 1. 架構概覽

```
┌─────────────────────────────────────────────────────────────────┐
│                          common-api（共享模組）                    │
│  ├── api/        Feign API 介面定義（@FeignClient）                │
│  ├── vo/         請求物件（Value Object）                          │
│  └── dto/        回應物件（Data Transfer Object）                  │
└────────────────────────────┬────────────────────────────────────┘
                             │ Maven 依賴
         ┌───────────────────┼───────────────────┐
         ▼                                       ▼
┌─────────────────────┐               ┌─────────────────────────┐
│ order-service        │               │ 消費端微服務              │
│ （服務端實作）         │               │ （呼叫端）               │
│ ├── controller/     │               │                         │
│ │   implements API  │◄──── Feign ───│ @Autowired              │
│ ├── service/        │    HTTP Call  │ OrderApi api;           │
│ └── service/impl/   │               │                         │
└─────────────────────┘               └─────────────────────────┘
```

**核心設計**：API 介面定義在 `common-api`，服務端 `implements` 該介面作為 Controller，消費端直接 `@Autowired` 注入即可透過 Feign 呼叫。

---

## 2. 完整開發步驟

### Step 1：在 `common-api` 定義 VO（請求物件）

**目錄**：`common-api/src/main/java/com/example/commonapi/order/vo/`

VO 用於封裝 API 的請求參數，規範：

- 實作 `Serializable`，定義 `serialVersionUID`
- 使用 Lombok 四件組：`@Data @Builder @AllArgsConstructor @NoArgsConstructor`
- 使用 `javax.validation` 註解做參數驗證

**範例：`CreateOrderVO.java`**

```java
package com.example.commonapi.order.vo;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
import java.io.Serializable;

/**
 * 建立訂單請求 VO
 */
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class CreateOrderVO implements Serializable {

    private static final long serialVersionUID = 1L;

    @NotBlank(message = "顧客ID不能為空")
    private String customerId;

    @NotBlank(message = "商品ID不能為空")
    private String productId;

    @NotNull(message = "數量不能為空")
    private Integer quantity;

    @NotBlank(message = "金額不能為空")
    private String amount;

    /** 備註，可為空 */
    private String remark;

    @NotNull(message = "下單時間不能為空")
    private Long requestTime;
}
```

**範例：`UpdateOrderStatusVO.java`**

```java
package com.example.commonapi.order.vo;

import com.example.commonapi.order.enums.OrderStatusEnum;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
import java.io.Serializable;

@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class UpdateOrderStatusVO implements Serializable {
    private static final long serialVersionUID = 1L;

    @NotBlank(message = "OrderId 不可為空")
    String orderId;

    @NotNull(message = "訂單狀態不可為空")
    OrderStatusEnum status;
}
```

---

### Step 2：在 `common-api` 定義 DTO（回應物件）

**目錄**：`common-api/src/main/java/com/example/commonapi/order/dto/`

DTO 用於封裝 API 的回應資料，規範與 VO 相同。

**範例：`OrderResultDTO.java`**

```java
package com.example.commonapi.order.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

/**
 * 訂單處理結果 DTO
 */
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class OrderResultDTO implements Serializable {

    private static final long serialVersionUID = 1L;

    /** 是否成功受理 */
    private boolean accepted;

    /** 結果說明 */
    private String message;

    /** 訂單ID，用於追蹤 */
    private String orderId;
}
```

---

### Step 3：在 `common-api` 定義 Feign API 介面

**目錄**：`common-api/src/main/java/com/example/commonapi/order/api/`

這是整個架構的核心——用 `@FeignClient` 定義介面，服務端實作它，消費端注入它。

**範例：`OrderApi.java`**

```java
package com.example.commonapi.order.api;

import com.example.commonapi.order.dto.OrderResultDTO;
import com.example.commonapi.order.vo.CreateOrderVO;
import com.example.commonapi.order.vo.UpdateOrderStatusVO;
import com.example.common.Result;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

/**
 * 訂單服務 API
 * 提供建立訂單與狀態更新功能
 */
@FeignClient(name = "order-service", path = "/api/order")
public interface OrderApi {

    @PostMapping("/createOrder")
    Result<OrderResultDTO> createOrder(
            @RequestBody @Validated CreateOrderVO vo);

    @PostMapping("/updateOrderStatus")
    Result<Void> updateOrderStatus(
            @RequestBody @Validated UpdateOrderStatusVO vo);
}
```

**`@FeignClient` 參數說明**：

| 參數 | 說明 | 範例 |
|---|---|---|
| `name` | 目標服務在 Eureka 註冊的名稱 | `"order-service"` |
| `path` | Controller 層的統一路徑前綴 | `"/api/order"` |

**方法註解說明**：

| 註解 | 說明 |
|---|---|
| `@PostMapping` | 定義 HTTP 端點路徑 |
| `@RequestBody` | 參數以 JSON body 傳遞 |
| `@Validated` | 啟用參數驗證（觸發 VO 中的 `@NotBlank` 等） |

---

### Step 4：在服務端建立 Controller 實作 API 介面

**目錄**：`order-service/src/main/java/.../controller/`

Controller **直接 implements API 介面**，不需要再寫 `@RequestMapping`、`@PostMapping` 等路徑註解——全部繼承自介面定義。

**範例：`OrderController.java`**

```java
package com.example.orderservice.controller;

import com.example.commonapi.order.api.OrderApi;
import com.example.commonapi.order.dto.OrderResultDTO;
import com.example.commonapi.order.vo.CreateOrderVO;
import com.example.commonapi.order.vo.UpdateOrderStatusVO;
import com.example.common.Result;
import com.example.common.ResultUtil;
import com.example.orderservice.service.OrderService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RestController
@RequiredArgsConstructor
public class OrderController implements OrderApi {

    private final OrderService orderService;

    @Override
    public Result<OrderResultDTO> createOrder(CreateOrderVO vo) {
        log.info("收到建立訂單請求: customerId={}, productId={}, amount={}",
                vo.getCustomerId(), vo.getProductId(), vo.getAmount());

        OrderResultDTO result = orderService.createOrder(vo);
        return ResultUtil.success(result);
    }

    @Override
    public Result<Void> updateOrderStatus(UpdateOrderStatusVO vo) {
        log.info("收到訂單狀態更新請求: orderId={}, status={}",
                vo.getOrderId(), vo.getStatus().getCode());
        return orderService.updateOrderStatus(
                vo.getOrderId(), vo.getStatus());
    }
}
```

**重點**：

- `@RestController` 即可，**不需要** `@RequestMapping`
- 路徑由介面的 `@FeignClient(path=...)` + `@PostMapping(...)` 組合決定
- 使用 `@RequiredArgsConstructor` 搭配 `private final` 做建構子注入

---

### Step 5：在服務端建立 Service 介面與實作

**Service 介面** — `order-service/src/main/java/.../service/`

```java
package com.example.orderservice.service;

import com.example.commonapi.order.dto.OrderResultDTO;
import com.example.commonapi.order.vo.CreateOrderVO;
import com.example.commonapi.order.enums.OrderStatusEnum;
import com.example.common.Result;

public interface OrderService {

    OrderResultDTO createOrder(CreateOrderVO vo);

    Result<Void> updateOrderStatus(String orderId, OrderStatusEnum status);
}
```

**Service 實作** — `order-service/src/main/java/.../service/impl/`

```java
package com.example.orderservice.service.impl;

import com.example.commonapi.order.dto.OrderResultDTO;
import com.example.commonapi.order.vo.CreateOrderVO;
import com.example.orderservice.service.OrderService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {

    @Override
    public OrderResultDTO createOrder(CreateOrderVO vo) {
        // 實作業務邏輯...
    }

    @Override
    public Result<Void> updateOrderStatus(String orderId, OrderStatusEnum status) {
        // 實作業務邏輯...
    }
}
```

---

### Step 6：在消費端微服務注入使用

#### 6.1 啟動類配置 `@EnableFeignClients`

消費端的 Spring Boot 啟動類必須掃描到 `com.example.commonapi` 才能自動裝配 Feign Client。

**範例：`ShopServiceApplication.java`**

```java
package com.example.shopservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@EnableEurekaClient
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
@EnableDiscoveryClient
@EnableFeignClients(basePackages = {
    "com.example.shopservice",  // 本服務的 Feign Client
    "com.example.commonapi"     // ← 關鍵！掃描 common-api 中的 API 介面
})
public class ShopServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShopServiceApplication.class, args);
    }
}
```

#### 6.2 在 Service 中注入 API 介面

直接用建構子注入 `OrderApi`，像呼叫本地方法一樣使用：

```java
package com.example.shopservice.service.impl;

import com.example.commonapi.order.api.OrderApi;
import com.example.commonapi.order.dto.OrderResultDTO;
import com.example.commonapi.order.vo.CreateOrderVO;
import com.example.common.Result;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class CheckoutServiceImpl implements CheckoutService {

    private final OrderApi orderApi;  // ← Feign 自動注入

    private void someMethod() {
        // 構建請求
        CreateOrderVO orderVO = CreateOrderVO.builder()
                .customerId("12345")
                .productId("SKU-001")
                .quantity(2)
                .amount("1000")
                .requestTime(System.currentTimeMillis())
                .build();

        // 呼叫遠端服務（透過 Feign，看起來像本地呼叫）
        Result<OrderResultDTO> response = orderApi.createOrder(orderVO);

        // 處理回應
        if (response.getSuccess() && response.getData() != null) {
            OrderResultDTO result = response.getData();
            if (result.isAccepted()) {
                // 後續處理...
            }
        }
    }
}
```

---

## 3. 檔案總覽

| 層級 | 檔案 | 路徑 |
|---|---|---|
| VO | `CreateOrderVO.java` | `common-api/.../order/vo/` |
| VO | `UpdateOrderStatusVO.java` | `common-api/.../order/vo/` |
| DTO | `OrderResultDTO.java` | `common-api/.../order/dto/` |
| API 介面 | `OrderApi.java` | `common-api/.../order/api/` |
| Controller | `OrderController.java` | `order-service/.../controller/` |
| Service | `OrderService.java` | `order-service/.../service/` |
| Service Impl | `OrderServiceImpl.java` | `order-service/.../service/impl/` |
| 消費端使用 | `CheckoutServiceImpl.java` | `shop-service/.../service/impl/` |
| 啟動類 | `ShopServiceApplication.java` | `shop-service/` |

---

## 4. 關鍵注意事項

### Package 結構慣例

```
com.example.commonapi.{domain}
├── api/        API 介面（@FeignClient）
├── vo/         請求物件
└── dto/        回應物件
```

- `{domain}` 依功能領域命名，如 `order`、`product`、`notify`
- VO 和 DTO 都放在 `common-api`，讓服務端和消費端都能使用

### 命名慣例

| 類型 | 命名規則 | 範例 |
|---|---|---|
| API 介面 | `{Domain}Api` | `OrderApi` |
| 請求物件 | `{Action}{Domain}VO` | `CreateOrderVO` |
| 回應物件 | `{Domain}{Action}DTO` | `OrderResultDTO` |
| Controller | `{Domain}Controller` | `OrderController` |
| Service | `{Domain}Service` | `OrderService` |
| Service Impl | `{Domain}ServiceImpl` | `OrderServiceImpl` |

### 驗證

- 在 VO 中使用 `javax.validation` 註解（`@NotBlank`、`@NotNull` 等）
- 在 API 介面方法參數加 `@Validated` 啟用驗證
- Controller **不需要**再加 `@Validated`，繼承自介面即可

### 序列化

- VO 和 DTO 都必須實作 `Serializable`
- 定義 `serialVersionUID` 確保版本相容
- 金額這類需要精度的欄位，避免用浮點數（可用 `String` 或 `BigDecimal`），防止精度問題
- 時間戳使用 `Long`（毫秒級，13 位數）

### 回傳值

- 所有 API 方法統一回傳 `Result<T>` 包裝器
- 無回傳資料時使用 `Result<Void>`
- 服務端使用 `ResultUtil.success(data)` / `ResultUtil.error(...)` 構建回應

### `@FeignClient` 配置

- `name` 必須與目標服務在 Eureka 的註冊名稱一致
- `path` 對應目標服務 Controller 的統一路徑前綴
- **不要**在 API 介面上加 `@RequestMapping`（只在方法級別用 `@PostMapping`）

### 消費端必要配置

`@EnableFeignClients` 必須包含 `"com.example.commonapi"` 才能掃描到 common-api 中的 API 介面：

```java
@EnableFeignClients(basePackages = {
    "com.your.service",     // 本服務
    "com.example.commonapi" // common-api 的 API
})
```

遺漏此配置會導致 `NoSuchBeanDefinitionException`。
