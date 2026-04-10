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




Perfect — you’re looking at how to configure Resilience4j in your Spring Boot microservices. The YAML you wrote for circuitbreaker can be extended in a similar way for retry and timeout. Here’s a high-level example:

📌 application.yml Example
yaml
resilience4j:
  circuitbreaker:
    instances:
      slowService:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3

  retry:
    instances:
      slowService:
        max-attempts: 3
        wait-duration: 2s
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException

  timelimiter:
    instances:
      slowService:
        timeout-duration: 3s
        cancel-running-future: true
🔑 What Each Section Does
CircuitBreaker

Monitors calls to slowService.

Opens the circuit if failure rate > 50% in a sliding window of 10 calls.

Stays open for 10s, then moves to half-open to test recovery.

Retry

Retries failed calls to slowService up to 3 times.

Waits 2s between attempts.

Only retries on specific exceptions (like IOException).

TimeLimiter

Limits execution time of slowService calls to 3s.

Cancels the future if timeout is exceeded.

⚙️ Usage in Code
You typically annotate methods with @CircuitBreaker, @Retry, or @TimeLimiter:

java
@Service
public class OrderService {

    @CircuitBreaker(name = "slowService", fallbackMethod = "fallbackOrder")
    @Retry(name = "slowService")
    @TimeLimiter(name = "slowService")
    public CompletableFuture<String> placeOrder(String productId) {
        return CompletableFuture.supplyAsync(() -> {
            // call to product service
            return "Order placed for product " + productId;
        });
    }

    public CompletableFuture<String> fallbackOrder(String productId, Throwable t) {
        return CompletableFuture.completedFuture("Fallback: Product service unavailable for " + productId);
    }
}
👉 This way, Resilience4j gives you fault tolerance at three levels:

Circuit breaker (stop hammering a failing service),

Retry (try again on transient errors),

Timeout (don’t hang forever).

Would you like me to also show how to combine these with Feign clients in your Service A → Service B calls? That’s a common pattern in microservices.

if using retstemple insated of feign cllient which annotation to yse name=servcie=b in ordersErvce
If you switch from FeignClient to using RestTemplate, you don’t use @FeignClient(name="service-b") anymore. Instead, you rely on Spring Cloud LoadBalancer with @LoadBalanced to resolve service names (like service-b) into actual instances registered in Eureka.

🔑 Key Annotation
java
@Bean
@LoadBalanced   // <-- This is the critical annotation
public RestTemplate restTemplate() {
    return new RestTemplate();
}
@LoadBalanced tells Spring to intercept calls made with RestTemplate and resolve the service name (http://service-b/...) via Eureka + LoadBalancer.

Without it, RestTemplate would expect a fixed hostname/IP and wouldn’t know how to discover services dynamically.

📌 Example in Service A (Order Service)
java
@SpringBootApplication
public class ServiceAApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceAApplication.class, args);
    }
}

@RestController
@RequestMapping("/orders")
public class OrderController {

    private final RestTemplate restTemplate;

    public OrderController(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @GetMapping("/{id}")
    public String getOrder(@PathVariable String id) {
        // Call Service B using service name, not hardcoded URL
        Product product = restTemplate.getForObject(
            "http://service-b/products/" + id, Product.class);

        return "Order for product: " + product.getName();
    }
}
⚙️ Flow
Service A calls http://service-b/products/{id} via RestTemplate.

@LoadBalanced RestTemplate consults Eureka to find all instances of service-b.

Spring Cloud LoadBalancer picks one instance.

Request routed → response returned.
