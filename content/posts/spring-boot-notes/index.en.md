---
title: "Spring Boot for Beginners: My Study Notes"
date: 2022-09-01
draft: false
summary: "My notes from self-studying Spring Boot: IoC/DI, AOP, Spring MVC, RESTful APIs, request validation, Spring JDBC, the three-tier architecture, and unit testing with JUnit 5 and Mockito."
tags: ["Java", "Spring Boot", "Notes"]
---

These are the notes I made while self-studying Spring Boot, following the Hahow
course
[*Spring Boot for Java Engineers: A Beginner's Course*](https://hahow.in/courses/5fe22e7fe810e10fc483dd78).
I'm putting them up as my own cheat sheet — hopefully useful to other beginners too.

## Spring IoC

IoC (Inversion of Control): hand control of your objects to an external Spring
container.

Benefits:

1. Loose coupling
2. Lifecycle management
3. More testable

DI (Dependency Injection): obtain objects from the external container.

- `@Component`: put on a class to make it a Spring-managed bean.
- `@Autowired`: put on a field to get a bean from the container (dependency
  injection).

### Injecting beans

- `@Autowired`: finds a bean in the container by the field's type.
- `@Qualifier`: when two classes implement the same interface, use
  `@Qualifier("{bean}")` to pick one. (The bean name is the class name with its
  first letter lowercased.)

### Creating beans

1. `@Configuration` + `@Bean`
   - `@Configuration`: marks the class as Spring configuration.
   - `@Bean`: only valid on methods inside a `@Configuration` class.
2. The `@Component` approach above.

### Initializing beans

1. Use `@PostConstruct`.
2. Implement `InitializingBean`'s `afterPropertiesSet()` method.

`@PostConstruct` requirements: the method must be `public`, return `void`, and
take no parameters.

### Bean lifecycle

1. Create → initialize → use.
2. If a bean depends on others at creation, Spring creates and initializes those
   dependencies first.
3. Avoid circular dependencies.

### Configuration file

`application.properties`, read with `@Value("${key:default_value}")`.

## Spring AOP

AOP (Aspect-Oriented Programming):

1. `pom.xml`: add `spring-boot-starter-aop`.
2. `@Aspect` + `@Component`: put on a class to declare it as an aspect.
3. Common aspect annotations: `@Before` / `@After` / `@Around`, on the declared
   methods.

Common AOP uses:

- Authorization: Spring Security
- Centralized exception handling: `@ControllerAdvice`
- Logging

## Spring MVC

### `@RequestMapping`

- Usage: on a class or a method; the URL path goes in the parentheses.
- Purpose: map a URL path to a method.
- Note: the class must have `@Controller` / `@RestController`.

### `@Controller` / `@RestController`

- Usage: on a class only.
- Purpose: make the class a bean and enable `@RequestMapping` (a beefed-up
  `@Component`).

## RESTful API

- Goal: reduce the communication cost between engineers.
- An API that follows REST style is a RESTful API.
- It's **not a strict spec** — use whatever is most appropriate.

### Reading request parameters

**1. `@RequestParam`**

- Usage: on a method parameter only.
- Purpose: read a URL query parameter.
- Options: `name` (or `value`), `required` (default true), `defaultValue`.
- Extra parameters in the URL are ignored by Spring Boot.

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

- Usage: on a method parameter only.
- Purpose: read the request body (deserialize JSON into a Java object).

```java
@RequestMapping("/test2")
public String test2(@RequestBody Student student) {
    System.out.println(student);
    return "Hello test2";
}
```

**3. `@RequestHeader`**

- Usage: on a method parameter only.
- Purpose: read a request header.
- Options: `name` (or `value` — handy because headers often contain `-`),
  `required`, `defaultValue`.

Common request headers:

| Request header | Meaning | Common values |
| --- | --- | --- |
| Content-Type | the format of the request body | `application/json` (most common), `application/octet-stream` (file upload), `multipart/form-data` (image upload) |
| Authorization | authentication | |

```java
@RequestMapping("/test3")
public String test3(@RequestHeader String info) {
    System.out.println("info: " + info);
    return "Hello Test3";
}
```

**4. `@PathVariable`**

- Usage: on a method parameter only.
- Purpose: read a value from the URL path.

```java
@RequestMapping("/test4/{id}")
public String test4(@PathVariable Integer id) {
    System.out.println("id: " + id);
    return "Hello Test4";
}
```

> The four can be mixed.

### Response status

Spring Boot's default HTTP status codes: 200 when a method completes normally; 500
when an exception is thrown.

`ResponseEntity<?>`: use as the method return type to customize the HTTP response.

```java
@RequestMapping("/test")
public ResponseEntity<String> test() {
    return ResponseEntity.status(HttpStatus.ACCEPTED)
                         .body("Hello World");
}
```

`@ControllerAdvice`: on a class, makes it a bean and lets you use
`@ExceptionHandler` inside (implemented under the hood with Spring AOP).

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

### Interceptor

Purpose: decide whether to let an HTTP request through to the controller method.

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
        System.out.println("MyInterceptor.preHandle invoked");
        response.setStatus(401);
        return false;
    }
}
```

### REST style

Use the HTTP method to express the action:

| HTTP method | CRUD | Description |
| --- | --- | --- |
| POST | **C**reate | create a resource |
| GET | **R**ead | fetch a resource |
| PUT | **U**pdate | update an existing resource |
| DELETE | **D**elete | delete a resource |

Use the URL path to express the hierarchy (`/`) between resources:

| HTTP method + URL | Description |
| --- | --- |
| `GET /users` | all users |
| `GET /users/123` | the user with id 123 |
| `GET /users/123/articles` | all articles written by that user |
| `GET /users/123/articles/456` | the user's article with id 456 |
| `GET /users/123/videos` | all videos recorded by that user |

The response body returns JSON or XML.

## Validating request parameters: `@Valid` / `@Validated` / `@NotNull`…

- With `@RequestBody`: add **`@Valid`** on the parameter so the validation
  annotations inside the class take effect.
- With `@RequestParam` / `@RequestHeader` / `@PathVariable`: add **`@Validated`**
  on the controller so the validation annotations take effect.

| Annotation | Meaning |
| --- | --- |
| `@NotNull` | must not be null |
| `@NotBlank` | not null and not a blank string (for String) |
| `@NotEmpty` | not null and size > 0 (for List, Set, Map) |
| `@Min(value)` | must be >= value (for numbers) |
| `@Max(value)` | must be <= value (for numbers) |

## RestTemplate

Purpose: make a REST-style HTTP request (GET, POST, PUT, DELETE), and deserialize
the JSON response body into a Java object.

## Spring JDBC

`NamedParameterJdbcTemplate` splits into two kinds by SQL statement:

**1. `update` (INSERT / UPDATE / DELETE)**

```java
@Autowired
private NamedParameterJdbcTemplate namedParameterJdbcTemplate;

@DeleteMapping("/students/{studentId}")
public String delete(@PathVariable Integer studentId) {
    String sql = "DELETE FROM student WHERE id = :studentId";

    Map<String, Object> map = new HashMap<>();
    map.put("studentId", studentId);

    namedParameterJdbcTemplate.update(sql, map);
    return "delete executed";
}
```

**2. `query` (SELECT)**

- Downsides of `SELECT *`: extra network traffic, no query speedup.
- `query` always returns a List: check whether it has rows before reading.
- `RowMapper`: convert each queried row into a Java object, using
  `resultSet.getXXX(column)`.
- `ResultSetExtractor`: similar to RowMapper, but can combine data across rows.

## Controller-Service-Dao three-tier architecture

1. **Controller**: receives the HTTP request and validates parameters.
2. **Service**: business logic.
3. **Dao**: talks to the database.

Notes:

- Name classes with the `Controller` / `Service` / `Dao` suffix to indicate their
  layer.
- Make all three beans and inject them with `@Autowired`.
- The controller must not call the Dao directly — only the Service, which then
  calls the Dao.
- The Dao only runs SQL and accesses data; no business logic.

## Unit testing

Goal: **automatically** test that your code is correct.

Traits: test one thing at a time (a method or an API); automatable; tests are
independent of each other; results are stable and unaffected by external services.

Notes: put tests in the `test` folder; name a test class after the original class
plus `Test`; mirror the original package structure.

### JUnit 5

- Put `@Test` on a method to make it a unit test (only inside the `test` folder).
- The method name can be anything — use it to describe what the test verifies;
  this is the important part.

Common asserts:

| Usage | Purpose |
| --- | --- |
| `assertNull(A)` | A is null |
| `assertNotNull(A)` | A is not null |
| `assertEquals(expected, actual)` | equal (via `equals()`) |
| `assertTrue(A)` | A is true |
| `assertFalse(A)` | A is false |
| `assertThrows(exception, method)` | running method throws exception |

Other common annotations:

- `@BeforeEach` / `@AfterEach`: run once before/after each `@Test`.
- `@BeforeAll` / `@AfterAll`: run once before/after all `@Test`s (must be
  `public static void`).
- `@Disabled`: skip the `@Test`.
- `@DisplayName`: custom display name.

### Testing Spring Boot with JUnit 5

Put `@SpringBootTest` on the test class; at runtime Spring Boot starts the
container and creates all beans (all `@Configuration` also runs — equivalent to
running the whole app).

`@Transactional`:

1. In the `test` folder: on a method or class; rolls back all DB operations after
   the test, restoring the original state.
2. In `main`: rolls back already-executed DB operations if an error occurs
   mid-run.

#### Controller-layer testing: MockMvc

Simulate a real API call: add `@AutoConfigureMockMvc` on the test class and inject
`MockMvc`.

```java
@Test
public void getById() throws Exception {
    // Build the request (URL, params, headers)
    RequestBuilder requestBuilder = MockMvcRequestBuilders.get("/students/3");

    // mockMvc.perform fires the request
    // .andDo / .andExpect / .andReturn handle and verify the response
    // jsonPath reference: https://jsonpath.com/
    mockMvc.perform(requestBuilder)
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id", equalTo(3)));
}
```

#### Mocking (Mockito)

- Goal: avoid wiring up a whole bean dependency graph just to test one unit.
- Approach: create a fake bean to replace the real one in the container.
- `@MockBean`: produce a fake bean; undefined methods return null by default.

```java
// Stub a return value
Mockito.when(...).thenReturn(...);
Mockito.doReturn(...).when(...);

// Stub a thrown exception
Mockito.when(...).thenThrow(...);   // non-void method
Mockito.doThrow(...).when(...);     // void method
```

- You can also record call counts and order.
- Limits: can't mock static methods, private methods, or final classes.
- `@SpyBean`: replace only some methods; the rest stay the real bean.

Other takeaways:

- "Run test with coverage" shows what's covered, but **don't write tests just to
  raise the number** — think about which scenarios you've missed; don't be fooled
  by the figure.
- Too many `@SpyBean`s suggests your code isn't well separated.
- From here you can move toward Test-Driven Development (TDD).

## Handy IntelliJ shortcuts (Mac)

| Action | Shortcut |
| --- | --- |
| Show intention actions | `⌥ + Enter` |
| Go to definition | `⌘ + click` |
| Find in files | `⌘ + ⇧ + F` |
| Back to previous caret position | `⌘ + ⌥ + ←` |
| Comment / uncomment | `⌘ + /` |
| Optimize imports | `⌃ + ⌥ + O` |
| Reformat code | `⌘ + ⌥ + L` |
| Search Everywhere | double-tap `⇧` |
| Recent files | `⌘ + E` |
| Generate | `⌘ + N` |
| Column selection | `⌥ + drag` |

Also: the Endpoints tool window lists every URL mapping; double-click a tab to
maximize it; "split to right" to split tabs; `File → New → Module from Existing
Sources → pick pom.xml` to load several Spring projects at once.

## Maven lifecycle

1. `clean`: delete the `target` folder.
2. `compile`: compile the program.
3. `test`: run unit tests.
4. `package`: package into a `.jar` (under `target/`).
5. `install`: put the `.jar` into the local repository (`~/.m2/repository`).
6. `deploy`: upload the `.jar` to a remote repository.
