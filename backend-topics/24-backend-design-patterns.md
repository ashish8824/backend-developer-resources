# Backend Design Patterns

## Why Design Patterns?

Design patterns are reusable solutions to common software design problems. They're not code you copy — they're blueprints for structuring code to be maintainable, extensible, and testable.

In backend interviews, knowing WHAT a pattern is matters less than knowing WHEN and WHY to use it, and being able to implement it.

---

## 1. Repository Pattern

### What is it?

An abstraction layer between the domain/business logic and the data access layer. The business logic works with a `Repository` interface and doesn't know or care whether the data comes from a PostgreSQL database, MongoDB, a file, or a mock.

```
Without Repository:
OrderService -> directly calls jdbcTemplate.query("SELECT ...") / EntityManager.find(...)
            -> business logic knows about SQL, JPA, DB structure
            -> hard to test (need real DB), hard to switch storage

With Repository:
OrderService -> OrderRepository interface -> PostgresOrderRepository (production)
                                          -> InMemoryOrderRepository (testing)
```

### Java Implementation

```java
// Repository interface — domain layer, knows nothing about DB
public interface OrderRepository {
    Optional<Order> findById(Long id);
    List<Order> findByUserId(Long userId);
    List<Order> findByStatus(OrderStatus status);
    Order save(Order order);
    void delete(Long id);
}

// JPA implementation
@Repository
public class JpaOrderRepository implements OrderRepository {

    @PersistenceContext
    private EntityManager em;

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(em.find(Order.class, id));
    }

    @Override
    public List<Order> findByUserId(Long userId) {
        return em.createQuery(
                "SELECT o FROM Order o WHERE o.userId = :userId", Order.class)
                .setParameter("userId", userId)
                .getResultList();
    }

    @Override
    public Order save(Order order) {
        if (order.getId() == null) {
            em.persist(order);
            return order;
        }
        return em.merge(order);
    }

    @Override
    public void delete(Long id) {
        Order order = em.find(Order.class, id);
        if (order != null) em.remove(order);
    }
}

// In-memory implementation for testing
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> store = new HashMap<>();
    private long idSequence = 1L;

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public Order save(Order order) {
        if (order.getId() == null) order.setId(idSequence++);
        store.put(order.getId(), order);
        return order;
    }
    // ...
}

// Service — only knows the interface
@Service
public class OrderService {

    private final OrderRepository orderRepo;  // injected — doesn't know which implementation

    public OrderService(OrderRepository orderRepo) {
        this.orderRepo = orderRepo;
    }

    public Order getOrder(Long id) {
        return orderRepo.findById(id)
                .orElseThrow(() -> new OrderNotFoundException(id));
    }
}
```

**Spring Data JPA** implements this pattern for free:

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByUserId(Long userId);
    List<Order> findByStatus(OrderStatus status);
    // Spring Data generates the SQL automatically
}
```

---

## 2. Strategy Pattern

### What is it?

Define a family of algorithms, put each in its own class, and make them interchangeable. The client selects the algorithm at runtime without changing the code that uses it.

```
Without Strategy:
if (paymentType == "CREDIT_CARD") { ... }
else if (paymentType == "PAYPAL") { ... }
else if (paymentType == "UPI") { ... }
// Add new payment type -> modify this if-else chain everywhere

With Strategy:
PaymentStrategy creditCard = new CreditCardStrategy();
PaymentStrategy paypal = new PaypalStrategy();
PaymentStrategy upi = new UpiStrategy();
// Add new payment type -> new class, zero change to existing code
```

### Java Implementation

```java
// Strategy interface
public interface PaymentStrategy {
    PaymentResult pay(BigDecimal amount, PaymentDetails details);
    String getType();
}

// Concrete strategies
@Component
public class CreditCardStrategy implements PaymentStrategy {
    @Override
    public PaymentResult pay(BigDecimal amount, PaymentDetails details) {
        // Call Stripe API
        return stripeService.charge(amount, details.getCardToken());
    }
    @Override
    public String getType() { return "CREDIT_CARD"; }
}

@Component
public class UpiStrategy implements PaymentStrategy {
    @Override
    public PaymentResult pay(BigDecimal amount, PaymentDetails details) {
        // Call UPI gateway
        return upiGateway.initiate(amount, details.getUpiId());
    }
    @Override
    public String getType() { return "UPI"; }
}

@Component
public class PaypalStrategy implements PaymentStrategy {
    @Override
    public PaymentResult pay(BigDecimal amount, PaymentDetails details) {
        return paypalService.charge(amount, details.getPaypalToken());
    }
    @Override
    public String getType() { return "PAYPAL"; }
}

// Context — holds reference to strategy, executes it
@Service
public class PaymentService {

    private final Map<String, PaymentStrategy> strategies;

    // Spring injects all PaymentStrategy beans automatically
    public PaymentService(List<PaymentStrategy> strategyList) {
        this.strategies = strategyList.stream()
                .collect(Collectors.toMap(PaymentStrategy::getType, s -> s));
    }

    public PaymentResult processPayment(String paymentType,
                                         BigDecimal amount,
                                         PaymentDetails details) {
        PaymentStrategy strategy = strategies.get(paymentType);
        if (strategy == null) {
            throw new UnsupportedPaymentTypeException(paymentType);
        }
        return strategy.pay(amount, details);
    }
}
```

Other common uses: sorting algorithms, discount calculation, export formats (PDF/CSV/Excel), notification channels (Email/SMS/Push).

---

## 3. Observer Pattern

### What is it?

When one object (Subject/Publisher) changes state, all dependent objects (Observers/Subscribers) are automatically notified and updated. The publisher doesn't know who the subscribers are — it just fires the event.

```
UserService creates a new user
  -> Without Observer: UserService must call EmailService.sendWelcome(), 
                        AnalyticsService.trackSignup(), RewardService.giveBonus()
                        -> UserService depends on all these services!

  -> With Observer: UserService publishes "USER_REGISTERED" event
                    EmailService, AnalyticsService, RewardService each subscribe
                    -> UserService knows nothing about them
```

### Java (Spring Events)

Spring's built-in ApplicationEventPublisher is a clean implementation of the Observer pattern.

```java
// Event class
public class UserRegisteredEvent {
    private final String userId;
    private final String email;
    private final String name;

    public UserRegisteredEvent(String userId, String email, String name) {
        this.userId = userId;
        this.email = email;
        this.name = name;
    }
    // getters
}

// Publisher (Subject)
@Service
public class UserService {

    @Autowired
    private UserRepository userRepo;

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Transactional
    public User registerUser(RegisterRequest request) {
        User user = new User(request.getEmail(), encode(request.getPassword()));
        userRepo.save(user);

        // Publish event — doesn't know who handles it
        eventPublisher.publishEvent(
                new UserRegisteredEvent(user.getId().toString(), user.getEmail(), user.getName())
        );

        return user;
    }
}

// Observers (Listeners) — completely independent
@Component
public class WelcomeEmailListener {

    @EventListener
    @Async  // runs in separate thread — doesn't block the registration
    public void onUserRegistered(UserRegisteredEvent event) {
        emailService.sendWelcomeEmail(event.getEmail(), event.getName());
    }
}

@Component
public class AnalyticsListener {

    @EventListener
    @Async
    public void onUserRegistered(UserRegisteredEvent event) {
        analyticsService.trackSignup(event.getUserId());
    }
}

@Component
public class LoyaltyRewardListener {

    @EventListener
    public void onUserRegistered(UserRegisteredEvent event) {
        rewardService.giveSignupBonus(event.getUserId());
    }
}
```

Adding a new "send Slack notification to team" behavior = add a new `@EventListener` class. Zero changes to `UserService`.

---

## 4. Singleton Pattern

### What is it?

Ensures a class has only one instance and provides a global access point to it.

In Spring Boot, every `@Bean` and `@Component` is a Singleton by default — Spring manages the single instance for you. You rarely need to implement the pattern manually.

### Thread-Safe Singleton (when you need it without Spring)

```java
public class DatabaseConnection {

    // volatile: ensures visibility across threads (no caching in CPU registers)
    private static volatile DatabaseConnection instance;

    private final Connection connection;

    private DatabaseConnection() {
        this.connection = DriverManager.getConnection("jdbc:postgresql://...");
    }

    // Double-checked locking — efficient and thread-safe
    public static DatabaseConnection getInstance() {
        if (instance == null) {                      // first check (no lock)
            synchronized (DatabaseConnection.class) {
                if (instance == null) {              // second check (with lock)
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }

    public Connection getConnection() { return connection; }
}
```

### Enum Singleton (Simplest, Thread-Safe)

```java
public enum AppConfig {
    INSTANCE;

    private final String apiKey = System.getenv("API_KEY");
    private final int maxRetries = 3;

    public String getApiKey() { return apiKey; }
    public int getMaxRetries() { return maxRetries; }
}

// Usage
AppConfig.INSTANCE.getApiKey();
```

Enum singletons are inherently thread-safe and serialization-safe.

---

## 5. Factory Pattern

### What is it?

A factory method creates objects without exposing the creation logic. The client asks for an object; the factory decides which class to instantiate.

```java
// Interface
public interface NotificationSender {
    void send(String to, String message);
}

// Implementations
@Component("EMAIL_SENDER")
public class EmailSender implements NotificationSender {
    public void send(String to, String message) {
        // send email
    }
}

@Component("SMS_SENDER")
public class SmsSender implements NotificationSender {
    public void send(String to, String message) {
        // send SMS
    }
}

@Component("PUSH_SENDER")
public class PushNotificationSender implements NotificationSender {
    public void send(String to, String message) {
        // send push notification
    }
}

// Factory
@Component
public class NotificationSenderFactory {

    @Autowired
    private ApplicationContext context;

    public NotificationSender getSender(String channel) {
        return switch (channel) {
            case "EMAIL" -> context.getBean("EMAIL_SENDER", NotificationSender.class);
            case "SMS"   -> context.getBean("SMS_SENDER", NotificationSender.class);
            case "PUSH"  -> context.getBean("PUSH_SENDER", NotificationSender.class);
            default -> throw new IllegalArgumentException("Unknown channel: " + channel);
        };
    }
}

// Usage
@Service
public class NotificationService {
    @Autowired
    private NotificationSenderFactory factory;

    public void notify(String channel, String to, String message) {
        factory.getSender(channel).send(to, message);
    }
}
```

---

## 6. Middleware Pattern

### What is it?

A chain of processing steps (middleware) where each step can inspect, modify, or short-circuit a request before it reaches the handler. Common in web frameworks.

```
Request -> [Logger] -> [Auth] -> [Rate Limiter] -> [Validator] -> Handler
Response <- [Logger] <- [Auth] <- [Rate Limiter] <- [Validator] <- Handler
```

### Express.js Middleware

```javascript
// Custom middleware functions
const logger = (req, res, next) => {
    console.log(`${req.method} ${req.path} at ${new Date().toISOString()}`);
    next();  // pass to next middleware
};

const authenticate = (req, res, next) => {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) return res.status(401).json({ error: 'Unauthorized' });
    try {
        req.user = jwt.verify(token, process.env.JWT_SECRET);
        next();
    } catch {
        res.status(401).json({ error: 'Invalid token' });
    }
};

const requireRole = (role) => (req, res, next) => {
    if (!req.user.roles.includes(role)) {
        return res.status(403).json({ error: 'Forbidden' });
    }
    next();
};

// Apply middleware
app.use(logger);                      // global
app.use('/api', authenticate);        // route-scoped
app.delete('/api/users/:id', requireRole('ADMIN'), deleteUser);  // endpoint-scoped
```

### Spring Boot Filter / Interceptor

```java
// Filter (Servlet level — runs before Spring MVC)
@Component
@Order(1)
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        long start = System.currentTimeMillis();
        chain.doFilter(request, response);  // execute next filter / handler
        long duration = System.currentTimeMillis() - start;

        log.info("{} {} -> {} in {}ms",
                request.getMethod(), request.getRequestURI(),
                response.getStatus(), duration);
    }
}

// HandlerInterceptor (Spring MVC level — runs inside DispatcherServlet)
@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String token = request.getHeader("Authorization");
        if (token == null) {
            response.setStatus(401);
            return false;  // stop processing
        }
        return true;  // continue to controller
    }
}
```

---

## 7. Dependency Injection (DI)

### What is it?

Instead of a class creating its own dependencies, the dependencies are provided (injected) from outside. This decouples classes, makes them testable, and allows swapping implementations.

```java
// Without DI (tightly coupled)
public class OrderService {
    private OrderRepository repo = new PostgresOrderRepository();  // hardcoded!
    // Can't test without a real DB; can't swap to another DB
}

// With DI (loosely coupled)
@Service
public class OrderService {
    private final OrderRepository repo;  // interface, not implementation

    @Autowired  // Spring injects the appropriate implementation
    public OrderService(OrderRepository repo) {
        this.repo = repo;
    }
    // Can inject InMemoryOrderRepository for tests; real DB for production
}
```

### Constructor vs Field vs Setter Injection

```java
// Constructor Injection (RECOMMENDED)
@Service
public class OrderService {
    private final OrderRepository repo;
    private final PaymentService payment;

    // All required deps in constructor — immutable, easy to test, clear dependencies
    public OrderService(OrderRepository repo, PaymentService payment) {
        this.repo = repo;
        this.payment = payment;
    }
}

// Field Injection (NOT RECOMMENDED — can't use without Spring, hides dependencies)
@Service
public class OrderService {
    @Autowired private OrderRepository repo;  // hidden dependency, not final
}
```

Always use constructor injection — it makes dependencies explicit and enables easy unit testing without Spring:

```java
// Unit test without Spring context
OrderRepository mockRepo = mock(OrderRepository.class);
PaymentService mockPayment = mock(PaymentService.class);
OrderService orderService = new OrderService(mockRepo, mockPayment);  // no Spring needed
```

---

## Pattern Summary

| Pattern | Problem Solved | Key Benefit |
|---|---|---|
| Repository | Decouple business logic from data access | Testability, swap storage layer |
| Strategy | Replace if-else chains for interchangeable algorithms | Open/Closed Principle |
| Observer | Notify multiple components of state changes | Decoupling, extensibility |
| Singleton | One shared instance of a resource | Resource efficiency (Spring handles this) |
| Factory | Create objects without exposing creation logic | Encapsulation, extensibility |
| Middleware | Process requests through a chain of steps | Separation of cross-cutting concerns |
| Dependency Injection | Provide dependencies from outside a class | Testability, loose coupling |

## Interview Questions

**Q1: What is the difference between Strategy and Factory patterns?**

Strategy defines interchangeable algorithms selected at runtime — the context executes whichever strategy is chosen (e.g. which payment method to use). Factory is about object creation — it decides which class to instantiate without exposing construction logic. Strategy is about behavior; Factory is about instantiation. They're often used together: a Factory creates the right Strategy object.

**Q2: Why prefer constructor injection over field injection in Spring?**

Constructor injection makes dependencies explicit (visible in the constructor signature), allows the fields to be `final` (immutable after construction), and enables instantiation without Spring (critical for unit tests). Field injection hides dependencies, prevents `final` fields, and makes the class unusable outside a Spring context.

**Q3: How is the Repository pattern different from just using a DAO?**

The Repository pattern represents a collection of domain objects and speaks the domain language (`findByStatus(OrderStatus.PLACED)`, not `executeQuery("SELECT ...")`). A DAO (Data Access Object) is more tied to the data source — it exposes CRUD operations that mirror DB operations. Repository provides a domain-focused abstraction; DAO is a lower-level data access abstraction. In practice, Spring Data JPA repositories blend both concepts.

**Q4: When would you use the Observer pattern vs an Event Queue like Kafka?**

Spring's in-process Observer (ApplicationEventPublisher) is for the same JVM — synchronous or async within the same application. Kafka is for cross-service communication — events that need to survive process restarts, be consumed by multiple services on different machines, or be replayed. Use in-process Observer for decoupling within a service; use Kafka for decoupling between services.

**Q5: What is the Open/Closed Principle and which pattern demonstrates it best?**

Open/Closed Principle: software entities should be open for extension but closed for modification. The Strategy pattern is the clearest demonstration — adding a new payment method (extension) requires a new class implementing PaymentStrategy, with zero changes to PaymentService (closed for modification). The Factory and Observer patterns also follow this principle.
