# Spring Boot Filters, Interceptors & Exception Handling

## Overview — Request Processing Pipeline

Every HTTP request in Spring Boot passes through a series of layers before reaching a controller, and the response passes back through the same layers. Understanding this pipeline is critical for interviews.

```
HTTP Request
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  Servlet Container (Tomcat/Undertow)                │
│  ┌───────────────────────────────────────────────┐  │
│  │  Filter Chain  (javax.servlet.Filter)         │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │  Spring Security Filters               │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │  Custom Filters (Logging, CORS, etc.)  │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  DispatcherServlet                                  │
│  ┌───────────────────────────────────────────────┐  │
│  │  HandlerInterceptor Chain                     │  │
│  │  (preHandle → Controller → postHandle →       │  │
│  │   afterCompletion)                            │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  @RestController / @Controller                      │
│  @ExceptionHandler / @ControllerAdvice              │
└─────────────────────────────────────────────────────┘
```

---

## PART 1 — FILTERS

## What is a Filter?

A Filter is a Servlet-level component that intercepts ALL HTTP requests before they reach the DispatcherServlet. It operates at the lowest layer — before Spring MVC's request mapping, before security context, before anything Spring-specific.

**Key characteristics:**
- Runs for ALL requests (static resources, actuator, everything)
- Can modify request AND response
- Can short-circuit the request (prevent it from reaching the controller)
- Part of the Servlet specification — not Spring-specific
- Ordered by `@Order` or `Ordered` interface

## Implementing a Filter

### Option 1: `OncePerRequestFilter` (Most Common)

```java
@Component
@Order(1)  // lower number = runs earlier
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString();

        // ── PRE-processing (runs before controller) ──
        request.setAttribute("requestId", requestId);
        response.setHeader("X-Request-Id", requestId);
        log.info("[{}] → {} {}", requestId, request.getMethod(), request.getRequestURI());

        // ── Pass to next filter / DispatcherServlet ──
        filterChain.doFilter(request, response);

        // ── POST-processing (runs after controller returns) ──
        long duration = System.currentTimeMillis() - startTime;
        log.info("[{}] ← {} {} in {}ms",
                requestId,
                response.getStatus(),
                request.getRequestURI(),
                duration);
    }

    // Skip filter for certain paths
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getRequestURI();
        return path.startsWith("/actuator") || path.startsWith("/health");
    }
}
```

### Option 2: JWT Authentication Filter

```java
@Component
@Order(2)
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtService jwtService;

    @Autowired
    private UserDetailsService userDetailsService;

    private static final List<String> PUBLIC_PATHS =
            List.of("/api/auth/login", "/api/auth/register", "/api/public");

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        // Skip JWT check for public paths
        if (isPublicPath(request)) {
            filterChain.doFilter(request, response);
            return;
        }

        String authHeader = request.getHeader(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);  // let Spring Security handle
            return;
        }

        String token = authHeader.substring(7);

        try {
            String username = jwtService.extractUsername(token);

            if (username != null &&
                    SecurityContextHolder.getContext().getAuthentication() == null) {

                UserDetails userDetails =
                        userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(token, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken =
                            new UsernamePasswordAuthenticationToken(
                                    userDetails, null, userDetails.getAuthorities());
                    authToken.setDetails(
                            new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (JwtException e) {
            // Invalid token — don't set auth context, Spring Security will reject
            log.warn("Invalid JWT token: {}", e.getMessage());
        }

        filterChain.doFilter(request, response);
    }

    private boolean isPublicPath(HttpServletRequest request) {
        String path = request.getRequestURI();
        return PUBLIC_PATHS.stream().anyMatch(path::startsWith);
    }
}
```

### Option 3: Filter via FilterRegistrationBean (More Control)

```java
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<RequestLoggingFilter> loggingFilter() {
        FilterRegistrationBean<RequestLoggingFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new RequestLoggingFilter());
        bean.addUrlPatterns("/api/*");  // only apply to /api/** paths
        bean.setOrder(1);
        bean.setName("requestLoggingFilter");
        return bean;
    }
}
```

---

## PART 2 — INTERCEPTORS

## What is a HandlerInterceptor?

An Interceptor is a Spring MVC component that intercepts requests at the Spring level — AFTER filters and AFTER the DispatcherServlet has mapped the request to a controller. Interceptors have access to the handler (controller method) and ModelAndView.

**Key characteristics:**
- Runs only for requests handled by Spring MVC (not static resources, unless configured)
- Access to handler method metadata (controller class, method, annotations)
- Three interception points: `preHandle`, `postHandle`, `afterCompletion`
- Can be applied to specific URL patterns

## Interceptor Lifecycle

```
Filter → DispatcherServlet → preHandle() → Controller → postHandle() → afterCompletion()
                                ↓
                    returns false? → stop, no controller call
```

## Implementing an Interceptor

```java
@Component
public class AuthenticationInterceptor implements HandlerInterceptor {

    @Autowired
    private JwtService jwtService;

    // Runs BEFORE the controller method
    // Return true = continue; false = abort request
    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) throws Exception {

        // Access handler method metadata
        if (handler instanceof HandlerMethod handlerMethod) {

            // Check for @Public annotation — skip auth
            if (handlerMethod.hasMethodAnnotation(Public.class) ||
                handlerMethod.getBeanType().isAnnotationPresent(Public.class)) {
                return true;
            }

            // Check for required role
            RequiresRole requiresRole = handlerMethod.getMethodAnnotation(RequiresRole.class);
            if (requiresRole != null) {
                String token = extractToken(request);
                if (!jwtService.hasRole(token, requiresRole.value())) {
                    response.setStatus(HttpStatus.FORBIDDEN.value());
                    response.getWriter().write("{\"error\": \"Insufficient role\"}");
                    return false;  // abort
                }
            }
        }

        return true;  // continue
    }

    // Runs AFTER controller method but BEFORE response is written
    // Only called if preHandle returned true
    @Override
    public void postHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            ModelAndView modelAndView) throws Exception {

        // Can add response headers here
        response.setHeader("X-Processed-By", "api-gateway");
    }

    // Runs AFTER response is fully written (even if exception occurred)
    // Good for cleanup
    @Override
    public void afterCompletion(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            Exception ex) throws Exception {

        // Cleanup — e.g. clear ThreadLocal
        RequestContextHolder.resetRequestAttributes();

        if (ex != null) {
            log.error("Request failed: {} {}", request.getMethod(), request.getRequestURI(), ex);
        }
    }

    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        return header != null && header.startsWith("Bearer ")
                ? header.substring(7) : null;
    }
}
```

### Rate Limiting Interceptor

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    @Autowired
    private StringRedisTemplate redis;

    private static final int MAX_REQUESTS = 100;
    private static final int WINDOW_SECONDS = 60;

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {

        String clientIp = getClientIp(request);
        String key = "rate_limit:" + clientIp;

        Long count = redis.opsForValue().increment(key);
        if (count == 1) {
            redis.expire(key, Duration.ofSeconds(WINDOW_SECONDS));
        }

        response.setHeader("X-RateLimit-Limit", String.valueOf(MAX_REQUESTS));
        response.setHeader("X-RateLimit-Remaining", String.valueOf(MAX_REQUESTS - count));

        if (count > MAX_REQUESTS) {
            response.setStatus(429);
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
            return false;
        }

        return true;
    }

    private String getClientIp(HttpServletRequest request) {
        String forwarded = request.getHeader("X-Forwarded-For");
        return forwarded != null ? forwarded.split(",")[0].trim() : request.getRemoteAddr();
    }
}
```

### Registering Interceptors

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired
    private AuthenticationInterceptor authInterceptor;

    @Autowired
    private RateLimitInterceptor rateLimitInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {

        // Auth interceptor — applied to all /api/** except auth endpoints
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/auth/**", "/api/public/**");

        // Rate limit — applied to all endpoints
        registry.addInterceptor(rateLimitInterceptor)
                .addPathPatterns("/**")
                .order(1);  // runs first
    }
}
```

## Filters vs Interceptors — When to Use Which

| | Filter | Interceptor |
|---|---|---|
| Level | Servlet (low-level) | Spring MVC (high-level) |
| Spring beans accessible | Not by default (use `DelegatingFilterProxy`) | Yes — full DI |
| Access to handler/controller | No | Yes (`HandlerMethod`) |
| Runs for | ALL requests (static files, actuator) | Spring MVC mapped requests only |
| Request/Response modification | Yes | Yes |
| Short-circuit request | Yes (`filterChain` not called) | Yes (return `false`) |
| Use for | JWT auth, CORS, request logging, compression | Role checks, audit logging, annotation-based behavior |

---

## PART 3 — EXCEPTION HANDLING

## Why Centralized Exception Handling?

Without centralized handling:

```java
// Every controller has try-catch:
@GetMapping("/{id}")
public Order getOrder(@PathVariable Long id) {
    try {
        return orderService.findById(id);
    } catch (OrderNotFoundException e) {
        return ResponseEntity.notFound()...  // repeated everywhere
    } catch (UnauthorizedException e) {
        return ResponseEntity.status(403)...  // repeated everywhere
    }
}
```

**With `@ControllerAdvice`:** one class handles all exceptions for all controllers.

## @ControllerAdvice — Global Exception Handler

```java
@RestControllerAdvice  // = @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    // ── Domain Exceptions ──

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiError handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ApiError.of(404, ex.getMessage(), request.getRequestURI());
    }

    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleValidation(ValidationException ex, HttpServletRequest request) {
        return ApiError.of(400, ex.getMessage(), request.getRequestURI());
    }

    @ExceptionHandler(UnauthorizedException.class)
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    public ApiError handleUnauthorized(UnauthorizedException ex, HttpServletRequest request) {
        return ApiError.of(401, "Unauthorized", request.getRequestURI());
    }

    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ApiError handleForbidden(AccessDeniedException ex, HttpServletRequest request) {
        return ApiError.of(403, "Access denied", request.getRequestURI());
    }

    // ── Spring Validation Exceptions ──

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleValidationErrors(MethodArgumentNotValidException ex,
                                           HttpServletRequest request) {
        List<String> errors = ex.getBindingResult()
                .getFieldErrors()
                .stream()
                .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
                .toList();

        return ApiError.builder()
                .status(400)
                .message("Validation failed")
                .errors(errors)
                .path(request.getRequestURI())
                .timestamp(LocalDateTime.now())
                .build();
    }

    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleConstraintViolation(ConstraintViolationException ex,
                                              HttpServletRequest request) {
        List<String> errors = ex.getConstraintViolations()
                .stream()
                .map(cv -> cv.getPropertyPath() + ": " + cv.getMessage())
                .toList();

        return ApiError.of(400, "Constraint violation", request.getRequestURI(), errors);
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleMalformedJson(HttpMessageNotReadableException ex,
                                        HttpServletRequest request) {
        return ApiError.of(400, "Malformed JSON request", request.getRequestURI());
    }

    @ExceptionHandler(MethodNotAllowedException.class)
    @ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
    public ApiError handleMethodNotAllowed(MethodNotAllowedException ex,
                                           HttpServletRequest request) {
        return ApiError.of(405, "Method not allowed: " + request.getMethod(),
                request.getRequestURI());
    }

    // ── Fallback — catch all unhandled exceptions ──

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiError handleGenericException(Exception ex, HttpServletRequest request) {
        // Log full stack trace internally — NEVER expose to client
        log.error("Unhandled exception on {} {}: {}",
                request.getMethod(), request.getRequestURI(), ex.getMessage(), ex);

        return ApiError.of(500, "An internal error occurred", request.getRequestURI());
        // Never send ex.getMessage() to client — may leak DB errors, stack traces, etc.
    }
}
```

### Standard Error Response DTO

```java
@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiError {
    private int status;
    private String message;
    private String path;
    private LocalDateTime timestamp;
    private List<String> errors;  // for validation errors

    public static ApiError of(int status, String message, String path) {
        return ApiError.builder()
                .status(status)
                .message(message)
                .path(path)
                .timestamp(LocalDateTime.now())
                .build();
    }

    public static ApiError of(int status, String message, String path, List<String> errors) {
        return ApiError.builder()
                .status(status)
                .message(message)
                .path(path)
                .errors(errors)
                .timestamp(LocalDateTime.now())
                .build();
    }
}
```

Sample error responses:

```json
// 404 Not Found
{
  "status": 404,
  "message": "Order with id 123 not found",
  "path": "/api/orders/123",
  "timestamp": "2025-01-15T10:30:00"
}

// 400 Validation Error
{
  "status": 400,
  "message": "Validation failed",
  "path": "/api/orders",
  "timestamp": "2025-01-15T10:30:00",
  "errors": [
    "amount: must be greater than 0",
    "userId: must not be null"
  ]
}
```

### Custom Domain Exceptions

```java
// Base exception
public abstract class AppException extends RuntimeException {
    private final HttpStatus status;

    protected AppException(String message, HttpStatus status) {
        super(message);
        this.status = status;
    }

    public HttpStatus getStatus() { return status; }
}

// Specific exceptions
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resource, Object id) {
        super(resource + " with id " + id + " not found", HttpStatus.NOT_FOUND);
    }
}

public class DuplicateResourceException extends AppException {
    public DuplicateResourceException(String message) {
        super(message, HttpStatus.CONFLICT);
    }
}

public class UnauthorizedException extends AppException {
    public UnauthorizedException(String message) {
        super(message, HttpStatus.UNAUTHORIZED);
    }
}

public class BusinessRuleException extends AppException {
    public BusinessRuleException(String message) {
        super(message, HttpStatus.UNPROCESSABLE_ENTITY);
    }
}

// Usage in service
public Order placeOrder(OrderRequest req) {
    if (req.getTotal().compareTo(BigDecimal.ZERO) <= 0) {
        throw new BusinessRuleException("Order total must be greater than zero");
    }

    User user = userRepo.findById(req.getUserId())
            .orElseThrow(() -> new ResourceNotFoundException("User", req.getUserId()));

    if (orderRepo.existsByIdempotencyKey(req.getIdempotencyKey())) {
        throw new DuplicateResourceException("Order already placed with this idempotency key");
    }

    return orderRepo.save(new Order(req));
}
```

### Controller-Level Exception Handler

For exceptions specific to one controller:

```java
@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    // Only applies to this controller — overrides GlobalExceptionHandler for this class
    @ExceptionHandler(PaymentDeclinedException.class)
    @ResponseStatus(HttpStatus.PAYMENT_REQUIRED)
    public ApiError handlePaymentDeclined(PaymentDeclinedException ex) {
        return ApiError.of(402, ex.getMessage(), "/api/payments");
    }
}
```

### Validation on Request Body

```java
@Data
public class CreateOrderRequest {

    @NotNull(message = "User ID is required")
    private Long userId;

    @NotEmpty(message = "Order must have at least one item")
    private List<@Valid OrderItemRequest> items;

    @DecimalMin(value = "0.01", message = "Total must be greater than 0")
    @NotNull
    private BigDecimal total;

    @Email(message = "Invalid email format")
    private String contactEmail;
}

@Data
public class OrderItemRequest {
    @NotNull
    private Long productId;

    @Min(value = 1, message = "Quantity must be at least 1")
    private Integer quantity;
}

@RestController
@Validated  // enables @Validated for path variable / request param validation
public class OrderController {

    @PostMapping("/api/orders")
    public ResponseEntity<Order> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {
        // @Valid triggers validation; MethodArgumentNotValidException on failure
        Order order = orderService.createOrder(request);
        return ResponseEntity.status(201).body(order);
    }

    @GetMapping("/api/orders/{id}")
    public Order getOrder(
            @PathVariable @Min(value = 1, message = "ID must be positive") Long id) {
        // @Validated on class enables this; ConstraintViolationException on failure
        return orderService.findById(id);
    }
}
```

## Filter vs Interceptor vs AOP for Cross-Cutting Concerns

```java
// AOP — method-level aspect for any Spring bean (not just controllers)
@Aspect
@Component
public class PerformanceAspect {

    @Around("@annotation(TrackPerformance)")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            log.info("{} took {}ms", pjp.getSignature(), duration);
        }
    }
}

// Annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TrackPerformance {}

// Usage
@Service
public class OrderService {

    @TrackPerformance
    public Order placeOrder(OrderRequest req) {
        // AOP logs execution time automatically
    }
}
```

| Approach | Use For |
|---|---|
| Filter | Request/response modification, JWT auth, CORS, logging, rate limiting (all requests) |
| Interceptor | Controller-specific logic, annotation-driven behavior, pre/post handler hooks |
| AOP `@Around` | Service/repository layer concerns, performance tracking, transaction management |
| `@ControllerAdvice` | Exception handling, global response transformation |

## Interview Questions

**Q1: What is the difference between a Filter and an Interceptor in Spring Boot?**

Filters are Servlet-level — they run before the DispatcherServlet for ALL requests (static files, actuator endpoints) and are not Spring-managed by default. Interceptors are Spring MVC-level — they run after DispatcherServlet maps the request to a controller, only for Spring-handled requests, and have full access to Spring beans and handler metadata. Use Filters for JWT auth and CORS (need to run before Spring Security); use Interceptors for controller-specific concerns like role checking or audit logging.

**Q2: What are the three methods of HandlerInterceptor and when does each run?**

`preHandle` runs before the controller method — return `false` to abort. `postHandle` runs after the controller returns but before the response is written — can add headers or modify ModelAndView. `afterCompletion` runs after the response is fully written, even if an exception occurred — use for cleanup (ThreadLocal clearing, metric recording). Note: `postHandle` and `afterCompletion` only run if `preHandle` returned `true`.

**Q3: How does @ControllerAdvice work and what are its advantages?**

`@ControllerAdvice` is a specialization of `@Component` that makes `@ExceptionHandler`, `@InitBinder`, and `@ModelAttribute` methods apply globally to all controllers. It centralizes error handling — instead of try-catch in every controller, you define exception-to-response mappings once. It makes error responses consistent, keeps controllers clean, and allows different handlers for different exception types with appropriate HTTP status codes.

**Q4: What is the difference between @ExceptionHandler in a controller vs in @ControllerAdvice?**

`@ExceptionHandler` inside a `@Controller` class only handles exceptions from that specific controller. `@ExceptionHandler` inside a `@ControllerAdvice` class handles exceptions from all controllers globally. The controller-local handler takes precedence for its own controller's exceptions.

**Q5: How do you ensure internal errors (DB exceptions, stack traces) aren't exposed to clients?**

In the `@ExceptionHandler(Exception.class)` fallback, log the full exception internally (with stack trace) but return only a generic message to the client (`"An internal error occurred"`). Never include `ex.getMessage()` in responses for unexpected exceptions — it may expose table names, SQL errors, or implementation details. Only expose safe, curated messages for known domain exceptions.

**Q6: What is `@Validated` on a controller class and when is it needed?**

`@Validated` (different from `@Valid`) enables Spring's method-level validation — required when you want to validate path variables, request parameters, or method return values (not just request bodies). Without it on the class, `@Min`, `@NotNull` etc. on individual parameters are silently ignored. Violations throw `ConstraintViolationException` (handled in `@ControllerAdvice`) rather than `MethodArgumentNotValidException`.
