---
layout: default
---

# Chapter 02: Foundational Knowledge Before Development

Before we dive into writing code, let's establish a solid foundation of concepts that will make you a more effective Spring Boot developer. Understanding these principles will help you make better architectural decisions and write more maintainable applications.

## 2.1 Server-to-Server Communication

In modern distributed systems, applications rarely work in isolation. Understanding how servers communicate with each other is crucial for building robust microservices and integrations.

### HTTP Communication Patterns

The most common pattern for server-to-server communication is HTTP-based REST APIs. Here's how different communication patterns work:

**Synchronous Communication:**
```kotlin
@Service
class InventoryService(
    private val restClient: RestClient
) {
    fun checkProductAvailability(productId: String): ProductAvailability {
        // Blocking call - waits for response
        return restClient.get()
            .uri("http://inventory-service/products/{id}/availability", productId)
            .retrieve()
            .body(ProductAvailability::class.java)
            ?: throw ProductNotFoundException("Product $productId not found")
    }
}
```

**Asynchronous Communication with Coroutines:**
```kotlin
@Service
class OrderProcessingService(
    private val webClient: WebClient
) {
    suspend fun processOrderAsync(order: Order): OrderResult {
        // Non-blocking call using coroutines
        val inventory = checkInventory(order.items)
        val payment = processPayment(order.payment)
        
        // Await both results concurrently
        return coroutineScope {
            val inventoryResult = async { inventory }
            val paymentResult = async { payment }
            
            OrderResult(
                inventoryStatus = inventoryResult.await(),
                paymentStatus = paymentResult.await()
            )
        }
    }
    
    private suspend fun checkInventory(items: List<OrderItem>): InventoryStatus {
        return webClient.post()
            .uri("http://inventory-service/check")
            .bodyValue(items)
            .awaitBody()
    }
    
    private suspend fun processPayment(payment: PaymentInfo): PaymentStatus {
        return webClient.post()
            .uri("http://payment-service/process")
            .bodyValue(payment)
            .awaitBody()
    }
}
```

### Message-Based Communication

Beyond HTTP, message queues provide reliable asynchronous communication:

```kotlin
@Component
class OrderEventPublisher(
    private val rabbitTemplate: RabbitTemplate
) {
    fun publishOrderCreated(order: Order) {
        val event = OrderCreatedEvent(
            orderId = order.id,
            customerId = order.customerId,
            totalAmount = order.totalAmount,
            timestamp = Instant.now()
        )
        
        rabbitTemplate.convertAndSend(
            "order-events",  // exchange
            "order.created",  // routing key
            event
        )
    }
}

@Component
class OrderEventListener {
    @RabbitListener(queues = ["order-processing-queue"])
    fun handleOrderCreated(event: OrderCreatedEvent) {
        println("Processing order: ${event.orderId}")
        // Process the order asynchronously
    }
}
```

## 2.2 How Spring Boot Works

Understanding Spring Boot's internals helps you troubleshoot issues and optimize your applications. Let's peek under the hood.

### The Startup Process

When you run a Spring Boot application, here's what happens:

1. **JVM Starts**: The Java Virtual Machine loads your application
2. **Main Method Executes**: Your `main` function calls `runApplication`
3. **SpringApplication Initializes**: Spring Boot prepares the application context
4. **Auto-configuration Runs**: Spring Boot configures beans based on classpath
5. **Application Context Refreshes**: All beans are instantiated and wired
6. **Embedded Server Starts**: Tomcat (or another server) begins listening
7. **Application Ready**: Your application is ready to handle requests

```kotlin
@SpringBootApplication
class Application

fun main(args: Array<String>) {
    // This single line triggers the entire startup process
    runApplication<Application>(*args) {
        // You can customize the startup here
        setBannerMode(Banner.Mode.OFF)
        
        // Add custom initializers
        addInitializers(
            ApplicationContextInitializer<GenericApplicationContext> { context ->
                context.registerBean<CustomService>()
            }
        )
    }
}
```

### Component Scanning

Spring Boot automatically discovers and registers components in your application:

```kotlin
// Spring Boot scans from the package containing @SpringBootApplication
// and all sub-packages

package com.example.myapp  // Root package

@SpringBootApplication  // Enables component scanning from here
class Application

// This will be found automatically
package com.example.myapp.service

@Service
class UserService {
    // Automatically registered as a Spring bean
}

// This will also be found
package com.example.myapp.repository

@Repository
interface UserRepository : JpaRepository<User, Long>
```

### Conditional Configuration

Spring Boot's power comes from conditional configuration:

```kotlin
@Configuration
class DatabaseConfiguration {
    
    @Bean
    @ConditionalOnProperty(
        name = ["app.database.cache.enabled"],
        havingValue = "true"
    )
    fun cacheManager(): CacheManager {
        return CaffeineCacheManager().apply {
            setCaffeine(Caffeine.newBuilder()
                .maximumSize(1000)
                .expireAfterWrite(10, TimeUnit.MINUTES))
        }
    }
    
    @Bean
    @ConditionalOnMissingBean(DataSource::class)
    @ConditionalOnClass(HikariDataSource::class)
    fun dataSource(): DataSource {
        return HikariDataSource().apply {
            jdbcUrl = "jdbc:h2:mem:testdb"
            username = "sa"
            password = ""
        }
    }
}
```

## 2.3 Layered Architecture

A well-structured Spring Boot application follows a layered architecture. Each layer has a specific responsibility and communicates only with adjacent layers.

### The Three-Layer Architecture

```kotlin
// Presentation Layer - Handles HTTP requests and responses
@RestController
@RequestMapping("/api/products")
class ProductController(
    private val productService: ProductService
) {
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: Long): ResponseEntity<ProductDTO> {
        return productService.findById(id)
            ?.let { ResponseEntity.ok(it.toDTO()) }
            ?: ResponseEntity.notFound().build()
    }
    
    @PostMapping
    fun createProduct(@RequestBody @Valid request: CreateProductRequest): ProductDTO {
        return productService.createProduct(request).toDTO()
    }
}

// Business Layer - Contains business logic
@Service
@Transactional
class ProductService(
    private val productRepository: ProductRepository,
    private val inventoryService: InventoryService,
    private val pricingService: PricingService
) {
    fun findById(id: Long): Product? {
        return productRepository.findById(id).orElse(null)
    }
    
    fun createProduct(request: CreateProductRequest): Product {
        // Business logic: validation, calculations, orchestration
        val price = pricingService.calculatePrice(request.basePrice, request.category)
        
        val product = Product(
            name = request.name,
            description = request.description,
            price = price,
            category = request.category
        )
        
        val savedProduct = productRepository.save(product)
        inventoryService.initializeStock(savedProduct.id, request.initialStock)
        
        return savedProduct
    }
}

// Data Access Layer - Handles database operations
@Repository
interface ProductRepository : JpaRepository<Product, Long> {
    fun findByCategory(category: String): List<Product>
    
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :minPrice AND :maxPrice")
    fun findByPriceRange(minPrice: BigDecimal, maxPrice: BigDecimal): List<Product>
}
```

### Why Layered Architecture?

1. **Separation of Concerns**: Each layer focuses on its specific responsibility
2. **Testability**: Layers can be tested independently with mocks
3. **Maintainability**: Changes in one layer don't affect others
4. **Reusability**: Business logic can be reused across different presentations

## 2.4 Design Patterns

Design patterns are proven solutions to common problems. Let's explore patterns you'll use frequently in Spring Boot applications.

### 2.4.1 Types of Design Patterns

Design patterns are categorized into three main types:
- **Creational**: How objects are created
- **Structural**: How objects are composed
- **Behavioral**: How objects interact

### 2.4.2 Creational Patterns

**Factory Pattern**
```kotlin
// Factory pattern for creating different types of notifications
interface NotificationFactory {
    fun createNotification(type: NotificationType, message: String): Notification
}

@Component
class NotificationFactoryImpl(
    private val emailService: EmailService,
    private val smsService: SmsService,
    private val pushService: PushService
) : NotificationFactory {
    
    override fun createNotification(type: NotificationType, message: String): Notification {
        return when (type) {
            NotificationType.EMAIL -> EmailNotification(emailService, message)
            NotificationType.SMS -> SmsNotification(smsService, message)
            NotificationType.PUSH -> PushNotification(pushService, message)
        }
    }
}
```

**Builder Pattern**
```kotlin
// Kotlin data classes with default parameters provide builder-like functionality
data class ApiClient(
    val baseUrl: String,
    val timeout: Duration = Duration.ofSeconds(30),
    val retryCount: Int = 3,
    val headers: Map<String, String> = emptyMap(),
    val interceptors: List<ClientInterceptor> = emptyList()
) {
    companion object {
        // Fluent builder for complex configurations
        fun builder() = Builder()
    }
    
    class Builder {
        private var baseUrl: String = ""
        private var timeout: Duration = Duration.ofSeconds(30)
        private var retryCount: Int = 3
        private val headers = mutableMapOf<String, String>()
        private val interceptors = mutableListOf<ClientInterceptor>()
        
        fun baseUrl(url: String) = apply { baseUrl = url }
        fun timeout(duration: Duration) = apply { timeout = duration }
        fun retryCount(count: Int) = apply { retryCount = count }
        fun addHeader(key: String, value: String) = apply { headers[key] = value }
        fun addInterceptor(interceptor: ClientInterceptor) = apply { interceptors.add(interceptor) }
        
        fun build(): ApiClient {
            require(baseUrl.isNotEmpty()) { "Base URL is required" }
            return ApiClient(baseUrl, timeout, retryCount, headers.toMap(), interceptors.toList())
        }
    }
}

// Usage
val client = ApiClient.builder()
    .baseUrl("https://api.example.com")
    .timeout(Duration.ofSeconds(60))
    .addHeader("Authorization", "Bearer token")
    .build()
```

**Singleton Pattern**
```kotlin
// Spring beans are singletons by default
@Component
class ConfigurationManager {
    private val properties = mutableMapOf<String, String>()
    
    init {
        // Initialization happens once
        loadProperties()
    }
    
    private fun loadProperties() {
        // Load configuration from various sources
    }
    
    fun getProperty(key: String): String? = properties[key]
}
```

### 2.4.3 Structural Patterns

**Adapter Pattern**
```kotlin
// Adapting external API responses to our domain model
interface PaymentGateway {
    fun processPayment(amount: BigDecimal, currency: String): PaymentResult
}

@Component
class StripeAdapter(
    private val stripeClient: StripeClient
) : PaymentGateway {
    
    override fun processPayment(amount: BigDecimal, currency: String): PaymentResult {
        // Adapt our interface to Stripe's API
        val stripeAmount = (amount * BigDecimal(100)).toInt() // Stripe uses cents
        val charge = stripeClient.createCharge(
            amount = stripeAmount,
            currency = currency.lowercase(),
            source = "tok_visa" // Would come from request
        )
        
        // Adapt Stripe's response to our domain model
        return PaymentResult(
            transactionId = charge.id,
            status = if (charge.paid) PaymentStatus.SUCCESS else PaymentStatus.FAILED,
            amount = amount,
            currency = currency
        )
    }
}
```

**Decorator Pattern**
```kotlin
// Adding behavior to services without modifying them
interface PricingService {
    fun calculatePrice(basePrice: BigDecimal, quantity: Int): BigDecimal
}

@Component
@Primary
class DiscountPricingDecorator(
    @Qualifier("basic") private val basicPricing: PricingService,
    private val discountService: DiscountService
) : PricingService {
    
    override fun calculatePrice(basePrice: BigDecimal, quantity: Int): BigDecimal {
        val basicPrice = basicPricing.calculatePrice(basePrice, quantity)
        val discount = discountService.getApplicableDiscount(quantity)
        return basicPrice * (BigDecimal.ONE - discount)
    }
}
```

### 2.4.4 Behavioral Patterns

**Strategy Pattern**
```kotlin
// Different strategies for calculating shipping costs
interface ShippingStrategy {
    fun calculateShipping(weight: Double, distance: Double): BigDecimal
}

@Component
class StandardShipping : ShippingStrategy {
    override fun calculateShipping(weight: Double, distance: Double): BigDecimal {
        return BigDecimal(weight * 0.5 + distance * 0.1)
    }
}

@Component
class ExpressShipping : ShippingStrategy {
    override fun calculateShipping(weight: Double, distance: Double): BigDecimal {
        return BigDecimal(weight * 1.0 + distance * 0.2 + 10) // Base fee + higher rates
    }
}

@Service
class ShippingService(
    private val strategies: Map<String, ShippingStrategy>
) {
    fun calculateShipping(
        type: String,
        weight: Double,
        distance: Double
    ): BigDecimal {
        val strategy = strategies[type] 
            ?: throw IllegalArgumentException("Unknown shipping type: $type")
        return strategy.calculateShipping(weight, distance)
    }
}
```

**Observer Pattern**
```kotlin
// Spring's event system implements the observer pattern
data class OrderCompletedEvent(
    val orderId: Long,
    val customerId: Long,
    val totalAmount: BigDecimal
)

@Component
class OrderService(
    private val eventPublisher: ApplicationEventPublisher
) {
    fun completeOrder(order: Order) {
        // Process order...
        
        // Publish event for interested listeners
        eventPublisher.publishEvent(
            OrderCompletedEvent(order.id, order.customerId, order.totalAmount)
        )
    }
}

@Component
class EmailNotificationListener {
    @EventListener
    fun handleOrderCompleted(event: OrderCompletedEvent) {
        println("Sending email for order ${event.orderId}")
    }
}

@Component
class InventoryUpdateListener {
    @EventListener
    @Async
    fun handleOrderCompleted(event: OrderCompletedEvent) {
        println("Updating inventory for order ${event.orderId}")
    }
}
```

## 2.5 REST API

REST (Representational State Transfer) is the architectural style that powers most modern web APIs. Understanding REST principles is essential for building APIs that are intuitive, scalable, and maintainable.

### 2.5.1 What is REST?

REST is an architectural style that treats resources as the primary abstraction. Each resource is identified by a URI, and we interact with resources using standard HTTP methods.

### 2.5.2 What is a REST API?

A REST API is an application programming interface that follows REST principles. It provides a uniform interface for clients to interact with your application's resources.

```kotlin
// A RESTful resource controller
@RestController
@RequestMapping("/api/v1/books")
class BookController(
    private val bookService: BookService
) {
    // GET /api/v1/books - Retrieve all books
    @GetMapping
    fun getAllBooks(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): Page<BookDTO> {
        return bookService.findAll(PageRequest.of(page, size))
    }
    
    // GET /api/v1/books/{id} - Retrieve a specific book
    @GetMapping("/{id}")
    fun getBook(@PathVariable id: Long): BookDTO {
        return bookService.findById(id)
            ?: throw ResponseStatusException(HttpStatus.NOT_FOUND, "Book not found")
    }
    
    // POST /api/v1/books - Create a new book
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun createBook(@RequestBody @Valid book: CreateBookRequest): BookDTO {
        return bookService.create(book)
    }
    
    // PUT /api/v1/books/{id} - Update an entire book
    @PutMapping("/{id}")
    fun updateBook(
        @PathVariable id: Long,
        @RequestBody @Valid book: UpdateBookRequest
    ): BookDTO {
        return bookService.update(id, book)
    }
    
    // PATCH /api/v1/books/{id} - Partially update a book
    @PatchMapping("/{id}")
    fun patchBook(
        @PathVariable id: Long,
        @RequestBody updates: Map<String, Any>
    ): BookDTO {
        return bookService.partialUpdate(id, updates)
    }
    
    // DELETE /api/v1/books/{id} - Delete a book
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun deleteBook(@PathVariable id: Long) {
        bookService.delete(id)
    }
}
```

### 2.5.3 Characteristics of REST

**1. Client-Server Architecture**
The client and server are independent. The client doesn't need to know how data is stored, and the server doesn't care about the user interface.

**2. Statelessness**
Each request contains all information needed to process it. The server doesn't maintain client state between requests.

```kotlin
// Good: Stateless authentication
@GetMapping("/api/orders")
fun getOrders(@RequestHeader("Authorization") token: String): List<Order> {
    val userId = tokenService.validateAndGetUserId(token)
    return orderService.findByUserId(userId)
}

// Bad: Relying on server-side session
@GetMapping("/api/orders")
fun getOrders(session: HttpSession): List<Order> {
    val userId = session.getAttribute("userId") as Long // Avoid this
    return orderService.findByUserId(userId)
}
```

**3. Cacheability**
Responses should indicate whether they can be cached.

```kotlin
@GetMapping("/api/products/{id}")
fun getProduct(@PathVariable id: Long): ResponseEntity<ProductDTO> {
    val product = productService.findById(id)
    
    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS))
        .eTag(product.version.toString())
        .body(product)
}
```

**4. Uniform Interface**
Use standard HTTP methods consistently:
- GET: Retrieve resources (safe, idempotent)
- POST: Create new resources
- PUT: Update entire resources (idempotent)
- PATCH: Partial updates
- DELETE: Remove resources (idempotent)

**5. Layered System**
The architecture can have multiple layers (caches, proxies, gateways) without the client knowing.

**6. HATEOAS (Hypermedia as the Engine of Application State)**
Responses include links to related resources:

```kotlin
data class BookResource(
    val id: Long,
    val title: String,
    val author: String,
    val links: List<Link>
) {
    companion object {
        fun from(book: Book): BookResource {
            return BookResource(
                id = book.id,
                title = book.title,
                author = book.author,
                links = listOf(
                    Link("self", "/api/books/${book.id}"),
                    Link("author", "/api/authors/${book.authorId}"),
                    Link("reviews", "/api/books/${book.id}/reviews")
                )
            )
        }
    }
}

data class Link(val rel: String, val href: String)
```

### 2.5.4 REST URI Design Rules

Good URI design makes your API intuitive and predictable.

**Use Nouns, Not Verbs**
```kotlin
// Good
GET /api/users
GET /api/users/123
POST /api/users

// Bad
GET /api/getUsers
GET /api/getUserById?id=123
POST /api/createUser
```

**Use Plural Names for Collections**
```kotlin
// Good
GET /api/products      // Collection
GET /api/products/1    // Single resource

// Confusing
GET /api/product       // Is this one product or all?
```

**Use Hierarchical Structure for Relationships**
```kotlin
// Clear parent-child relationship
GET /api/users/123/orders       // All orders for user 123
GET /api/users/123/orders/456   // Specific order 456 for user 123

@GetMapping("/users/{userId}/orders")
fun getUserOrders(@PathVariable userId: Long): List<Order> {
    return orderService.findByUserId(userId)
}

@GetMapping("/users/{userId}/orders/{orderId}")
fun getUserOrder(
    @PathVariable userId: Long,
    @PathVariable orderId: Long
): Order {
    return orderService.findByUserIdAndOrderId(userId, orderId)
        ?: throw ResponseStatusException(HttpStatus.NOT_FOUND)
}
```

**Use Query Parameters for Filtering, Sorting, and Pagination**
```kotlin
@GetMapping("/api/products")
fun getProducts(
    @RequestParam(required = false) category: String?,
    @RequestParam(defaultValue = "name") sortBy: String,
    @RequestParam(defaultValue = "ASC") direction: Sort.Direction,
    @RequestParam(defaultValue = "0") page: Int,
    @RequestParam(defaultValue = "20") size: Int
): Page<Product> {
    val pageable = PageRequest.of(page, size, Sort.by(direction, sortBy))
    
    return if (category != null) {
        productRepository.findByCategory(category, pageable)
    } else {
        productRepository.findAll(pageable)
    }
}

// Usage: GET /api/products?category=electronics&sortBy=price&direction=DESC&page=0&size=10
```

**Version Your API**
```kotlin
// URL versioning
@RequestMapping("/api/v1/products")
class ProductControllerV1

@RequestMapping("/api/v2/products")
class ProductControllerV2

// Header versioning
@GetMapping("/api/products", headers = ["API-Version=1"])
fun getProductsV1(): List<ProductV1>

@GetMapping("/api/products", headers = ["API-Version=2"])
fun getProductsV2(): List<ProductV2>
```

## 2.6 Kotlin Basics for Spring Boot Developers

Kotlin brings powerful features that make Spring Boot development more enjoyable and less error-prone. Let's explore the key features you'll use daily.

### 2.6.1 Data Classes

Data classes eliminate boilerplate for POJOs and DTOs:

```kotlin
// Kotlin data class - concise and feature-rich
data class UserDTO(
    val id: Long,
    val username: String,
    val email: String,
    val role: UserRole = UserRole.USER,  // Default value
    val createdAt: Instant = Instant.now()
)

// Automatically provides:
// - equals() and hashCode()
// - toString()
// - copy() function
// - componentN() functions for destructuring

fun example() {
    val user = UserDTO(1, "john", "john@example.com")
    
    // Copy with modifications
    val updatedUser = user.copy(email = "newemail@example.com")
    
    // Destructuring
    val (id, username, email) = user
    
    println(user) // UserDTO(id=1, username=john, email=john@example.com, role=USER, createdAt=...)
}
```

### 2.6.2 Null Safety

Kotlin's type system prevents null pointer exceptions at compile time:

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository
) {
    // Return type explicitly allows null
    fun findUser(id: Long): User? {
        return userRepository.findById(id).orElse(null)
    }
    
    // Safe navigation
    fun getUserEmail(id: Long): String {
        val user = findUser(id)
        return user?.email ?: "no-email@example.com"  // Elvis operator provides default
    }
    
    // Force non-null with !! (use sparingly!)
    fun getUserOrThrow(id: Long): User {
        return findUser(id) ?: throw UserNotFoundException("User $id not found")
    }
    
    // Smart casting
    fun processUser(id: Long) {
        val user = findUser(id)
        if (user != null) {
            // user is automatically cast to non-null User here
            println(user.email)  // No safe navigation needed
        }
    }
    
    // let scope function for null-safe operations
    fun sendEmailIfExists(id: Long) {
        findUser(id)?.let { user ->
            emailService.send(user.email, "Welcome!")
        }
    }
}
```

### 2.6.3 Extension Functions

Extension functions let you add functionality to existing classes:

```kotlin
// Add functions to Spring's ResponseEntity
fun <T> T.toOkResponse(): ResponseEntity<T> = ResponseEntity.ok(this)

fun <T> T?.toResponseEntity(): ResponseEntity<T> {
    return this?.let { ResponseEntity.ok(it) }
        ?: ResponseEntity.notFound().build()
}

// Extension for validation
fun String.isValidEmail(): Boolean {
    return this.matches(Regex("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"))
}

// Extensions for collections
fun <T> List<T>.secondOrNull(): T? = if (size >= 2) this[1] else null

// Usage in controller
@GetMapping("/{id}")
fun getProduct(@PathVariable id: Long): ResponseEntity<ProductDTO> {
    return productService.findById(id)
        ?.toDTO()
        .toResponseEntity()  // Clean and expressive
}
```

### 2.6.4 Top-level Functions

Functions don't need to be in classes:

```kotlin
// utils/ValidationUtils.kt
package com.example.utils

// Top-level functions - no class needed
fun validatePassword(password: String): ValidationResult {
    return when {
        password.length < 8 -> ValidationResult.Error("Password too short")
        !password.any { it.isDigit() } -> ValidationResult.Error("Password must contain a digit")
        !password.any { it.isUpperCase() } -> ValidationResult.Error("Password must contain uppercase")
        else -> ValidationResult.Success
    }
}

sealed class ValidationResult {
    object Success : ValidationResult()
    data class Error(val message: String) : ValidationResult()
}

// Use directly without class reference
@Service
class AuthService {
    fun register(username: String, password: String): User {
        when (val result = validatePassword(password)) {
            is ValidationResult.Error -> throw ValidationException(result.message)
            is ValidationResult.Success -> {
                // Proceed with registration
            }
        }
    }
}
```

### 2.6.5 Kotlin vs Java Syntax (for common Spring use cases)

Let's compare common Spring patterns:

**Dependency Injection**
```kotlin
// Kotlin - Constructor injection is concise
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val emailService: EmailService
) {
    // Properties are automatically initialized
}

// Java equivalent would require:
// - Constructor with assignments
// - Or field injection with @Autowired
// - Or setter injection
```

**Configuration Properties**
```kotlin
// Kotlin - Use data class for type-safe configuration
@ConfigurationProperties("app.email")
data class EmailProperties(
    val host: String,
    val port: Int,
    val username: String,
    val password: String,
    val timeout: Duration = Duration.ofSeconds(30)
)

// Usage
@Configuration
@EnableConfigurationProperties(EmailProperties::class)
class EmailConfig(
    private val emailProperties: EmailProperties
) {
    @Bean
    fun emailClient(): EmailClient {
        return EmailClient(
            host = emailProperties.host,
            port = emailProperties.port
        )
    }
}
```

**Collections and Streams**
```kotlin
// Kotlin collections are more intuitive than Java streams
@Service
class ProductAnalytics(
    private val productRepository: ProductRepository
) {
    fun getTopProducts(limit: Int): List<ProductSummary> {
        return productRepository.findAll()
            .filter { it.rating >= 4.0 }
            .sortedByDescending { it.soldCount }
            .take(limit)
            .map { ProductSummary(it.id, it.name, it.soldCount) }
    }
    
    fun getCategoryStats(): Map<String, Int> {
        return productRepository.findAll()
            .groupingBy { it.category }
            .eachCount()
    }
}
```

**Sealed Classes for Result Types**
```kotlin
// Better than exceptions for expected errors
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val message: String, val code: String) : ApiResult<Nothing>()
    object Loading : ApiResult<Nothing>()
}

@Service
class ApiService {
    suspend fun fetchData(id: Long): ApiResult<Data> {
        return try {
            val data = webClient.get()
                .uri("/data/{id}", id)
                .awaitBody<Data>()
            ApiResult.Success(data)
        } catch (e: WebClientResponseException) {
            ApiResult.Error(e.message ?: "Unknown error", e.statusCode.toString())
        }
    }
}
```

**Coroutines for Async Operations**
```kotlin
@RestController
class AsyncController(
    private val service: AsyncService
) {
    // Suspend function for async endpoint
    @GetMapping("/async-data")
    suspend fun getAsyncData(): List<Data> {
        return coroutineScope {
            val data1 = async { service.fetchData1() }
            val data2 = async { service.fetchData2() }
            val data3 = async { service.fetchData3() }
            
            // Wait for all and combine
            listOf(data1.await(), data2.await(), data3.await()).flatten()
        }
    }
}
```

## Summary

This chapter has laid the groundwork for effective Spring Boot development with Kotlin. We've covered:

- **Server-to-Server Communication**: Understanding synchronous and asynchronous patterns, HTTP-based REST, and message-based communication
- **Spring Boot Internals**: How the startup process works, component scanning, and conditional configuration
- **Layered Architecture**: Organizing code into presentation, business, and data access layers for maintainability
- **Design Patterns**: Essential creational, structural, and behavioral patterns you'll use in Spring applications
- **REST Principles**: Building intuitive, scalable APIs following REST conventions and best practices
- **Kotlin Features**: Leveraging data classes, null safety, extension functions, and other Kotlin features to write cleaner Spring Boot code

With this foundation, you're ready to set up your development environment and start building Spring Boot applications. In the next chapter, we'll get your development environment configured and ready for Kotlin and Spring Boot development.