Create an updated Spring Boot microservices architecture by adding an **API Gateway** in front of the existing Order Service and Inventory Service.

The system must use **three completely separate Spring Boot projects**, not a Maven multi-module project.

## Existing Services

1. `order-service`
   - Port: `8081`
   - Database: `orderdb`
   - Owns order data
   - Uses Spring Cloud OpenFeign to call Inventory Service

2. `inventory-service`
   - Port: `8082`
   - Database: `inventorydb`
   - Owns product and stock data

3. Create a new service:
   - Project name: `api-gateway`
   - Port: `8080`
   - No database

## Architecture

Implement this flow:

```text
Client / Postman
      |
      v
API Gateway :8080
      |
      +------------------------+
      |                        |
      v                        v
Order Service :8081      Inventory Service :8082
      |                        |
      v                        v
   orderdb                 inventorydb

Order Service
      |
      | OpenFeign
      v
Inventory Service
```

Important:

- The client must call only the API Gateway.
- The API Gateway should route external requests to the correct microservice.
- Order Service must continue calling Inventory Service directly using OpenFeign.
- Do not route the internal Feign call through the API Gateway.
- Do not allow Order Service to access `inventorydb` directly.
- Do not allow Inventory Service to access `orderdb` directly.

# API Gateway Requirements

Create a completely separate Spring Boot application named:

```text
api-gateway
```

Use:

- Java 17+
- Spring Boot
- Spring Cloud Gateway
- Maven

Do not add:

- JPA
- PostgreSQL
- OpenFeign
- Spring Security
- Eureka
- Kafka

for the initial version.

Add the required Spring Cloud Gateway dependency and compatible Spring Cloud BOM directly inside the API Gateway `pom.xml`.

## Gateway Configuration

Use port:

```text
8080
```

Configure these routes.

### Order Route

Client request:

```http
POST http://localhost:8080/api/orders
```

Gateway should forward to:

```http
http://localhost:8081/api/orders
```

Use route predicate:

```text
Path=/api/orders/**
```

### Inventory Route

Client request:

```http
GET http://localhost:8080/api/inventory/P1001?quantity=2
```

Gateway should forward to:

```http
http://localhost:8082/api/inventory/P1001?quantity=2
```

Use route predicate:

```text
Path=/api/inventory/**
```

Create `application.yml` similar to:

```yaml
server:
  port: 8080

spring:
  application:
    name: api-gateway

  cloud:
    gateway:
      routes:
        - id: order-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/orders/**

        - id: inventory-service
          uri: http://localhost:8082
          predicates:
            - Path=/api/inventory/**
```

If the current Spring Boot / Spring Cloud Gateway version requires a different property structure, use the correct current configuration and explain the difference.

# Order Service Requirements

Keep Order Service as a separate Spring Boot project.

Port:

```text
8081
```

Database:

```text
orderdb
```

Keep the API:

```http
POST /api/orders
```

The Order Service must still call Inventory Service using OpenFeign.

Use:

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

Keep:

```properties
inventory.service.url=http://localhost:8082
```

Do not change the Feign URL to the API Gateway for this version.

The internal communication must remain:

```text
Order Service
     |
     | Feign
     v
Inventory Service
```

not:

```text
Order Service
     |
     v
API Gateway
     |
     v
Inventory Service
```

# Inventory Service Requirements

Keep Inventory Service as a separate Spring Boot project.

Port:

```text
8082
```

Database:

```text
inventorydb
```

Keep API:

```http
GET /api/inventory/{productCode}?quantity={quantity}
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

# Order Request Flow

Implement and explain the complete flow:

```text
1. Client sends:
   POST http://localhost:8080/api/orders

2. API Gateway receives request.

3. Gateway matches:
   /api/orders/**

4. Gateway forwards request to:
   http://localhost:8081/api/orders

5. OrderController receives request.

6. OrderService validates order.

7. OrderService calls Inventory Service using OpenFeign.

8. Feign sends:
   GET http://localhost:8082/api/inventory/P1001?quantity=2

9. Inventory Service checks product and stock in inventorydb.

10. Inventory Service returns stock response.

11. Order Service checks:
    available = true / false

12. If available:
    save order into orderdb
    status = CONFIRMED

13. Order Service returns response to API Gateway.

14. API Gateway returns response to client.
```

# Example Client Request

The client must use API Gateway URL:

```http
POST http://localhost:8080/api/orders
```

Request body:

```json
{
  "productCode": "P1001",
  "quantity": 2,
  "price": 75000
}
```

Expected success response:

```json
{
  "orderId": 1,
  "productCode": "P1001",
  "quantity": 2,
  "price": 75000,
  "status": "CONFIRMED",
  "createdAt": "2026-08-24T13:30:00"
}
```

# Direct Inventory Testing Through Gateway

Allow testing Inventory Service through API Gateway:

```http
GET http://localhost:8080/api/inventory/P1001?quantity=2
```

The gateway should forward it to Inventory Service on port `8082`.

# Project Structure

Create three independent folders:

```text
workspace
│
├── api-gateway
│   ├── pom.xml
│   └── src
│
├── order-service
│   ├── pom.xml
│   └── src
│
└── inventory-service
    ├── pom.xml
    └── src
```

Do not create:

```text
parent-project
pom.xml
<modules>
```

Each application must be independently runnable from STS or IntelliJ.

# API Gateway Output Requirements

Generate:

1. Complete `api-gateway/pom.xml`
2. Main application class
3. `application.yml`
4. Route configuration
5. Optional logging configuration
6. Explanation of each route
7. Sample requests
8. Sample responses

# Existing Services Output Requirements

Show only the required updates to:

- `order-service`
- `inventory-service`

Do not unnecessarily rewrite all existing code unless required.

For Order Service, verify:

- `@EnableFeignClients`
- `InventoryClient`
- `inventory.service.url`
- Order API
- Feign communication

For Inventory Service, verify:

- Inventory API
- Port `8082`
- Inventory database

# Error Scenarios

Explain what happens in these cases:

## Order Service Down

Client calls:

```http
POST http://localhost:8080/api/orders
```

Gateway cannot reach Order Service.

Return or explain an appropriate gateway/service unavailable response.

## Inventory Service Down

Flow:

```text
Client
  ->
API Gateway
  ->
Order Service
  ->
Feign call fails
```

Order Service should return a meaningful error such as:

```json
{
  "status": 503,
  "error": "SERVICE_UNAVAILABLE",
  "message": "Inventory service is currently unavailable"
}
```

API Gateway should forward that response to the client.

## Product Not Found

Inventory Service should return a structured 404 response.

## Insufficient Stock

Order Service must not save the order.

Return a proper error response.

# Important Concept to Explain

Clearly explain this distinction:

```text
API Gateway
= Client -> Microservices communication

OpenFeign
= Microservice -> Microservice communication
```

Explain why the client should not normally call each microservice directly once an API Gateway is introduced.

Also explain why the Order Service's internal Feign call should normally go directly to Inventory Service instead of passing through the API Gateway.

# Testing Steps

Provide the exact startup sequence:

```text
1. Start PostgreSQL
2. Start inventory-service on 8082
3. Verify inventory API directly
4. Start order-service on 8081
5. Verify Feign communication
6. Start api-gateway on 8080
7. Call Order API using gateway URL
8. Call Inventory API using gateway URL
9. Stop one service and demonstrate failure behavior
```

# Final Architecture

End with this architecture:

```text
                Client / Postman
                       |
                       v
               +----------------+
               |  API Gateway   |
               |     :8080      |
               +----------------+
                  /          \
                 /            \
                v              v
      +----------------+   +-------------------+
      | Order Service  |   | Inventory Service |
      |     :8081      |   |      :8082        |
      +----------------+   +-------------------+
             |                    |
             v                    v
          orderdb             inventorydb
             |
             |
             +------ OpenFeign ------> Inventory Service
```

Generate the solution in a beginner-friendly way, but follow proper microservice architecture practices.

Use full compilable code and do not skip imports.