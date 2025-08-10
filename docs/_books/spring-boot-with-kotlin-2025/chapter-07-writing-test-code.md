# Chapter 07: Writing Test Code

Testing is crucial for building reliable Spring Boot applications. In this chapter, we'll explore comprehensive testing strategies using modern Kotlin testing frameworks. You'll learn about unit testing, integration testing, test patterns, and how to use Kotest, MockK, and other tools to create maintainable test suites. We'll also cover test-driven development practices and code coverage analysis.

## 7.1 Why Testing Matters

Testing is not just about catching bugs—it's about designing better software, documenting behavior, and enabling confident refactoring. Let's explore why testing is essential for Spring Boot applications.

### 7.1.1 The Value of Testing

```kotlin
// Example: What happens without tests?
class ProductService(private val repository: ProductRepository) {
    
    fun calculateDiscount(product: Product, customerType: CustomerType): Double {
        return when (customerType) {
            CustomerType.PREMIUM -> product.price * 0.15  // 15% discount
            CustomerType.REGULAR -> product.price * 0.05  // 5% discount
            CustomerType.NEW -> 0.0  // No discount... or is it?
        }
    }
}

// Without tests, we might miss:
// - What if product.price is negative?
// - What if we add a new CustomerType?
// - What if the business rules change?
// - What happens with null values?
```

**With proper testing, we ensure:**
- **Correctness**: Code behaves as expected
- **Regression Prevention**: Changes don't break existing functionality
- **Design Quality**: Testable code is usually better designed
- **Documentation**: Tests serve as executable specifications
- **Confidence**: Safe refactoring and feature additions

### 7.1.2 Testing Pyramid for Spring Boot

```
              /\
             /  \
            / UI \
           /Tests \
          /________\
         /          \
        /Integration \
       /    Tests     \
      /________________\
     /                  \
    /    Unit Tests      \
   /____________________\
```

**Unit Tests (70%)**:
- Fast execution (< 1 second)
- Test individual components in isolation
- Use mocking for dependencies
- Focus on business logic

**Integration Tests (20%)**:
- Test component interactions
- Include database, web layer, or external services
- Slower execution but higher confidence
- Test real scenarios

**UI/End-to-End Tests (10%)**:
- Test complete user journeys
- Slowest but most comprehensive
- Usually automated browser testing

### 7.1.3 Testing Philosophy

```kotlin
// Good test characteristics: F.I.R.S.T
// Fast - Tests should run quickly
// Independent - Tests should not depend on each other
// Repeatable - Same result every time
// Self-Validating - Clear pass/fail result
// Timely - Written before or with the production code

class ProductServiceTest {
    
    @Test
    fun `should calculate premium discount correctly`() {
        // Given (Arrange)
        val product = Product(name = "Laptop", price = 1000.0)
        val customerType = CustomerType.PREMIUM
        val service = ProductService(mockRepository)
        
        // When (Act)
        val discount = service.calculateDiscount(product, customerType)
        
        // Then (Assert)
        discount shouldBe 150.0
    }
}
```

## 7.2 Unit vs Integration Testing

Understanding when to use unit tests versus integration tests is crucial for building an effective test suite.

### 7.2.1 Unit Testing Principles

Unit tests focus on testing individual components in isolation:

```kotlin
// src/test/kotlin/com/example/api/service/ProductServiceUnitTest.kt
package com.example.api.service

import com.example.api.dto.CreateProductRequest
import com.example.api.entity.Category
import com.example.api.entity.Product
import com.example.api.exception.CategoryNotFoundException
import com.example.api.exception.ProductNotFoundException
import com.example.api.repository.CategoryRepository
import com.example.api.repository.ProductRepository
import com.example.api.service.mapper.ProductMapper
import io.kotest.assertions.throwables.shouldThrow
import io.kotest.core.spec.style.BehaviorSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import io.mockk.*
import org.springframework.data.repository.findByIdOrNull

class ProductServiceUnitTest : BehaviorSpec({
    
    // Mocked dependencies
    val productRepository = mockk<ProductRepository>()
    val categoryRepository = mockk<CategoryRepository>()
    val productMapper = mockk<ProductMapper>()
    
    // System under test
    val productService = ProductService(productRepository, categoryRepository, productMapper)
    
    beforeEach {
        clearAllMocks()
    }
    
    Given("a product service") {
        
        When("finding a product by valid ID") {
            val productId = 1L
            val mockProduct = mockk<Product>()
            val mockProductDto = mockk<ProductDTO>()
            
            every { productRepository.findByIdOrNull(productId) } returns mockProduct
            every { productMapper.toDto(mockProduct) } returns mockProductDto
            
            Then("should return the product DTO") {
                val result = productService.findById(productId)
                result shouldBe mockProductDto
                
                verify(exactly = 1) { productRepository.findByIdOrNull(productId) }
                verify(exactly = 1) { productMapper.toDto(mockProduct) }
            }
        }
        
        When("finding a product by invalid ID") {
            val productId = 999L
            
            every { productRepository.findByIdOrNull(productId) } returns null
            
            Then("should throw ProductNotFoundException") {
                shouldThrow<ProductNotFoundException> {
                    productService.findById(productId)
                }
                
                verify(exactly = 1) { productRepository.findByIdOrNull(productId) }
                verify(exactly = 0) { productMapper.toDto(any()) }
            }
        }
        
        When("creating a new product") {
            val request = CreateProductRequest(
                name = "Test Product",
                description = "A test product",
                price = 99.99,
                brand = "TestBrand",
                stockQuantity = 10,
                categoryId = 1L,
                tags = listOf("test")
            )
            
            val mockCategory = mockk<Category>()
            val mockProduct = mockk<Product>()
            val mockSavedProduct = mockk<Product>()
            val mockProductDto = mockk<ProductDTO>()
            
            every { categoryRepository.findByIdOrNull(request.categoryId) } returns mockCategory
            every { productRepository.existsByNameAndBrand(request.name, request.brand) } returns false
            every { Product.create(any(), any(), any(), any(), any(), any(), any()) } returns mockProduct
            every { productRepository.save(mockProduct) } returns mockSavedProduct
            every { productMapper.toDto(mockSavedProduct) } returns mockProductDto
            
            Then("should create and return the product") {
                val result = productService.create(request)
                result shouldBe mockProductDto
                
                verify(exactly = 1) { categoryRepository.findByIdOrNull(request.categoryId) }
                verify(exactly = 1) { productRepository.existsByNameAndBrand(request.name, request.brand) }
                verify(exactly = 1) { productRepository.save(mockProduct) }
            }
        }
        
        When("creating a product with invalid category") {
            val request = CreateProductRequest(
                name = "Test Product",
                description = "A test product", 
                price = 99.99,
                brand = "TestBrand",
                stockQuantity = 10,
                categoryId = 999L,
                tags = emptyList()
            )
            
            every { categoryRepository.findByIdOrNull(request.categoryId) } returns null
            
            Then("should throw CategoryNotFoundException") {
                shouldThrow<CategoryNotFoundException> {
                    productService.create(request)
                }
                
                verify(exactly = 1) { categoryRepository.findByIdOrNull(request.categoryId) }
                verify(exactly = 0) { productRepository.save(any()) }
            }
        }
        
        When("creating a duplicate product") {
            val request = CreateProductRequest(
                name = "Duplicate Product",
                description = "A duplicate product",
                price = 99.99,
                brand = "TestBrand",
                stockQuantity = 10,
                categoryId = 1L,
                tags = emptyList()
            )
            
            val mockCategory = mockk<Category>()
            
            every { categoryRepository.findByIdOrNull(request.categoryId) } returns mockCategory
            every { productRepository.existsByNameAndBrand(request.name, request.brand) } returns true
            
            Then("should throw IllegalArgumentException") {
                shouldThrow<IllegalArgumentException> {
                    productService.create(request)
                }
                
                verify(exactly = 1) { categoryRepository.findByIdOrNull(request.categoryId) }
                verify(exactly = 1) { productRepository.existsByNameAndBrand(request.name, request.brand) }
                verify(exactly = 0) { productRepository.save(any()) }
            }
        }
    }
})
```

### 7.2.2 Integration Testing Principles

Integration tests verify that multiple components work together correctly:

```kotlin
// src/test/kotlin/com/example/api/service/ProductServiceIntegrationTest.kt
package com.example.api.service

import com.example.api.dto.CreateProductRequest
import com.example.api.entity.Category
import com.example.api.repository.CategoryRepository
import com.example.api.repository.ProductRepository
import io.kotest.core.spec.style.BehaviorSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.TestPropertySource
import org.springframework.transaction.annotation.Transactional

@SpringBootTest
@TestPropertySource(
    properties = [
        "spring.datasource.url=jdbc:h2:mem:testdb",
        "spring.jpa.hibernate.ddl-auto=create-drop"
    ]
)
@Transactional
class ProductServiceIntegrationTest(
    private val productService: ProductService,
    private val categoryRepository: CategoryRepository,
    private val productRepository: ProductRepository
) : BehaviorSpec({
    
    Given("a product service with real database") {
        
        When("creating and retrieving a product") {
            val category = Category.create("Electronics", "Electronic devices")
            val savedCategory = categoryRepository.save(category)
            
            val request = CreateProductRequest(
                name = "Integration Test Product",
                description = "A product for integration testing",
                price = 149.99,
                brand = "TestBrand",
                stockQuantity = 25,
                categoryId = savedCategory.id,
                tags = listOf("integration", "test")
            )
            
            Then("should persist the product correctly") {
                val createdProduct = productService.create(request)
                
                createdProduct.id shouldNotBe 0L
                createdProduct.name shouldBe request.name
                createdProduct.price shouldBe request.price
                createdProduct.category?.id shouldBe savedCategory.id
                createdProduct.tags shouldBe request.tags
                
                // Verify it was actually saved to database
                val retrievedProduct = productService.findById(createdProduct.id)
                retrievedProduct.name shouldBe createdProduct.name
                retrievedProduct.price shouldBe createdProduct.price
            }
        }
        
        When("updating product stock") {
            val category = Category.create("Books", "Book category")
            val savedCategory = categoryRepository.save(category)
            
            val request = CreateProductRequest(
                name = "Stock Test Product",
                description = "Testing stock updates",
                price = 29.99,
                brand = "BookBrand",
                stockQuantity = 100,
                categoryId = savedCategory.id,
                tags = emptyList()
            )
            
            val createdProduct = productService.create(request)
            val newStockQuantity = 150
            
            Then("should update stock correctly") {
                val updatedProduct = productService.updateStock(createdProduct.id, newStockQuantity)
                
                updatedProduct.stockQuantity shouldBe newStockQuantity
                
                // Verify persistence
                val retrievedProduct = productService.findById(createdProduct.id)
                retrievedProduct.stockQuantity shouldBe newStockQuantity
            }
        }
        
        When("searching products with complex criteria") {
            // Create test data
            val electronics = categoryRepository.save(Category.create("Electronics", "Electronic devices"))
            val books = categoryRepository.save(Category.create("Books", "Book category"))
            
            // Create products with varying prices and categories
            listOf(
                CreateProductRequest("Laptop", "Gaming laptop", 1299.99, "TechBrand", 5, electronics.id, listOf("gaming", "premium")),
                CreateProductRequest("Mouse", "Gaming mouse", 79.99, "TechBrand", 20, electronics.id, listOf("gaming", "accessory")),
                CreateProductRequest("Keyboard", "Mechanical keyboard", 159.99, "TechBrand", 15, electronics.id, listOf("gaming", "mechanical")),
                CreateProductRequest("Novel", "Fiction novel", 19.99, "BookBrand", 50, books.id, listOf("fiction", "bestseller"))
            ).forEach { request ->
                productService.create(request)
            }
            
            Then("should filter by price range correctly") {
                val criteria = ProductSearchCriteriaDTO(
                    minPrice = 50.0,
                    maxPrice = 200.0,
                    page = 0,
                    size = 10
                )
                
                val results = productService.search(criteria)
                
                results.content.size shouldBe 2 // Mouse and Keyboard
                results.content.all { it.price in 50.0..200.0 } shouldBe true
            }
            
            Then("should filter by category correctly") {
                val criteria = ProductSearchCriteriaDTO(
                    categoryIds = listOf(electronics.id),
                    page = 0,
                    size = 10
                )
                
                val results = productService.search(criteria)
                
                results.content.size shouldBe 3 // All electronics
                results.content.all { it.category?.id == electronics.id } shouldBe true
            }
            
            Then("should filter by tags correctly") {
                val criteria = ProductSearchCriteriaDTO(
                    tags = listOf("gaming"),
                    page = 0,
                    size = 10
                )
                
                val results = productService.search(criteria)
                
                results.content.size shouldBe 3 // All gaming products
                results.content.all { it.tags.contains("gaming") } shouldBe true
            }
        }
    }
})
```

### 7.2.3 When to Use Each Type

**Use Unit Tests When:**
- Testing business logic in isolation
- Validating edge cases and error conditions
- Testing pure functions or methods with clear inputs/outputs
- Need fast feedback during development

**Use Integration Tests When:**
- Testing database interactions
- Validating component interactions
- Testing configuration and wiring
- Verifying end-to-end scenarios

## 7.3 Test Code Patterns and Quality

Writing high-quality tests requires following established patterns and principles.

### 7.3.1 Given-When-Then (GWT) Pattern

The GWT pattern structures tests for clarity and consistency:

```kotlin
class ProductDiscountCalculatorTest : BehaviorSpec({
    
    val calculator = ProductDiscountCalculator()
    
    Given("a premium customer") {
        val customer = Customer(type = CustomerType.PREMIUM, loyaltyYears = 3)
        
        When("purchasing a product worth $100") {
            val product = Product(price = 100.0, category = "Electronics")
            
            Then("should receive 15% base discount plus loyalty bonus") {
                val discount = calculator.calculateDiscount(customer, product)
                
                // Base premium discount (15%) + loyalty bonus (3%)
                discount shouldBe 18.0
            }
        }
        
        When("purchasing a discounted product") {
            val product = Product(price = 100.0, category = "Electronics", onSale = true, saleDiscount = 10.0)
            
            Then("should receive the better of the two discounts") {
                val discount = calculator.calculateDiscount(customer, product)
                
                // Should choose premium discount (18%) over sale discount (10%)
                discount shouldBe 18.0
            }
        }
    }
    
    Given("a new customer") {
        val customer = Customer(type = CustomerType.NEW, loyaltyYears = 0)
        
        When("purchasing their first product") {
            val product = Product(price = 50.0, category = "Books")
            
            Then("should receive welcome discount") {
                val discount = calculator.calculateDiscount(customer, product)
                
                discount shouldBe 5.0 // Welcome discount of 10%
            }
        }
    }
})
```

### 7.3.2 FIRST Principles in Practice

**Fast Tests:**
```kotlin
class FastUnitTest : StringSpec({
    
    "should validate email format quickly" {
        val validator = EmailValidator()
        
        // These should complete in milliseconds
        validator.isValid("user@example.com") shouldBe true
        validator.isValid("invalid.email") shouldBe false
        validator.isValid("") shouldBe false
        validator.isValid("user@") shouldBe false
    }
    
    "should calculate tax efficiently" {
        val calculator = TaxCalculator()
        
        measureTime {
            repeat(1000) {
                calculator.calculateTax(Random.nextDouble(1.0, 1000.0))
            }
        }.inWholeMilliseconds shouldBeLessThan 100 // Should be very fast
    }
})
```

**Independent Tests:**
```kotlin
class IndependentTests : StringSpec({
    
    // Each test creates its own data - no shared state
    "test 1 should not affect test 2" {
        val service = createServiceWithFreshState()
        service.addItem("test1")
        service.getItems() shouldHaveSize 1
    }
    
    "test 2 should start fresh" {
        val service = createServiceWithFreshState()
        service.getItems() shouldHaveSize 0 // Starts empty, regardless of test 1
    }
    
    private fun createServiceWithFreshState(): ItemService {
        return ItemService(InMemoryRepository())
    }
})
```

**Repeatable Tests:**
```kotlin
class RepeatableTests : BehaviorSpec({
    
    Given("a date-based service") {
        
        When("calculating age") {
            val fixedDate = LocalDate.of(2023, 1, 1)
            val birthDate = LocalDate.of(1990, 1, 1)
            
            // Use dependency injection or mocking for current date
            val service = AgeCalculatorService(FixedClock(fixedDate))
            
            Then("should always return the same age") {
                repeat(10) {
                    service.calculateAge(birthDate) shouldBe 33
                }
            }
        }
    }
    
    Given("a random number generator with fixed seed") {
        val random = Random(42) // Fixed seed for repeatability
        val service = LotteryService(random)
        
        When("generating lottery numbers") {
            Then("should generate the same sequence every time") {
                val firstRun = service.generateNumbers()
                
                // Reset with same seed
                val service2 = LotteryService(Random(42))
                val secondRun = service2.generateNumbers()
                
                firstRun shouldBe secondRun
            }
        }
    }
})
```

**Self-Validating Tests:**
```kotlin
class SelfValidatingTests : StringSpec({
    
    "should clearly indicate success or failure" {
        val calculator = Calculator()
        
        // Good: Clear assertions
        calculator.add(2, 3) shouldBe 5
        calculator.divide(10, 2) shouldBe 5.0
        
        // Avoid: Manual verification required
        // println("Result: ${calculator.add(2, 3)}") // Don't do this
    }
    
    "should provide meaningful error messages" {
        val validator = PasswordValidator()
        
        // Good: Descriptive assertion
        validator.validate("weak") shouldBe ValidationResult.Invalid("Password must be at least 8 characters")
        
        // Better: Use custom matchers for domain-specific assertions
        "strongPassword123!" should beValidPassword()
        "weak" shouldNot beValidPassword()
    }
})

// Custom matcher for domain-specific assertions
fun beValidPassword() = object : Matcher<String> {
    override fun test(value: String): MatcherResult {
        val validator = PasswordValidator()
        val result = validator.validate(value)
        return MatcherResult(
            result is ValidationResult.Valid,
            { "Expected '$value' to be a valid password, but got: $result" },
            { "Expected '$value' to be an invalid password" }
        )
    }
}
```

### 7.3.3 Test Data Builders Pattern

Create reusable test data builders for complex objects:

```kotlin
// Test data builders for maintainable test data
class ProductBuilder {
    private var id: Long = 1L
    private var name: String = "Default Product"
    private var price: Double = 100.0
    private var brand: String = "Default Brand"
    private var category: Category? = null
    private var stockQuantity: Int = 10
    private var tags: MutableSet<String> = mutableSetOf()
    
    fun withId(id: Long) = apply { this.id = id }
    fun withName(name: String) = apply { this.name = name }
    fun withPrice(price: Double) = apply { this.price = price }
    fun withBrand(brand: String) = apply { this.brand = brand }
    fun withCategory(category: Category) = apply { this.category = category }
    fun withStockQuantity(quantity: Int) = apply { this.stockQuantity = quantity }
    fun withTags(vararg tags: String) = apply { this.tags.addAll(tags) }
    fun withNoStock() = apply { this.stockQuantity = 0 }
    fun withHighPrice() = apply { this.price = 999.99 }
    fun withLowPrice() = apply { this.price = 9.99 }
    
    fun build(): Product {
        val product = Product.create(
            name = name,
            description = "Test product description",
            price = price,
            brand = brand,
            stockQuantity = stockQuantity,
            category = category ?: CategoryBuilder().build(),
            tags = tags
        )
        // Use reflection or test helper to set ID if needed
        return product
    }
}

class CategoryBuilder {
    private var id: Long = 1L
    private var name: String = "Default Category"
    private var description: String = "Default description"
    
    fun withId(id: Long) = apply { this.id = id }
    fun withName(name: String) = apply { this.name = name }
    fun withDescription(description: String) = apply { this.description = description }
    
    fun build(): Category {
        return Category.create(name, description)
    }
}

// Usage in tests
class ProductServiceTest : BehaviorSpec({
    
    Given("various product scenarios") {
        val productRepository = mockk<ProductRepository>()
        val categoryRepository = mockk<CategoryRepository>()
        val service = ProductService(productRepository, categoryRepository, mockk())
        
        When("calculating discount for expensive electronics") {
            val expensiveElectronics = ProductBuilder()
                .withName("Gaming Laptop")
                .withCategory(CategoryBuilder().withName("Electronics").build())
                .withHighPrice()
                .withTags("gaming", "premium")
                .build()
            
            Then("should apply premium discount") {
                // Test logic here
                val discount = service.calculateDiscount(expensiveElectronics)
                discount shouldBeGreaterThan 100.0
            }
        }
        
        When("processing out of stock products") {
            val outOfStockProduct = ProductBuilder()
                .withName("Sold Out Item")
                .withNoStock()
                .build()
            
            Then("should handle unavailable products") {
                // Test logic here
                service.isAvailable(outOfStockProduct) shouldBe false
            }
        }
    }
})
```

### 7.3.4 Test Fixtures and Setup

Organize test data and setup efficiently:

```kotlin
// Shared test fixtures
object TestFixtures {
    
    fun createTestCategory(
        name: String = "Test Category",
        description: String = "A test category"
    ): Category = Category.create(name, description)
    
    fun createTestProduct(
        name: String = "Test Product",
        price: Double = 99.99,
        category: Category = createTestCategory()
    ): Product = Product.create(
        name = name,
        description = "A test product",
        price = price,
        brand = "TestBrand",
        stockQuantity = 10,
        category = category
    )
    
    fun createPremiumCustomer(): Customer = Customer(
        type = CustomerType.PREMIUM,
        loyaltyYears = 5,
        email = "premium@example.com"
    )
    
    fun createTestProducts(count: Int): List<Product> = 
        (1..count).map { createTestProduct("Product $it", 50.0 + it * 10) }
}

// Base test class for common setup
abstract class ServiceTestBase : BehaviorSpec() {
    
    protected val productRepository = mockk<ProductRepository>()
    protected val categoryRepository = mockk<CategoryRepository>()
    protected val productMapper = mockk<ProductMapper>()
    
    init {
        beforeEach {
            clearAllMocks()
        }
        
        afterEach {
            confirmVerified(productRepository, categoryRepository, productMapper)
        }
    }
    
    protected fun mockProductExists(product: Product) {
        every { productRepository.findByIdOrNull(product.id) } returns product
    }
    
    protected fun mockProductNotFound(id: Long) {
        every { productRepository.findByIdOrNull(id) } returns null
    }
}

// Concrete test class
class ProductServiceSpecificTest : ServiceTestBase() {
    
    private val productService = ProductService(productRepository, categoryRepository, productMapper)
    
    init {
        Given("a product service") {
            When("finding existing product") {
                val product = TestFixtures.createTestProduct()
                mockProductExists(product)
                
                Then("should return product") {
                    // Test implementation
                }
            }
        }
    }
}
```

## 7.4 Using Kotest (Submodules, Spring Boot Configuration, Testing Lifecycle, Controller/Service/Repository Testing, Using mockk)

Kotest is a powerful testing framework for Kotlin that provides excellent Spring Boot integration and expressive test syntax.

### 7.4.1 Kotest Setup and Configuration

Add Kotest dependencies to your `build.gradle.kts`:

```kotlin
dependencies {
    testImplementation("io.kotest:kotest-runner-junit5:5.7.2")
    testImplementation("io.kotest:kotest-assertions-core:5.7.2")
    testImplementation("io.kotest:kotest-property:5.7.2")
    testImplementation("io.kotest:kotest-extensions-spring:1.1.3")
    testImplementation("io.mockk:mockk:1.13.7")
    testImplementation("com.ninja-squad:springmockk:4.0.2")
    
    // Spring Boot test dependencies
    testImplementation("org.springframework.boot:spring-boot-starter-test") {
        exclude(group = "org.mockito", module = "mockito-core")
    }
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
    testImplementation("org.testcontainers:postgresql")
    testImplementation("org.testcontainers:junit-jupiter")
}
```

### 7.4.2 Kotest Configuration

```kotlin
// src/test/kotlin/kotest.config.kt
package com.example.api

import io.kotest.core.config.AbstractProjectConfig
import io.kotest.core.extensions.Extension
import io.kotest.extensions.spring.SpringExtension

object ProjectConfig : AbstractProjectConfig() {
    override fun extensions(): List<Extension> = listOf(SpringExtension)
    
    override val parallelism: Int = Runtime.getRuntime().availableProcessors()
    override val testCaseOrder = TestCaseOrder.Random
    override val globalAssertSoftly = true
}
```

### 7.4.3 Repository Testing with Kotest

```kotlin
// src/test/kotlin/com/example/api/repository/ProductRepositoryTest.kt
package com.example.api.repository

import com.example.api.entity.Category
import com.example.api.entity.Product
import io.kotest.core.spec.style.BehaviorSpec
import io.kotest.matchers.collections.shouldHaveSize
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager
import org.springframework.data.domain.PageRequest
import org.springframework.test.context.TestPropertySource

@DataJpaTest
@TestPropertySource(properties = [
    "spring.jpa.hibernate.ddl-auto=create-drop",
    "spring.datasource.url=jdbc:h2:mem:testdb"
])
class ProductRepositoryTest(
    private val productRepository: ProductRepository,
    private val categoryRepository: CategoryRepository,
    private val entityManager: TestEntityManager
) : BehaviorSpec({
    
    Given("a product repository") {
        
        When("saving a new product") {
            val category = Category.create("Electronics", "Electronic devices")
            val savedCategory = categoryRepository.save(category)
            
            val product = Product.create(
                name = "Test Laptop",
                description = "A laptop for testing",
                price = 999.99,
                brand = "TestBrand",
                stockQuantity = 5,
                category = savedCategory,
                tags = setOf("computer", "portable")
            )
            
            Then("should persist the product with correct attributes") {
                val savedProduct = productRepository.save(product)
                
                savedProduct.id shouldNotBe 0L
                savedProduct.name shouldBe "Test Laptop"
                savedProduct.price shouldBe 999.99
                savedProduct.category?.id shouldBe savedCategory.id
                savedProduct.tags shouldHaveSize 2
                savedProduct.active shouldBe true
                savedProduct.createdAt shouldNotBe null
                savedProduct.updatedAt shouldNotBe null
            }
        }
        
        When("finding products by category") {
            // Setup test data
            val electronics = categoryRepository.save(Category.create("Electronics", "Electronic devices"))
            val books = categoryRepository.save(Category.create("Books", "Book category"))
            
            val laptop = productRepository.save(Product.create(
                name = "Laptop", description = "A laptop", price = 1000.0, 
                brand = "TechBrand", stockQuantity = 10, category = electronics
            ))
            
            val book = productRepository.save(Product.create(
                name = "Programming Book", description = "A book about programming", price = 50.0,
                brand = "BookPublisher", stockQuantity = 20, category = books
            ))
            
            entityManager.flush()
            entityManager.clear()
            
            Then("should return only products from specified category") {
                val electronicsProducts = productRepository.findByCategoryIdAndActiveTrue(
                    electronics.id, PageRequest.of(0, 10)
                )
                
                electronicsProducts.content shouldHaveSize 1
                electronicsProducts.content.first().category?.id shouldBe electronics.id
                electronicsProducts.content.first().name shouldBe "Laptop"
            }
        }
        
        When("searching products by price range") {
            // Create products with different prices
            val category = categoryRepository.save(Category.create("Test Category", "For testing"))
            
            val cheapProduct = productRepository.save(Product.create(
                name = "Cheap Product", description = "Affordable item", price = 25.0,
                brand = "BudgetBrand", stockQuantity = 50, category = category
            ))
            
            val expensiveProduct = productRepository.save(Product.create(
                name = "Expensive Product", description = "Premium item", price = 500.0,
                brand = "PremiumBrand", stockQuantity = 5, category = category
            ))
            
            val midRangeProduct = productRepository.save(Product.create(
                name = "Mid Range Product", description = "Balanced item", price = 100.0,
                brand = "StandardBrand", stockQuantity = 15, category = category
            ))
            
            entityManager.flush()
            entityManager.clear()
            
            Then("should find products within price range") {
                val productsInRange = productRepository.findByPriceBetweenAndActiveTrue(50.0, 200.0)
                
                productsInRange shouldHaveSize 1
                productsInRange.first().name shouldBe "Mid Range Product"
            }
        }
        
        When("using custom query methods") {
            val category = categoryRepository.save(Category.create("Gaming", "Gaming products"))
            
            // Create products with low stock
            repeat(3) { index ->
                productRepository.save(Product.create(
                    name = "Low Stock Product $index",
                    description = "Running low on stock",
                    price = 75.0 + index * 25,
                    brand = "TestBrand",
                    stockQuantity = 2 + index, // Stock: 2, 3, 4
                    category = category
                ))
            }
            
            // Create product with high stock
            productRepository.save(Product.create(
                name = "High Stock Product",
                description = "Plenty in stock",
                price = 150.0,
                brand = "TestBrand",
                stockQuantity = 100,
                category = category
            ))
            
            entityManager.flush()
            entityManager.clear()
            
            Then("should find low stock products correctly") {
                val lowStockProducts = productRepository.findLowStockProducts(5)
                
                lowStockProducts shouldHaveSize 3
                lowStockProducts.all { it.stockQuantity < 5 } shouldBe true
                lowStockProducts.all { it.name.contains("Low Stock") } shouldBe true
            }
        }
        
        When("testing product statistics") {
            val electronics = categoryRepository.save(Category.create("Electronics", "Electronic devices"))
            val books = categoryRepository.save(Category.create("Books", "Book category"))
            
            // Electronics products
            productRepository.save(Product.create(
                name = "Laptop", description = "High-end laptop", price = 1500.0,
                brand = "TechBrand", stockQuantity = 10, category = electronics
            ))
            productRepository.save(Product.create(
                name = "Mouse", description = "Gaming mouse", price = 100.0,
                brand = "TechBrand", stockQuantity = 25, category = electronics
            ))
            
            // Books
            productRepository.save(Product.create(
                name = "Programming Guide", description = "Learn to code", price = 45.0,
                brand = "BookPublisher", stockQuantity = 30, category = books
            ))
            
            entityManager.flush()
            entityManager.clear()
            
            Then("should calculate statistics correctly") {
                val statistics = productRepository.getProductStatistics()
                
                statistics shouldHaveSize 2
                
                val electronicsStats = statistics.first { it.categoryName == "Electronics" }
                electronicsStats.productCount shouldBe 2L
                electronicsStats.averagePrice shouldBe 800.0 // (1500 + 100) / 2
                electronicsStats.totalStock shouldBe 35L // 10 + 25
                
                val booksStats = statistics.first { it.categoryName == "Books" }
                booksStats.productCount shouldBe 1L
                booksStats.averagePrice shouldBe 45.0
                booksStats.totalStock shouldBe 30L
            }
        }
    }
})
```

### 7.4.4 Service Layer Testing with Kotest and MockK

```kotlin
// src/test/kotlin/com/example/api/service/ProductServiceKotestTest.kt
package com.example.api.service

import com.example.api.dto.*
import com.example.api.entity.Category
import com.example.api.entity.Product
import com.example.api.exception.CategoryNotFoundException
import com.example.api.exception.ProductNotFoundException
import com.example.api.repository.CategoryRepository
import com.example.api.repository.ProductRepository
import com.example.api.service.mapper.ProductMapper
import io.kotest.assertions.throwables.shouldThrow
import io.kotest.core.spec.style.DescribeSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import io.mockk.*
import org.springframework.data.domain.PageImpl
import org.springframework.data.domain.PageRequest
import org.springframework.data.repository.findByIdOrNull
import java.time.LocalDateTime

class ProductServiceKotestTest : DescribeSpec({
    
    val productRepository = mockk<ProductRepository>()
    val categoryRepository = mockk<CategoryRepository>()
    val productMapper = mockk<ProductMapper>()
    
    val productService = ProductService(productRepository, categoryRepository, productMapper)
    
    beforeEach {
        clearAllMocks()
    }
    
    describe("ProductService.findById") {
        
        context("when product exists") {
            val productId = 1L
            val mockProduct = mockk<Product> {
                every { id } returns productId
                every { name } returns "Test Product"
                every { active } returns true
            }
            val mockProductDto = mockk<ProductDTO> {
                every { id } returns productId
                every { name } returns "Test Product"
            }
            
            beforeEach {
                every { productRepository.findByIdOrNull(productId) } returns mockProduct
                every { productMapper.toDto(mockProduct) } returns mockProductDto
            }
            
            it("should return the product DTO") {
                val result = productService.findById(productId)
                
                result shouldBe mockProductDto
                verify(exactly = 1) { productRepository.findByIdOrNull(productId) }
                verify(exactly = 1) { productMapper.toDto(mockProduct) }
            }
        }
        
        context("when product does not exist") {
            val productId = 999L
            
            beforeEach {
                every { productRepository.findByIdOrNull(productId) } returns null
            }
            
            it("should throw ProductNotFoundException") {
                shouldThrow<ProductNotFoundException> {
                    productService.findById(productId)
                }.message shouldBe "Product not found with ID: $productId"
                
                verify(exactly = 1) { productRepository.findByIdOrNull(productId) }
                verify(exactly = 0) { productMapper.toDto(any()) }
            }
        }
    }
    
    describe("ProductService.create") {
        
        context("with valid request") {
            val categoryId = 1L
            val request = CreateProductRequest(
                name = "New Product",
                description = "A new product for testing",
                price = 199.99,
                brand = "TestBrand",
                stockQuantity = 15,
                categoryId = categoryId,
                tags = listOf("new", "test")
            )
            
            val mockCategory = mockk<Category> {
                every { id } returns categoryId
                every { name } returns "Test Category"
            }
            
            val mockProduct = mockk<Product> {
                every { id } returns 0L // Before saving
            }
            
            val savedProduct = mockk<Product> {
                every { id } returns 1L
                every { name } returns request.name
                every { price } returns request.price
            }
            
            val productDto = mockk<ProductDTO> {
                every { id } returns 1L
                every { name } returns request.name
                every { price } returns request.price
            }
            
            beforeEach {
                every { categoryRepository.findByIdOrNull(categoryId) } returns mockCategory
                every { productRepository.existsByNameAndBrand(request.name, request.brand) } returns false
                every { Product.create(any(), any(), any(), any(), any(), any(), any()) } returns mockProduct
                every { productRepository.save(mockProduct) } returns savedProduct
                every { productMapper.toDto(savedProduct) } returns productDto
            }
            
            it("should create and return the product") {
                val result = productService.create(request)
                
                result shouldBe productDto
                result.id shouldNotBe 0L
                
                verify(exactly = 1) { categoryRepository.findByIdOrNull(categoryId) }
                verify(exactly = 1) { productRepository.existsByNameAndBrand(request.name, request.brand) }
                verify(exactly = 1) { productRepository.save(mockProduct) }
                verify(exactly = 1) { productMapper.toDto(savedProduct) }
            }
        }
        
        context("with invalid category") {
            val invalidCategoryId = 999L
            val request = CreateProductRequest(
                name = "Product",
                description = "Description",
                price = 100.0,
                brand = "Brand",
                stockQuantity = 10,
                categoryId = invalidCategoryId,
                tags = emptyList()
            )
            
            beforeEach {
                every { categoryRepository.findByIdOrNull(invalidCategoryId) } returns null
            }
            
            it("should throw CategoryNotFoundException") {
                shouldThrow<CategoryNotFoundException> {
                    productService.create(request)
                }.message shouldBe "Category not found with ID: $invalidCategoryId"
                
                verify(exactly = 1) { categoryRepository.findByIdOrNull(invalidCategoryId) }
                verify(exactly = 0) { productRepository.save(any()) }
            }
        }
        
        context("with duplicate product") {
            val request = CreateProductRequest(
                name = "Duplicate Product",
                description = "Already exists",
                price = 100.0,
                brand = "ExistingBrand",
                stockQuantity = 10,
                categoryId = 1L,
                tags = emptyList()
            )
            
            val mockCategory = mockk<Category>()
            
            beforeEach {
                every { categoryRepository.findByIdOrNull(1L) } returns mockCategory
                every { productRepository.existsByNameAndBrand(request.name, request.brand) } returns true
            }
            
            it("should throw IllegalArgumentException") {
                shouldThrow<IllegalArgumentException> {
                    productService.create(request)
                }.message shouldBe "Product with name '${request.name}' and brand '${request.brand}' already exists"
                
                verify(exactly = 1) { categoryRepository.findByIdOrNull(1L) }
                verify(exactly = 1) { productRepository.existsByNameAndBrand(request.name, request.brand) }
                verify(exactly = 0) { productRepository.save(any()) }
            }
        }
    }
    
    describe("ProductService.search") {
        
        context("with complex search criteria") {
            val criteria = ProductSearchCriteriaDTO(
                name = "Gaming",
                minPrice = 100.0,
                maxPrice = 500.0,
                categoryIds = listOf(1L, 2L),
                tags = listOf("gaming", "premium"),
                page = 0,
                size = 10,
                sortBy = "price",
                sortDirection = "DESC"
            )
            
            val mockProducts = listOf(
                mockk<Product> {
                    every { id } returns 1L
                    every { name } returns "Gaming Laptop"
                    every { price } returns 399.99
                },
                mockk<Product> {
                    every { id } returns 2L
                    every { name } returns "Gaming Mouse"
                    every { price } returns 149.99
                }
            )
            
            val mockDtos = mockProducts.map { product ->
                mockk<ProductDTO> {
                    every { id } returns product.id
                    every { name } returns product.name
                    every { price } returns product.price
                }
            }
            
            val mockPage = PageImpl(mockProducts, PageRequest.of(0, 10), 2)
            
            beforeEach {
                every { 
                    productRepository.findByComplexCriteria(
                        name = criteria.name,
                        minPrice = criteria.minPrice,
                        maxPrice = criteria.maxPrice,
                        categoryIds = criteria.categoryIds,
                        tags = criteria.tags,
                        pageable = any()
                    )
                } returns mockPage
                
                mockProducts.forEachIndexed { index, product ->
                    every { productMapper.toDto(product) } returns mockDtos[index]
                }
            }
            
            it("should return filtered and sorted results") {
                val result = productService.search(criteria)
                
                result.content.size shouldBe 2
                result.totalElements shouldBe 2L
                result.content.map { it.name } shouldBe listOf("Gaming Laptop", "Gaming Mouse")
                
                verify(exactly = 1) { 
                    productRepository.findByComplexCriteria(
                        name = criteria.name,
                        minPrice = criteria.minPrice,
                        maxPrice = criteria.maxPrice,
                        categoryIds = criteria.categoryIds,
                        tags = criteria.tags,
                        pageable = any()
                    )
                }
            }
        }
    }
    
    describe("ProductService.updateStock") {
        
        context("with valid product and quantity") {
            val productId = 1L
            val newQuantity = 50
            
            beforeEach {
                every { productRepository.updateStock(productId, newQuantity) } returns 1
                // Mock the subsequent findById call
                val mockProduct = mockk<Product> {
                    every { id } returns productId
                    every { stockQuantity } returns newQuantity
                }
                val mockDto = mockk<ProductDTO> {
                    every { id } returns productId
                    every { stockQuantity } returns newQuantity
                }
                every { productRepository.findByIdOrNull(productId) } returns mockProduct
                every { productMapper.toDto(mockProduct) } returns mockDto
            }
            
            it("should update stock and return updated product") {
                val result = productService.updateStock(productId, newQuantity)
                
                result.id shouldBe productId
                result.stockQuantity shouldBe newQuantity
                
                verify(exactly = 1) { productRepository.updateStock(productId, newQuantity) }
                verify(exactly = 1) { productRepository.findByIdOrNull(productId) }
            }
        }
        
        context("with non-existent product") {
            val productId = 999L
            val newQuantity = 50
            
            beforeEach {
                every { productRepository.updateStock(productId, newQuantity) } returns 0
            }
            
            it("should throw ProductNotFoundException") {
                shouldThrow<ProductNotFoundException> {
                    productService.updateStock(productId, newQuantity)
                }.message shouldBe "Product not found with ID: $productId"
                
                verify(exactly = 1) { productRepository.updateStock(productId, newQuantity) }
                verify(exactly = 0) { productRepository.findByIdOrNull(any()) }
            }
        }
    }
})
```

### 7.4.5 Controller Testing with Kotest

```kotlin
// src/test/kotlin/com/example/api/controller/ProductControllerTest.kt
package com.example.api.controller

import com.example.api.dto.*
import com.example.api.service.ProductService
import com.fasterxml.jackson.databind.ObjectMapper
import com.ninjasquad.springmockk.MockkBean
import io.kotest.core.spec.style.BehaviorSpec
import io.mockk.*
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.data.domain.PageImpl
import org.springframework.data.domain.PageRequest
import org.springframework.http.MediaType
import org.springframework.test.web.servlet.*
import java.time.LocalDateTime

@WebMvcTest(ProductController::class)
class ProductControllerTest(
    private val mockMvc: MockMvc,
    private val objectMapper: ObjectMapper
) : BehaviorSpec({
    
    @MockkBean
    lateinit var productService: ProductService
    
    Given("a product controller") {
        
        When("getting a product by ID") {
            val productId = 1L
            val productDto = ProductDTO(
                id = productId,
                name = "Test Product",
                description = "A test product",
                price = 99.99,
                brand = "TestBrand",
                stockQuantity = 10,
                inStock = true,
                active = true,
                tags = listOf("test"),
                category = CategoryDTO(
                    id = 1L,
                    name = "Test Category",
                    description = "A test category",
                    active = true,
                    productCount = 1,
                    createdAt = LocalDateTime.now(),
                    updatedAt = LocalDateTime.now()
                ),
                createdAt = LocalDateTime.now(),
                updatedAt = LocalDateTime.now()
            )
            
            every { productService.findById(productId) } returns productDto
            
            Then("should return the product") {
                mockMvc.get("/v1/products/$productId") {
                    contentType = MediaType.APPLICATION_JSON
                }.andExpect {
                    status { isOk() }
                    content { 
                        contentType(MediaType.APPLICATION_JSON)
                        json(objectMapper.writeValueAsString(productDto))
                    }
                    jsonPath("$.id") { value(productId) }
                    jsonPath("$.name") { value("Test Product") }
                    jsonPath("$.price") { value(99.99) }
                }
                
                verify(exactly = 1) { productService.findById(productId) }
            }
        }
        
        When("getting a non-existent product") {
            val productId = 999L
            
            every { productService.findById(productId) } throws ProductNotFoundException("Product not found")
            
            Then("should return 404 Not Found") {
                mockMvc.get("/v1/products/$productId") {
                    contentType = MediaType.APPLICATION_JSON
                }.andExpect {
                    status { isNotFound() }
                }
                
                verify(exactly = 1) { productService.findById(productId) }
            }
        }
        
        When("creating a new product") {
            val request = CreateProductRequest(
                name = "New Product",
                description = "A brand new product",
                price = 149.99,
                brand = "NewBrand",
                stockQuantity = 25,
                categoryId = 1L,
                tags = listOf("new", "featured")
            )
            
            val createdProduct = ProductDTO(
                id = 1L,
                name = request.name,
                description = request.description,
                price = request.price,
                brand = request.brand,
                stockQuantity = request.stockQuantity,
                inStock = true,
                active = true,
                tags = request.tags,
                category = null,
                createdAt = LocalDateTime.now(),
                updatedAt = LocalDateTime.now()
            )
            
            every { productService.create(request) } returns createdProduct
            
            Then("should create and return the product") {
                mockMvc.post("/v1/products") {
                    contentType = MediaType.APPLICATION_JSON
                    content = objectMapper.writeValueAsString(request)
                }.andExpect {
                    status { isCreated() }
                    content {
                        contentType(MediaType.APPLICATION_JSON)
                        json(objectMapper.writeValueAsString(createdProduct))
                    }
                    jsonPath("$.id") { value(1L) }
                    jsonPath("$.name") { value(request.name) }
                }
                
                verify(exactly = 1) { productService.create(request) }
            }
        }
        
        When("creating a product with validation errors") {
            val invalidRequest = CreateProductRequest(
                name = "", // Invalid: blank name
                description = "Valid description",
                price = -10.0, // Invalid: negative price
                brand = "",  // Invalid: blank brand
                stockQuantity = -5, // Invalid: negative stock
                categoryId = 1L,
                tags = emptyList()
            )
            
            Then("should return 400 Bad Request with validation errors") {
                mockMvc.post("/v1/products") {
                    contentType = MediaType.APPLICATION_JSON
                    content = objectMapper.writeValueAsString(invalidRequest)
                }.andExpect {
                    status { isBadRequest() }
                    jsonPath("$.errors") { isArray() }
                }
                
                verify(exactly = 0) { productService.create(any()) }
            }
        }
        
        When("searching products") {
            val searchCriteria = ProductSearchCriteriaDTO(
                name = "Gaming",
                minPrice = 100.0,
                maxPrice = 500.0,
                categoryIds = listOf(1L),
                tags = listOf("gaming"),
                page = 0,
                size = 10
            )
            
            val searchResults = PageImpl(
                listOf(
                    ProductDTO(
                        id = 1L, name = "Gaming Mouse", description = "RGB Gaming Mouse",
                        price = 79.99, brand = "GamerBrand", stockQuantity = 15,
                        inStock = true, active = true, tags = listOf("gaming", "rgb"),
                        category = null, createdAt = LocalDateTime.now(), updatedAt = LocalDateTime.now()
                    ),
                    ProductDTO(
                        id = 2L, name = "Gaming Keyboard", description = "Mechanical Gaming Keyboard",
                        price = 129.99, brand = "GamerBrand", stockQuantity = 10,
                        inStock = true, active = true, tags = listOf("gaming", "mechanical"),
                        category = null, createdAt = LocalDateTime.now(), updatedAt = LocalDateTime.now()
                    )
                ),
                PageRequest.of(0, 10),
                2L
            )
            
            every { productService.search(searchCriteria) } returns searchResults
            
            Then("should return filtered products") {
                mockMvc.post("/v1/products/search") {
                    contentType = MediaType.APPLICATION_JSON
                    content = objectMapper.writeValueAsString(searchCriteria)
                }.andExpected {
                    status { isOk() }
                    jsonPath("$.content") { isArray() }
                    jsonPath("$.content.length()") { value(2) }
                    jsonPath("$.content[0].name") { value("Gaming Mouse") }
                    jsonPath("$.content[1].name") { value("Gaming Keyboard") }
                    jsonPath("$.totalElements") { value(2) }
                    jsonPath("$.size") { value(10) }
                }
                
                verify(exactly = 1) { productService.search(searchCriteria) }
            }
        }
        
        When("updating product stock") {
            val productId = 1L
            val newQuantity = 100
            val updatedProduct = ProductDTO(
                id = productId,
                name = "Updated Product",
                description = "Updated description",
                price = 99.99,
                brand = "UpdatedBrand",
                stockQuantity = newQuantity,
                inStock = true,
                active = true,
                tags = emptyList(),
                category = null,
                createdAt = LocalDateTime.now(),
                updatedAt = LocalDateTime.now()
            )
            
            every { productService.updateStock(productId, newQuantity) } returns updatedProduct
            
            Then("should update and return the product") {
                mockMvc.patch("/v1/products/$productId/stock") {
                    param("quantity", newQuantity.toString())
                }.andExpect {
                    status { isOk() }
                    jsonPath("$.id") { value(productId) }
                    jsonPath("$.stockQuantity") { value(newQuantity) }
                }
                
                verify(exactly = 1) { productService.updateStock(productId, newQuantity) }
            }
        }
        
        When("deleting a product") {
            val productId = 1L
            
            every { productService.delete(productId) } just runs
            
            Then("should delete the product and return no content") {
                mockMvc.delete("/v1/products/$productId").andExpect {
                    status { isNoContent() }
                }
                
                verify(exactly = 1) { productService.delete(productId) }
            }
        }
    }
})
```

### 7.4.6 Integration Testing with Test Containers

```kotlin
// src/test/kotlin/com/example/api/integration/ProductIntegrationTest.kt
package com.example.api.integration

import com.example.api.dto.CreateProductRequest
import com.example.api.entity.Category
import com.example.api.repository.CategoryRepository
import com.fasterxml.jackson.databind.ObjectMapper
import io.kotest.core.spec.style.BehaviorSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureWebMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.http.MediaType
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.springframework.test.web.servlet.*
import org.springframework.transaction.annotation.Transactional
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebMvc
@Testcontainers
@Transactional
class ProductIntegrationTest(
    private val mockMvc: MockMvc,
    private val objectMapper: ObjectMapper,
    private val categoryRepository: CategoryRepository
) : BehaviorSpec({
    
    companion object {
        @Container
        @JvmStatic
        val postgresContainer = PostgreSQLContainer<Nothing>("postgres:14-alpine").apply {
            withDatabaseName("testdb")
            withUsername("testuser")
            withPassword("testpass")
        }
        
        @DynamicPropertySource
        @JvmStatic
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgresContainer::getJdbcUrl)
            registry.add("spring.datasource.username", postgresContainer::getUsername)
            registry.add("spring.datasource.password", postgresContainer::getPassword)
        }
    }
    
    Given("a running application with real database") {
        
        When("creating a complete product workflow") {
            // First, create a category
            val category = Category.create("Integration Test Category", "For integration testing")
            val savedCategory = categoryRepository.save(category)
            
            val productRequest = CreateProductRequest(
                name = "Integration Test Product",
                description = "A product created during integration testing",
                price = 299.99,
                brand = "IntegrationBrand",
                stockQuantity = 50,
                categoryId = savedCategory.id,
                tags = listOf("integration", "test", "api")
            )
            
            Then("should create, retrieve, update, and delete successfully") {
                // Create product
                val createResponse = mockMvc.post("/v1/products") {
                    contentType = MediaType.APPLICATION_JSON
                    content = objectMapper.writeValueAsString(productRequest)
                }.andExpect {
                    status { isCreated() }
                    jsonPath("$.name") { value(productRequest.name) }
                    jsonPath("$.price") { value(productRequest.price) }
                    jsonPath("$.stockQuantity") { value(productRequest.stockQuantity) }
                }.andReturn()
                
                val createdProduct = objectMapper.readTree(createResponse.response.contentAsString)
                val productId = createdProduct.get("id").asLong()
                
                productId shouldNotBe 0L
                
                // Retrieve product
                mockMvc.get("/v1/products/$productId").andExpect {
                    status { isOk() }
                    jsonPath("$.id") { value(productId) }
                    jsonPath("$.name") { value(productRequest.name) }
                    jsonPath("$.category.id") { value(savedCategory.id) }
                    jsonPath("$.tags.length()") { value(3) }
                }
                
                // Update stock
                val newStockQuantity = 75
                mockMvc.patch("/v1/products/$productId/stock") {
                    param("quantity", newStockQuantity.toString())
                }.andExpect {
                    status { isOk() }
                    jsonPath("$.stockQuantity") { value(newStockQuantity) }
                }
                
                // Verify stock update persisted
                mockMvc.get("/v1/products/$productId").andExpect {
                    status { isOk() }
                    jsonPath("$.stockQuantity") { value(newStockQuantity) }
                }
                
                // Delete product
                mockMvc.delete("/v1/products/$productId").andExpect {
                    status { isNoContent() }
                }
                
                // Verify product is no longer accessible (soft deleted)
                mockMvc.get("/v1/products/$productId").andExpect {
                    status { isNotFound() }
                }
            }
        }
        
        When("performing complex search operations") {
            // Setup test data
            val electronics = categoryRepository.save(Category.create("Electronics", "Electronic devices"))
            val books = categoryRepository.save(Category.create("Books", "Books and publications"))
            
            val productsToCreate = listOf(
                CreateProductRequest("Gaming Laptop", "High-end gaming laptop", 1999.99, "TechCorp", 5, electronics.id, listOf("gaming", "premium", "laptop")),
                CreateProductRequest("Office Mouse", "Ergonomic office mouse", 29.99, "TechCorp", 100, electronics.id, listOf("office", "ergonomic")),
                CreateProductRequest("Mechanical Keyboard", "RGB mechanical keyboard", 149.99, "TechCorp", 25, electronics.id, listOf("gaming", "rgb", "mechanical")),
                CreateProductRequest("Programming Guide", "Complete programming guide", 45.99, "EduPress", 30, books.id, listOf("programming", "education")),
                CreateProductRequest("Fiction Novel", "Best-selling fiction novel", 12.99, "BookCorp", 200, books.id, listOf("fiction", "bestseller"))
            )
            
            // Create all products
            productsToCreate.forEach { request ->
                mockMvc.post("/v1/products") {
                    contentType = MediaType.APPLICATION_JSON
                    content = objectMapper.writeValueAsString(request)
                }.andExpect {
                    status { isCreated() }
                }
            }
            
            Then("should filter products correctly by various criteria") {
                // Search by price range
                val priceRangeSearch = mapOf(
                    "minPrice" to 30.0,
                    "maxPrice" to 200.0,
                    "page" to 0,
                    "size" to 10
                )
                
                mockMvc.post("/v1/products/search") {
                    contentType = MediaType.APPLICATION_JSON
                    content = objectMapper.writeValueAsString(priceRangeSearch)
                }.andExpect {
                    status { isOk() }
                    jsonPath("$.content.length()") { value(2) } // Office Mouse and Mechanical Keyboard
                    jsonPath("$.content[*].price").value(org.hamcrest.Matchers.everyItem(
                        org.hamcrest.Matchers.allOf(
                            org.hamcrest.Matchers.greaterThanOrEqualTo(30.0),
                            org.hamcrest.Matchers.lessThanOrEqualTo(200.0)
                        )
                    ))
                }
                
                // Search by category
                val categorySearch = mapOf(
                    "categoryIds" to listOf(electronics.id),
                    "page" to 0,
                    "size" to 10
                )
                
                mockMvc.post("/v1/products/search") {
                    contentType = MediaType.APPLICATION_JSON
                    content = objectMapper.writeValueAsString(categorySearch)
                }.andExpect {
                    status { isOk() }
                    jsonPath("$.content.length()") { value(3) } // All electronics
                    jsonPath("$.content[*].category.id").value(
                        org.hamcrest.Matchers.everyItem(
                            org.hamcrest.Matchers.equalTo(electronics.id.toInt())
                        )
                    )
                }
                
                // Search by tags
                val tagSearch = mapOf(
                    "tags" to listOf("gaming"),
                    "page" to 0,
                    "size" to 10
                )
                
                mockMvc.post("/v1/products/search") {
                    contentType = MediaType.APPLICATION_JSON
                    content = objectMapper.writeValueAsString(tagSearch)
                }.andExpect {
                    status { isOk() }
                    jsonPath("$.content.length()") { value(2) } // Gaming Laptop and Mechanical Keyboard
                }
                
                // Combined search
                val combinedSearch = mapOf(
                    "name" to "keyboard",
                    "categoryIds" to listOf(electronics.id),
                    "tags" to listOf("gaming"),
                    "maxPrice" to 200.0,
                    "page" to 0,
                    "size" to 10
                )
                
                mockMvc.post("/v1/products/search") {
                    contentType = MediaType.APPLICATION_JSON
                    content = objectMapper.writeValueAsString(combinedSearch)
                }.andExpect {
                    status { isOk() }
                    jsonPath("$.content.length()") { value(1) } // Only Mechanical Keyboard matches all criteria
                    jsonPath("$.content[0].name") { value("Mechanical Keyboard") }
                }
            }
        }
    }
})
```

## 7.5 Test Coverage with Kover

Code coverage helps ensure that your tests exercise all parts of your codebase. Kover is JetBrains' code coverage tool for Kotlin.

### 7.5.1 Kover Setup

Add Kover to your `build.gradle.kts`:

```kotlin
plugins {
    id("org.jetbrains.kotlinx.kover") version "0.7.4"
}

kover {
    reports {
        total {
            xml {
                onCheck = false
            }
            html {
                onCheck = false
            }
        }
    }
}

koverReport {
    filters {
        excludes {
            classes(
                "com.example.api.ApiApplication*",
                "com.example.api.config.*",
                "com.example.api.dto.*",
                "com.example.api.entity.*"
            )
            packages("com.example.api.generated")
        }
    }
    
    verify {
        rule {
            name = "Minimal line coverage rate in percent"
            bound {
                minValue = 80
                valueType = kotlinx.kover.gradle.plugin.dsl.MetricType.LINE
            }
        }
        
        rule {
            name = "Minimal branch coverage rate in percent"
            bound {
                minValue = 75
                valueType = kotlinx.kover.gradle.plugin.dsl.MetricType.BRANCH
            }
        }
    }
}
```

### 7.5.2 Coverage-Driven Test Development

```kotlin
// Example: Ensuring comprehensive coverage of business logic
class PricingEngineTest : BehaviorSpec({
    
    val pricingEngine = PricingEngine()
    
    Given("a pricing engine") {
        
        // Test all paths through the pricing logic
        context("calculating standard pricing") {
            
            it("should apply base pricing for new customers") {
                val customer = Customer(type = CustomerType.NEW)
                val product = Product(basePrice = 100.0)
                
                val price = pricingEngine.calculatePrice(customer, product)
                
                price shouldBe 100.0 // No discount for new customers
            }
            
            it("should apply regular customer discount") {
                val customer = Customer(type = CustomerType.REGULAR, loyaltyYears = 2)
                val product = Product(basePrice = 100.0)
                
                val price = pricingEngine.calculatePrice(customer, product)
                
                price shouldBe 95.0 // 5% discount
            }
            
            it("should apply premium customer discount with loyalty bonus") {
                val customer = Customer(type = CustomerType.PREMIUM, loyaltyYears = 3)
                val product = Product(basePrice = 100.0)
                
                val price = pricingEngine.calculatePrice(customer, product)
                
                price shouldBe 82.0 // 15% base + 3% loyalty = 18% total discount
            }
            
            it("should cap loyalty bonus at maximum") {
                val customer = Customer(type = CustomerType.PREMIUM, loyaltyYears = 10)
                val product = Product(basePrice = 100.0)
                
                val price = pricingEngine.calculatePrice(customer, product)
                
                price shouldBe 75.0 // 15% base + 10% max loyalty = 25% total discount
            }
        }
        
        context("handling edge cases") {
            
            it("should handle zero price products") {
                val customer = Customer(type = CustomerType.PREMIUM)
                val product = Product(basePrice = 0.0)
                
                val price = pricingEngine.calculatePrice(customer, product)
                
                price shouldBe 0.0
            }
            
            it("should handle negative prices") {
                val customer = Customer(type = CustomerType.REGULAR)
                val product = Product(basePrice = -50.0)
                
                shouldThrow<IllegalArgumentException> {
                    pricingEngine.calculatePrice(customer, product)
                }.message shouldBe "Product price cannot be negative"
            }
            
            it("should handle null customer") {
                val product = Product(basePrice = 100.0)
                
                shouldThrow<IllegalArgumentException> {
                    pricingEngine.calculatePrice(null, product)
                }.message shouldBe "Customer cannot be null"
            }
        }
        
        context("bulk pricing scenarios") {
            
            it("should apply bulk discount for large orders") {
                val customer = Customer(type = CustomerType.REGULAR)
                val products = List(10) { Product(basePrice = 50.0) }
                
                val totalPrice = pricingEngine.calculateBulkPrice(customer, products)
                
                // Regular discount (5%) + bulk discount (10%) = 15% total
                totalPrice shouldBe 425.0 // 500 * 0.85
            }
            
            it("should prioritize better discount") {
                val customer = Customer(type = CustomerType.PREMIUM, loyaltyYears = 5)
                val products = List(5) { Product(basePrice = 100.0) }
                
                val totalPrice = pricingEngine.calculateBulkPrice(customer, products)
                
                // Premium discount (20%) is better than bulk discount (5% for 5 items)
                totalPrice shouldBe 400.0 // 500 * 0.8
            }
        }
    }
})

// Corresponding implementation to achieve full coverage
class PricingEngine {
    
    fun calculatePrice(customer: Customer?, product: Product): Double {
        requireNotNull(customer) { "Customer cannot be null" }
        require(product.basePrice >= 0) { "Product price cannot be negative" }
        
        if (product.basePrice == 0.0) return 0.0
        
        val discountPercentage = when (customer.type) {
            CustomerType.NEW -> 0.0
            CustomerType.REGULAR -> 5.0
            CustomerType.PREMIUM -> {
                val baseDiscount = 15.0
                val loyaltyBonus = minOf(customer.loyaltyYears * 1.0, 10.0)
                baseDiscount + loyaltyBonus
            }
        }
        
        return product.basePrice * (1.0 - discountPercentage / 100.0)
    }
    
    fun calculateBulkPrice(customer: Customer, products: List<Product>): Double {
        val baseTotal = products.sumOf { it.basePrice }
        val individualTotal = products.sumOf { calculatePrice(customer, it) }
        
        // Bulk discount based on quantity
        val bulkDiscountPercentage = when {
            products.size >= 10 -> 10.0
            products.size >= 5 -> 5.0
            else -> 0.0
        }
        
        val bulkTotal = baseTotal * (1.0 - bulkDiscountPercentage / 100.0)
        
        // Return the better price for the customer
        return minOf(individualTotal, bulkTotal)
    }
}
```

### 7.5.3 Running and Analyzing Coverage

```bash
# Generate coverage report
./gradlew koverHtmlReport

# Verify coverage meets requirements
./gradlew koverVerify

# Generate XML report for CI/CD
./gradlew koverXmlReport
```

The coverage report will show:
- **Line Coverage**: Percentage of lines executed by tests
- **Branch Coverage**: Percentage of decision branches taken
- **Method Coverage**: Percentage of methods called by tests

### 7.5.4 Coverage Analysis Example

```kotlin
// Example: Analyzing low coverage areas
class PaymentProcessorTest : StringSpec({
    
    val mockGateway = mockk<PaymentGateway>()
    val processor = PaymentProcessor(mockGateway)
    
    "should process successful payment" {
        val payment = Payment(amount = 100.0, currency = "USD")
        
        every { mockGateway.processPayment(payment) } returns PaymentResult.Success("tx123")
        
        val result = processor.processPayment(payment)
        
        result should beInstanceOf<ProcessingResult.Success>()
    }
    
    "should handle payment failures" {
        val payment = Payment(amount = 100.0, currency = "USD")
        
        every { mockGateway.processPayment(payment) } returns PaymentResult.Failure("Insufficient funds")
        
        val result = processor.processPayment(payment)
        
        result should beInstanceOf<ProcessingResult.Failure>()
    }
    
    // This test was added after coverage analysis showed missing branch
    "should handle gateway timeouts" {
        val payment = Payment(amount = 100.0, currency = "USD")
        
        every { mockGateway.processPayment(payment) } throws TimeoutException("Gateway timeout")
        
        val result = processor.processPayment(payment)
        
        result should beInstanceOf<ProcessingResult.Error>()
        (result as ProcessingResult.Error).message shouldContain "timeout"
    }
    
    // Another test added after coverage analysis
    "should validate payment amount" {
        val invalidPayment = Payment(amount = -10.0, currency = "USD")
        
        shouldThrow<IllegalArgumentException> {
            processor.processPayment(invalidPayment)
        }.message shouldBe "Payment amount must be positive"
    }
})
```

## 7.6 Test-Driven Development (TDD)

TDD is a development practice where you write tests before implementing the functionality. This approach leads to better design and higher code coverage.

### 7.6.1 The TDD Cycle

The TDD cycle consists of three phases:

1. **Red**: Write a failing test
2. **Green**: Write minimal code to make the test pass
3. **Refactor**: Improve the code while keeping tests passing

### 7.6.2 TDD Example: Shopping Cart Feature

Let's implement a shopping cart feature using TDD:

```kotlin
// Step 1: Write failing tests first
class ShoppingCartTDDTest : BehaviorSpec({
    
    Given("an empty shopping cart") {
        val cart = ShoppingCart()
        
        When("checking if empty") {
            Then("should be empty") {
                cart.isEmpty() shouldBe true
                cart.itemCount shouldBe 0
                cart.totalValue shouldBe 0.0
            }
        }
        
        When("adding a product") {
            val product = Product(id = 1, name = "Laptop", price = 999.99)
            cart.addProduct(product, quantity = 1)
            
            Then("should contain the product") {
                cart.isEmpty() shouldBe false
                cart.itemCount shouldBe 1
                cart.totalValue shouldBe 999.99
                cart.getQuantity(product) shouldBe 1
            }
        }
        
        When("adding multiple quantities of same product") {
            val product = Product(id = 1, name = "Mouse", price = 29.99)
            cart.addProduct(product, quantity = 3)
            
            Then("should aggregate quantities") {
                cart.itemCount shouldBe 1  // One unique product
                cart.getQuantity(product) shouldBe 3
                cart.totalValue shouldBe 89.97 // 29.99 * 3
            }
        }
    }
    
    Given("a cart with existing products") {
        val cart = ShoppingCart()
        val laptop = Product(id = 1, name = "Laptop", price = 999.99)
        val mouse = Product(id = 2, name = "Mouse", price = 29.99)
        
        cart.addProduct(laptop, 1)
        cart.addProduct(mouse, 2)
        
        When("removing a product completely") {
            cart.removeProduct(mouse)
            
            Then("should remove all quantities") {
                cart.itemCount shouldBe 1
                cart.getQuantity(mouse) shouldBe 0
                cart.totalValue shouldBe 999.99
            }
        }
        
        When("reducing product quantity") {
            cart.updateQuantity(mouse, 1)
            
            Then("should update quantity and total") {
                cart.itemCount shouldBe 2
                cart.getQuantity(mouse) shouldBe 1
                cart.totalValue shouldBe 1029.98 // 999.99 + 29.99
            }
        }
        
        When("applying a discount") {
            val discount = PercentageDiscount(10.0) // 10% off
            cart.applyDiscount(discount)
            
            Then("should calculate discounted total") {
                val originalTotal = 999.99 + (29.99 * 2) // 1059.97
                val expectedTotal = originalTotal * 0.9   // 953.97
                cart.totalValue shouldBe expectedTotal
                cart.appliedDiscount shouldBe discount
            }
        }
    }
    
    Given("cart business rules") {
        val cart = ShoppingCart()
        
        When("adding product with zero quantity") {
            val product = Product(id = 1, name = "Test", price = 10.0)
            
            Then("should throw exception") {
                shouldThrow<IllegalArgumentException> {
                    cart.addProduct(product, 0)
                }.message shouldBe "Quantity must be positive"
            }
        }
        
        When("updating to negative quantity") {
            val product = Product(id = 1, name = "Test", price = 10.0)
            cart.addProduct(product, 5)
            
            Then("should throw exception") {
                shouldThrow<IllegalArgumentException> {
                    cart.updateQuantity(product, -1)
                }.message shouldBe "Quantity cannot be negative"
            }
        }
        
        When("applying multiple discounts") {
            val product = Product(id = 1, name = "Test", price = 100.0)
            cart.addProduct(product, 1)
            
            val discount1 = PercentageDiscount(10.0)
            val discount2 = PercentageDiscount(5.0)
            
            cart.applyDiscount(discount1)
            cart.applyDiscount(discount2)
            
            Then("should keep only the last discount") {
                cart.appliedDiscount shouldBe discount2
                cart.totalValue shouldBe 95.0 // 100 * 0.95
            }
        }
    }
})

// Step 2: Implement minimal code to make tests pass
data class Product(
    val id: Long,
    val name: String,
    val price: Double
) {
    init {
        require(price >= 0) { "Price cannot be negative" }
    }
}

interface Discount {
    fun apply(amount: Double): Double
}

class PercentageDiscount(private val percentage: Double) : Discount {
    init {
        require(percentage in 0.0..100.0) { "Percentage must be between 0 and 100" }
    }
    
    override fun apply(amount: Double): Double {
        return amount * (1.0 - percentage / 100.0)
    }
    
    override fun equals(other: Any?): Boolean {
        return other is PercentageDiscount && other.percentage == percentage
    }
    
    override fun hashCode(): Int = percentage.hashCode()
}

class ShoppingCart {
    private val items = mutableMapOf<Product, Int>()
    var appliedDiscount: Discount? = null
        private set
    
    fun isEmpty(): Boolean = items.isEmpty()
    
    val itemCount: Int
        get() = items.size
    
    val totalValue: Double
        get() {
            val subtotal = items.entries.sumOf { (product, quantity) ->
                product.price * quantity
            }
            return appliedDiscount?.apply(subtotal) ?: subtotal
        }
    
    fun addProduct(product: Product, quantity: Int) {
        require(quantity > 0) { "Quantity must be positive" }
        
        val currentQuantity = items[product] ?: 0
        items[product] = currentQuantity + quantity
    }
    
    fun removeProduct(product: Product) {
        items.remove(product)
    }
    
    fun updateQuantity(product: Product, quantity: Int) {
        require(quantity >= 0) { "Quantity cannot be negative" }
        
        if (quantity == 0) {
            removeProduct(product)
        } else {
            items[product] = quantity
        }
    }
    
    fun getQuantity(product: Product): Int {
        return items[product] ?: 0
    }
    
    fun applyDiscount(discount: Discount) {
        this.appliedDiscount = discount
    }
}
```

### 7.6.3 TDD Benefits in Practice

**Better Design:**
```kotlin
// TDD forces you to think about the API first
class UserServiceTDDTest : StringSpec({
    
    "should register new user with valid email" {
        val userService = UserService(mockUserRepository, mockEmailValidator, mockPasswordEncoder)
        
        // This test drives the UserService constructor design
        every { mockEmailValidator.isValid("user@example.com") } returns true
        every { mockPasswordEncoder.encode("password123") } returns "encoded_password"
        every { mockUserRepository.existsByEmail("user@example.com") } returns false
        every { mockUserRepository.save(any()) } returns mockUser
        
        val result = userService.registerUser("user@example.com", "password123")
        
        result should beInstanceOf<RegistrationResult.Success>()
    }
    
    "should reject registration with invalid email" {
        // This drives error handling design
        every { mockEmailValidator.isValid("invalid-email") } returns false
        
        val result = userService.registerUser("invalid-email", "password123")
        
        result should beInstanceOf<RegistrationResult.ValidationError>()
    }
})

// Implementation emerges from tests
class UserService(
    private val userRepository: UserRepository,
    private val emailValidator: EmailValidator,
    private val passwordEncoder: PasswordEncoder
) {
    
    fun registerUser(email: String, password: String): RegistrationResult {
        if (!emailValidator.isValid(email)) {
            return RegistrationResult.ValidationError("Invalid email format")
        }
        
        if (userRepository.existsByEmail(email)) {
            return RegistrationResult.ValidationError("Email already exists")
        }
        
        val encodedPassword = passwordEncoder.encode(password)
        val user = User(email = email, passwordHash = encodedPassword)
        val savedUser = userRepository.save(user)
        
        return RegistrationResult.Success(savedUser)
    }
}
```

### 7.6.4 TDD Anti-patterns to Avoid

**Don't Write Implementation-Dependent Tests:**
```kotlin
// Bad: Testing implementation details
class BadServiceTest : StringSpec({
    "should call repository.findById exactly once" {
        val service = ProductService(mockRepository)
        
        every { mockRepository.findById(1L) } returns mockProduct
        
        service.getProduct(1L)
        
        // This test is fragile - it breaks if implementation changes
        verify(exactly = 1) { mockRepository.findById(1L) }
    }
})

// Good: Testing behavior
class GoodServiceTest : StringSpec({
    "should return product when found" {
        val service = ProductService(mockRepository)
        val expectedProduct = Product(id = 1L, name = "Test")
        
        every { mockRepository.findById(1L) } returns expectedProduct
        
        val result = service.getProduct(1L)
        
        // This test is robust - it focuses on behavior
        result shouldBe expectedProduct
    }
})
```

## 7.7 Summary

In this comprehensive chapter, we've explored testing strategies and practices for Spring Boot Kotlin applications:

### Key Concepts Covered:

1. **Testing Philosophy and Value**:
   - Understanding why testing matters beyond bug detection
   - The testing pyramid and when to use different test types
   - F.I.R.S.T principles for effective test design

2. **Unit vs Integration Testing**:
   - Clear distinctions and when to use each approach
   - Isolated unit tests with mocking for fast feedback
   - Integration tests for component interactions and real scenarios

3. **Test Code Quality and Patterns**:
   - Given-When-Then (GWT) structure for readable tests
   - Test data builders for maintainable test fixtures
   - Avoiding common testing anti-patterns

4. **Kotest Framework Mastery**:
   - Comprehensive setup and configuration
   - Repository, service, and controller testing patterns
   - Integration with Spring Boot and MockK
   - Test containers for database integration testing

5. **Test Coverage Analysis**:
   - Kover setup and configuration
   - Coverage-driven development approach
   - Analyzing and improving coverage gaps
   - Setting appropriate coverage thresholds

6. **Test-Driven Development (TDD)**:
   - Red-Green-Refactor cycle
   - Practical TDD implementation examples
   - Benefits for design and code quality
   - Avoiding common TDD pitfalls

### Best Practices Implemented:

- **Test Organization**: Clear test structure with appropriate naming and grouping
- **Mock Management**: Proper use of MockK for isolation and verification
- **Data Management**: Test data builders and fixtures for maintainability
- **Coverage Goals**: Meaningful coverage targets with appropriate exclusions
- **Performance**: Fast unit tests and comprehensive integration tests
- **Readability**: Expressive test syntax using Kotest's BDD-style specifications

### Testing Strategy Recommendations:

1. **Start with Unit Tests**: Write fast, isolated tests for business logic
2. **Add Integration Tests**: Verify component interactions and data persistence
3. **Use TDD for Complex Logic**: Drive design through test-first development
4. **Monitor Coverage**: Maintain high coverage while focusing on meaningful tests
5. **Maintain Test Quality**: Treat test code with the same care as production code

### What's Next:

With a solid testing foundation in place, you now have the tools and knowledge to build reliable Spring Boot Kotlin applications. The testing practices covered in this chapter will serve as the foundation for maintaining code quality as your application grows and evolves.

The combination of comprehensive unit tests, integration tests, and a test-driven development approach ensures that your code is not only correct but also well-designed and maintainable. These practices will enable confident refactoring and feature development throughout your application's lifecycle.