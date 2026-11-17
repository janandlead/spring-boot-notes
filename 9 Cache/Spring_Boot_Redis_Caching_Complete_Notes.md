# Spring Boot Caching with Redis — Complete Notes

**Technology stack:** Java 21 | Spring Boot 3 | Spring Data JPA | PostgreSQL | Redis | REST API | Maven | Postman

## 1. Introduction to caching

Caching temporarily stores frequently accessed data in a fast storage system so subsequent requests can reuse it without repeatedly accessing the original source.

### Example

For `GET /api/products/101`, without caching each request executes:

```sql
SELECT * FROM products WHERE id = 101;
```

With Redis, the initial request loads the product from PostgreSQL and puts the response in Redis. Subsequent requests use the cached response while it is valid.

### Architecture without caching

```text
Client -> Spring Boot Product Service -> PostgreSQL
               (every request)           (every request)
```

### Architecture with caching

```text
                         +-------------------+
Client -> Spring Boot -> | Redis Cache       |
                         +-------------------+
                           | hit: return cached response
                           | miss
                           v
                         PostgreSQL
                           |
                           +--> store result in Redis --> return response
```

## 2. Why caching is required

| Problem | How caching helps |
|---|---|
| Repeated database queries | Reuses previous results |
| Slow API responses | Avoids repeated expensive reads |
| High database load | Reduces database traffic |
| Repeated computations | Stores computed results |
| Expensive third-party API requests | Reuses eligible responses |
| High traffic | Reduces load on backend dependencies |

Example: if 10,000 requests fetch the same unchanged product, caching can substantially reduce database queries. Illustrative timings might be 100 ms for a database-backed request versus 5 ms for a cache hit; actual values vary by infrastructure.

**Suitable candidates:** product catalogs, reference data, settings, expensive calculations and frequently requested reports.

**Use with care:** rapidly changing prices or stock, confidential/personalized responses, and data that must be immediately consistent. Design keys, authorization, TTL and invalidation appropriately.

## 3. Introduction to Redis

Redis is an in-memory data store commonly used for caching, session storage, counters, messaging and fast key-based access.

### Redis features

- In-memory reads/writes.
- Key-value storage and data structures such as strings, hashes, lists and sets.
- TTL (time to live) expiration.
- Atomic operations.
- Replication/high-availability options.
- Integration with Spring Data Redis.

### Redis versus PostgreSQL

| Aspect | Redis in this project | PostgreSQL |
|---|---|---|
| Role | Cache | Source of truth |
| Main access pattern | Key lookup | SQL queries |
| Typical retention | Temporary (TTL) | Durable |
| Primary application data | Copies of responses | Product records |
| Built-in key expiry | Yes | Not the same TTL model |

Redis can support persistence, but the cache in this exercise does **not** replace PostgreSQL.

## 4. Spring Cache abstraction

Spring Cache is an abstraction for caching method results without writing provider-specific cache commands throughout the business logic.

```java
@Cacheable(value = "products", key = "#id")
public ProductResponse getProduct(Long id) {
    return loadProductFromDatabase(id);
}
```

### Internal execution flow

```text
Controller
   |
   v
Spring-managed Service Proxy / Cache Interceptor
   |
   v
CacheManager
   |
   +---- Cache HIT ----> return cached value
   |
   +---- Cache MISS ---> invoke service method ---> PostgreSQL
                                          |
                                          v
                                    store result in Redis
```

### Main annotations

| Annotation | Purpose | Important behavior |
|---|---|---|
| `@EnableCaching` | Enables annotation-driven caching | Add to configuration/application |
| `@Cacheable` | Look up or populate an entry | Skips method body on cache hit |
| `@CachePut` | Update cache with returned result | Always executes method |
| `@CacheEvict` | Remove cached entries | By default evicts after successful method execution |
| `@Caching` | Group several cache operations | Useful for item + list caches |
| `@CacheConfig` | Define common cache settings | Applied at class level |

## 5. Cache hit, miss and expiration

**Cache hit:** Redis has a valid key; the cached result is returned without executing the database read.

**Cache miss:** Redis has no valid key; the application executes the method, loads from PostgreSQL, stores the result, and returns it.

**TTL:** The lifetime of an entry. Example with 10-minute TTL:

```text
09:00  First read: miss, store products::101
09:05  Read again: hit
09:10  Entry expires
09:11  Read again: miss, refresh cache
```

The ordinary TTL does not automatically reset on every read.

---

# Part II — Complete Spring Boot CRUD Project

## 6. Project setup

### Technology and project structure

```text
redis-cache-demo/
├── pom.xml
└── src/main/
    ├── java/com/javacodeex/cache/
    │   ├── RedisCacheDemoApplication.java
    │   ├── controller/ProductController.java
    │   ├── service/ProductService.java
    │   ├── repository/ProductRepository.java
    │   ├── entity/Product.java
    │   └── dto/
    │       ├── ProductRequest.java
    │       └── ProductResponse.java
    └── resources/application.properties
```

### Step 1: `pom.xml`

This uses Spring Boot **3.4.0** as a fixed teaching baseline. Choose a currently supported patched version for new production projects.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.0</version>
        <relativePath/>
    </parent>
    <groupId>com.javacodeex</groupId>
    <artifactId>redis-cache-demo</artifactId>
    <version>1.0.0</version>
    <properties><java.version>21</java.version></properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

`spring-boot-starter-cache` supplies the Spring Cache infrastructure; `spring-boot-starter-data-redis` supplies Redis integration.

### Step 2: Start Redis using Docker

```bash
docker run -d --name redis-cache -p 6379:6379 redis:7
```

Windows CMD: enter the command on one line.

```bash
docker exec -it redis-cache redis-cli
```

```redis
PING
```

Expected: `PONG`.

Try basic Redis commands:

```redis
SET username Anand
GET username
DEL username
SET sample "Spring Boot" EX 60
TTL sample
```

### Step 3: Create PostgreSQL database

```sql
CREATE DATABASE redis_demo;
```

### Step 4: `application.properties`

```properties
spring.application.name=redis-cache-demo
server.port=8080

spring.datasource.url=jdbc:postgresql://localhost:5432/redis_demo
spring.datasource.username=postgres
spring.datasource.password=postgres

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.cache.type=redis
spring.cache.redis.time-to-live=10m
spring.cache.redis.cache-null-values=false

logging.level.org.springframework.cache=TRACE
```

Use your actual PostgreSQL username/password. `ddl-auto=update` is for classroom demonstration; use migrations for production.

| Property | Meaning |
|---|---|
| `spring.data.redis.host` | Redis host |
| `spring.data.redis.port` | Redis port |
| `spring.cache.type` | Redis cache provider |
| `spring.cache.redis.time-to-live` | Time before entry expiration |
| `spring.cache.redis.cache-null-values` | Whether null values can be cached |
| `spring.jpa.show-sql` | Print SQL for testing |

### Step 5: Main application

```java
package com.javacodeex.cache;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching
public class RedisCacheDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(RedisCacheDemoApplication.class, args);
    }
}
```

### Step 6: Product entity

```java
package com.javacodeex.cache.entity;

import jakarta.persistence.*;

@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_name")
    private String productName;

    private Double price;

    public Product() {}

    public Product(String productName, Double price) {
        this.productName = productName;
        this.price = price;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getProductName() { return productName; }
    public void setProductName(String productName) { this.productName = productName; }
    public Double getPrice() { return price; }
    public void setPrice(Double price) { this.price = price; }
}
```

### Step 7: Repository

```java
package com.javacodeex.cache.repository;

import com.javacodeex.cache.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
}
```

### Step 8: DTOs

`ProductRequest.java`

```java
package com.javacodeex.cache.dto;

public record ProductRequest(String productName, Double price) {
}
```

`ProductResponse.java`

```java
package com.javacodeex.cache.dto;

import java.io.Serializable;

public record ProductResponse(
        Long id,
        String productName,
        Double price
) implements Serializable {
}
```

`Serializable` is necessary here for the default JDK serialization of cache values. JSON-based serialization can be configured separately.

### Step 9: Complete `ProductService.java`

```java
package com.javacodeex.cache.service;

import com.javacodeex.cache.dto.ProductRequest;
import com.javacodeex.cache.dto.ProductResponse;
import com.javacodeex.cache.entity.Product;
import com.javacodeex.cache.repository.ProductRepository;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Caching;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

import java.util.List;

@Service
public class ProductService {
    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    private ProductResponse toResponse(Product product) {
        return new ProductResponse(
                product.getId(), product.getProductName(), product.getPrice());
    }

    // CREATE: save to PostgreSQL, invalidate cached list
    @CacheEvict(value = "productList", allEntries = true)
    public ProductResponse createProduct(ProductRequest request) {
        System.out.println("DATABASE: Creating product");
        Product product = new Product(request.productName(), request.price());
        return toResponse(repository.save(product));
    }

    // READ ONE: cache individual product by ID
    @Cacheable(value = "products", key = "#id")
    public ProductResponse getProduct(Long id) {
        System.out.println("DATABASE: Fetching product " + id);
        Product product = repository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(
                        HttpStatus.NOT_FOUND, "Product not found: " + id));
        return toResponse(product);
    }

    // READ ALL: cache list separately
    @Cacheable(value = "productList", key = "'all'")
    public List<ProductResponse> getAllProducts() {
        System.out.println("DATABASE: Fetching all products");
        return repository.findAll().stream().map(this::toResponse).toList();
    }

    // UPDATE: refresh item cache and evict list cache
    @Caching(
        put = { @CachePut(value = "products", key = "#id") },
        evict = { @CacheEvict(value = "productList", allEntries = true) }
    )
    public ProductResponse updateProduct(Long id, ProductRequest request) {
        System.out.println("DATABASE: Updating product " + id);
        Product product = repository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(
                        HttpStatus.NOT_FOUND, "Product not found: " + id));
        product.setProductName(request.productName());
        product.setPrice(request.price());
        return toResponse(repository.save(product));
    }

    // DELETE: invalidate item and list cache
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#id"),
        @CacheEvict(value = "productList", allEntries = true)
    })
    public void deleteProduct(Long id) {
        System.out.println("DATABASE: Deleting product " + id);
        Product product = repository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(
                        HttpStatus.NOT_FOUND, "Product not found: " + id));
        repository.delete(product);
    }
}
```

Why invalidate the list as well? Updating `products::1` does not automatically change a previously cached `productList::all` containing the same record.

### Step 10: REST controller

```java
package com.javacodeex.cache.controller;

import com.javacodeex.cache.dto.ProductRequest;
import com.javacodeex.cache.dto.ProductResponse;
import com.javacodeex.cache.service.ProductService;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/products")
public class ProductController {
    private final ProductService service;

    public ProductController(ProductService service) {
        this.service = service;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductResponse create(@RequestBody ProductRequest request) {
        return service.createProduct(request);
    }

    @GetMapping("/{id}")
    public ProductResponse getById(@PathVariable Long id) {
        return service.getProduct(id);
    }

    @GetMapping
    public List<ProductResponse> getAll() {
        return service.getAllProducts();
    }

    @PutMapping("/{id}")
    public ProductResponse update(@PathVariable Long id,
                                  @RequestBody ProductRequest request) {
        return service.updateProduct(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        service.deleteProduct(id);
    }
}
```

### API summary

| Method | URL | Description |
|---|---|---|
| POST | `/api/products` | Create product |
| GET | `/api/products/{id}` | Retrieve by ID |
| GET | `/api/products` | Retrieve all |
| PUT | `/api/products/{id}` | Update product |
| DELETE | `/api/products/{id}` | Delete product |

---

# Part III — Annotation Deep Dive

## 7. `@Cacheable`

```java
@Cacheable(value = "products", key = "#id")
```

For `id=101`, with the default Redis key prefix, the key is `products::101`.

Execution:

1. Look up `products::101`.
2. If present, return it; the method body is not invoked.
3. Otherwise, invoke the method and query PostgreSQL.
4. Cache the returned response for later calls.

First GET console:

```text
DATABASE: Fetching product 101
Hibernate: select ... from products where id=?
```

Second GET while cached: no new service method log or product SELECT.

### Useful attributes

| Attribute | Use | Example |
|---|---|---|
| `value` / `cacheNames` | Cache region | `"products"` |
| `key` | Cache key expression | `"#id"` |
| `condition` | Before-invocation condition | `"#id > 100"` |
| `unless` | Prevent caching based on result | `"#result == null"` |
| `sync` | Coordinate same-key loads where supported | `true` |

```java
@Cacheable(
    value = "products",
    key = "#id",
    condition = "#id > 100",
    unless = "#result == null"
)
public ProductResponse getProduct(Long id) {
    // database query
}
```

## 8. `@CachePut`

`@CachePut` **always executes** the method and stores its successful result under the cache key.

```java
@CachePut(value = "products", key = "#id")
public ProductResponse updateProduct(Long id, ProductRequest request) {
    Product product = repository.findById(id).orElseThrow();
    product.setProductName(request.productName());
    product.setPrice(request.price());
    return toResponse(repository.save(product));
}
```

```text
PUT /api/products/101
 -> execute update method
 -> update PostgreSQL
 -> write updated response to products::101
 -> next GET can return updated value from Redis
```

| Behavior | `@Cacheable` | `@CachePut` |
|---|---|---|
| Executes method on cache hit | No | Yes |
| Reads cache first | Yes | Does not skip based on cached value |
| Common use | Read methods | Update methods |

Avoid casually combining `@Cacheable` and `@CachePut` on the same method.

## 9. `@CacheEvict`

Evicts one key:

```java
@CacheEvict(value = "products", key = "#id")
public void deleteProduct(Long id) {
    repository.deleteById(id);
}
```

Evicts every entry in a **named cache**:

```java
@CacheEvict(value = "products", allEntries = true)
public void clearProductsCache() {
}
```

By default `beforeInvocation=false`, so after-invocation eviction is skipped if the method throws. `beforeInvocation=true` performs eviction before the method executes. Note that cache operations and database transactions are not automatically atomic.

## 10. `@Caching`

Use `@Caching` to combine multiple cache operations. An update must maintain both an individual product cache and a cached product list:

```java
@Caching(
    put = { @CachePut(value = "products", key = "#id") },
    evict = { @CacheEvict(value = "productList", allEntries = true) }
)
```

---

# Part IV — Practical Testing with Postman and Redis CLI

## 11. Prepare testing

1. Start PostgreSQL and create `redis_demo`.
2. Start Redis and confirm `PING` -> `PONG`.
3. Run the Spring Boot application.
4. Keep the Spring Boot console visible (`spring.jpa.show-sql=true`).
5. Open Postman and Redis CLI (`docker exec -it redis-cache redis-cli`).
6. Substitute the actual auto-generated IDs for the sample IDs below.

### Test 1: Create a product

```http
POST http://localhost:8080/api/products
Content-Type: application/json
```

```json
{"productName":"Laptop","price":65000.0}
```

Expected HTTP **201**, sample response:

```json
{"id":1,"productName":"Laptop","price":65000.0}
```

Confirm PostgreSQL:

```sql
SELECT * FROM products;
```

Expected: a Laptop row. The individual cache is not populated by `createProduct`; the cached product list is invalidated if it existed.

### Test 2: First GET — cache miss

```http
GET http://localhost:8080/api/products/1
```

Expect HTTP 200 with Laptop JSON, console `DATABASE: Fetching product 1` and a Hibernate SELECT.

Redis CLI:

```redis
EXISTS products::1
TTL products::1
```

Expected: `EXISTS` = `1`, TTL is positive while entry is unexpired.

### Test 3: Second GET — cache hit

Repeat:

```http
GET http://localhost:8080/api/products/1
```

Expect the same JSON, but **no new** `DATABASE: Fetching product 1` line and no product SELECT. This is the primary proof that caching works.

### Test 4: Another product uses another key

Create Mouse with price 800, then GET its generated ID (assume `2`).

Redis should contain independent keys:

```text
products::1
products::2
```

### Test 5: Update product — `@CachePut`

```http
PUT http://localhost:8080/api/products/1
Content-Type: application/json
```

```json
{"productName":"Gaming Laptop","price":75000.0}
```

Expect updated JSON, PostgreSQL update, and replacement of `products::1`. The cached product list is evicted. A subsequent GET by ID should return the new value without another read.

### Test 6: Delete product — `@CacheEvict`

```http
DELETE http://localhost:8080/api/products/1
```

Expect HTTP **204**.

```redis
EXISTS products::1
```

Expected `0`.

Subsequent GET of product `1` should return HTTP **404** because the entry was removed from both the DB and cache.

### Test 7: First and second GET all products

```http
GET http://localhost:8080/api/products
```

First request queries the database and populates `productList::all`. The next request uses that cached list.

```redis
EXISTS productList::all
```

Expected: `1` after the list has been fetched successfully.

### Test 8: Creating a product invalidates the list

1. Call GET all to populate `productList::all`.
2. POST `{"productName":"Monitor","price":12000.0}`.
3. Check `EXISTS productList::all`: expected `0`.
4. Call GET all again: a fresh SELECT includes Monitor.

### Test 9: TTL expiration

For a short demonstration, change:

```properties
spring.cache.redis.time-to-live=30s
```

Restart application and clear relevant old test keys if necessary. GET an existing product. Check `TTL products::<id>` immediately. After 30 seconds, the key expires. The next GET queries PostgreSQL and writes a new entry.

### Test 10: Cache isolation

1. Cache product 2 and 3.
2. Delete product 2.
3. Check `EXISTS products::2` -> `0`.
4. Check `EXISTS products::3` -> `1` (assuming it has not expired).

### Test 11: Missing product

```http
GET http://localhost:8080/api/products/9999
```

Expect **404**. An exception terminates the method, so the missing product does not get cached as a successful result.

### Test 12: Failed update

```http
PUT http://localhost:8080/api/products/9999
Content-Type: application/json
```

```json
{"productName":"Unavailable","price":1000.0}
```

Expect **404**. The service throws before a successful return, so its normal after-invocation `@CachePut` does not run.

### Testing matrix

| ID | Request/operation | Database expectation | Cache expectation |
|---|---|---|---|
| TC01 | POST product | INSERT | Invalidate list |
| TC02 | GET ID first time | SELECT | Create item entry |
| TC03 | GET same ID | No SELECT | Hit |
| TC04 | GET different ID | SELECT | New item entry |
| TC05 | PUT existing ID | SELECT + UPDATE | Refresh item, evict list |
| TC06 | GET updated ID | No SELECT | Hit updated item |
| TC07 | DELETE existing ID | SELECT + DELETE | Evict item and list |
| TC08 | GET deleted ID | SELECT then 404 | Do not cache item |
| TC09 | GET all first time | SELECT | Cache list |
| TC10 | GET all again | No SELECT | Hit list |
| TC11 | POST after caching list | INSERT | Evict list |
| TC12 | GET after TTL | SELECT | Repopulate |
| TC13 | PUT missing ID | SELECT then 404 | No cache put |
| TC14 | Delete one of two IDs | SELECT + DELETE | Preserve unrelated item |

### Written walkthrough replacing the interactive simulator

```text
Initial state:
  PostgreSQL: Product #1 = Laptop, ₹65,000
  Redis:      no products::1 key

GET #1  -> cache miss -> SELECT -> Redis stores Laptop -> miss count 1
GET #1  -> cache hit  -> no SELECT -> hit count 1
PUT #1  -> DB changes to Gaming Laptop, ₹75,000
        -> @CachePut updates Redis entry
GET #1  -> cache hit, returns Gaming Laptop -> hit count 2
DELETE #1 -> DB delete + @CacheEvict removes cache key
GET #1  -> miss -> database returns no row -> 404
```

---

# Part V — Cache Invalidation and Advanced Configuration

## 12. Cache invalidation basics

Cache invalidation means deleting or refreshing stale cached information when the authoritative data changes.

### Stale-data example

```text
PostgreSQL: Product 101 price changes from ₹50,000 to ₹45,000
Redis:      products::101 still says ₹50,000
Client GET: may incorrectly receive ₹50,000 from Redis
```

### Strategies

| Strategy | Explanation | Example |
|---|---|---|
| TTL | Automatic expiration | 10 minutes |
| Explicit eviction | Remove after mutation | `@CacheEvict` |
| Cache refresh | Replace on update | `@CachePut` |
| Versioned keys | Change namespace/version | `products:v2::101` |
| Event-driven invalidation | Respond to updates | message/event after inventory update |

### Multiple cached representations

If both `products::101` and `productList::all` contain product 101, updating the individual key alone does not update the list. Use `@Caching` to refresh item cache and invalidate the list.

### Direct database updates

Changes made directly in PostgreSQL do not trigger Spring caching annotations. Redis may remain stale until TTL expiry or external invalidation. Other services that write data also need a consistent invalidation strategy.

## 13. Redis CLI debugging commands

| Command | Purpose |
|---|---|
| `PING` | Verify connection |
| `SCAN 0 MATCH products::* COUNT 100` | Incrementally discover matching keys |
| `EXISTS products::1` | Check key presence |
| `TTL products::1` | Remaining lifetime |
| `TYPE products::1` | Redis value type |
| `DEL products::1` | Delete single key |
| `DBSIZE` | Count keys in selected database |
| `INFO memory` | Memory statistics |

Default Java value serialization stores binary bytes; `GET products::1` may not be readable as JSON. Avoid `KEYS *` or destructive commands on large/shared production Redis systems.

## 14. Custom TTL per cache

Example optional class: `config/RedisCacheConfig.java`.

```java
package com.javacodeex.cache.config;

import java.time.Duration;
import java.util.Map;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;

@Configuration
@EnableCaching
public class RedisCacheConfig {
    @Bean
    public RedisCacheManager cacheManager(
            RedisConnectionFactory connectionFactory) {

        RedisCacheConfiguration defaults =
                RedisCacheConfiguration.defaultCacheConfig()
                        .entryTtl(Duration.ofMinutes(10))
                        .disableCachingNullValues();

        Map<String, RedisCacheConfiguration> settings = Map.of(
                "products", defaults.entryTtl(Duration.ofMinutes(10)),
                "productList", defaults.entryTtl(Duration.ofMinutes(2))
        );

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(defaults)
                .withInitialCacheConfigurations(settings)
                .build();
    }
}
```

`@EnableCaching` need only appear once; if present in this configuration, it can be removed from the application class. Custom cache manager settings override relevant automatic manager configuration.

## 15. Optional JSON value serialization (Spring Data Redis 3.x)

The earlier example uses the default JDK serializer. With Spring Data Redis 3.x, JSON serialization can be configured using `GenericJackson2JsonRedisSerializer`:

```java
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;

RedisCacheConfiguration defaults =
    RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10))
        .disableCachingNullValues()
        .serializeValuesWith(
            RedisSerializationContext.SerializationPair.fromSerializer(
                new GenericJackson2JsonRedisSerializer()
            )
        );
```

Use this in the cache manager builder instead of the earlier `defaults` expression. API details differ in Spring Data Redis 4.x / Jackson 3.

## 16. Troubleshooting

| Symptom | Likely reason | Check |
|---|---|---|
| Every GET queries DB | Caching disabled or proxy bypassed | `@EnableCaching`, bean call paths |
| No keys created | Redis connectivity/config error | `PING`, logs, provider selection |
| Serialization fails | DTO not compatible with serializer | `Serializable` or JSON configuration |
| Outdated response | Missing eviction or direct DB update | Ensure every writer invalidates |
| Unexpected expiration | TTL too short | Redis `TTL` and config |
| Binary output in Redis GET | JDK serializer | Expected or configure JSON |
| PUT appears stale | Different cache keys between GET/PUT | Check cache names and SpEL keys |

### Self-invocation limitation

```java
@Service
public class ProductService {
    public void process() {
        getProduct(1L); // self-invocation bypasses default cache proxy
    }

    @Cacheable(value = "products", key = "#id")
    public ProductResponse getProduct(Long id) {
        // database operation
        return null;
    }
}
```

Spring's default proxy-based caching does not intercept internal calls from one method of the same bean to another. Put the cacheable method in a separate Spring-managed bean if necessary.

### Redis failure and transaction consistency

- By default, cache connection problems can affect requests. Production applications need explicit timeout, error handling and monitoring choices.
- Database writes and Redis mutations are not automatically a single atomic transaction.
- A DB transaction rollback or an external writer may cause an inconsistency unless the design coordinates invalidation at the right point.
- Consider after-commit invalidation and events for more demanding correctness requirements.

---

# Part VI — Testing, Practice, Interviews

## 17. Automated testing ideas

Use Spring integration tests with a proxied service and available Redis to prove caching. A repository mock can count calls, provided the test actually goes through the cache interceptor.

Pseudo-assertion:

```java
ProductResponse first = service.getProduct(1L);
ProductResponse second = service.getProduct(1L);
assertEquals(first, second);
verify(repository, times(1)).findById(1L);
```

The `service` above must be a Spring-managed **proxied** bean; simply using `new ProductService(...)` does not exercise Spring Cache.

Suggested tests:

| Test | Expected assertion |
|---|---|
| First GET | Repository called once |
| Second same GET | Repository remains called once |
| Different IDs | Each unique ID loaded separately |
| Update | Subsequent cached read sees new value |
| Delete | Cache key removed |
| Create | Cached list invalidated |
| Missing product | HTTP 404 |
| TTL expiry | Read after expiry loads again |
| Redis unavailable | Defined error/fallback behavior |

## 18. Student assignment: Employee API

Build an Employee Management REST API using Java 21, Spring Boot 3, PostgreSQL and Redis.

Employee fields: `id`, `name`, `email`, `department`, `salary`.

Requirements:

1. Create CRUD REST endpoints.
2. Persist Employee records using Spring Data JPA.
3. Use `@Cacheable` for employee-by-ID reads.
4. Use `@CachePut` for employee updates.
5. Use `@CacheEvict` for deletes.
6. Cache the employee list separately and invalidate it after writes.
7. Set an employee TTL of 5 minutes.
8. Show that two same-ID GET calls execute one SELECT.
9. Verify that deleted records do not remain readable from cache.
10. Bonus: different TTLs for individual employees and the employee list.

## 19. Interview questions and answers

**1. What is caching in Spring Boot?** Temporarily storing method results to avoid repeating expensive operations.

**2. What is Redis?** An in-memory data store often used as a distributed cache, session store and low-latency key-value database.

**3. Spring Cache vs Redis?** Spring Cache is the API/abstraction; Redis is the storage implementation used in this example.

**4. `@Cacheable` vs `@CachePut`?** `@Cacheable` can skip execution on a hit; `@CachePut` executes and updates the cache.

**5. What is eviction?** Removal of cached entries that are stale or no longer needed.

**6. What is TTL?** The duration before a cache entry automatically expires.

**7. What happens after TTL?** The next request misses, runs the service method and may repopulate the cache.

**8. Can Spring Cache be used without Redis?** Yes, with in-memory or other supported cache providers.

**9. What is cache-aside?** Check the cache first, load from source on miss, and then populate the cache.

**10. Why is cache invalidation difficult?** Multiple writers, cached representations and transaction boundaries can leave old values in cache.

## 20. Quick revision

```text
@Cacheable  -> use existing cache or execute/load/store on miss
@CachePut  -> execute method, then update cached result
@CacheEvict -> remove cache entry or named cache region
@EnableCaching -> activate annotation-driven cache behavior

Spring Cache = caching abstraction
Redis        = storage for cached values
PostgreSQL   = source of truth
```

## Official references

- [Spring Framework — Cache annotations](https://docs.spring.io/spring-framework/reference/integration/cache/annotations.html)
- [Spring Boot 3.4 — Caching configuration](https://docs.spring.io/spring-boot/3.4/reference/io/caching.html)
- [Spring Data Redis — Redis Cache](https://docs.spring.io/spring-data/redis/reference/redis/redis-cache.html)
- [Redis documentation](https://redis.io/docs/latest/)
