---
layout: default
parent: Spring Boot with Kotlin (2025)
nav_exclude: true
---
# Chapter 06: Database Integration
- TOC
{:toc}

In this chapter, we'll explore how to integrate Spring Boot applications with PostgreSQL using Kotlin. We'll cover the challenges of using Kotlin with JPA/Hibernate and provide practical solutions. You'll learn how to set up database connections, design entities, implement repositories, and integrate with your service layer while maintaining type safety and leveraging Kotlin's unique features.

## 6.1 Installing PostgreSQL

Before we can integrate our Spring Boot application with a database, we need to set up PostgreSQL. We'll cover both local installation and containerized approaches.

### 6.1.1 Local PostgreSQL Installation

**macOS (using Homebrew):**
```bash
# Install PostgreSQL
brew install postgresql@14

# Start PostgreSQL service
brew services start postgresql@14

# Create a database user
createuser --interactive --pwprompt api_user

# Create a database
createdb -O api_user api_db
```

**Ubuntu/Debian:**
```bash
# Install PostgreSQL
sudo apt update
sudo apt install postgresql postgresql-contrib

# Switch to postgres user
sudo -i -u postgres

# Create user and database
createuser --interactive --pwprompt api_user
createdb -O api_user api_db
```

**Windows:**
Download and install from [PostgreSQL Official Website](https://www.postgresql.org/download/windows/)

### 6.1.2 Docker-based PostgreSQL Setup

For development consistency, we recommend using Docker:

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: api_db
      POSTGRES_USER: api_user
      POSTGRES_PASSWORD: api_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U api_user -d api_db"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Optional: PgAdmin for database administration
  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "8081:80"
    depends_on:
      - postgres

volumes:
  postgres_data:
```

**Starting the Database:**
```bash
# Start the PostgreSQL container
docker-compose up -d postgres

# Connect to the database
docker exec -it <container_name> psql -U api_user -d api_db
```

### 6.1.3 Database Initialization Scripts

Create initialization scripts in `init-scripts/01-init.sql`:

```sql
-- Create extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Create schemas
CREATE SCHEMA IF NOT EXISTS app_data;
CREATE SCHEMA IF NOT EXISTS audit;

-- Set default search path
ALTER USER api_user SET search_path = app_data, public;

-- Create audit table for tracking changes
CREATE TABLE IF NOT EXISTS audit.audit_log (
    id BIGSERIAL PRIMARY KEY,
    table_name VARCHAR(255) NOT NULL,
    operation VARCHAR(10) NOT NULL,
    old_values JSONB,
    new_values JSONB,
    user_id VARCHAR(255),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create basic tables
CREATE TABLE IF NOT EXISTS app_data.categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample categories
INSERT INTO app_data.categories (name, description) VALUES
('Electronics', 'Electronic devices and accessories'),
('Books', 'Books and publications'),
('Clothing', 'Apparel and fashion items')
ON CONFLICT (name) DO NOTHING;
```

## 6.2 ORM (Kotlin + JPA Challenges, Workarounds)

Working with JPA and Hibernate in Kotlin presents unique challenges due to Kotlin's design principles. Let's explore these challenges and their solutions.

### 6.2.1 The Kotlin-JPA Challenge

**Challenge 1: Final Classes by Default**
```kotlin
// This won't work with JPA - classes are final by default
data class Product(
    val id: Long,
    val name: String,
    val price: Double
)
```

JPA/Hibernate requires classes to be open for proxy creation, but Kotlin classes are final by default.

**Solution: kotlin-jpa Plugin**

Add to your `build.gradle.kts`:
```kotlin
plugins {
    kotlin("plugin.jpa") version "1.9.10"
    kotlin("plugin.spring") version "1.9.10"
}
```

The `kotlin-jpa` plugin automatically opens JPA entities and their properties.

**Challenge 2: No-Args Constructor Requirement**
```kotlin
// JPA requires a no-args constructor, but this doesn't have one
data class Product(
    val name: String,
    val price: Double
)
```

**Solution: kotlin-noarg Plugin**
```kotlin
plugins {
    kotlin("plugin.noarg") version "1.9.10"
}

noArg {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}
```

### 6.2.2 Recommended Entity Design Pattern

```kotlin
// src/main/kotlin/com/example/api/entity/BaseEntity.kt
package com.example.api.entity

import jakarta.persistence.*
import org.springframework.data.annotation.CreatedDate
import org.springframework.data.annotation.LastModifiedDate
import org.springframework.data.jpa.domain.support.AuditingEntityListener
import java.time.LocalDateTime

@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    open val id: Long = 0

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    open var createdAt: LocalDateTime? = null

    @LastModifiedDate
    @Column(name = "updated_at")
    open var updatedAt: LocalDateTime? = null

    @Version
    open var version: Long = 0

    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (javaClass != other?.javaClass) return false
        other as BaseEntity
        return id != 0L && id == other.id
    }

    override fun hashCode(): Int {
        return javaClass.hashCode()
    }
}
```

### 6.2.3 Kotlin-Friendly Entity Design

```kotlin
// src/main/kotlin/com/example/api/entity/Product.kt
package com.example.api.entity

import jakarta.persistence.*

@Entity
@Table(
    name = "products",
    schema = "app_data",
    indexes = [
        Index(name = "idx_product_name", columnList = "name"),
        Index(name = "idx_product_category", columnList = "category_id"),
        Index(name = "idx_product_active", columnList = "active")
    ]
)
class Product : BaseEntity() {

    @Column(name = "name", nullable = false, length = 255)
    var name: String = ""
        protected set

    @Column(name = "description", columnDefinition = "TEXT")
    var description: String = ""
        protected set

    @Column(name = "price", nullable = false, precision = 10, scale = 2)
    var price: Double = 0.0
        protected set

    @Column(name = "brand", nullable = false, length = 100)
    var brand: String = ""
        protected set

    @Column(name = "stock_quantity", nullable = false)
    var stockQuantity: Int = 0
        protected set

    @Column(name = "active", nullable = false)
    var active: Boolean = true
        protected set

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    var category: Category? = null
        protected set

    @ElementCollection
    @CollectionTable(
        name = "product_tags",
        schema = "app_data",
        joinColumns = [JoinColumn(name = "product_id")]
    )
    @Column(name = "tag")
    var tags: MutableSet<String> = mutableSetOf()
        protected set

    // Factory method for creation
    companion object {
        fun create(
            name: String,
            description: String,
            price: Double,
            brand: String,
            stockQuantity: Int,
            category: Category,
            tags: Set<String> = emptySet()
        ): Product {
            return Product().apply {
                this.name = name
                this.description = description
                this.price = price
                this.brand = brand
                this.stockQuantity = stockQuantity
                this.category = category
                this.tags.addAll(tags)
            }
        }
    }

    // Business methods
    fun updateDetails(
        name: String? = null,
        description: String? = null,
        price: Double? = null,
        brand: String? = null
    ) {
        name?.let { this.name = it }
        description?.let { this.description = it }
        price?.let { this.price = it }
        brand?.let { this.brand = it }
    }

    fun updateStock(quantity: Int) {
        require(quantity >= 0) { "Stock quantity cannot be negative" }
        this.stockQuantity = quantity
    }

    fun addTags(newTags: Set<String>) {
        this.tags.addAll(newTags)
    }

    fun removeTags(tagsToRemove: Set<String>) {
        this.tags.removeAll(tagsToRemove)
    }

    fun deactivate() {
        this.active = false
    }

    fun activate() {
        this.active = true
    }

    val isInStock: Boolean
        get() = stockQuantity > 0 && active

    override fun toString(): String {
        return "Product(id=$id, name='$name', price=$price, brand='$brand')"
    }
}
```

### 6.2.4 Category Entity

```kotlin
// src/main/kotlin/com/example/api/entity/Category.kt
package com.example.api.entity

import jakarta.persistence.*

@Entity
@Table(
    name = "categories",
    schema = "app_data",
    indexes = [
        Index(name = "idx_category_name", columnList = "name", unique = true),
        Index(name = "idx_category_active", columnList = "active")
    ]
)
class Category : BaseEntity() {

    @Column(name = "name", nullable = false, unique = true, length = 255)
    var name: String = ""
        protected set

    @Column(name = "description", columnDefinition = "TEXT")
    var description: String = ""
        protected set

    @Column(name = "active", nullable = false)
    var active: Boolean = true
        protected set

    @OneToMany(mappedBy = "category", cascade = [CascadeType.ALL], fetch = FetchType.LAZY)
    var products: MutableList<Product> = mutableListOf()
        protected set

    companion object {
        fun create(name: String, description: String = ""): Category {
            return Category().apply {
                this.name = name
                this.description = description
            }
        }
    }

    fun updateDetails(name: String? = null, description: String? = null) {
        name?.let { this.name = it }
        description?.let { this.description = it }
    }

    fun deactivate() {
        this.active = false
    }

    fun activate() {
        this.active = true
    }

    override fun toString(): String {
        return "Category(id=$id, name='$name', active=$active)"
    }
}
```

## 6.3 JPA and Hibernate (Spring Data JPA)

Spring Data JPA provides a powerful abstraction over JPA, making database operations more convenient and type-safe in Kotlin.

### 6.3.1 JPA Configuration

```kotlin
// src/main/kotlin/com/example/api/config/JpaConfig.kt
package com.example.api.config

import org.springframework.context.annotation.Configuration
import org.springframework.data.jpa.repository.config.EnableJpaAuditing
import org.springframework.data.jpa.repository.config.EnableJpaRepositories
import org.springframework.transaction.annotation.EnableTransactionManagement

@Configuration
@EnableJpaRepositories(basePackages = ["com.example.api.repository"])
@EnableJpaAuditing
@EnableTransactionManagement
class JpaConfig
```

### 6.3.2 Database Configuration

Update your `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/api_db
    username: api_user
    password: api_password
    driver-class-name: org.postgresql.Driver
    hikari:
      connection-timeout: 20000
      maximum-pool-size: 10
      minimum-idle: 5
      idle-timeout: 300000
      max-lifetime: 1200000
      leak-detection-threshold: 60000

  jpa:
    database: postgresql
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: validate  # Use 'create-drop' for development only
      naming:
        physical-strategy: org.hibernate.boot.model.naming.CamelCaseToUnderscoresNamingStrategy
        implicit-strategy: org.springframework.boot.orm.jpa.hibernate.SpringImplicitNamingStrategy
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true
        jdbc:
          batch_size: 25
          order_inserts: true
          order_updates: true
        connection:
          provider_disables_autocommit: true
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory

  # Database migration with Flyway
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
    schemas: app_data
```

### 6.3.3 Flyway Database Migration

Add Flyway to your dependencies:
```kotlin
dependencies {
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
}
```

Create migration files in `src/main/resources/db/migration/`:

**V1__Create_initial_tables.sql:**
```sql
-- Create categories table
CREATE TABLE app_data.categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0
);

-- Create products table
CREATE TABLE app_data.products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    brand VARCHAR(100) NOT NULL,
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    active BOOLEAN NOT NULL DEFAULT true,
    category_id BIGINT NOT NULL,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0,
    CONSTRAINT fk_products_category FOREIGN KEY (category_id) REFERENCES app_data.categories(id)
);

-- Create product_tags table
CREATE TABLE app_data.product_tags (
    product_id BIGINT NOT NULL,
    tag VARCHAR(255) NOT NULL,
    CONSTRAINT fk_product_tags_product FOREIGN KEY (product_id) REFERENCES app_data.products(id) ON DELETE CASCADE
);

-- Create indexes
CREATE INDEX idx_product_name ON app_data.products(name);
CREATE INDEX idx_product_category ON app_data.products(category_id);
CREATE INDEX idx_product_active ON app_data.products(active);
CREATE INDEX idx_category_name ON app_data.categories(name);
CREATE INDEX idx_category_active ON app_data.categories(active);
```

## 6.4 Persistence Context (EntityManager, Entity Lifecycle)

Understanding the persistence context is crucial for effective JPA usage in Kotlin.

### 6.4.1 Entity Lifecycle and States

```kotlin
// src/main/kotlin/com/example/api/service/EntityLifecycleService.kt
package com.example.api.service

import com.example.api.entity.Product
import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional
class EntityLifecycleService {

    @PersistenceContext
    private lateinit var entityManager: EntityManager

    fun demonstrateEntityLifecycle() {
        // 1. Transient state - not associated with persistence context
        val newProduct = Product.create(
            name = "New Product",
            description = "A new product",
            price = 99.99,
            brand = "TestBrand",
            stockQuantity = 10,
            category = findCategory(1L)!!
        )

        println("Is managed: ${entityManager.contains(newProduct)}") // false

        // 2. Persistent state - associated with persistence context
        entityManager.persist(newProduct)
        println("Is managed: ${entityManager.contains(newProduct)}") // true

        // 3. Changes are tracked automatically
        newProduct.updateDetails(name = "Updated Product")
        // No need to call save() - changes are automatically tracked

        // 4. Detached state - entity exists in DB but not managed
        entityManager.detach(newProduct)
        println("Is managed: ${entityManager.contains(newProduct)}") // false

        // Changes to detached entities are not tracked
        newProduct.updateDetails(name = "Detached Update") // This won't be persisted

        // 5. Merge detached entity back to persistent state
        val managedProduct = entityManager.merge(newProduct)
        println("Is managed: ${entityManager.contains(managedProduct)}") // true

        // 6. Remove entity (becomes removed state)
        entityManager.remove(managedProduct)
    }

    private fun findCategory(id: Long) = entityManager.find(com.example.api.entity.Category::class.java, id)
}
```

### 6.4.2 Custom Repository with EntityManager

```kotlin
// src/main/kotlin/com/example/api/repository/custom/CustomProductRepository.kt
package com.example.api.repository.custom

import com.example.api.entity.Product
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable

interface CustomProductRepository {
    fun findByComplexCriteria(
        name: String?,
        minPrice: Double?,
        maxPrice: Double?,
        categoryIds: List<Long>,
        tags: List<String>,
        pageable: Pageable
    ): Page<Product>

    fun findProductsWithStockBelow(threshold: Int): List<Product>
    fun updatePricesInCategory(categoryId: Long, percentage: Double): Int
    fun findTopSellingProducts(limit: Int): List<Product>
}
```

```kotlin
// src/main/kotlin/com/example/api/repository/custom/CustomProductRepositoryImpl.kt
package com.example.api.repository.custom

import com.example.api.entity.Product
import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import jakarta.persistence.criteria.*
import org.springframework.data.domain.Page
import org.springframework.data.domain.PageImpl
import org.springframework.data.domain.Pageable

class CustomProductRepositoryImpl : CustomProductRepository {

    @PersistenceContext
    private lateinit var entityManager: EntityManager

    override fun findByComplexCriteria(
        name: String?,
        minPrice: Double?,
        maxPrice: Double?,
        categoryIds: List<Long>,
        tags: List<String>,
        pageable: Pageable
    ): Page<Product> {
        val cb = entityManager.criteriaBuilder
        val query = cb.createQuery(Product::class.java)
        val root = query.from(Product::class.java)

        val predicates = mutableListOf<Predicate>()

        // Name search (case-insensitive partial match)
        name?.let { searchName ->
            predicates.add(
                cb.like(
                    cb.lower(root.get("name")),
                    "%${searchName.lowercase()}%"
                )
            )
        }

        // Price range
        minPrice?.let { min ->
            predicates.add(cb.greaterThanOrEqualTo(root.get("price"), min))
        }

        maxPrice?.let { max ->
            predicates.add(cb.lessThanOrEqualTo(root.get("price"), max))
        }

        // Category filter
        if (categoryIds.isNotEmpty()) {
            predicates.add(root.get<Any>("category").get<Any>("id").`in`(categoryIds))
        }

        // Tag search
        if (tags.isNotEmpty()) {
            val tagJoin = root.join<Product, String>("tags")
            predicates.add(tagJoin.`in`(tags))
        }

        // Always filter active products
        predicates.add(cb.isTrue(root.get("active")))

        // Apply predicates
        if (predicates.isNotEmpty()) {
            query.where(cb.and(*predicates.toTypedArray()))
        }

        // Apply ordering
        val orders = pageable.sort.map { order ->
            val path = root.get<Any>(order.property)
            if (order.isAscending) cb.asc(path) else cb.desc(path)
        }
        query.orderBy(orders)

        // Execute query with pagination
        val typedQuery = entityManager.createQuery(query)
        typedQuery.firstResult = pageable.offset.toInt()
        typedQuery.maxResults = pageable.pageSize

        val results = typedQuery.resultList

        // Count query for total elements
        val countQuery = cb.createQuery(Long::class.java)
        val countRoot = countQuery.from(Product::class.java)
        countQuery.select(cb.count(countRoot))

        if (predicates.isNotEmpty()) {
            countQuery.where(cb.and(*predicates.toTypedArray()))
        }

        val total = entityManager.createQuery(countQuery).singleResult

        return PageImpl(results, pageable, total)
    }

    override fun findProductsWithStockBelow(threshold: Int): List<Product> {
        return entityManager.createQuery("""
            SELECT p FROM Product p
            WHERE p.stockQuantity < :threshold
            AND p.active = true
            ORDER BY p.stockQuantity ASC
        """, Product::class.java)
            .setParameter("threshold", threshold)
            .resultList
    }

    override fun updatePricesInCategory(categoryId: Long, percentage: Double): Int {
        return entityManager.createQuery("""
            UPDATE Product p
            SET p.price = p.price * :multiplier
            WHERE p.category.id = :categoryId
        """)
            .setParameter("multiplier", 1.0 + (percentage / 100.0))
            .setParameter("categoryId", categoryId)
            .executeUpdate()
    }

    override fun findTopSellingProducts(limit: Int): List<Product> {
        // This would typically involve a join with an orders table
        // For demonstration, we'll return products with highest stock
        return entityManager.createQuery("""
            SELECT p FROM Product p
            WHERE p.active = true
            ORDER BY p.stockQuantity DESC
        """, Product::class.java)
            .setMaxResults(limit)
            .resultList
    }
}
```

## 6.5 Database Connection Setup

Let's configure robust database connection management with connection pooling and monitoring.

### 6.5.1 Advanced DataSource Configuration

```kotlin
// src/main/kotlin/com/example/api/config/DatabaseConfig.kt
package com.example.api.config

import com.zaxxer.hikari.HikariConfig
import com.zaxxer.hikari.HikariDataSource
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Primary
import javax.sql.DataSource

@Configuration
class DatabaseConfig {

    @Bean
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.hikari")
    fun hikariConfig(): HikariConfig {
        return HikariConfig().apply {
            // Connection pool settings
            maximumPoolSize = 20
            minimumIdle = 5
            connectionTimeout = 30000
            idleTimeout = 600000
            maxLifetime = 1800000
            leakDetectionThreshold = 60000

            // Performance settings
            isAutoCommit = false
            connectionTestQuery = "SELECT 1"

            // Monitoring
            poolName = "SpringBootKotlinHikariCP"
            isRegisterMbeans = true
        }
    }

    @Bean
    @Primary
    fun dataSource(config: HikariConfig): DataSource {
        return HikariDataSource(config)
    }
}
```

### 6.5.2 Health Check Configuration

```kotlin
// src/main/kotlin/com/example/api/health/DatabaseHealthIndicator.kt
package com.example.api.health

import org.springframework.boot.actuator.health.Health
import org.springframework.boot.actuator.health.HealthIndicator
import org.springframework.stereotype.Component
import javax.sql.DataSource

@Component
class DatabaseHealthIndicator(private val dataSource: DataSource) : HealthIndicator {

    override fun health(): Health {
        return try {
            dataSource.connection.use { connection ->
                val isValid = connection.isValid(5)
                if (isValid) {
                    Health.up()
                        .withDetail("database", "PostgreSQL")
                        .withDetail("validationQuery", "SELECT 1")
                        .build()
                } else {
                    Health.down()
                        .withDetail("error", "Database connection validation failed")
                        .build()
                }
            }
        } catch (e: Exception) {
            Health.down(e)
                .withDetail("error", e.message)
                .build()
        }
    }
}
```

## 6.6 Designing Entities (Kotlin data class style)

While we can't use data classes directly for JPA entities, we can create Kotlin-friendly entity designs that maintain immutability principles.

### 6.6.1 Value Objects and Embeddables

```kotlin
// src/main/kotlin/com/example/api/entity/vo/Money.kt
package com.example.api.entity.vo

import jakarta.persistence.Embeddable
import java.math.BigDecimal
import java.util.Currency

@Embeddable
data class Money(
    val amount: BigDecimal = BigDecimal.ZERO,
    val currency: String = "USD"
) {
    init {
        require(amount >= BigDecimal.ZERO) { "Amount cannot be negative" }
        require(currency.isNotBlank()) { "Currency cannot be blank" }
    }

    val currencyInstance: Currency
        get() = Currency.getInstance(currency)

    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return copy(amount = amount + other.amount)
    }

    operator fun minus(other: Money): Money {
        require(currency == other.currency) { "Cannot subtract different currencies" }
        return copy(amount = amount - other.amount)
    }

    operator fun times(multiplier: BigDecimal): Money {
        return copy(amount = amount * multiplier)
    }

    companion object {
        fun usd(amount: Double): Money = Money(BigDecimal.valueOf(amount), "USD")
        fun eur(amount: Double): Money = Money(BigDecimal.valueOf(amount), "EUR")
    }
}
```

```kotlin
// src/main/kotlin/com/example/api/entity/vo/Address.kt
package com.example.api.entity.vo

import jakarta.persistence.Embeddable

@Embeddable
data class Address(
    val street: String = "",
    val city: String = "",
    val state: String = "",
    val zipCode: String = "",
    val country: String = ""
) {
    val isComplete: Boolean
        get() = street.isNotBlank() && city.isNotBlank() &&
                state.isNotBlank() && zipCode.isNotBlank() && country.isNotBlank()

    override fun toString(): String {
        return "$street, $city, $state $zipCode, $country"
    }
}
```

### 6.6.2 Enhanced Product Entity with Value Objects

```kotlin
// src/main/kotlin/com/example/api/entity/ProductV2.kt
package com.example.api.entity

import com.example.api.entity.vo.Money
import jakarta.persistence.*
import java.time.LocalDate

@Entity
@Table(
    name = "products_v2",
    schema = "app_data",
    uniqueConstraints = [
        UniqueConstraint(columnNames = ["name", "brand"])
    ]
)
class ProductV2 : BaseEntity() {

    @Column(name = "name", nullable = false)
    var name: String = ""
        protected set

    @Column(name = "description", columnDefinition = "TEXT")
    var description: String = ""
        protected set

    @Embedded
    @AttributeOverrides(
        AttributeOverride(name = "amount", column = Column(name = "price_amount")),
        AttributeOverride(name = "currency", column = Column(name = "price_currency"))
    )
    var price: Money = Money.usd(0.0)
        protected set

    @Column(name = "brand", nullable = false)
    var brand: String = ""
        protected set

    @Column(name = "stock_quantity", nullable = false)
    var stockQuantity: Int = 0
        protected set

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    var status: ProductStatus = ProductStatus.DRAFT
        protected set

    @Column(name = "launch_date")
    var launchDate: LocalDate? = null
        protected set

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    var category: Category? = null
        protected set

    @ElementCollection
    @CollectionTable(name = "product_specifications", joinColumns = [JoinColumn(name = "product_id")])
    @MapKeyColumn(name = "spec_name")
    @Column(name = "spec_value")
    var specifications: MutableMap<String, String> = mutableMapOf()
        protected set

    companion object {
        fun create(
            name: String,
            description: String,
            price: Money,
            brand: String,
            category: Category,
            launchDate: LocalDate? = null
        ): ProductV2 {
            require(name.isNotBlank()) { "Product name cannot be blank" }
            require(brand.isNotBlank()) { "Brand cannot be blank" }

            return ProductV2().apply {
                this.name = name
                this.description = description
                this.price = price
                this.brand = brand
                this.category = category
                this.launchDate = launchDate
                this.status = ProductStatus.DRAFT
            }
        }
    }

    fun updatePrice(newPrice: Money) {
        require(newPrice.amount > BigDecimal.ZERO) { "Price must be positive" }
        this.price = newPrice
    }

    fun addSpecification(name: String, value: String) {
        require(name.isNotBlank()) { "Specification name cannot be blank" }
        specifications[name] = value
    }

    fun removeSpecification(name: String) {
        specifications.remove(name)
    }

    fun launch(date: LocalDate = LocalDate.now()) {
        require(status == ProductStatus.DRAFT) { "Only draft products can be launched" }
        this.status = ProductStatus.ACTIVE
        this.launchDate = date
    }

    fun discontinue() {
        require(status == ProductStatus.ACTIVE) { "Only active products can be discontinued" }
        this.status = ProductStatus.DISCONTINUED
    }

    fun updateStock(quantity: Int) {
        require(quantity >= 0) { "Stock quantity cannot be negative" }
        this.stockQuantity = quantity
    }

    val isAvailable: Boolean
        get() = status == ProductStatus.ACTIVE && stockQuantity > 0

    val isLaunched: Boolean
        get() = launchDate != null && !launchDate!!.isAfter(LocalDate.now())
}

enum class ProductStatus {
    DRAFT, ACTIVE, DISCONTINUED, OUT_OF_STOCK
}
```

## 6.7 Repository Interfaces

Spring Data JPA provides powerful repository abstractions that work well with Kotlin's type system.

### 6.7.1 Basic Repository Interfaces

```kotlin
// src/main/kotlin/com/example/api/repository/CategoryRepository.kt
package com.example.api.repository

import com.example.api.entity.Category
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository

@Repository
interface CategoryRepository : JpaRepository<Category, Long> {

    // Query methods using method name conventions
    fun findByName(name: String): Category?
    fun findByNameIgnoreCase(name: String): Category?
    fun findByActiveTrue(): List<Category>
    fun existsByNameIgnoreCase(name: String): Boolean

    // Custom queries with JPQL
    @Query("SELECT c FROM Category c WHERE c.active = true ORDER BY c.name")
    fun findAllActive(): List<Category>

    @Query("SELECT c FROM Category c WHERE SIZE(c.products) > :minProducts")
    fun findCategoriesWithMinimumProducts(@Param("minProducts") minProducts: Int): List<Category>

    // Native queries
    @Query(
        value = """
            SELECT c.* FROM app_data.categories c
            WHERE c.active = true
            AND EXISTS (SELECT 1 FROM app_data.products p WHERE p.category_id = c.id)
        """,
        nativeQuery = true
    )
    fun findActiveCategoriesWithProducts(): List<Category>
}
```

```kotlin
// src/main/kotlin/com/example/api/repository/ProductRepository.kt
package com.example.api.repository

import com.example.api.entity.Product
import com.example.api.entity.ProductStatus
import com.example.api.repository.custom.CustomProductRepository
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Modifying
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import java.time.LocalDateTime

@Repository
interface ProductRepository : JpaRepository<Product, Long>, CustomProductRepository {

    // Basic finder methods
    fun findByActiveTrue(pageable: Pageable): Page<Product>
    fun findByBrand(brand: String): List<Product>
    fun findByBrandAndActiveTrue(brand: String): List<Product>

    // Complex queries
    fun findByNameContainingIgnoreCaseAndActiveTrue(name: String, pageable: Pageable): Page<Product>
    fun findByCategoryIdAndActiveTrue(categoryId: Long, pageable: Pageable): Page<Product>
    fun findByPriceBetweenAndActiveTrue(minPrice: Double, maxPrice: Double): List<Product>

    // Existence checks
    fun existsByNameAndBrand(name: String, brand: String): Boolean
    fun existsByCategoryIdAndActiveTrue(categoryId: Long): Boolean

    // Custom JPQL queries
    @Query("""
        SELECT p FROM Product p
        WHERE p.active = true
        AND (:categoryId IS NULL OR p.category.id = :categoryId)
        AND (:name IS NULL OR LOWER(p.name) LIKE LOWER(CONCAT('%', :name, '%')))
        AND (:minPrice IS NULL OR p.price >= :minPrice)
        AND (:maxPrice IS NULL OR p.price <= :maxPrice)
        ORDER BY p.createdAt DESC
    """)
    fun findBySearchCriteria(
        @Param("categoryId") categoryId: Long?,
        @Param("name") name: String?,
        @Param("minPrice") minPrice: Double?,
        @Param("maxPrice") maxPrice: Double?,
        pageable: Pageable
    ): Page<Product>

    @Query("SELECT p FROM Product p WHERE p.stockQuantity < :threshold AND p.active = true")
    fun findLowStockProducts(@Param("threshold") threshold: Int): List<Product>

    @Query("""
        SELECT p FROM Product p
        WHERE p.active = true
        AND SIZE(p.tags) > 0
        AND EXISTS (
            SELECT t FROM p.tags t WHERE t IN :tags
        )
    """)
    fun findByTagsIn(@Param("tags") tags: List<String>): List<Product>

    // Aggregation queries
    @Query("SELECT COUNT(p) FROM Product p WHERE p.category.id = :categoryId AND p.active = true")
    fun countActiveByCategoryId(@Param("categoryId") categoryId: Long): Long

    @Query("SELECT AVG(p.price) FROM Product p WHERE p.category.id = :categoryId AND p.active = true")
    fun averagePriceByCategoryId(@Param("categoryId") categoryId: Long): Double?

    // Update queries
    @Modifying
    @Query("UPDATE Product p SET p.active = false WHERE p.id IN :ids")
    fun deactivateByIds(@Param("ids") ids: List<Long>): Int

    @Modifying
    @Query("UPDATE Product p SET p.stockQuantity = p.stockQuantity + :quantity WHERE p.id = :id")
    fun updateStock(@Param("id") id: Long, @Param("quantity") quantity: Int): Int

    // Date-based queries
    fun findByCreatedAtAfter(date: LocalDateTime): List<Product>
    fun findByCreatedAtBetween(start: LocalDateTime, end: LocalDateTime): List<Product>
}
```

### 6.7.2 Projection Interfaces

```kotlin
// src/main/kotlin/com/example/api/repository/projection/ProductProjections.kt
package com.example.api.repository.projection

import java.time.LocalDateTime

// Interface-based projections
interface ProductSummary {
    val id: Long
    val name: String
    val price: Double
    val brand: String
    val categoryName: String
    val inStock: Boolean
}

interface ProductStatistics {
    val categoryId: Long
    val categoryName: String
    val productCount: Long
    val averagePrice: Double
    val totalStock: Long
}

// Closed projections with computed properties
interface ProductWithCategory {
    val id: Long
    val name: String
    val price: Double
    val category: CategoryInfo

    // Computed property
    val displayName: String
        get() = "$name ($${String.format("%.2f", price)})"

    interface CategoryInfo {
        val id: Long
        val name: String
    }
}

// Class-based projections for complex operations
data class ProductReportDto(
    val id: Long,
    val name: String,
    val brand: String,
    val price: Double,
    val categoryName: String,
    val stockQuantity: Int,
    val createdAt: LocalDateTime,
    val tags: List<String>
) {
    val stockStatus: String
        get() = when {
            stockQuantity == 0 -> "OUT_OF_STOCK"
            stockQuantity < 10 -> "LOW_STOCK"
            stockQuantity < 50 -> "MEDIUM_STOCK"
            else -> "HIGH_STOCK"
        }

    val priceCategory: String
        get() = when {
            price < 25.0 -> "BUDGET"
            price < 100.0 -> "MID_RANGE"
            price < 500.0 -> "PREMIUM"
            else -> "LUXURY"
        }
}
```

### 6.7.3 Repository with Projections

```kotlin
// Enhanced repository with projections
@Repository
interface ProductRepository : JpaRepository<Product, Long>, CustomProductRepository {

    // Interface-based projections
    @Query("""
        SELECT p.id as id, p.name as name, p.price as price, p.brand as brand,
               c.name as categoryName, (p.stockQuantity > 0) as inStock
        FROM Product p JOIN p.category c
        WHERE p.active = true
    """)
    fun findAllProductSummaries(): List<ProductSummary>

    @Query("""
        SELECT c.id as categoryId, c.name as categoryName,
               COUNT(p) as productCount, AVG(p.price) as averagePrice,
               SUM(p.stockQuantity) as totalStock
        FROM Product p JOIN p.category c
        WHERE p.active = true
        GROUP BY c.id, c.name
    """)
    fun getProductStatistics(): List<ProductStatistics>

    // Class-based projections with constructor expression
    @Query("""
        SELECT new com.example.api.repository.projection.ProductReportDto(
            p.id, p.name, p.brand, p.price, c.name,
            p.stockQuantity, p.createdAt, p.tags
        )
        FROM Product p JOIN p.category c
        WHERE p.active = true
        ORDER BY p.createdAt DESC
    """)
    fun getProductReports(): List<ProductReportDto>

    // Dynamic projections
    fun <T> findByActiveTrue(type: Class<T>): List<T>
    fun <T> findByCategoryId(categoryId: Long, type: Class<T>): List<T>
}
```

## 6.8 DAO and Service Layer Integration

Let's create a comprehensive service layer that integrates with our repositories while maintaining clean architecture principles.

### 6.8.1 Service Layer Design

```kotlin
// src/main/kotlin/com/example/api/service/ProductService.kt
package com.example.api.service

import com.example.api.dto.*
import com.example.api.entity.Product
import com.example.api.exception.ProductNotFoundException
import com.example.api.exception.CategoryNotFoundException
import com.example.api.repository.CategoryRepository
import com.example.api.repository.ProductRepository
import com.example.api.repository.projection.ProductSummary
import com.example.api.service.mapper.ProductMapper
import org.slf4j.LoggerFactory
import org.springframework.data.domain.Page
import org.springframework.data.domain.PageRequest
import org.springframework.data.domain.Sort
import org.springframework.data.repository.findByIdOrNull
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDateTime

@Service
@Transactional(readOnly = true)
class ProductService(
    private val productRepository: ProductRepository,
    private val categoryRepository: CategoryRepository,
    private val productMapper: ProductMapper
) {

    companion object {
        private val logger = LoggerFactory.getLogger(ProductService::class.java)
    }

    // Read operations
    fun findById(id: Long): ProductDTO {
        logger.debug("Finding product by ID: {}", id)

        val product = productRepository.findByIdOrNull(id)
            ?: throw ProductNotFoundException("Product not found with ID: $id")

        return productMapper.toDto(product)
    }

    fun findAll(pageable: PageRequest): Page<ProductDTO> {
        logger.debug("Finding all products with pagination: {}", pageable)

        return productRepository.findByActiveTrue(pageable)
            .map(productMapper::toDto)
    }

    fun findByCategory(categoryId: Long, pageable: PageRequest): Page<ProductDTO> {
        logger.debug("Finding products by category ID: {} with pagination: {}", categoryId, pageable)

        if (!categoryRepository.existsById(categoryId)) {
            throw CategoryNotFoundException("Category not found with ID: $categoryId")
        }

        return productRepository.findByCategoryIdAndActiveTrue(categoryId, pageable)
            .map(productMapper::toDto)
    }

    fun search(criteria: ProductSearchCriteriaDTO): Page<ProductDTO> {
        logger.debug("Searching products with criteria: {}", criteria)

        val pageable = PageRequest.of(
            criteria.page,
            criteria.size,
            Sort.by(
                if (criteria.sortDirection == "DESC") Sort.Direction.DESC else Sort.Direction.ASC,
                criteria.sortBy
            )
        )

        return productRepository.findByComplexCriteria(
            name = criteria.name,
            minPrice = criteria.minPrice,
            maxPrice = criteria.maxPrice,
            categoryIds = criteria.categoryIds,
            tags = criteria.tags,
            pageable = pageable
        ).map(productMapper::toDto)
    }

    fun findProductSummaries(): List<ProductSummary> {
        logger.debug("Finding all product summaries")
        return productRepository.findAllProductSummaries()
    }

    fun findLowStockProducts(threshold: Int = 10): List<ProductDTO> {
        logger.debug("Finding low stock products with threshold: {}", threshold)

        return productRepository.findLowStockProducts(threshold)
            .map(productMapper::toDto)
    }

    // Write operations
    @Transactional
    fun create(request: CreateProductRequest): ProductDTO {
        logger.info("Creating new product: {}", request.name)

        // Validate category exists
        val category = categoryRepository.findByIdOrNull(request.categoryId)
            ?: throw CategoryNotFoundException("Category not found with ID: ${request.categoryId}")

        // Check for duplicate
        if (productRepository.existsByNameAndBrand(request.name, request.brand)) {
            throw IllegalArgumentException("Product with name '${request.name}' and brand '${request.brand}' already exists")
        }

        val product = Product.create(
            name = request.name,
            description = request.description,
            price = request.price,
            brand = request.brand,
            stockQuantity = request.stockQuantity,
            category = category,
            tags = request.tags.toSet()
        )

        val savedProduct = productRepository.save(product)
        logger.info("Product created successfully with ID: {}", savedProduct.id)

        return productMapper.toDto(savedProduct)
    }

    @Transactional
    fun update(id: Long, request: UpdateProductRequest): ProductDTO {
        logger.info("Updating product with ID: {}", id)

        val product = productRepository.findByIdOrNull(id)
            ?: throw ProductNotFoundException("Product not found with ID: $id")

        // Update category if provided
        request.categoryId?.let { categoryId ->
            val category = categoryRepository.findByIdOrNull(categoryId)
                ?: throw CategoryNotFoundException("Category not found with ID: $categoryId")
            product.category = category
        }

        // Update product details
        product.updateDetails(
            name = request.name,
            description = request.description,
            price = request.price,
            brand = request.brand
        )

        request.stockQuantity?.let { stock ->
            product.updateStock(stock)
        }

        // Update tags
        request.tags?.let { newTags ->
            product.tags.clear()
            product.addTags(newTags.toSet())
        }

        val savedProduct = productRepository.save(product)
        logger.info("Product updated successfully: {}", savedProduct.id)

        return productMapper.toDto(savedProduct)
    }

    @Transactional
    fun updateStock(id: Long, quantity: Int): ProductDTO {
        logger.info("Updating stock for product ID: {} to quantity: {}", id, quantity)

        val updatedRows = productRepository.updateStock(id, quantity)
        if (updatedRows == 0) {
            throw ProductNotFoundException("Product not found with ID: $id")
        }

        // Fetch and return updated product
        return findById(id)
    }

    @Transactional
    fun delete(id: Long) {
        logger.info("Deleting product with ID: {}", id)

        val product = productRepository.findByIdOrNull(id)
            ?: throw ProductNotFoundException("Product not found with ID: $id")

        product.deactivate()
        productRepository.save(product)

        logger.info("Product deactivated successfully: {}", id)
    }

    @Transactional
    fun bulkDelete(ids: List<Long>): BulkOperationResult {
        logger.info("Bulk deleting products with IDs: {}", ids)

        val deactivatedCount = productRepository.deactivateByIds(ids)

        return BulkOperationResult(
            totalRequested = ids.size,
            successful = deactivatedCount,
            failed = ids.size - deactivatedCount
        )
    }

    // Analytics and reporting
    fun getProductStatistics(): ProductStatisticsDTO {
        logger.debug("Generating product statistics")

        val stats = productRepository.getProductStatistics()
        val totalProducts = productRepository.count()
        val activeProducts = productRepository.countByActiveTrue()

        return ProductStatisticsDTO(
            totalProducts = totalProducts,
            activeProducts = activeProducts,
            categoriesWithProducts = stats.size.toLong(),
            categoryStatistics = stats.map { stat ->
                CategoryStatisticsDTO(
                    categoryId = stat.categoryId,
                    categoryName = stat.categoryName,
                    productCount = stat.productCount,
                    averagePrice = stat.averagePrice,
                    totalStock = stat.totalStock
                )
            }
        )
    }

    fun findRecentProducts(days: Int = 7): List<ProductDTO> {
        logger.debug("Finding products created in last {} days", days)

        val cutoffDate = LocalDateTime.now().minusDays(days.toLong())
        return productRepository.findByCreatedAtAfter(cutoffDate)
            .map(productMapper::toDto)
    }
}
```

### 6.8.2 Category Service

```kotlin
// src/main/kotlin/com/example/api/service/CategoryService.kt
package com.example.api.service

import com.example.api.dto.*
import com.example.api.entity.Category
import com.example.api.exception.CategoryNotFoundException
import com.example.api.repository.CategoryRepository
import com.example.api.service.mapper.CategoryMapper
import org.slf4j.LoggerFactory
import org.springframework.data.repository.findByIdOrNull
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional(readOnly = true)
class CategoryService(
    private val categoryRepository: CategoryRepository,
    private val categoryMapper: CategoryMapper
) {

    companion object {
        private val logger = LoggerFactory.getLogger(CategoryService::class.java)
    }

    fun findAll(): List<CategoryDTO> {
        logger.debug("Finding all categories")
        return categoryRepository.findAll().map(categoryMapper::toDto)
    }

    fun findAllActive(): List<CategoryDTO> {
        logger.debug("Finding all active categories")
        return categoryRepository.findByActiveTrue().map(categoryMapper::toDto)
    }

    fun findById(id: Long): CategoryDTO {
        logger.debug("Finding category by ID: {}", id)

        val category = categoryRepository.findByIdOrNull(id)
            ?: throw CategoryNotFoundException("Category not found with ID: $id")

        return categoryMapper.toDto(category)
    }

    fun findByName(name: String): CategoryDTO? {
        logger.debug("Finding category by name: {}", name)

        return categoryRepository.findByNameIgnoreCase(name)
            ?.let(categoryMapper::toDto)
    }

    @Transactional
    fun create(request: CreateCategoryRequest): CategoryDTO {
        logger.info("Creating new category: {}", request.name)

        if (categoryRepository.existsByNameIgnoreCase(request.name)) {
            throw IllegalArgumentException("Category with name '${request.name}' already exists")
        }

        val category = Category.create(
            name = request.name,
            description = request.description
        )

        val savedCategory = categoryRepository.save(category)
        logger.info("Category created successfully with ID: {}", savedCategory.id)

        return categoryMapper.toDto(savedCategory)
    }

    @Transactional
    fun update(id: Long, request: UpdateCategoryRequest): CategoryDTO {
        logger.info("Updating category with ID: {}", id)

        val category = categoryRepository.findByIdOrNull(id)
            ?: throw CategoryNotFoundException("Category not found with ID: $id")

        // Check for name conflicts if name is being changed
        if (request.name != null && request.name != category.name) {
            if (categoryRepository.existsByNameIgnoreCase(request.name)) {
                throw IllegalArgumentException("Category with name '${request.name}' already exists")
            }
        }

        category.updateDetails(
            name = request.name,
            description = request.description
        )

        val savedCategory = categoryRepository.save(category)
        logger.info("Category updated successfully: {}", savedCategory.id)

        return categoryMapper.toDto(savedCategory)
    }

    @Transactional
    fun delete(id: Long) {
        logger.info("Deleting category with ID: {}", id)

        val category = categoryRepository.findByIdOrNull(id)
            ?: throw CategoryNotFoundException("Category not found with ID: $id")

        // Check if category has active products
        if (category.products.any { it.active }) {
            throw IllegalStateException("Cannot delete category with active products")
        }

        category.deactivate()
        categoryRepository.save(category)

        logger.info("Category deactivated successfully: {}", id)
    }
}
```

### 6.8.3 Mapper Classes

```kotlin
// src/main/kotlin/com/example/api/service/mapper/ProductMapper.kt
package com.example.api.service.mapper

import com.example.api.dto.ProductDTO
import com.example.api.entity.Product
import org.springframework.stereotype.Component

@Component
class ProductMapper(
    private val categoryMapper: CategoryMapper
) {

    fun toDto(entity: Product): ProductDTO {
        return ProductDTO(
            id = entity.id,
            name = entity.name,
            description = entity.description,
            price = entity.price,
            brand = entity.brand,
            stockQuantity = entity.stockQuantity,
            inStock = entity.isInStock,
            active = entity.active,
            tags = entity.tags.toList(),
            category = entity.category?.let(categoryMapper::toDto),
            createdAt = entity.createdAt!!,
            updatedAt = entity.updatedAt!!
        )
    }

    fun toDtoList(entities: List<Product>): List<ProductDTO> {
        return entities.map(::toDto)
    }
}

@Component
class CategoryMapper {

    fun toDto(entity: Category): CategoryDTO {
        return CategoryDTO(
            id = entity.id,
            name = entity.name,
            description = entity.description,
            active = entity.active,
            productCount = entity.products.count { it.active }.toLong(),
            createdAt = entity.createdAt!!,
            updatedAt = entity.updatedAt!!
        )
    }

    fun toDtoList(entities: List<Category>): List<CategoryDTO> {
        return entities.map(::toDto)
    }
}
```

## 6.9 Swagger-Based Verification

Let's enhance our API documentation to include database-specific operations and integrate it with our service layer.

### 6.9.1 Enhanced Controller with Database Operations

```kotlin
// src/main/kotlin/com/example/api/controller/DatabaseProductController.kt
package com.example.api.controller

import com.example.api.dto.*
import com.example.api.service.ProductService
import io.swagger.v3.oas.annotations.*
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import io.swagger.v3.oas.annotations.responses.ApiResponses
import io.swagger.v3.oas.annotations.tags.Tag
import jakarta.validation.Valid
import org.springframework.data.domain.Page
import org.springframework.data.domain.PageRequest
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/v1/products")
@Tag(
    name = "Products (Database Operations)",
    description = "Product management with full database integration"
)
class DatabaseProductController(
    private val productService: ProductService
) {

    @Operation(
        summary = "Search products with complex criteria",
        description = "Search products using multiple criteria including price range, categories, and tags"
    )
    @ApiResponses(
        value = [
            ApiResponse(
                responseCode = "200",
                description = "Search completed successfully",
                content = [Content(
                    mediaType = "application/json",
                    schema = Schema(implementation = Page::class)
                )]
            )
        ]
    )
    @PostMapping("/search")
    fun searchProducts(
        @Valid @RequestBody criteria: ProductSearchCriteriaDTO
    ): ResponseEntity<Page<ProductDTO>> {
        val results = productService.search(criteria)
        return ResponseEntity.ok(results)
    }

    @Operation(
        summary = "Get low stock products",
        description = "Retrieve products with stock below the specified threshold"
    )
    @GetMapping("/low-stock")
    fun getLowStockProducts(
        @Parameter(description = "Stock threshold", example = "10")
        @RequestParam(defaultValue = "10") threshold: Int
    ): ResponseEntity<List<ProductDTO>> {
        val lowStockProducts = productService.findLowStockProducts(threshold)
        return ResponseEntity.ok(lowStockProducts)
    }

    @Operation(
        summary = "Update product stock",
        description = "Update the stock quantity for a specific product"
    )
    @PatchMapping("/{id}/stock")
    fun updateStock(
        @Parameter(description = "Product ID", example = "1")
        @PathVariable id: Long,

        @Parameter(description = "New stock quantity", example = "100")
        @RequestParam quantity: Int
    ): ResponseEntity<ProductDTO> {
        val updatedProduct = productService.updateStock(id, quantity)
        return ResponseEntity.ok(updatedProduct)
    }

    @Operation(
        summary = "Bulk delete products",
        description = "Deactivate multiple products by their IDs"
    )
    @DeleteMapping("/bulk")
    fun bulkDelete(
        @Parameter(description = "List of product IDs to delete", example = "[1, 2, 3]")
        @RequestParam ids: List<Long>
    ): ResponseEntity<BulkOperationResult> {
        val result = productService.bulkDelete(ids)
        return ResponseEntity.ok(result)
    }

    @Operation(
        summary = "Get product statistics",
        description = "Retrieve comprehensive statistics about products and categories"
    )
    @GetMapping("/statistics")
    fun getStatistics(): ResponseEntity<ProductStatisticsDTO> {
        val statistics = productService.getProductStatistics()
        return ResponseEntity.ok(statistics)
    }

    @Operation(
        summary = "Get recent products",
        description = "Retrieve products created within the specified number of days"
    )
    @GetMapping("/recent")
    fun getRecentProducts(
        @Parameter(description = "Number of days to look back", example = "7")
        @RequestParam(defaultValue = "7") days: Int
    ): ResponseEntity<List<ProductDTO>> {
        val recentProducts = productService.findRecentProducts(days)
        return ResponseEntity.ok(recentProducts)
    }
}
```

### 6.9.2 Enhanced DTOs with Database Constraints

```kotlin
// src/main/kotlin/com/example/api/dto/DatabaseDTOs.kt
package com.example.api.dto

import io.swagger.v3.oas.annotations.media.Schema
import jakarta.validation.constraints.*
import java.time.LocalDateTime

@Schema(description = "Product search criteria with database-optimized filters")
data class ProductSearchCriteriaDTO(
    @Schema(description = "Product name search (partial match)", example = "iPhone")
    val name: String? = null,

    @Schema(description = "Minimum price", example = "50.0")
    @field:DecimalMin(value = "0.0", message = "Minimum price cannot be negative")
    val minPrice: Double? = null,

    @Schema(description = "Maximum price", example = "1000.0")
    @field:DecimalMin(value = "0.0", message = "Maximum price cannot be negative")
    val maxPrice: Double? = null,

    @Schema(description = "Category IDs to filter by", example = "[1, 2, 3]")
    val categoryIds: List<Long> = emptyList(),

    @Schema(description = "Product tags to search for", example = "[\"premium\", \"smartphone\"]")
    val tags: List<String> = emptyList(),

    @Schema(description = "Page number (0-based)", example = "0")
    @field:Min(value = 0, message = "Page number cannot be negative")
    val page: Int = 0,

    @Schema(description = "Page size", example = "20")
    @field:Min(value = 1, message = "Page size must be at least 1")
    @field:Max(value = 100, message = "Page size cannot exceed 100")
    val size: Int = 20,

    @Schema(description = "Sort field", example = "name", allowableValues = ["id", "name", "price", "createdAt", "stockQuantity"])
    val sortBy: String = "id",

    @Schema(description = "Sort direction", example = "ASC", allowableValues = ["ASC", "DESC"])
    val sortDirection: String = "ASC"
) {
    init {
        require(maxPrice == null || minPrice == null || maxPrice >= minPrice) {
            "Maximum price must be greater than or equal to minimum price"
        }
    }
}

@Schema(description = "Product statistics aggregated from database")
data class ProductStatisticsDTO(
    @Schema(description = "Total number of products in database", example = "1500")
    val totalProducts: Long,

    @Schema(description = "Number of active products", example = "1200")
    val activeProducts: Long,

    @Schema(description = "Number of categories with products", example = "25")
    val categoriesWithProducts: Long,

    @Schema(description = "Statistics by category")
    val categoryStatistics: List<CategoryStatisticsDTO>
)

@Schema(description = "Category-specific product statistics")
data class CategoryStatisticsDTO(
    @Schema(description = "Category ID", example = "1")
    val categoryId: Long,

    @Schema(description = "Category name", example = "Electronics")
    val categoryName: String,

    @Schema(description = "Number of products in category", example = "150")
    val productCount: Long,

    @Schema(description = "Average price in category", example = "299.99")
    val averagePrice: Double,

    @Schema(description = "Total stock in category", example = "5000")
    val totalStock: Long
)

@Schema(description = "Result of bulk operations")
data class BulkOperationResult(
    @Schema(description = "Total number of items requested for operation", example = "10")
    val totalRequested: Int,

    @Schema(description = "Number of items successfully processed", example = "8")
    val successful: Int,

    @Schema(description = "Number of items that failed", example = "2")
    val failed: Int
) {
    @Schema(description = "Success rate percentage", example = "80.0")
    val successRate: Double
        get() = if (totalRequested > 0) (successful.toDouble() / totalRequested) * 100 else 0.0
}

@Schema(description = "Enhanced product DTO with database relationships")
data class ProductDTO(
    @Schema(description = "Product ID", example = "1")
    val id: Long,

    @Schema(description = "Product name", example = "iPhone 13 Pro")
    val name: String,

    @Schema(description = "Product description", example = "Latest iPhone with advanced camera")
    val description: String,

    @Schema(description = "Product price", example = "999.99")
    val price: Double,

    @Schema(description = "Product brand", example = "Apple")
    val brand: String,

    @Schema(description = "Available stock quantity", example = "50")
    val stockQuantity: Int,

    @Schema(description = "Whether product is in stock", example = "true")
    val inStock: Boolean,

    @Schema(description = "Whether product is active", example = "true")
    val active: Boolean,

    @Schema(description = "Product tags", example = "[\"smartphone\", \"premium\"]")
    val tags: List<String>,

    @Schema(description = "Product category")
    val category: CategoryDTO?,

    @Schema(description = "Creation timestamp")
    val createdAt: LocalDateTime,

    @Schema(description = "Last update timestamp")
    val updatedAt: LocalDateTime
)

@Schema(description = "Category information")
data class CategoryDTO(
    @Schema(description = "Category ID", example = "1")
    val id: Long,

    @Schema(description = "Category name", example = "Electronics")
    val name: String,

    @Schema(description = "Category description", example = "Electronic devices and accessories")
    val description: String,

    @Schema(description = "Whether category is active", example = "true")
    val active: Boolean,

    @Schema(description = "Number of active products in category", example = "150")
    val productCount: Long,

    @Schema(description = "Creation timestamp")
    val createdAt: LocalDateTime,

    @Schema(description = "Last update timestamp")
    val updatedAt: LocalDateTime
)
```

### 6.9.3 Integration Test for Swagger Documentation

```kotlin
// src/test/kotlin/com/example/api/controller/SwaggerIntegrationTest.kt
package com.example.api.controller

import com.example.api.entity.Category
import com.example.api.entity.Product
import com.example.api.repository.CategoryRepository
import com.example.api.repository.ProductRepository
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureWebMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.TestPropertySource
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get
import org.springframework.transaction.annotation.Transactional

@SpringBootTest
@AutoConfigureWebMvc
@TestPropertySource(properties = ["springdoc.api-docs.enabled=true"])
@Transactional
class SwaggerIntegrationTest {

    @Autowired
    lateinit var mockMvc: MockMvc

    @Autowired
    lateinit var categoryRepository: CategoryRepository

    @Autowired
    lateinit var productRepository: ProductRepository

    @BeforeEach
    fun setup() {
        // Create test data
        val category = Category.create("Electronics", "Electronic devices")
        val savedCategory = categoryRepository.save(category)

        val product = Product.create(
            name = "Test Product",
            description = "A test product",
            price = 99.99,
            brand = "TestBrand",
            stockQuantity = 10,
            category = savedCategory
        )
        productRepository.save(product)
    }

    @Test
    fun `should generate OpenAPI documentation`() {
        mockMvc.get("/v3/api-docs") {
        }.andExpect {
            status { isOk() }
            content { contentType("application/json") }
            jsonPath("$.openapi") { value("3.0.1") }
            jsonPath("$.info.title") { value("Spring Boot Kotlin API") }
        }
    }

    @Test
    fun `should serve Swagger UI`() {
        mockMvc.get("/swagger-ui.html") {
        }.andExpect {
            status { isOk() }
        }
    }

    @Test
    fun `should document product search endpoint`() {
        mockMvc.get("/v3/api-docs") {
        }.andExpect {
            status { isOk() }
            jsonPath("$.paths./v1/products/search.post") { exists() }
            jsonPath("$.paths./v1/products/search.post.summary") {
                value("Search products with complex criteria")
            }
        }
    }
}
```

## Summary

In this comprehensive chapter, we've explored database integration with Spring Boot and Kotlin:

### Key Concepts Covered:

1. **PostgreSQL Installation**: Local and containerized setup with Docker Compose, including database initialization scripts and health checks.

2. **Kotlin-JPA Challenges and Solutions**:
   - Understanding the final class and no-args constructor challenges
   - Using kotlin-jpa and kotlin-noarg plugins
   - Designing Kotlin-friendly entity classes

3. **JPA and Hibernate Configuration**:
   - Advanced JPA configuration with connection pooling
   - Flyway database migrations
   - Performance optimization with second-level caching

4. **Persistence Context Understanding**:
   - Entity lifecycle management (transient, persistent, detached, removed)
   - Custom repository implementations with EntityManager
   - Criteria API usage for complex queries

5. **Database Connection Setup**:
   - HikariCP connection pool configuration
   - Database health checks and monitoring
   - Production-ready connection management

6. **Kotlin-Friendly Entity Design**:
   - Base entity patterns with auditing
   - Value objects and embeddables
   - Factory methods and business logic encapsulation
   - Proper equals/hashCode implementation

7. **Repository Interfaces**:
   - Spring Data JPA repository patterns
   - Custom query methods and JPQL
   - Projections for optimized data retrieval
   - Bulk operations and aggregations

8. **Service Layer Integration**:
   - Transaction management and error handling
   - Comprehensive service implementations
   - Mapping between entities and DTOs
   - Business logic encapsulation

9. **Swagger-Based Verification**:
   - Enhanced API documentation for database operations
   - Database-aware DTOs and validation
   - Integration testing for documentation

### Best Practices Implemented:

- **Entity Design**: Proper encapsulation with protected setters and factory methods
- **Transaction Management**: Read-only transactions for queries and proper transactional boundaries
- **Error Handling**: Custom exceptions and proper error propagation
- **Performance**: Optimized queries, projections, and connection pooling
- **Type Safety**: Leveraging Kotlin's null safety and type system
- **Testing**: Integration tests for database operations

### What's Next:

In the next chapter, we'll explore comprehensive testing strategies for your Spring Boot Kotlin application, including unit tests, integration tests, and test-driven development practices. We'll cover testing the database layer, service layer, and web layer with modern testing frameworks and tools.

The database integration patterns we've established in this chapter provide a solid foundation for building robust, scalable applications with proper data persistence and retrieval capabilities.