# Eureka Microservices Demo

A complete microservices demo showcasing **Service Registry**, **Service Discovery**, **Load Balancing**, and **API Gateway** using Spring Cloud Netflix Eureka.

---

## How They Work Together

### 1. Eureka Server — Service Registry
The Eureka Server acts as the **central service registry**. Every microservice registers itself here on startup and periodically sends heartbeats to stay listed. It is the single source of truth for all available services and their instances.

### 2. Service Discovery
Instead of hardcoding URLs, each microservice uses the **Eureka Client** to look up other services by name (e.g., `service-b`). This means services can find and talk to each other dynamically, even if their host or port changes.

### 3. Load Balancer
Both the API Gateway and Feign clients use the `lb://` prefix (e.g., `lb://service-a`, `lb://service-b`) to route requests. Spring Cloud LoadBalancer automatically distributes traffic across all running instances of a service registered in Eureka.

### 4. API Gateway — Single Entry Point
The API Gateway runs on **port 8080** and serves as the single entry point for all client requests. It uses Eureka discovery to resolve service names and routes incoming requests to the appropriate microservice, with built-in load balancing.

### 5. Feign Client — Inter-Service Communication
Service A uses a **Feign Client** to call Service B by name (`service-b`) without any hardcoded URLs. Feign integrates with Eureka and the Load Balancer automatically, making service-to-service calls clean and resilient.

---

## Project Structure

```
eureka-microservices-demo/
├── eureka-server/     ← Service Registry (Eureka Server)
├── api-gateway/       ← API Gateway + Routing + Load Balancing
├── service-a/         ← Microservice 1 (Eureka Client, Feign, LoadBalancer)
└── service-b/         ← Microservice 2 (Eureka Client)
```

All of the above components are fully implemented in this project.

---

## Architecture

```
                    ┌─────────────────────────────────────┐
                    │       API GATEWAY (Port 8080)        │
                    │   Single entry point, routing, LB   │
                    └─────────────────┬───────────────────┘
                                      │
                    ┌─────────────────┴───────────────────┐
                    │      EUREKA SERVER (Port 8761)       │
                    │    Service Registry + Discovery      │
                    └─────────────────┬───────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
              ▼                       ▼                       ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │   SERVICE A      │    │   SERVICE B      │    │   API GATEWAY    │
    │   (Port 8081)    │───▶│   (Port 8082)    │    │   (Port 8080)    │
    │   Order Service  │    │  Product Service │    │  Routes clients  │
    │   Feign + LB     │    │   Provider       │    │  to services     │
    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## Dependencies

| Module | Dependencies |
|--------|--------------|
| **Parent** | `spring-boot-starter-parent` (2.7.18), `spring-cloud-dependencies` (2021.0.8) |
| **eureka-server** | `spring-cloud-starter-netflix-eureka-server` |
| **api-gateway** | `spring-cloud-starter-gateway`, `spring-cloud-starter-netflix-eureka-client` |
| **service-a** | `spring-boot-starter-web`, `spring-cloud-starter-netflix-eureka-client`, `spring-cloud-starter-openfeign`, `spring-cloud-starter-loadbalancer` |
| **service-b** | `spring-boot-starter-web`, `spring-cloud-starter-netflix-eureka-client` |

---

## Key Annotations

| Module | Annotations |
|--------|-------------|
| **eureka-server** | `@SpringBootApplication`, `@EnableEurekaServer` |
| **api-gateway** | `@SpringBootApplication` |
| **service-a** | `@SpringBootApplication`, `@EnableFeignClients`, `@FeignClient(name="service-b")`, `@RestController`, `@RequestMapping`, `@GetMapping`, `@PathVariable` |
| **service-b** | `@SpringBootApplication`, `@RestController`, `@RequestMapping`, `@GetMapping`, `@PathVariable` |

---

## Run Order

Start the services in the following order:

```bash
# 1. Start Eureka Server first
mvn -pl eureka-server spring-boot:run

# 2. Start Service B (provider)
mvn -pl service-b spring-boot:run

# 3. Start Service A (consumer)
mvn -pl service-a spring-boot:run

# 4. Start API Gateway last
mvn -pl api-gateway spring-boot:run
```

---

## Ports & Endpoints

| Service | Port | URL |
|---------|------|-----|
| Eureka Server | 8761 | http://localhost:8761 |
| API Gateway | 8080 | http://localhost:8080 |
| Service A | 8081 | http://localhost:8081 |
| Service B | 8082 | http://localhost:8082 |

**Via Gateway:**
- http://localhost:8080/api/orders/products
- http://localhost:8080/api/products

**Direct:**
- http://localhost:8081/api/orders/products
- http://localhost:8082/api/products

---

## Load Balancing — Multiple Instances

To spin up an additional instance of Service B and see load balancing in action:

```bash
mvn -pl service-b spring-boot:run -Dserver.port=8083
```

The gateway and Feign client will automatically distribute requests across both instances.

---

## Apigee vs Spring Cloud Gateway

| | Spring Cloud Gateway | Apigee |
|---|---------------------|--------|
| Type | Open-source, self-hosted | Google Cloud managed |
| Use case | Internal microservices | Enterprise API management |
| Features | Routing, LB, filters | Full lifecycle, analytics, developer portal |



# Spring Boot Microservices Architecture

A microservices setup using **Eureka Server**, **API Gateway**, **Feign Client**, and two services communicating with each other.

---

## Architecture Overview

```
Client
  │
  ▼
API Gateway (port 8080)
  │
  ├──► Service A - Order Service (port 8081)
  │         │
  │         └──► Service B via Feign Client
  │
  └──► Service B - Product Service (port 8082)
  
All services register with → Eureka Server (port 8761)
```

---

## Services

### 🟩 Eureka Server

Service registry — all microservices register here so they can discover each other.

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

**`application.yml`**
```yaml
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  client:
    register-with-eureka: false   # don't register itself
    fetch-registry: false         # don't fetch registry from another server
  server:
    wait-time-in-ms-when-sync-empty: 0   # faster startup in dev
  instance:
    hostname: localhost
```

Dashboard available at: `http://localhost:8761`

---

### 🟩 API Gateway

Single entry point for all client requests. Routes traffic to the correct service using path-based routing.

```java
@SpringBootApplication
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

**`application.yml`**
```yaml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: service-a
          uri: lb://service-a          # lb = load balanced via Eureka
          predicates:
            - Path=/orders/**
          filters:
            - StripPrefix=0            # keeps /orders/** as-is

        - id: service-b
          uri: lb://service-b
          predicates:
            - Path=/products/**
          filters:
            - StripPrefix=0

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true

logging:
  level:
    org.springframework.cloud.gateway: DEBUG  # helpful during development
```

---

### 🟧 Service A — Order Service

Handles order requests. Uses **Feign Client** to call Service B internally.

**Main Class**
```java
@SpringBootApplication
@EnableFeignClients
public class ServiceAApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceAApplication.class, args);
    }
}
```

**`application.yml`**
```yaml
server:
  port: 8081

spring:
  application:
    name: service-a                  # must match lb://service-a in gateway

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true

# Feign client timeout settings
feign:
  client:
    config:
      default:
        connect-timeout: 5000
        read-timeout: 5000
  circuitbreaker:
    enabled: true                    # enable resilience4j fallback (optional)
```

**Feign Client** (calls Service B)
```java
@FeignClient(name = "service-b")
public interface ProductClient {
    @GetMapping("/products/{id}")
    Product getProduct(@PathVariable("id") String id);
}
```

**Controller**
```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final ProductClient productClient;

    public OrderController(ProductClient productClient) {
        this.productClient = productClient;
    }

    @GetMapping("/{id}")
    public String getOrder(@PathVariable String id) {
        Product product = productClient.getProduct(id);
        return "Order for product: " + product.getName();
    }
}
```

---

### 🟥 Service B — Product Service

Handles product data. Called directly by clients via the gateway, or internally by Service A via Feign.

**`application.yml`**
```yaml
server:
  port: 8082

spring:
  application:
    name: service-b                  # must match lb://service-b in gateway
  datasource:
    url: jdbc:h2:mem:productdb       # H2 in-memory DB (swap for MySQL/Postgres in prod)
    driver-class-name: org.h2.Driver
    username: sa
    password:
  h2:
    console:
      enabled: true                  # access at http://localhost:8082/h2-console
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

**Main Class**
```java
@SpringBootApplication
public class ServiceBApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceBApplication.class, args);
    }
}
```

**Controller**
```java
@RestController
@RequestMapping("/products")
public class ProductController {

    @GetMapping("/{id}")
    public Product getProduct(@PathVariable String id) {
        return new Product(id, "Sample Product " + id);
    }
}
```

**Model**
```java
class Product {
    private String id;
    private String name;

    public Product(String id, String name) {
        this.id = id;
        this.name = name;
    }

    // getters and setters
    public String getId()   { return id; }
    public String getName() { return name; }
}
```

---

## 🔄 Request Flow

```
1. Client calls → GET http://localhost:8080/orders/123

2. API Gateway
   └── matches Path=/orders/**
   └── routes to → Service A (lb://service-a)

3. Service A - OrderController
   └── calls ProductClient.getProduct("123")

4. Feign Client (Service A → Service B)
   └── resolves "service-b" from Eureka
   └── calls → GET http://service-b/products/123

5. Service B - ProductController
   └── returns Product("123", "Sample Product 123")

6. Service A
   └── returns "Order for product: Sample Product 123"

7. Response flows back to Client ✅
```

---

## Dependencies

Add these to each service's `pom.xml`:

```xml
<!-- Eureka Client (all services except Eureka Server) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<!-- API Gateway -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<!-- Feign Client (Service A) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

---

## Startup Order

1. **Eureka Server** — must start first
2. **Service B** (Product Service)
3. **Service A** (Order Service)
4. **API Gateway**

---

## Quick Test

```bash
# Get an order (triggers full flow)
curl http://localhost:8080/orders/123

# Expected response
Order for product: Sample Product 123

# Hit Service B directly via gateway
curl http://localhost:8080/products/123
```
