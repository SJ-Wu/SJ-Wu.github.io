---
title: "Sharing Feign API Contracts via a Common Module in Spring Cloud"
date: 2026-02-11
draft: false
summary: "Put the Feign interface, VOs, and DTOs in one shared module: the server implements the interface as its controller, the consumer just injects and calls it — removing the legacy duplication where the interface and its client drift apart."
tags: ["Java", "Spring Cloud", "Feign", "Microservices"]
---

In Spring Cloud microservices, cross-service calls usually go through OpenFeign.
The problem with the traditional (legacy) approach is that **the server-side
controller and the consumer-side Feign client are written separately** — the HTTP
paths and request/response objects live in two places, so any change to the
interface tends to drift between the two sides, and DTOs often get copy-pasted.

This post shares an improvement: **put the Feign API interface, VOs, and DTOs in a
single shared module** that serves as the one contract between services. The
server `implements` that interface as its controller; the consumer just
`@Autowired`-injects it and calls it. There is only one interface, both sides
depend on it, and nothing drifts.

The complete example below uses an e-commerce **order service** (the `order`
domain).

---

## 1. Architecture overview

```
┌─────────────────────────────────────────────────────────────────┐
│                       common-api (shared module)                  │
│  ├── api/        Feign API interface definitions (@FeignClient)   │
│  ├── vo/         request objects (Value Object)                   │
│  └── dto/        response objects (Data Transfer Object)          │
└────────────────────────────┬────────────────────────────────────┘
                             │ Maven dependency
         ┌───────────────────┼───────────────────┐
         ▼                                       ▼
┌─────────────────────┐               ┌─────────────────────────┐
│ order-service        │               │ consumer microservice    │
│ (server impl)        │               │ (caller)                 │
│ ├── controller/     │               │                         │
│ │   implements API  │◄──── Feign ───│ @Autowired              │
│ ├── service/        │    HTTP Call  │ OrderApi api;           │
│ └── service/impl/   │               │                         │
└─────────────────────┘               └─────────────────────────┘
```

**Core idea:** the API interface is defined in `common-api`; the server
`implements` it as a controller, and the consumer `@Autowired`-injects it to call
it over Feign.

---

## 2. Step-by-step

### Step 1: Define the VO (request object) in `common-api`

**Directory:** `common-api/src/main/java/com/example/commonapi/order/vo/`

A VO wraps the request parameters of an API. Conventions:

- Implement `Serializable` and define `serialVersionUID`.
- Use the Lombok quartet: `@Data @Builder @AllArgsConstructor @NoArgsConstructor`.
- Validate parameters with `javax.validation` annotations.

**Example: `CreateOrderVO.java`**

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
 * Create-order request VO
 */
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class CreateOrderVO implements Serializable {

    private static final long serialVersionUID = 1L;

    @NotBlank(message = "customerId must not be blank")
    private String customerId;

    @NotBlank(message = "productId must not be blank")
    private String productId;

    @NotNull(message = "quantity must not be null")
    private Integer quantity;

    @NotBlank(message = "amount must not be blank")
    private String amount;

    /** optional note */
    private String remark;

    @NotNull(message = "requestTime must not be null")
    private Long requestTime;
}
```

**Example: `UpdateOrderStatusVO.java`**

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

    @NotBlank(message = "orderId must not be blank")
    String orderId;

    @NotNull(message = "status must not be null")
    OrderStatusEnum status;
}
```

---

### Step 2: Define the DTO (response object) in `common-api`

**Directory:** `common-api/src/main/java/com/example/commonapi/order/dto/`

A DTO wraps the response data of an API; same conventions as the VO.

**Example: `OrderResultDTO.java`**

```java
package com.example.commonapi.order.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

/**
 * Order-processing result DTO
 */
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class OrderResultDTO implements Serializable {

    private static final long serialVersionUID = 1L;

    /** whether the request was accepted */
    private boolean accepted;

    /** result message */
    private String message;

    /** order ID, for tracking */
    private String orderId;
}
```

---

### Step 3: Define the Feign API interface in `common-api`

**Directory:** `common-api/src/main/java/com/example/commonapi/order/api/`

This is the heart of the architecture — define the interface with `@FeignClient`,
the server implements it, the consumer injects it.

**Example: `OrderApi.java`**

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
 * Order service API
 * Provides order creation and status updates.
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

**`@FeignClient` parameters:**

| Parameter | Meaning | Example |
|---|---|---|
| `name` | The target service's registered name in Eureka | `"order-service"` |
| `path` | The controller's shared path prefix | `"/api/order"` |

**Method annotations:**

| Annotation | Meaning |
|---|---|
| `@PostMapping` | Defines the HTTP endpoint path |
| `@RequestBody` | Pass the argument as a JSON body |
| `@Validated` | Enable validation (triggers the `@NotBlank` etc. in the VO) |

---

### Step 4: Implement the API interface as a controller on the server

**Directory:** `order-service/src/main/java/.../controller/`

The controller **implements the API interface directly** — no need for
`@RequestMapping` / `@PostMapping` path annotations; they're all inherited from
the interface definition.

**Example: `OrderController.java`**

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
        log.info("create-order request: customerId={}, productId={}, amount={}",
                vo.getCustomerId(), vo.getProductId(), vo.getAmount());

        OrderResultDTO result = orderService.createOrder(vo);
        return ResultUtil.success(result);
    }

    @Override
    public Result<Void> updateOrderStatus(UpdateOrderStatusVO vo) {
        log.info("update-status request: orderId={}, status={}",
                vo.getOrderId(), vo.getStatus().getCode());
        return orderService.updateOrderStatus(
                vo.getOrderId(), vo.getStatus());
    }
}
```

**Key points:**

- `@RestController` is enough — **no** `@RequestMapping`.
- The path is the combination of the interface's `@FeignClient(path=...)` +
  `@PostMapping(...)`.
- Use `@RequiredArgsConstructor` with `private final` for constructor injection.

---

### Step 5: Define the service interface and implementation on the server

**Service interface** — `order-service/src/main/java/.../service/`

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

**Service implementation** — `order-service/src/main/java/.../service/impl/`

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
        // business logic...
    }

    @Override
    public Result<Void> updateOrderStatus(String orderId, OrderStatusEnum status) {
        // business logic...
    }
}
```

---

### Step 6: Inject and use it in the consumer microservice

#### 6.1 Configure `@EnableFeignClients` on the application class

The consumer's Spring Boot application class must scan `com.example.commonapi` to
auto-wire the Feign client.

**Example: `ShopServiceApplication.java`**

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
    "com.example.shopservice",  // this service's own Feign clients
    "com.example.commonapi"     // ← key! scans the API interfaces in common-api
})
public class ShopServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShopServiceApplication.class, args);
    }
}
```

#### 6.2 Inject the API interface in a service

Constructor-inject `OrderApi` and call it like a local method:

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

    private final OrderApi orderApi;  // ← auto-injected by Feign

    private void someMethod() {
        // build the request
        CreateOrderVO orderVO = CreateOrderVO.builder()
                .customerId("12345")
                .productId("SKU-001")
                .quantity(2)
                .amount("1000")
                .requestTime(System.currentTimeMillis())
                .build();

        // call the remote service (via Feign, looks like a local call)
        Result<OrderResultDTO> response = orderApi.createOrder(orderVO);

        // handle the response
        if (response.getSuccess() && response.getData() != null) {
            OrderResultDTO result = response.getData();
            if (result.isAccepted()) {
                // follow-up...
            }
        }
    }
}
```

---

## 3. File overview

| Layer | File | Path |
|---|---|---|
| VO | `CreateOrderVO.java` | `common-api/.../order/vo/` |
| VO | `UpdateOrderStatusVO.java` | `common-api/.../order/vo/` |
| DTO | `OrderResultDTO.java` | `common-api/.../order/dto/` |
| API interface | `OrderApi.java` | `common-api/.../order/api/` |
| Controller | `OrderController.java` | `order-service/.../controller/` |
| Service | `OrderService.java` | `order-service/.../service/` |
| Service Impl | `OrderServiceImpl.java` | `order-service/.../service/impl/` |
| Consumer usage | `CheckoutServiceImpl.java` | `shop-service/.../service/impl/` |
| Application class | `ShopServiceApplication.java` | `shop-service/` |

---

## 4. Things to watch out for

### Package convention

```
com.example.commonapi.{domain}
├── api/        API interfaces (@FeignClient)
├── vo/         request objects
└── dto/        response objects
```

- Name `{domain}` after the functional area, e.g. `order`, `product`, `notify`.
- Both VO and DTO live in `common-api` so the server and consumer can both use
  them.

### Naming conventions

| Type | Rule | Example |
|---|---|---|
| API interface | `{Domain}Api` | `OrderApi` |
| Request object | `{Action}{Domain}VO` | `CreateOrderVO` |
| Response object | `{Domain}{Action}DTO` | `OrderResultDTO` |
| Controller | `{Domain}Controller` | `OrderController` |
| Service | `{Domain}Service` | `OrderService` |
| Service Impl | `{Domain}ServiceImpl` | `OrderServiceImpl` |

### Validation

- Use `javax.validation` annotations (`@NotBlank`, `@NotNull`, etc.) in the VO.
- Add `@Validated` to the API method parameter to enable validation.
- The controller **doesn't** need `@Validated` again — it inherits it from the
  interface.

### Serialization

- Both VO and DTO must implement `Serializable`.
- Define `serialVersionUID` to ensure version compatibility.
- For fields that need precision (such as money), avoid floating point (use
  `String` or `BigDecimal`) to prevent precision issues.
- Use `Long` for timestamps (millisecond, 13 digits).

### Return values

- All API methods return the `Result<T>` wrapper.
- Use `Result<Void>` when there's no return data.
- The server builds responses with `ResultUtil.success(data)` /
  `ResultUtil.error(...)`.

### `@FeignClient` configuration

- `name` must match the target service's registered name in Eureka.
- `path` corresponds to the target controller's shared path prefix.
- **Don't** put `@RequestMapping` on the API interface (only `@PostMapping` at the
  method level).

### Required consumer configuration

`@EnableFeignClients` must include `"com.example.commonapi"` so the API interfaces
in common-api are scanned:

```java
@EnableFeignClients(basePackages = {
    "com.your.service",     // this service
    "com.example.commonapi" // common-api's APIs
})
```

Missing this causes a `NoSuchBeanDefinitionException`.
