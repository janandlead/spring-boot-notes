Create a Spring Boot microservices use case using **two completely independent Spring Boot applications**.

IMPORTANT:

- Do NOT create a Maven multi-module project.
- Do NOT create a parent project containing both services.
- Create **two separate Spring Boot projects** that can be opened and run independently in IntelliJ/STS.
- Each project must have its own:
  - `pom.xml`
  - `src/main/java`
  - `src/main/resources`
  - `application.properties`
  - PostgreSQL database
  - Maven lifecycle
- The two applications should communicate only through REST APIs using **Spring Cloud OpenFeign**.

# Architecture

Create these two independent services:

```text
order-service
inventory-service
```

They should exist like this:

```text
workspace
│
├── order-service
│   ├── pom.xml
│   └── src
│
└── inventory-service
    ├── pom.xml
    └── src
```

These are **two standalone Maven Spring Boot projects**, not Maven modules.

---

# 1. Inventory Service

Create a standalone Spring Boot project named:

```text
inventory-service
```

Use:

```text
Port: 8082
Database: inventorydb
```

Technology:

- Java 17 or later
- Spring Boot
- Spring Web
- Spring Data JPA
- PostgreSQL
- Bean Validation
- Lombok if required
- Maven

Do NOT add OpenFeign dependency to Inventory Service.

## Inventory Entity

Create:

```java
Inventory
```

with fields:

```text
id              Long
productCode     String
productName     String
quantity        Integer
```

Use:

```java
@Entity
@Table(name = "inventory")
```

`productCode` should be unique and not null.

## Inventory Repository

Create:

```java
InventoryRepository
```

extending:

```java
JpaRepository<Inventory, Long>
```

Add:

```java
Optional<Inventory> findByProductCode(String productCode);
```

## Inventory API

Create API:

```http
GET /api/inventory/{productCode}?quantity={requestedQuantity}
```

Example:

```http
GET http://localhost:8082/api/inventory/P1001?quantity=2
```

Response:

```json
{
  "productCode": "P1001",
  "productName": "Laptop",
  "availableQuantity": 10,
  "available": true
}
```

If quantity is insufficient:

```json
{
  "productCode": "P1001",
  "productName": "Laptop",
  "availableQuantity": 1,
  "available": false
}
```

Create:

- InventoryController
- InventoryService
- InventoryRepository
- Inventory entity
- InventoryResponse record
- ProductNotFoundException
- GlobalExceptionHandler

Use constructor injection.

---

# 2. Inventory Database Configuration

Create PostgreSQL database:

```sql
CREATE DATABASE inventorydb;
```

Use:

```properties
spring.application.name=inventory-service

server.port=8082

spring.datasource.url=jdbc:postgresql://localhost:5432/inventorydb
spring.datasource.username=postgres
spring.datasource.password=postgres

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

The `inventory` table should be automatically created from the JPA entity.

Also provide sample insert data:

```sql
INSERT INTO inventory(product_code, product_name, quantity)
VALUES ('P1001', 'Laptop', 10);

INSERT INTO inventory(product_code, product_name, quantity)
VALUES ('P1002', 'Mobile', 20);

INSERT INTO inventory(product_code, product_name, quantity)
VALUES ('P1003', 'Headphones', 5);
```

---

# 3. Order Service

Create another completely separate Spring Boot project named:

```text
order-service
```

Use:

```text
Port: 8081
Database: orderdb
```

Technology:

- Java 17 or later
- Spring Boot
- Spring Web
- Spring Data JPA
- PostgreSQL
- Spring Cloud OpenFeign
- Bean Validation
- Maven

Order Service must not have direct database access to `inventorydb`.

It should communicate with Inventory Service only through HTTP using OpenFeign.

---

# 4. Order Entity

Create:

```java
Order
```

with fields:

```text
id
productCode
quantity
price
status
createdAt
```

Recommended types:

```text
id          Long
productCode String
quantity    Integer
price       BigDecimal
status      String or Enum
createdAt   LocalDateTime
```

Use:

```java
@Entity
@Table(name = "orders")
```

---

# 5. Order DTOs

Use Java records.

Create:

```java
public record OrderRequest(
        String productCode,
        Integer quantity,
        BigDecimal price
) {
}
```

Add validation:

```text
productCode -> @NotBlank
quantity -> @NotNull and @Positive
price -> @NotNull and @Positive
```

Create:

```java
OrderResponse
```

with:

```text
orderId
productCode
quantity
price
status
createdAt
```

Also create a DTO inside Order Service matching the Inventory Service response:

```java
public record InventoryResponse(
        String productCode,
        String productName,
        Integer availableQuantity,
        boolean available
) {
}
```

Do not share DTO classes through a common Maven module.

Each service should remain independent.

---

# 6. OpenFeign Dependency

Add OpenFeign only to `order-service`.

Use:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

Also configure the correct Spring Cloud BOM version that is compatible with the generated Spring Boot version.

Do not create a shared parent Maven project.

The Spring Cloud dependency management must exist directly inside the `order-service/pom.xml`.

---

# 7. Enable Feign Client

In:

```java
OrderServiceApplication
```

add:

```java
@SpringBootApplication
@EnableFeignClients
public class OrderServiceApplication {
}
```

---

# 8. Create Inventory Feign Client

Inside Order Service create package:

```text
client
```

Create:

```java
@FeignClient(
        name = "inventory-service",
        url = "${inventory.service.url}"
)
public interface InventoryClient {

    @GetMapping("/api/inventory/{productCode}")
    InventoryResponse checkStock(
            @PathVariable("productCode") String productCode,
            @RequestParam("quantity") Integer quantity
    );
}
```

Configure:

```properties
inventory.service.url=http://localhost:8082
```

---

# 9. Order Service Business Flow

Implement:

```java
placeOrder(OrderRequest request)
```

Business flow:

```text
Receive Order Request
        |
        v
Call Inventory Service using Feign Client
        |
        v
Check if product exists
        |
        v
Check requested quantity
        |
        +---- available = false
        |          |
        |          v
        |   Throw InsufficientStockException
        |
        v
available = true
        |
        v
Create Order
        |
        v
Set status = CONFIRMED
        |
        v
Set createdAt
        |
        v
Save Order into orderdb
        |
        v
Return OrderResponse
```

Example logic:

```java
InventoryResponse inventoryResponse =
        inventoryClient.checkStock(
                request.productCode(),
                request.quantity()
        );

if (!inventoryResponse.available()) {
    throw new InsufficientStockException(
            "Insufficient stock for product: " + request.productCode()
    );
}
```

Only save the order if stock is available.

---

# 10. Order API

Create:

```http
POST /api/orders
```

Example:

```http
POST http://localhost:8081/api/orders
```

Request:

```json
{
  "productCode": "P1001",
  "quantity": 2,
  "price": 75000
}
```

Response:

```json
{
  "orderId": 1,
  "productCode": "P1001",
  "quantity": 2,
  "price": 75000,
  "status": "CONFIRMED",
  "createdAt": "2026-08-24T12:30:00"
}
```

---

# 11. Order Database

Create PostgreSQL database:

```sql
CREATE DATABASE orderdb;
```

Use:

```properties
spring.application.name=order-service

server.port=8081

spring.datasource.url=jdbc:postgresql://localhost:5432/orderdb
spring.datasource.username=postgres
spring.datasource.password=postgres

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

inventory.service.url=http://localhost:8082
```

The `orders` table should be automatically generated from the entity.

---

# 12. Exception Handling

Order Service should contain:

```text
InsufficientStockException
InventoryServiceUnavailableException
GlobalExceptionHandler
```

Inventory Service should contain:

```text
ProductNotFoundException
GlobalExceptionHandler
```

Use:

```java
@RestControllerAdvice
```

Return structured error responses with:

```text
timestamp
status
error
message
path
```

Handle Feign failures appropriately.

If Inventory Service is unavailable, Order Service should not save the order.

Return a meaningful error such as:

```json
{
  "status": 503,
  "error": "SERVICE_UNAVAILABLE",
  "message": "Inventory service is currently unavailable"
}
```

---

# 13. Package Structure

Inventory Service:

```text
com.javacodeex.inventory
│
├── InventoryServiceApplication.java
├── controller
│   └── InventoryController.java
├── service
│   └── InventoryService.java
├── repository
│   └── InventoryRepository.java
├── entity
│   └── Inventory.java
├── dto
│   └── InventoryResponse.java
└── exception
    ├── ProductNotFoundException.java
    └── GlobalExceptionHandler.java
```

Order Service:

```text
com.javacodeex.order
│
├── OrderServiceApplication.java
├── controller
│   └── OrderController.java
├── service
│   └── OrderService.java
├── repository
│   └── OrderRepository.java
├── entity
│   └── Order.java
├── dto
│   ├── OrderRequest.java
│   ├── OrderResponse.java
│   └── InventoryResponse.java
├── client
│   └── InventoryClient.java
└── exception
    ├── InsufficientStockException.java
    ├── InventoryServiceUnavailableException.java
    └── GlobalExceptionHandler.java
```

---

# 14. Important Architectural Rules

Strictly follow these rules:

1. `order-service` and `inventory-service` must be two standalone Spring Boot projects.
2. Do not generate a root Maven `pom.xml`.
3. Do not use `<modules>`.
4. Do not create a common/shared module.
5. Do not share entity classes between services.
6. Do not share DTO JARs between services.
7. Each service must have its own `pom.xml`.
8. Each service must have its own PostgreSQL database.
9. Order Service must never directly query `inventorydb`.
10. Inventory Service must never directly query `orderdb`.
11. Communication between the services must happen using REST + OpenFeign.
12. Both applications must be independently runnable.

---

# 15. Expected Communication Flow

```text
Postman
   |
   | POST /api/orders
   v
OrderController
   |
   v
OrderService
   |
   v
InventoryClient
   |
   | HTTP GET
   v
InventoryController :8082
   |
   v
InventoryService
   |
   v
InventoryRepository
   |
   v
inventorydb
   |
   v
InventoryResponse
   |
   v
Feign Client
   |
   v
OrderService
   |
   | Stock Available
   v
OrderRepository
   |
   v
orderdb
   |
   v
OrderResponse
```

---

# 16. Generation Instructions

Generate the projects separately.

First generate the complete:

```text
inventory-service
```

including:

- complete `pom.xml`
- application.properties
- entity
- repository
- service
- controller
- DTO
- exception handling
- SQL scripts
- sample Postman calls

Then generate the complete:

```text
order-service
```

including:

- complete `pom.xml`
- application.properties
- entity
- repository
- service
- controller
- DTOs
- Feign Client
- exception handling
- sample Postman calls

Do not skip imports.

Provide full compilable Java classes.

Do not provide pseudo code where real Java code can be provided.

Finally provide the exact startup order:

```text
1. Start PostgreSQL
2. Create inventorydb
3. Create orderdb
4. Start inventory-service on 8082
5. Verify Inventory API
6. Start order-service on 8081
7. Call Order API
8. Verify Feign call in logs
9. Verify data in orderdb
```

The final solution must be directly importable as **two separate Maven projects in STS or IntelliJ**.