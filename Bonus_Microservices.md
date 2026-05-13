# Bonus — Microservices Architecture Overview
**Level:** Advanced | **Focus:** Architectural Concepts

---

## 🌟 Monolith vs Microservices

### The Monolith
A traditional architectural style where all application logic (UI, business layer, database access, background jobs) is packaged and deployed as a single, unified unit (e.g., a single `.war` file on Tomcat).

**Pros:** Easy to develop initially, simple to test, simple to deploy, easy to debug (one single stack trace).
**Cons:**
- Becomes a "Big Ball of Mud" as it grows.
- **Scaling:** You must scale the entire application, even if only one feature (like image processing) is under heavy load.
- **Technology Lock-in:** Stuck with the initial tech stack (e.g., Java 8). Hard to migrate to a new language.
- **Deployment Risk:** A bug in the reporting module can crash the entire payment system.

### Microservices
An architectural style that structures an application as a collection of small, loosely coupled, independently deployable services. Each service is modeled around a specific **Business Domain** (e.g., Payment Service, User Service, Inventory Service).

**Pros:**
- **Independent Scaling:** Scale only the heavily used services.
- **Technology Diversity:** Payment service in Java, Data Science service in Python, UI in Node.js.
- **Fault Isolation:** If the Email service crashes, the core E-commerce site stays up.
- **Agility:** Small, autonomous teams can deploy their services independently multiple times a day.

**Cons:**
- **Extreme Complexity:** Distributed systems are inherently hard.
- **Network Latency:** Method calls in a monolith (`order.getUser()`) become network calls across microservices (HTTP/REST), which are slower and can fail.
- **Data Consistency:** Distributed transactions are incredibly difficult to manage.
- **Debugging:** Tracing a request that jumps across 5 different services requires specialized observability tools.

---

## 🏗️ Core Microservice Patterns

To solve the complexities of distributed systems, several industry-standard patterns have emerged.

### 1. API Gateway
Clients (mobile apps, web apps) shouldn't have to know the IP address and port of 50 different microservices.
**Solution:** The API Gateway sits between the clients and the microservices. It acts as a single entry point (Reverse Proxy).
**Functions:**
- Routing (Send `/api/users` to User Service).
- Authentication/Authorization (Validate JWT token once, before passing request to internal services).
- Rate Limiting.
- Response Aggregation (Call 3 microservices and combine their responses into one JSON for the mobile app).

### 2. Service Discovery
In cloud environments (like Kubernetes), microservice IP addresses change constantly due to scaling and failures. Hardcoding IPs in configuration files is impossible.
**Solution:** A Service Registry (like Netflix Eureka or HashiCorp Consul).
1. When a service starts, it registers its IP with the Registry.
2. When Service A wants to call Service B, it asks the Registry: "Where is Service B currently located?"
*(Note: Kubernetes has built-in service discovery via internal DNS, reducing the need for Eureka).*

### 3. Circuit Breaker
If Service A calls Service B, and Service B is down or responding extremely slowly, Service A's threads will block, waiting for a response. Eventually, Service A runs out of threads and crashes too. This is a **Cascading Failure**.
**Solution:** The Circuit Breaker (like Resilience4j).
It monitors calls to Service B. If failure rates exceed a threshold (e.g., 50%), the circuit "opens" (trips). Further calls immediately fail fast or return a fallback default value, without waiting for a timeout. After a cooldown period, it lets a few requests through (half-open) to see if Service B has recovered.

### 4. Database Per Service
If multiple microservices read/write to the exact same database tables, they are tightly coupled at the database level. Changing a table schema breaks multiple services.
**Solution:** Each microservice must have its own private database. Service A cannot query Service B's database directly. It must call Service B's API.
*(Challenge: How do you perform a JOIN across two databases? You can't. You must fetch data from both services and combine them in memory, or use CQRS/Event Sourcing).*

### 5. Saga Pattern (Distributed Transactions)
In a monolith, placing an order (Update Inventory, Charge Card, Create Order) happens in a single ACID database transaction. If charging the card fails, everything rolls back automatically. In microservices, these are 3 different databases.
**Solution:** The Saga Pattern. A Saga is a sequence of local transactions. Each service updates its DB and publishes an event to trigger the next step. If a step fails, the Saga executes **Compensating Transactions** (e.g., if charging card fails, send an event to Inventory Service to restock the items).

### 6. Event-Driven Architecture (Asynchronous Communication)
Relying entirely on synchronous REST calls creates tight coupling and latency chains.
**Solution:** Use Message Brokers (Kafka, RabbitMQ) for asynchronous communication.
*Example:* User Service publishes a "UserCreatedEvent" to Kafka. The WelcomeEmail Service and the Recommendation Engine Service both listen for this event and process it independently. The User Service doesn't know or care they exist.

---

## ❓ Interview Questions & Answers

**Q1. When should you NOT use Microservices?**
> When the application is small, the domain is not well understood, or the team is very small. The operational overhead (CI/CD pipelines, Kubernetes, monitoring, networking) of microservices is massive. "Don't even consider microservices unless you have a system that's too complex to manage as a monolith." - Martin Fowler.

**Q2. What is CAP Theorem?**
> In a distributed data store, you can only guarantee two out of three properties simultaneously:
> - **C**onsistency: Every read receives the most recent write or an error.
> - **A**vailability: Every request receives a non-error response, without guarantee that it contains the most recent write.
> - **P**artition Tolerance: The system continues to operate despite an arbitrary number of messages being dropped/delayed by the network between nodes.
> *Because network partitions (P) are unavoidable in distributed systems, architects must trade-off between Consistency (CP) and Availability (AP).*

**Q3. How do you trace a request across multiple microservices?**
> Distributed Tracing using tools like Jaeger or Zipkin. The API Gateway generates a unique `Trace ID` (e.g., a UUID) when a request enters the system. This Trace ID is passed in the HTTP Headers to every subsequent microservice call. All logs printed by all services include this Trace ID. A centralized logging system (like ELK stack) allows you to search for the Trace ID and see the entire journey of the request.

**Q4. What is the "Strangler Fig" pattern?**
> It's a strategy for migrating a monolithic application to a microservices architecture gradually. You put an API Gateway in front of the monolith. You write a new feature as a microservice and route traffic for that specific feature to the new service, while everything else still goes to the monolith. Over time, you "strangle" the monolith by moving functionality piece by piece until the monolith can be deleted.

---

## ✅ Best Practices

1. **Model services around Business Domains (Bounded Contexts in DDD),** not technical layers. Have an "Order Service", not a "Database Access Service".
2. **Automate everything.** You cannot manually deploy or test 50 microservices. Robust CI/CD pipelines and Infrastructure as Code are mandatory prerequisites.
3. **Design for Failure.** Assume the network will fail, databases will go down, and other services will timeout. Implement retries, timeouts, and circuit breakers everywhere.
4. **Prefer Asynchronous messaging** over Synchronous REST calls between internal services to reduce coupling and improve resilience.
