
# Applying Pragmatic Principles to Learning and Building with Spring Boot

Spring Boot's ecosystem is *designed* to enable exactly the opposite of your principles - it encourages enterprise complexity, premature abstraction, and toy problems disguised as "best practices."

Let me show you how to fight against this.

## **Learning Spring Boot: The Two Paths**

### **The Enterprise Tutorial Path (What Most People Do)**

**Week 1-2:** Learn Spring Core
- Dependency injection concepts
- Bean lifecycle
- ApplicationContext internals
- Different types of autowiring
- @Component vs @Service vs @Repository

**Week 3-4:** Learn Spring MVC architecture
- DispatcherServlet
- Handler mappings
- View resolvers
- Interceptors

**Week 5-6:** Learn Spring Data JPA
- Entity relationships
- Custom repositories
- Query derivation
- Specifications

**Week 7-8:** Learn Spring Security
- Authentication vs authorization
- Filter chains
- Security contexts
- OAuth2/JWT

**Week 9-10:** Learn microservices patterns
- Spring Cloud
- Service discovery
- Circuit breakers
- Config servers

**Result after 10 weeks:** You understand a lot of concepts but haven't shipped anything. You're stuck in tutorial hell. You don't know what matters.

### **The Pragmatic Path (Fast Time-to-First-Output)**

**Day 1:** Build something that works
```bash
# Go to start.spring.io
# Pick: Web, JPA, H2, Lombok
# Download, unzip
```

```java
@RestController
@RequestMapping("/todos")
public class TodoController {
    @GetMapping
    public List<String> getTodos() {
        return List.of("Learn Spring Boot", "Build something real");
    }
}
```

Run it. Hit `localhost:8080/todos`. **You have a working API in 10 minutes.**

**Day 1 afternoon:** Add a database
```java
@Entity
class Todo {
    @Id @GeneratedValue
    Long id;
    String text;
    boolean done;
}

@Repository
interface TodoRepository extends JpaRepository<Todo, Long> {}

@RestController
@RequestMapping("/todos")
class TodoController {
    @Autowired TodoRepository repo;
    
    @GetMapping
    List<Todo> getAll() { return repo.findAll(); }
    
    @PostMapping
    Todo create(@RequestBody Todo todo) { return repo.save(todo); }
}
```

**You now have a CRUD API with persistence.** You don't understand dependency injection theory, but you see it working.

**Day 2-3:** Add something real people need
- Deploy to Heroku/Railway (change H2 to Postgres)
- Add basic auth with Spring Security (just `spring.security.user.name/password` in properties)
- Share the URL with a friend

**Week 2:** Now you have questions
- "Why is my API slow?" → Learn about N+1 queries, add `@EntityGraph`
- "How do I organize this code?" → Learn about service layers *when you feel the pain*
- "How do I test this?" → Learn about `@SpringBootTest` *when you need it*

**Result after 2 weeks:** You have a deployed application. You learned by solving real problems. You have 20% of Spring knowledge but it's the 20% that matters.

---

## **Designing Enterprise Backends: Real Scenarios**

### **Scenario 1: The E-commerce Order System**

You're building an order management system. Orders, products, users, payments.

#### **The Over-Engineered Approach (What "Enterprise" Often Looks Like)**

```
order-service/
  ├── domain/
  │   ├── model/
  │   ├── repository/
  │   ├── service/
  │   └── factory/
  ├── application/
  │   ├── dto/
  │   ├── mapper/
  │   └── usecase/
  ├── infrastructure/
  │   ├── persistence/
  │   ├── messaging/
  │   └── external/
  └── presentation/
      ├── rest/
      └── graphql/

product-service/
  └── [same structure]

user-service/
  └── [same structure]

payment-service/
  └── [same structure]

shared-library/
  ├── common-dtos/
  ├── common-exceptions/
  └── common-utils/
```

**What you've built:**
- 4 separate microservices that need to talk to each other
- Kafka/RabbitMQ for inter-service communication
- Service discovery (Eureka)
- API Gateway (Spring Cloud Gateway)
- Config server
- Distributed tracing
- Each service has 6 layers of abstraction

**The first feature: "Create an order"**

1. Request hits API Gateway
2. Routes to order-service
3. OrderController → OrderUseCase → OrderService → OrderFactory → OrderRepository
4. Publishes OrderCreated event to Kafka
5. Payment-service consumes event
6. Calls external payment API
7. Publishes PaymentProcessed event
8. Order-service consumes event, updates order
9. Product-service consumes event, updates inventory

**Time to implement:** 3-4 weeks for the infrastructure, then 1 week per feature.

**What breaks:**
- Kafka goes down → orders stuck in limbo
- Payment-service deploys with a bug → all orders fail
- Network timeout between services → partial state everywhere
- Debugging requires distributed tracing across 4 services
- Hiring new devs is hard because the cognitive load is massive

#### **The Pragmatic Approach (Embrace Simplicity)**

```
src/main/java/com/shop/
  ├── order/
  │   ├── Order.java
  │   ├── OrderController.java
  │   ├── OrderService.java
  │   └── OrderRepository.java
  ├── product/
  │   └── [same simple structure]
  ├── user/
  │   └── [same simple structure]
  └── payment/
      └── [same simple structure]
```

**One Spring Boot application. One database. One codebase.**

```java
@Service
public class OrderService {
    @Autowired OrderRepository orders;
    @Autowired ProductRepository products;
    @Autowired PaymentClient paymentClient;
    
    @Transactional
    public Order createOrder(CreateOrderRequest req) {
        // Validate product exists and has stock
        Product product = products.findById(req.productId())
            .orElseThrow(() -> new ProductNotFound());
            
        if (product.stock < req.quantity()) {
            throw new OutOfStock();
        }
        
        // Create order
        Order order = new Order(req.userId(), req.productId(), req.quantity());
        orders.save(order);
        
        // Process payment
        PaymentResult payment = paymentClient.charge(req.paymentMethod(), order.total());
        
        if (payment.success()) {
            order.markPaid();
            product.reduceStock(req.quantity());
            products.save(product);
        } else {
            order.markFailed();
        }
        
        return orders.save(order);
    }
}
```

**Time to implement:** 2 days for first version.

**Advantages:**
- Everything in one transaction
- Simple to understand
- Easy to debug (one log file)
- Fast to iterate
- Can hire junior devs

**Where it breaks:**
- 10K orders/second → Database becomes bottleneck
- Payment API is slow → Blocks your API
- One deployment takes down everything

**When to split:**
- When you actually hit scale (most companies never do)
- When you have revenue to justify the complexity
- When you understand your actual bottlenecks from production data

---

### **Scenario 2: The Abstraction Trap**

You need to integrate with external APIs (payment, shipping, email).

#### **The Over-Abstracted Approach**

```java
// "We might switch payment providers!"
public interface PaymentGateway {
    PaymentResult charge(PaymentRequest request);
    RefundResult refund(RefundRequest request);
}

public interface PaymentGatewayFactory {
    PaymentGateway create(PaymentProvider provider);
}

public class StripePaymentGateway implements PaymentGateway {
    // Wraps Stripe API in your abstraction
}

public class PaypalPaymentGateway implements PaymentGateway {
    // Wraps PayPal API in your abstraction
}

// Configuration hell
@Configuration
public class PaymentConfig {
    @Bean
    @ConditionalOnProperty("payment.provider=stripe")
    public PaymentGateway stripeGateway() { ... }
    
    @Bean
    @ConditionalOnProperty("payment.provider=paypal") 
    public PaymentGateway paypalGateway() { ... }
}

// DTOs that map to your abstraction
public class PaymentRequest {
    // Fields that work for BOTH Stripe and PayPal
    // Ends up being lowest common denominator
}
```

**Problems:**
- Stripe has webhooks, PayPal has IPN - your abstraction doesn't handle this
- Stripe supports 3D Secure, PayPal has different fraud tools - can't expose these features
- When Stripe API changes, you update your wrapper, but it's never quite right
- You've built a leaky abstraction that makes both providers harder to use

**Time cost:** 1 week to build the abstraction layer. You're using Stripe. You've never used PayPal in production.

#### **The Pragmatic Approach**

```java
@Service
public class StripePaymentService {
    private final Stripe stripe = new Stripe(apiKey);
    
    public PaymentResult charge(String paymentMethod, int amount) {
        try {
            PaymentIntent intent = stripe.paymentIntents.create(
                PaymentIntentCreateParams.builder()
                    .setAmount(amount)
                    .setCurrency("usd")
                    .setPaymentMethod(paymentMethod)
                    .build()
            );
            return new PaymentResult(true, intent.getId());
        } catch (StripeException e) {
            log.error("Payment failed", e);
            return new PaymentResult(false, null);
        }
    }
}
```

**If you ever switch to PayPal:**
- Create `PaypalPaymentService`
- Update the 5 places that call payment
- Takes 2 hours

**You've saved:** A week of building abstraction you didn't need. More importantly, you used Stripe's SDK directly, so you could easily implement 3D Secure, webhooks, subscriptions - all the features that would have been painful through an abstraction.

---

### **Scenario 3: The Microservices Question**

You're building a SaaS platform. Should you use microservices?

#### **When Enterprise Says "Yes"**

"We need to scale independently, have team autonomy, use different tech stacks per service, deploy independently..."

#### **The Reality Check**

**Question 1:** Do you have more than 20 engineers?
- **No:** You don't have enough people to staff separate teams anyway

**Question 2:** Do you have different parts of the system with wildly different scaling needs?
- **No:** Everything scales together for most businesses

**Question 3:** Do you have parts of the system that need to deploy multiple times per day while other parts deploy monthly?
- **No:** Most features touch multiple services anyway

**Question 4:** Have you actually hit the limits of vertical scaling?
- **No:** A $500/month server can handle millions of requests

#### **The Pragmatic Decision Tree**

```
Start: Monolith

                ↓
                
Hit 100K+ requests/min?
    No → Stay monolith
    Yes → Profile and optimize
    
            ↓
            
Optimization not enough?
    No → Stay monolith  
    Yes → Vertical scale (bigger server)
    
            ↓
            
Vertical scaling maxed out?
    No → Stay monolith
    Yes → NOW consider splitting
    
            ↓
            
Split based on REAL bottlenecks:
- CPU-heavy analytics → Separate service
- Image processing → Separate service
- Everything else → Stay together
```

**Real example:** Shopify ran as a monolith until $1B+ in revenue. They eventually split things out, but they had:
- Hundreds of engineers
- Real scaling data showing where bottlenecks were
- Money to hire distributed systems experts
- Years of domain knowledge about where boundaries made sense

---

### **Scenario 4: The Testing Pyramid Trap**

Enterprise best practices say you need:
- Unit tests (70%)
- Integration tests (20%)
- E2E tests (10%)

#### **The Over-Tested Approach**

```java
// Unit test for service (mocking everything)
@Test
void createOrder_whenProductExists_savesOrder() {
    // Mock repository
    when(productRepo.findById(1L)).thenReturn(Optional.of(product));
    when(orderRepo.save(any())).thenReturn(order);
    
    // Mock payment
    when(paymentClient.charge(any(), any())).thenReturn(success);
    
    // Test
    Order result = orderService.createOrder(request);
    
    // Verify
    verify(orderRepo).save(any());
    verify(productRepo).save(any());
    assertEquals(OrderStatus.PAID, result.status());
}

// Unit test for controller (mocking service)
@Test
void createOrderEndpoint_returns201() {
    when(orderService.createOrder(any())).thenReturn(order);
    
    mockMvc.perform(post("/orders")...)
        .andExpect(status().isCreated());
}

// Integration test (mocking external APIs)
@Test
@SpringBootTest
void createOrder_integration() {
    // Mock external payment API
    // Test full flow through real DB
}

// E2E test (with test containers)
@Test
@Testcontainers
void createOrder_e2e() {
    // Full stack test with real DB, mocked external APIs
}
```

**Result:** You have 50 tests. They all pass. You deploy. Payment processing is broken in production because your mock didn't match the real API behavior.

**Time cost:** 3 hours of testing per feature.

#### **The Pragmatic Approach**

```java
@SpringBootTest
@AutoConfigureMockMvc
class OrderFlowTest {
    @Autowired MockMvc mvc;
    @Autowired OrderRepository orders;
    @MockBean PaymentClient paymentClient; // Only mock external APIs
    
    @Test
    void completeOrderFlow() {
        // Setup
        when(paymentClient.charge(any(), any()))
            .thenReturn(PaymentResult.success("txn_123"));
        
        // Create product
        mvc.perform(post("/products")
            .content("{\"name\":\"Widget\",\"price\":1000,\"stock\":10}")
            .contentType(APPLICATION_JSON))
            .andExpect(status().isCreated());
        
        // Create order
        mvc.perform(post("/orders")
            .content("{\"productId\":1,\"quantity\":2}")
            .contentType(APPLICATION_JSON))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.status").value("PAID"));
        
        // Verify database state
        Order order = orders.findAll().get(0);
        assertEquals(OrderStatus.PAID, order.getStatus());
        assertEquals(2, order.getQuantity());
        
        // Verify product stock reduced
        Product product = products.findById(1L).get();
        assertEquals(8, product.getStock());
    }
}
```

**One test that:**
- Hits real controllers
- Uses real services
- Writes to real database (in-memory H2 or Testcontainers)
- Only mocks external APIs
- Tests the actual user flow

**Result:** Fewer tests, but they catch real bugs. When this test passes, you're confident the feature works.

**Time cost:** 30 minutes per feature.

---

## **The Spring Boot Learning Roadmap (Pragmatic)**

### **Week 1: Ship Something**
- Build a TODO API
- Deploy it
- Share with one person

**Learn:** @RestController, @RequestMapping, JPA basics

### **Week 2-3: Add Real Features**
- Authentication (Spring Security with in-memory users)
- File upload (MultipartFile)
- Email (Spring Mail)
- Scheduled tasks (@Scheduled)

**Learn only when you hit the problem:**
- "My API is slow" → Learn about query optimization
- "I need to process uploads async" → Learn about @Async
- "I need better error handling" → Learn about @ControllerAdvice

### **Month 2: Feel The Pain, Then Solve It**
Don't learn these until you need them:
- Transactions (@Transactional) - when you have data consistency bugs
- Caching (@Cacheable) - when you have performance issues
- Validation (@Valid) - when you have bad input
- Testing - when you break things in production

### **Month 3+: Complexity As Needed**
- Microservices - when monolith is actually too slow
- Message queues - when you need async processing
- Advanced security - when you have real users
- Observability - when you can't debug production

---

## **The Configuration Hell: How to Avoid It**

Spring Boot's biggest trap is configuration complexity.

### **Bad: Premature Abstraction**

```yaml
# application.yml
spring:
  profiles:
    active: ${ENV:dev}
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASS}
  jpa:
    hibernate:
      ddl-auto: ${DDL_AUTO:validate}
    properties:
      hibernate:
        dialect: ${DB_DIALECT}
        
---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb
    
---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://...
```

**Problem:** You have three users. Why do you need dev/staging/prod profiles?

### **Good: Start Simple**

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp
    username: postgres
    password: postgres
```

**When you deploy:** Use environment variables to override:
```bash
export SPRING_DATASOURCE_URL=jdbc:postgresql://prod.db/myapp
export SPRING_DATASOURCE_PASSWORD=actualsecret
```

Spring Boot automatically reads these. No profiles needed.

---

## **The Real-World Enterprise Pattern**

Here's what actually works in production:

### **Structure**
```
src/main/java/com/company/
  ├── order/
  │   ├── Order.java              # Entity
  │   ├── OrderRepository.java    # Data access
  │   ├── OrderService.java       # Business logic
  │   └── OrderController.java    # HTTP API
  ├── product/
  │   └── [same]
  └── shared/
      ├── exception/              # Global error handling
      ├── security/               # Auth config
      └── config/                 # Minimal config
```

### **Rules**
1. **One feature = one package** (order/, product/, user/)
2. **Max 4 layers:** Controller → Service → Repository → Entity
3. **No "common" or "utils"** packages until you have duplicate code in 3+ places
4. **Services can call other services directly** - no event bus until you need it
5. **Global concerns** (auth, errors) go in shared/

### **When to add complexity:**

| Add This | When You Hit This Pain |
|----------|----------------------|
| Service layer | Controller has business logic |
| DTO mapping | Exposing entities directly causes problems |
| Events | Services are too coupled |
| Caching | Database is slow despite optimization |
| Queue | API timeouts on long operations |
| Separate service | Part of system needs different scaling |

---

## **The Migration Path**

When your simple system needs to scale:

### **Step 1: Vertical Scale**
- Bigger server
- Read replicas
- Connection pooling

**Cost:** $200/month → $1000/month
**Handles:** 10x more traffic

### **Step 2: Extract Bottlenecks**
Profile and find the slowest parts:
- Heavy queries → Add caching
- Image processing → Async queue
- Reports → Separate read DB

**Cost:** Add Redis, queue system
**Handles:** Another 10x

### **Step 3: Split Services (If You Must)**
Extract only the parts that need it:
- Image service (CPU bound)
- Analytics service (different DB needs)
- Everything else stays monolith

**NOT:** Split into 20 microservices by "domain boundaries"

---

## **The Anti-Patterns to Avoid**

### **1. Repository-Service-Controller With Nothing In Between**

```java
@RestController
class OrderController {
    @Autowired OrderService service;
    
    @PostMapping("/orders")
    Order create(@RequestBody Order order) {
        return service.save(order);  // Just passes through
    }
}

@Service
class OrderService {
    @Autowired OrderRepository repo;
    
    Order save(Order order) {
        return repo.save(order);  // Just passes through
    }
}
```

**This is cargo cult programming.** Just use the repository directly in the controller until you have actual business logic.

### **2. DTO Hell**

```java
CreateOrderRequest → CreateOrderCommand → OrderEntity → OrderDTO → OrderResponse
```

Five objects for one concept. Each needs mapping code. Why?

**Do this instead:** Use the entity directly until you have a reason not to.

### **3. Generic Repository Abstraction**

```java
public interface GenericRepository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    T save(T entity);
    // etc
}
```

Spring Data JPA already gives you this! You've reinvented the wheel worse.

---

## **The Mental Model**

Think of complexity as **debt you're taking on**:

- **Microservices:** High interest rate, occasional large payoff
- **Layers/abstraction:** Low interest rate, rarely pays off
- **Simple code:** No debt

Most people take on huge debt (microservices, layers, abstractions) for startups that fail before the debt matters. The ones that succeed get big enough to pay off the debt before it kills them.

Your principles say: **Stay debt-free until you're forced to borrow.**

In Spring Boot terms:
- Start with one app, one database
- Add complexity only when you measure a problem
- Never add patterns "because enterprise does it"
- Optimize for speed of iteration until you hit actual scale

**The question:** How many Spring Boot projects die before they need microservices? 95%? 99%?

Build for the 95%, not the 5%.
