---
layout: default
---
# Chapter 05: Various Ways to Write APIs

In this chapter, we'll explore the different approaches to building REST APIs with Spring Boot and Kotlin. We'll cover everything from basic controller setup to advanced documentation and logging strategies. By the end of this chapter, you'll understand how to create robust, well-documented APIs that follow modern best practices.

## 5.1 Project Configuration

Before we dive into writing APIs, let's set up our project with the necessary dependencies and configuration for a comprehensive API development experience.

### 5.1.1 Dependencies Setup

First, let's add the essential dependencies to our `build.gradle.kts`:

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    
    // JSON processing
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("com.fasterxml.jackson.datatype:jackson-datatype-jsr310")
    
    // API Documentation (Swagger/OpenAPI)
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.2.0")
    
    // Database
    implementation("org.postgresql:postgresql")
    
    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.testcontainers:postgresql")
    testImplementation("org.testcontainers:junit-jupiter")
    
    // Kotlin-specific
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")
}
```

### 5.1.2 Application Configuration

Create a well-structured configuration for your API application:

```kotlin
// src/main/kotlin/com/example/api/Application.kt
package com.example.api

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class ApiApplication

fun main(args: Array<String>) {
    runApplication<ApiApplication>(*args)
}
```

### 5.1.3 Basic Configuration Properties

Configure your `application.yml`:

```yaml
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  application:
    name: spring-boot-kotlin-api
  
  datasource:
    url: jdbc:postgresql://localhost:5432/api_db
    username: api_user
    password: api_password
    driver-class-name: org.postgresql.Driver
    
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true

  jackson:
    property-naming-strategy: SNAKE_CASE
    default-property-inclusion: NON_NULL
    serialization:
      write-dates-as-timestamps: false

logging:
  level:
    com.example.api: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
```

### 5.1.4 Base Configuration Classes

Set up foundational configuration classes:

```kotlin
// src/main/kotlin/com/example/api/config/WebConfig.kt
package com.example.api.config

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.kotlin.registerKotlinModule
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.cors.CorsConfiguration
import org.springframework.web.cors.CorsConfigurationSource
import org.springframework.web.cors.UrlBasedCorsConfigurationSource

@Configuration
class WebConfig {
    
    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource {
        val configuration = CorsConfiguration().apply {
            allowedOriginPatterns = listOf("*")
            allowedMethods = listOf("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            allowedHeaders = listOf("*")
            allowCredentials = true
        }
        
        return UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/**", configuration)
        }
    }
    
    @Bean
    fun objectMapper(): ObjectMapper {
        return ObjectMapper().apply {
            registerKotlinModule()
            findAndRegisterModules()
        }
    }
}
```

## 5.2 Creating a GET API

Let's start with GET APIs, which are the foundation of REST services. We'll explore different approaches to handle various types of requests.

### 5.2.1 Basic GET API with @RequestMapping

```kotlin
// src/main/kotlin/com/example/api/controller/ProductController.kt
package com.example.api.controller

import com.example.api.dto.ProductDTO
import com.example.api.service.ProductService
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    /**
     * Simple GET endpoint without parameters
     * GET /api/v1/products/health
     */
    @RequestMapping(value = ["/health"], method = [RequestMethod.GET])
    fun healthCheck(): ResponseEntity<Map<String, String>> {
        return ResponseEntity.ok(
            mapOf(
                "status" to "UP",
                "service" to "ProductController",
                "timestamp" to java.time.Instant.now().toString()
            )
        )
    }
    
    /**
     * Get all products without parameters
     * GET /api/v1/products
     */
    @RequestMapping(method = [RequestMethod.GET])
    fun getAllProducts(): ResponseEntity<List<ProductDTO>> {
        val products = productService.findAll()
        return ResponseEntity.ok(products)
    }
}
```

### 5.2.2 GET API with PathVariable

```kotlin
@RestController
@RequestMapping("/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    /**
     * Get product by ID using PathVariable
     * GET /api/v1/products/123
     */
    @RequestMapping(value = ["/{id}"], method = [RequestMethod.GET])
    fun getProductById(@PathVariable id: Long): ResponseEntity<ProductDTO> {
        val product = productService.findById(id)
        return ResponseEntity.ok(product)
    }
    
    /**
     * Get products by category using PathVariable
     * GET /api/v1/products/category/electronics
     */
    @RequestMapping(value = ["/category/{categoryName}"], method = [RequestMethod.GET])
    fun getProductsByCategory(@PathVariable categoryName: String): ResponseEntity<List<ProductDTO>> {
        val products = productService.findByCategory(categoryName)
        return ResponseEntity.ok(products)
    }
    
    /**
     * Complex path with multiple variables
     * GET /api/v1/products/category/electronics/brand/apple
     */
    @RequestMapping(value = ["/category/{category}/brand/{brand}"], method = [RequestMethod.GET])
    fun getProductsByCategoryAndBrand(
        @PathVariable category: String,
        @PathVariable brand: String
    ): ResponseEntity<List<ProductDTO>> {
        val products = productService.findByCategoryAndBrand(category, brand)
        return ResponseEntity.ok(products)
    }
}
```

### 5.2.3 GET API with RequestParam

```kotlin
@RestController
@RequestMapping("/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    /**
     * Get products with pagination using RequestParam
     * GET /api/v1/products/search?page=0&size=10
     */
    @RequestMapping(value = ["/search"], method = [RequestMethod.GET])
    fun searchProducts(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int,
        @RequestParam(required = false) name: String?,
        @RequestParam(required = false) category: String?,
        @RequestParam(defaultValue = "id") sortBy: String,
        @RequestParam(defaultValue = "ASC") sortDirection: String
    ): ResponseEntity<PagedProductResponse> {
        val searchCriteria = ProductSearchCriteria(
            name = name,
            category = category,
            page = page,
            size = size,
            sortBy = sortBy,
            sortDirection = sortDirection
        )
        
        val result = productService.search(searchCriteria)
        return ResponseEntity.ok(result)
    }
    
    /**
     * Get products with price filtering
     * GET /api/v1/products/filter?minPrice=10&maxPrice=100&inStock=true
     */
    @RequestMapping(value = ["/filter"], method = [RequestMethod.GET])
    fun filterProducts(
        @RequestParam(required = false) minPrice: Double?,
        @RequestParam(required = false) maxPrice: Double?,
        @RequestParam(defaultValue = "true") inStock: Boolean,
        @RequestParam(required = false) tags: List<String>?
    ): ResponseEntity<List<ProductDTO>> {
        val filterCriteria = ProductFilterCriteria(
            minPrice = minPrice,
            maxPrice = maxPrice,
            inStock = inStock,
            tags = tags ?: emptyList()
        )
        
        val products = productService.filter(filterCriteria)
        return ResponseEntity.ok(products)
    }
}
```

### 5.2.4 DTO Classes for GET Operations

```kotlin
// src/main/kotlin/com/example/api/dto/ProductDTO.kt
package com.example.api.dto

import java.time.LocalDateTime

data class ProductDTO(
    val id: Long,
    val name: String,
    val description: String,
    val price: Double,
    val category: String,
    val brand: String,
    val inStock: Boolean,
    val stockQuantity: Int,
    val tags: List<String> = emptyList(),
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
)

// Search and filter DTOs
data class ProductSearchCriteria(
    val name: String?,
    val category: String?,
    val page: Int,
    val size: Int,
    val sortBy: String,
    val sortDirection: String
)

data class ProductFilterCriteria(
    val minPrice: Double?,
    val maxPrice: Double?,
    val inStock: Boolean,
    val tags: List<String>
)

data class PagedProductResponse(
    val content: List<ProductDTO>,
    val page: Int,
    val size: Int,
    val totalElements: Long,
    val totalPages: Int,
    val first: Boolean,
    val last: Boolean
)
```

## 5.3 Creating a POST API

POST APIs are used for creating resources. Let's explore different patterns for handling request bodies and data validation.

### 5.3.1 Basic POST API with @RequestMapping

```kotlin
@RestController
@RequestMapping("/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    /**
     * Create a new product using POST
     * POST /api/v1/products
     */
    @RequestMapping(method = [RequestMethod.POST])
    fun createProduct(@RequestBody createProductRequest: CreateProductRequest): ResponseEntity<ProductDTO> {
        val product = productService.create(createProductRequest)
        return ResponseEntity.status(201).body(product)
    }
    
    /**
     * Bulk create products
     * POST /api/v1/products/bulk
     */
    @RequestMapping(value = ["/bulk"], method = [RequestMethod.POST])
    fun createProducts(@RequestBody createRequests: List<CreateProductRequest>): ResponseEntity<BulkProductResponse> {
        val result = productService.createBulk(createRequests)
        return ResponseEntity.status(201).body(result)
    }
}
```

### 5.3.2 POST API with Request Body Validation

```kotlin
import jakarta.validation.Valid

@RestController
@RequestMapping("/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    /**
     * Create product with validation
     * POST /api/v1/products
     */
    @RequestMapping(method = [RequestMethod.POST])
    fun createProductWithValidation(
        @Valid @RequestBody createProductRequest: CreateProductRequest
    ): ResponseEntity<ProductDTO> {
        val product = productService.create(createProductRequest)
        return ResponseEntity.status(201).body(product)
    }
    
    /**
     * Complex POST with additional parameters
     * POST /api/v1/products?notify=true&category=electronics
     */
    @RequestMapping(method = [RequestMethod.POST])
    fun createProductWithParams(
        @Valid @RequestBody createProductRequest: CreateProductRequest,
        @RequestParam(defaultValue = "false") notify: Boolean,
        @RequestParam(required = false) category: String?
    ): ResponseEntity<ProductDTO> {
        // Override category from request param if provided
        val finalRequest = if (category != null) {
            createProductRequest.copy(category = category)
        } else {
            createProductRequest
        }
        
        val product = productService.create(finalRequest)
        
        if (notify) {
            productService.sendNotification(product)
        }
        
        return ResponseEntity.status(201).body(product)
    }
}
```

### 5.3.3 Request DTOs for POST Operations

```kotlin
// src/main/kotlin/com/example/api/dto/CreateProductRequest.kt
package com.example.api.dto

import jakarta.validation.constraints.*

data class CreateProductRequest(
    @field:NotBlank(message = "Product name is required")
    @field:Size(min = 2, max = 100, message = "Product name must be between 2 and 100 characters")
    val name: String,
    
    @field:NotBlank(message = "Description is required")
    @field:Size(max = 500, message = "Description cannot exceed 500 characters")
    val description: String,
    
    @field:NotNull(message = "Price is required")
    @field:Min(value = 0, message = "Price cannot be negative")
    val price: Double,
    
    @field:NotBlank(message = "Category is required")
    val category: String,
    
    @field:NotBlank(message = "Brand is required")
    val brand: String,
    
    @field:Min(value = 0, message = "Stock quantity cannot be negative")
    val stockQuantity: Int = 0,
    
    val tags: List<String> = emptyList()
)

data class BulkProductResponse(
    val created: List<ProductDTO>,
    val errors: List<ProductCreationError>,
    val totalRequested: Int,
    val totalCreated: Int,
    val totalErrors: Int
)

data class ProductCreationError(
    val index: Int,
    val request: CreateProductRequest,
    val error: String
)
```

## 5.4 Creating a PUT API

PUT APIs are used for updating entire resources. Let's explore different patterns for handling updates.

### 5.4.1 Basic PUT API with ResponseEntity

```kotlin
@RestController
@RequestMapping("/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    /**
     * Update entire product using PUT
     * PUT /api/v1/products/123
     */
    @RequestMapping(value = ["/{id}"], method = [RequestMethod.PUT])
    fun updateProduct(
        @PathVariable id: Long,
        @Valid @RequestBody updateProductRequest: UpdateProductRequest
    ): ResponseEntity<ProductDTO> {
        val updatedProduct = productService.update(id, updateProductRequest)
        return ResponseEntity.ok(updatedProduct)
    }
    
    /**
     * Update with conditional logic
     * PUT /api/v1/products/123?force=true
     */
    @RequestMapping(value = ["/{id}"], method = [RequestMethod.PUT])
    fun updateProductConditional(
        @PathVariable id: Long,
        @Valid @RequestBody updateProductRequest: UpdateProductRequest,
        @RequestParam(defaultValue = "false") force: Boolean
    ): ResponseEntity<ProductDTO> {
        return try {
            val updatedProduct = productService.updateConditional(id, updateProductRequest, force)
            ResponseEntity.ok(updatedProduct)
        } catch (e: ProductNotFoundException) {
            ResponseEntity.notFound().build()
        } catch (e: ProductConflictException) {
            ResponseEntity.status(409).build()
        }
    }
}
```

### 5.4.2 PUT API with Different Response Scenarios

```kotlin
@RestController
@RequestMapping("/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    /**
     * Update with comprehensive response handling
     * PUT /api/v1/products/123
     */
    @RequestMapping(value = ["/{id}"], method = [RequestMethod.PUT])
    fun updateProductComprehensive(
        @PathVariable id: Long,
        @Valid @RequestBody updateProductRequest: UpdateProductRequest
    ): ResponseEntity<Any> {
        return when (val result = productService.updateWithResult(id, updateProductRequest)) {
            is UpdateResult.Success -> ResponseEntity.ok(result.product)
            is UpdateResult.NotFound -> ResponseEntity.notFound().build()
            is UpdateResult.Conflict -> ResponseEntity.status(409).body(
                mapOf("error" to "Product version conflict", "message" to result.message)
            )
            is UpdateResult.ValidationError -> ResponseEntity.badRequest().body(
                mapOf("error" to "Validation failed", "details" to result.errors)
            )
        }
    }
    
    /**
     * Upsert operation (create or update)
     * PUT /api/v1/products/upsert/123
     */
    @RequestMapping(value = ["/upsert/{id}"], method = [RequestMethod.PUT])
    fun upsertProduct(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpsertProductRequest
    ): ResponseEntity<ProductDTO> {
        val result = productService.upsert(id, request)
        val status = if (result.wasCreated) 201 else 200
        return ResponseEntity.status(status).body(result.product)
    }
}
```

### 5.4.3 Update DTOs and Response Types

```kotlin
// src/main/kotlin/com/example/api/dto/UpdateProductRequest.kt
package com.example.api.dto

import jakarta.validation.constraints.*

data class UpdateProductRequest(
    @field:NotBlank(message = "Product name is required")
    @field:Size(min = 2, max = 100, message = "Product name must be between 2 and 100 characters")
    val name: String,
    
    @field:NotBlank(message = "Description is required")
    @field:Size(max = 500, message = "Description cannot exceed 500 characters")
    val description: String,
    
    @field:NotNull(message = "Price is required")
    @field:Min(value = 0, message = "Price cannot be negative")
    val price: Double,
    
    @field:NotBlank(message = "Category is required")
    val category: String,
    
    @field:NotBlank(message = "Brand is required")
    val brand: String,
    
    @field:Min(value = 0, message = "Stock quantity cannot be negative")
    val stockQuantity: Int,
    
    val tags: List<String> = emptyList(),
    
    val version: Long? = null // For optimistic locking
)

data class UpsertProductRequest(
    val name: String,
    val description: String,
    val price: Double,
    val category: String,
    val brand: String,
    val stockQuantity: Int = 0,
    val tags: List<String> = emptyList()
)

sealed class UpdateResult {
    data class Success(val product: ProductDTO) : UpdateResult()
    object NotFound : UpdateResult()
    data class Conflict(val message: String) : UpdateResult()
    data class ValidationError(val errors: List<String>) : UpdateResult()
}

data class UpsertResult(
    val product: ProductDTO,
    val wasCreated: Boolean
)
```

## 5.5 Creating a DELETE API

DELETE APIs are used for removing resources. Let's explore different patterns for handling deletions.

### 5.5.1 DELETE API with PathVariable

```kotlin
@RestController
@RequestMapping("/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    /**
     * Delete product by ID
     * DELETE /api/v1/products/123
     */
    @RequestMapping(value = ["/{id}"], method = [RequestMethod.DELETE])
    fun deleteProduct(@PathVariable id: Long): ResponseEntity<Void> {
        productService.delete(id)
        return ResponseEntity.noContent().build()
    }
    
    /**
     * Delete with confirmation
     * DELETE /api/v1/products/123?confirm=true
     */
    @RequestMapping(value = ["/{id}"], method = [RequestMethod.DELETE])
    fun deleteProductWithConfirmation(
        @PathVariable id: Long,
        @RequestParam(defaultValue = "false") confirm: Boolean
    ): ResponseEntity<Any> {
        if (!confirm) {
            return ResponseEntity.badRequest().body(
                mapOf("error" to "Deletion requires confirmation parameter")
            )
        }
        
        return try {
            productService.delete(id)
            ResponseEntity.noContent().build()
        } catch (e: ProductNotFoundException) {
            ResponseEntity.notFound().build()
        }
    }
}
```

### 5.5.2 DELETE API with RequestParam

```kotlin
@RestController
@RequestMapping("/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    /**
     * Bulk delete by IDs
     * DELETE /api/v1/products?ids=1,2,3
     */
    @RequestMapping(method = [RequestMethod.DELETE])
    fun deleteProducts(@RequestParam ids: List<Long>): ResponseEntity<BulkDeleteResponse> {
        val result = productService.deleteBulk(ids)
        return ResponseEntity.ok(result)
    }
    
    /**
     * Delete by category
     * DELETE /api/v1/products/category?name=electronics&confirm=true
     */
    @RequestMapping(value = ["/category"], method = [RequestMethod.DELETE])
    fun deleteProductsByCategory(
        @RequestParam name: String,
        @RequestParam(defaultValue = "false") confirm: Boolean
    ): ResponseEntity<Any> {
        if (!confirm) {
            return ResponseEntity.badRequest().body(
                mapOf("error" to "Category deletion requires confirmation")
            )
        }
        
        val deletedCount = productService.deleteByCategory(name)
        return ResponseEntity.ok(
            mapOf(
                "category" to name,
                "deletedCount" to deletedCount
            )
        )
    }
    
    /**
     * Soft delete with parameters
     * DELETE /api/v1/products/soft?ids=1,2,3&reason=discontinued
     */
    @RequestMapping(value = ["/soft"], method = [RequestMethod.DELETE])
    fun softDeleteProducts(
        @RequestParam ids: List<Long>,
        @RequestParam(required = false) reason: String?
    ): ResponseEntity<SoftDeleteResponse> {
        val result = productService.softDelete(ids, reason)
        return ResponseEntity.ok(result)
    }
}
```

### 5.5.3 Delete Response DTOs

```kotlin
// src/main/kotlin/com/example/api/dto/DeleteResponses.kt
package com.example.api.dto

import java.time.LocalDateTime

data class BulkDeleteResponse(
    val requestedIds: List<Long>,
    val deletedIds: List<Long>,
    val notFoundIds: List<Long>,
    val errorIds: List<Long>,
    val errors: Map<Long, String>,
    val totalRequested: Int,
    val totalDeleted: Int,
    val totalErrors: Int
)

data class SoftDeleteResponse(
    val deletedProducts: List<SoftDeletedProduct>,
    val errors: List<DeleteError>,
    val totalProcessed: Int,
    val totalDeleted: Int,
    val totalErrors: Int
)

data class SoftDeletedProduct(
    val id: Long,
    val name: String,
    val deletedAt: LocalDateTime,
    val reason: String?
)

data class DeleteError(
    val id: Long,
    val error: String,
    val reason: String?
)
```

## 5.6 [Advanced] Documenting with Swagger

API documentation is crucial for developer experience. Spring Boot integrates seamlessly with OpenAPI (formerly Swagger) to provide interactive API documentation.

### 5.6.1 OpenAPI Configuration

```kotlin
// src/main/kotlin/com/example/api/config/OpenApiConfig.kt
package com.example.api.config

import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.info.Contact
import io.swagger.v3.oas.models.info.Info
import io.swagger.v3.oas.models.info.License
import io.swagger.v3.oas.models.servers.Server
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiConfig {
    
    @Bean
    fun customOpenAPI(): OpenAPI {
        return OpenAPI()
            .info(
                Info()
                    .title("Spring Boot Kotlin API")
                    .description("Comprehensive REST API built with Spring Boot and Kotlin")
                    .version("1.0.0")
                    .contact(
                        Contact()
                            .name("API Support Team")
                            .email("support@example.com")
                            .url("https://example.com/support")
                    )
                    .license(
                        License()
                            .name("Apache 2.0")
                            .url("https://springdoc.org")
                    )
            )
            .servers(
                listOf(
                    Server().url("http://localhost:8080/api").description("Local Development Server"),
                    Server().url("https://staging-api.example.com").description("Staging Server"),
                    Server().url("https://api.example.com").description("Production Server")
                )
            )
    }
}
```

### 5.6.2 Comprehensive API Documentation

```kotlin
// Enhanced ProductController with detailed documentation
package com.example.api.controller

import io.swagger.v3.oas.annotations.*
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.parameters.RequestBody
import io.swagger.v3.oas.annotations.responses.ApiResponse
import io.swagger.v3.oas.annotations.responses.ApiResponses
import io.swagger.v3.oas.annotations.tags.Tag

@RestController
@RequestMapping("/v1/products")
@Tag(
    name = "Products", 
    description = "Product management operations including CRUD operations, search, and bulk operations"
)
class ProductController(
    private val productService: ProductService
) {
    
    @Operation(
        summary = "Get all products",
        description = "Retrieve a paginated list of all products with optional filtering and sorting"
    )
    @ApiResponses(
        value = [
            ApiResponse(
                responseCode = "200",
                description = "Successfully retrieved products",
                content = [Content(
                    mediaType = "application/json",
                    schema = Schema(implementation = PagedProductResponse::class)
                )]
            ),
            ApiResponse(
                responseCode = "400", 
                description = "Invalid request parameters",
                content = [Content()]
            )
        ]
    )
    @GetMapping
    fun getAllProducts(
        @Parameter(description = "Page number (0-based)", example = "0")
        @RequestParam(defaultValue = "0") page: Int,
        
        @Parameter(description = "Page size", example = "10")
        @RequestParam(defaultValue = "10") size: Int,
        
        @Parameter(description = "Filter by product name", example = "iPhone")
        @RequestParam(required = false) name: String?,
        
        @Parameter(description = "Filter by category", example = "electronics")
        @RequestParam(required = false) category: String?,
        
        @Parameter(description = "Sort field", example = "name", schema = Schema(allowableValues = ["id", "name", "price", "createdAt"]))
        @RequestParam(defaultValue = "id") sortBy: String,
        
        @Parameter(description = "Sort direction", example = "ASC", schema = Schema(allowableValues = ["ASC", "DESC"]))
        @RequestParam(defaultValue = "ASC") sortDirection: String
    ): ResponseEntity<PagedProductResponse> {
        // Implementation
        val searchCriteria = ProductSearchCriteria(name, category, page, size, sortBy, sortDirection)
        val result = productService.search(searchCriteria)
        return ResponseEntity.ok(result)
    }
    
    @Operation(
        summary = "Get product by ID",
        description = "Retrieve a specific product by its unique identifier"
    )
    @ApiResponses(
        value = [
            ApiResponse(
                responseCode = "200",
                description = "Product found",
                content = [Content(
                    mediaType = "application/json",
                    schema = Schema(implementation = ProductDTO::class)
                )]
            ),
            ApiResponse(
                responseCode = "404",
                description = "Product not found",
                content = [Content()]
            )
        ]
    )
    @GetMapping("/{id}")
    fun getProductById(
        @Parameter(description = "Product ID", example = "1")
        @PathVariable id: Long
    ): ResponseEntity<ProductDTO> {
        val product = productService.findById(id)
        return ResponseEntity.ok(product)
    }
    
    @Operation(
        summary = "Create new product",
        description = "Create a new product with the provided information"
    )
    @ApiResponses(
        value = [
            ApiResponse(
                responseCode = "201",
                description = "Product created successfully",
                content = [Content(
                    mediaType = "application/json",
                    schema = Schema(implementation = ProductDTO::class)
                )]
            ),
            ApiResponse(
                responseCode = "400",
                description = "Invalid product data",
                content = [Content()]
            ),
            ApiResponse(
                responseCode = "409",
                description = "Product already exists",
                content = [Content()]
            )
        ]
    )
    @PostMapping
    fun createProduct(
        @RequestBody(
            description = "Product creation data",
            required = true,
            content = [Content(schema = Schema(implementation = CreateProductRequest::class))]
        )
        @Valid @org.springframework.web.bind.annotation.RequestBody createProductRequest: CreateProductRequest
    ): ResponseEntity<ProductDTO> {
        val product = productService.create(createProductRequest)
        return ResponseEntity.status(201).body(product)
    }
}
```

### 5.6.3 Schema Documentation

```kotlin
// Enhanced DTOs with schema documentation
package com.example.api.dto

import io.swagger.v3.oas.annotations.media.Schema
import jakarta.validation.constraints.*
import java.time.LocalDateTime

@Schema(description = "Product information")
data class ProductDTO(
    @Schema(description = "Unique product identifier", example = "1")
    val id: Long,
    
    @Schema(description = "Product name", example = "iPhone 13 Pro")
    val name: String,
    
    @Schema(description = "Product description", example = "Latest iPhone with advanced camera system")
    val description: String,
    
    @Schema(description = "Product price in USD", example = "999.99")
    val price: Double,
    
    @Schema(description = "Product category", example = "electronics")
    val category: String,
    
    @Schema(description = "Product brand", example = "Apple")
    val brand: String,
    
    @Schema(description = "Whether product is in stock", example = "true")
    val inStock: Boolean,
    
    @Schema(description = "Available stock quantity", example = "50")
    val stockQuantity: Int,
    
    @Schema(description = "Product tags for categorization", example = "[\"smartphone\", \"premium\", \"latest\"]")
    val tags: List<String> = emptyList(),
    
    @Schema(description = "Product creation timestamp")
    val createdAt: LocalDateTime,
    
    @Schema(description = "Product last update timestamp")
    val updatedAt: LocalDateTime
)

@Schema(description = "Product creation request")
data class CreateProductRequest(
    @Schema(description = "Product name", example = "iPhone 13 Pro", requiredMode = Schema.RequiredMode.REQUIRED)
    @field:NotBlank(message = "Product name is required")
    @field:Size(min = 2, max = 100, message = "Product name must be between 2 and 100 characters")
    val name: String,
    
    @Schema(description = "Product description", example = "Latest iPhone with advanced camera system", requiredMode = Schema.RequiredMode.REQUIRED)
    @field:NotBlank(message = "Description is required")
    @field:Size(max = 500, message = "Description cannot exceed 500 characters")
    val description: String,
    
    @Schema(description = "Product price in USD", example = "999.99", requiredMode = Schema.RequiredMode.REQUIRED)
    @field:NotNull(message = "Price is required")
    @field:Min(value = 0, message = "Price cannot be negative")
    val price: Double,
    
    @Schema(description = "Product category", example = "electronics", requiredMode = Schema.RequiredMode.REQUIRED)
    @field:NotBlank(message = "Category is required")
    val category: String,
    
    @Schema(description = "Product brand", example = "Apple", requiredMode = Schema.RequiredMode.REQUIRED)
    @field:NotBlank(message = "Brand is required")
    val brand: String,
    
    @Schema(description = "Initial stock quantity", example = "100", defaultValue = "0")
    @field:Min(value = 0, message = "Stock quantity cannot be negative")
    val stockQuantity: Int = 0,
    
    @Schema(description = "Product tags", example = "[\"smartphone\", \"premium\"]")
    val tags: List<String> = emptyList()
)
```

## 5.7 [Advanced] Logging with Logback

Proper logging is essential for monitoring, debugging, and maintaining your APIs in production.

### 5.7.1 Logback Configuration

Create `logback-spring.xml` in `src/main/resources`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProfile name="!prod">
        <!-- Development configuration -->
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %highlight(%-5level) %cyan(%logger{36}) - %msg%n</pattern>
            </encoder>
        </appender>
        
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/application.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>logs/application.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
                <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                    <maxFileSize>10MB</maxFileSize>
                </timeBasedFileNamingAndTriggeringPolicy>
                <maxHistory>30</maxHistory>
            </rollingPolicy>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>
    
    <springProfile name="prod">
        <!-- Production configuration -->
        <appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/application.json</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>logs/application.%d{yyyy-MM-dd}.%i.json</fileNamePattern>
                <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                    <maxFileSize>100MB</maxFileSize>
                </timeBasedFileNamingAndTriggeringPolicy>
                <maxHistory>90</maxHistory>
            </rollingPolicy>
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeContext>true</includeContext>
                <includeMdc>true</includeMdc>
            </encoder>
        </appender>
        
        <root level="WARN">
            <appender-ref ref="JSON_FILE"/>
        </root>
    </springProfile>
    
    <!-- Application specific loggers -->
    <logger name="com.example.api" level="DEBUG" additivity="false">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </logger>
    
    <!-- HTTP request logging -->
    <logger name="org.springframework.web.filter.CommonsRequestLoggingFilter" level="DEBUG"/>
    
    <!-- Database query logging -->
    <logger name="org.hibernate.SQL" level="DEBUG"/>
    <logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE"/>
</configuration>
```

### 5.7.2 Request/Response Logging

```kotlin
// src/main/kotlin/com/example/api/config/LoggingConfig.kt
package com.example.api.config

import org.slf4j.LoggerFactory
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.filter.CommonsRequestLoggingFilter

@Configuration
class LoggingConfig {
    
    @Bean
    fun requestLoggingFilter(): CommonsRequestLoggingFilter {
        return CommonsRequestLoggingFilter().apply {
            setIncludeQueryString(true)
            setIncludePayload(true)
            setMaxPayloadLength(10000)
            setIncludeHeaders(false)
            setAfterMessagePrefix("REQUEST DATA : ")
        }
    }
}
```

### 5.7.3 Structured Logging in Controllers

```kotlin
// Enhanced controller with comprehensive logging
package com.example.api.controller

import org.slf4j.LoggerFactory
import org.slf4j.MDC
import org.springframework.web.bind.annotation.*
import java.util.*

@RestController
@RequestMapping("/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    companion object {
        private val logger = LoggerFactory.getLogger(ProductController::class.java)
    }
    
    @GetMapping("/{id}")
    fun getProductById(@PathVariable id: Long): ResponseEntity<ProductDTO> {
        val requestId = UUID.randomUUID().toString()
        
        // Add request context to MDC for structured logging
        MDC.put("requestId", requestId)
        MDC.put("operation", "getProductById")
        MDC.put("productId", id.toString())
        
        return try {
            logger.info("Fetching product with ID: {}", id)
            val startTime = System.currentTimeMillis()
            
            val product = productService.findById(id)
            
            val duration = System.currentTimeMillis() - startTime
            logger.info("Successfully retrieved product with ID: {} in {}ms", id, duration)
            
            // Log business metrics
            logger.info("Product retrieved - name: '{}', category: '{}', price: {}", 
                       product.name, product.category, product.price)
            
            ResponseEntity.ok(product)
        } catch (e: ProductNotFoundException) {
            logger.warn("Product not found with ID: {}", id, e)
            ResponseEntity.notFound().build()
        } catch (e: Exception) {
            logger.error("Error retrieving product with ID: {}", id, e)
            ResponseEntity.internalServerError().build()
        } finally {
            MDC.clear() // Always clear MDC to prevent memory leaks
        }
    }
    
    @PostMapping
    fun createProduct(@Valid @RequestBody createProductRequest: CreateProductRequest): ResponseEntity<ProductDTO> {
        val requestId = UUID.randomUUID().toString()
        
        MDC.put("requestId", requestId)
        MDC.put("operation", "createProduct")
        MDC.put("productName", createProductRequest.name)
        MDC.put("category", createProductRequest.category)
        
        return try {
            logger.info("Creating new product: {}", createProductRequest.name)
            val startTime = System.currentTimeMillis()
            
            val product = productService.create(createProductRequest)
            
            val duration = System.currentTimeMillis() - startTime
            logger.info("Successfully created product with ID: {} in {}ms", product.id, duration)
            
            // Log business metrics
            logger.info("Product created - ID: {}, name: '{}', category: '{}', price: {}", 
                       product.id, product.name, product.category, product.price)
            
            ResponseEntity.status(201).body(product)
        } catch (e: ProductAlreadyExistsException) {
            logger.warn("Attempt to create duplicate product: {}", createProductRequest.name, e)
            ResponseEntity.status(409).build()
        } catch (e: ValidationException) {
            logger.warn("Validation failed for product creation: {}", e.message)
            ResponseEntity.badRequest().build()
        } catch (e: Exception) {
            logger.error("Error creating product: {}", createProductRequest.name, e)
            ResponseEntity.internalServerError().build()
        } finally {
            MDC.clear()
        }
    }
}
```

### 5.7.4 Service Layer Logging

```kotlin
// src/main/kotlin/com/example/api/service/ProductService.kt
package com.example.api.service

import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service

@Service
class ProductService(
    private val productRepository: ProductRepository
) {
    
    companion object {
        private val logger = LoggerFactory.getLogger(ProductService::class.java)
    }
    
    fun findById(id: Long): ProductDTO {
        logger.debug("Searching for product with ID: {}", id)
        
        return productRepository.findById(id)?.let { product ->
            logger.debug("Found product: {} (category: {})", product.name, product.category)
            product.toDTO()
        } ?: run {
            logger.warn("Product not found with ID: {}", id)
            throw ProductNotFoundException("Product not found with ID: $id")
        }
    }
    
    fun create(request: CreateProductRequest): ProductDTO {
        logger.debug("Creating product: {} in category: {}", request.name, request.category)
        
        // Check for duplicates
        productRepository.findByNameAndCategory(request.name, request.category)?.let {
            logger.warn("Duplicate product creation attempt: {} in category: {}", request.name, request.category)
            throw ProductAlreadyExistsException("Product '${request.name}' already exists in category '${request.category}'")
        }
        
        val product = Product.from(request)
        val savedProduct = productRepository.save(product)
        
        logger.info("Product created successfully - ID: {}, name: '{}'", savedProduct.id, savedProduct.name)
        
        return savedProduct.toDTO()
    }
    
    fun search(criteria: ProductSearchCriteria): PagedProductResponse {
        logger.debug("Searching products with criteria: {}", criteria)
        
        val startTime = System.currentTimeMillis()
        val result = productRepository.search(criteria)
        val duration = System.currentTimeMillis() - startTime
        
        logger.info("Search completed in {}ms - found {} products out of {} total", 
                   duration, result.content.size, result.totalElements)
        
        return result
    }
}
```

### 5.7.5 Performance and Audit Logging

```kotlin
// src/main/kotlin/com/example/api/aspect/LoggingAspect.kt
package com.example.api.aspect

import org.aspectj.lang.ProceedingJoinPoint
import org.aspectj.lang.annotation.Around
import org.aspectj.lang.annotation.Aspect
import org.slf4j.LoggerFactory
import org.slf4j.MDC
import org.springframework.stereotype.Component

@Aspect
@Component
class LoggingAspect {
    
    companion object {
        private val logger = LoggerFactory.getLogger(LoggingAspect::class.java)
    }
    
    @Around("@annotation(Timed)")
    fun logExecutionTime(joinPoint: ProceedingJoinPoint): Any? {
        val methodName = joinPoint.signature.name
        val className = joinPoint.signature.declaringType.simpleName
        
        val startTime = System.currentTimeMillis()
        MDC.put("method", "$className.$methodName")
        
        return try {
            logger.debug("Starting execution of {}.{}", className, methodName)
            val result = joinPoint.proceed()
            val duration = System.currentTimeMillis() - startTime
            
            logger.info("Method {}.{} executed in {}ms", className, methodName, duration)
            
            // Log slow operations
            if (duration > 1000) {
                logger.warn("Slow operation detected: {}.{} took {}ms", className, methodName, duration)
            }
            
            result
        } catch (e: Exception) {
            val duration = System.currentTimeMillis() - startTime
            logger.error("Method {}.{} failed after {}ms", className, methodName, duration, e)
            throw e
        } finally {
            MDC.remove("method")
        }
    }
}

@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Timed
```

## 5.8 Summary

In this comprehensive chapter, we've explored the various ways to write APIs with Spring Boot and Kotlin:

### Key Concepts Covered:

1. **Project Configuration**: Setting up a robust foundation with proper dependencies, configuration, and structure for API development.

2. **GET APIs**: 
   - Using `@RequestMapping` with different patterns
   - Handling `@PathVariable` for resource identification
   - Managing `@RequestParam` for filtering and pagination
   - Creating comprehensive DTOs for data transfer

3. **POST APIs**:
   - Implementing resource creation with `@RequestBody`
   - Adding validation with Bean Validation annotations
   - Handling bulk operations and complex request scenarios

4. **PUT APIs**:
   - Full resource updates with proper `ResponseEntity` usage
   - Implementing conditional updates and upsert operations
   - Managing different response scenarios and error handling

5. **DELETE APIs**:
   - Resource deletion with `@PathVariable` and `@RequestParam`
   - Implementing soft deletes and bulk operations
   - Proper response handling for different deletion scenarios

6. **Advanced Documentation with Swagger/OpenAPI**:
   - Comprehensive API documentation setup
   - Detailed schema documentation for DTOs
   - Interactive API testing through Swagger UI

7. **Advanced Logging with Logback**:
   - Structured logging configuration for different environments
   - Request/response logging and MDC for correlation
   - Performance monitoring and audit logging
   - Aspect-oriented logging for cross-cutting concerns

### Best Practices Implemented:

- **Consistent Response Patterns**: Using `ResponseEntity` for proper HTTP status codes
- **Validation**: Comprehensive input validation with meaningful error messages
- **Error Handling**: Proper exception handling with appropriate HTTP responses
- **Documentation**: Self-documenting APIs with detailed OpenAPI annotations
- **Logging**: Structured logging for monitoring, debugging, and audit trails
- **Kotlin Idioms**: Leveraging Kotlin's features like data classes, default parameters, and null safety

### What's Next:

In the next chapter, we'll dive deep into database integration, exploring how to connect your APIs to PostgreSQL using JPA and Hibernate, handling the unique challenges of Kotlin with ORM frameworks, and implementing robust data access patterns.

The foundation we've built in this chapter provides a solid base for creating maintainable, well-documented, and observable REST APIs. These patterns and practices will serve you well as we move into more complex topics like database integration and testing strategies.