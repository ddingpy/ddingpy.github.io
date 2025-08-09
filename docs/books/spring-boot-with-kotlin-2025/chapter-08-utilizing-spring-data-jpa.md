# Chapter 08: Utilizing Spring Data JPA

Spring Data JPA is a powerful abstraction layer that simplifies database access while providing sophisticated query capabilities. In this chapter, we'll explore advanced features that go beyond basic CRUD operations, helping you leverage the full power of Spring Data JPA with Kotlin.

## 8.1 Creating the Project

Let's create a project that showcases advanced Spring Data JPA features. We'll build a book management system that demonstrates various query techniques.

### Project Setup

First, create a new Spring Boot project with these dependencies:

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    
    // Kotlin support
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    
    // Database
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    
    // Querydsl
    implementation("com.querydsl:querydsl-jpa:5.0.0:jakarta")
    kapt("com.querydsl:querydsl-apt:5.0.0:jakarta")
    
    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.testcontainers:postgresql:1.19.3")
    testImplementation("io.mockk:mockk:1.13.8")
}
```

### Domain Model

Let's create a comprehensive domain model for our book management system:

```kotlin
package com.example.bookstore.domain

import jakarta.persistence.*
import org.hibernate.annotations.CreationTimestamp
import org.hibernate.annotations.UpdateTimestamp
import java.math.BigDecimal
import java.time.Instant
import java.time.LocalDate

@Entity
@Table(name = "books")
class Book(
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "book_seq")
    @SequenceGenerator(name = "book_seq", sequenceName = "book_sequence", allocationSize = 1)
    var id: Long? = null,
    
    @Column(nullable = false, unique = true)
    var isbn: String,
    
    @Column(nullable = false)
    var title: String,
    
    @Column(length = 2000)
    var description: String? = null,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    var author: Author,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "publisher_id")
    var publisher: Publisher? = null,
    
    @Column(nullable = false)
    var publicationDate: LocalDate,
    
    @Column(nullable = false, precision = 10, scale = 2)
    var price: BigDecimal,
    
    @Column(nullable = false)
    var stockQuantity: Int = 0,
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    var status: BookStatus = BookStatus.AVAILABLE,
    
    @ElementCollection
    @CollectionTable(name = "book_categories", joinColumns = [JoinColumn(name = "book_id")])
    @Column(name = "category")
    var categories: MutableSet<String> = mutableSetOf(),
    
    @OneToMany(mappedBy = "book", cascade = [CascadeType.ALL], orphanRemoval = true)
    var reviews: MutableList<Review> = mutableListOf(),
    
    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    var createdAt: Instant? = null,
    
    @UpdateTimestamp
    @Column(nullable = false)
    var updatedAt: Instant? = null,
    
    @Version
    var version: Long? = null
) {
    fun addReview(review: Review) {
        reviews.add(review)
        review.book = this
    }
    
    fun removeReview(review: Review) {
        reviews.remove(review)
        review.book = null
    }
    
    fun getAverageRating(): Double {
        return if (reviews.isEmpty()) 0.0
        else reviews.map { it.rating }.average()
    }
}

@Entity
@Table(name = "authors")
class Author(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var firstName: String,
    
    @Column(nullable = false)
    var lastName: String,
    
    @Column(unique = true)
    var email: String? = null,
    
    @Column(length = 1000)
    var biography: String? = null,
    
    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    var books: MutableList<Book> = mutableListOf()
) {
    val fullName: String
        get() = "$firstName $lastName"
}

@Entity
@Table(name = "publishers")
class Publisher(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false, unique = true)
    var name: String,
    
    @Column(nullable = false)
    var country: String,
    
    @OneToMany(mappedBy = "publisher", fetch = FetchType.LAZY)
    var books: MutableList<Book> = mutableListOf()
)

@Entity
@Table(name = "reviews")
class Review(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "book_id")
    var book: Book? = null,
    
    @Column(nullable = false)
    var reviewerName: String,
    
    @Column(nullable = false)
    var rating: Int,
    
    @Column(length = 2000)
    var comment: String? = null,
    
    @CreationTimestamp
    @Column(nullable = false)
    var reviewDate: Instant? = null
)

enum class BookStatus {
    AVAILABLE,
    OUT_OF_STOCK,
    DISCONTINUED,
    COMING_SOON
}
```

## 8.2 JPQL

JPQL (Java Persistence Query Language) is JPA's query language that operates on entities rather than tables. It's database-independent and type-safe when used with Spring Data JPA.

### Basic JPQL Queries

```kotlin
package com.example.bookstore.repository

import com.example.bookstore.domain.Book
import com.example.bookstore.domain.BookStatus
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import java.math.BigDecimal
import java.time.LocalDate

interface BookRepository : JpaRepository<Book, Long> {
    
    // Simple JPQL query
    @Query("SELECT b FROM Book b WHERE b.title = :title")
    fun findByTitleJpql(@Param("title") title: String): List<Book>
    
    // JPQL with JOIN FETCH to avoid N+1 problem
    @Query("""
        SELECT DISTINCT b FROM Book b 
        LEFT JOIN FETCH b.author 
        LEFT JOIN FETCH b.publisher 
        WHERE b.status = :status
    """)
    fun findByStatusWithAuthorAndPublisher(@Param("status") status: BookStatus): List<Book>
    
    // JPQL with complex conditions
    @Query("""
        SELECT b FROM Book b 
        WHERE b.price BETWEEN :minPrice AND :maxPrice 
        AND b.publicationDate >= :fromDate 
        AND b.stockQuantity > 0
        ORDER BY b.price ASC
    """)
    fun findAvailableBooksByPriceRange(
        @Param("minPrice") minPrice: BigDecimal,
        @Param("maxPrice") maxPrice: BigDecimal,
        @Param("fromDate") fromDate: LocalDate
    ): List<Book>
    
    // JPQL with aggregation
    @Query("""
        SELECT b.author.id, COUNT(b), AVG(b.price) 
        FROM Book b 
        GROUP BY b.author.id 
        HAVING COUNT(b) > :minBooks
    """)
    fun findAuthorStatistics(@Param("minBooks") minBooks: Long): List<Array<Any>>
    
    // JPQL with subquery
    @Query("""
        SELECT b FROM Book b 
        WHERE b.price > (
            SELECT AVG(b2.price) FROM Book b2 
            WHERE b2.author = b.author
        )
    """)
    fun findBooksAboveAuthorAveragePrice(): List<Book>
    
    // JPQL with constructor expression (DTO projection)
    @Query("""
        SELECT NEW com.example.bookstore.dto.BookSummary(
            b.id, b.isbn, b.title, b.author.fullName, b.price
        )
        FROM Book b
        WHERE b.status = :status
    """)
    fun findBookSummariesByStatus(@Param("status") status: BookStatus): List<BookSummary>
    
    // JPQL with CASE expression
    @Query("""
        SELECT b.title,
            CASE 
                WHEN b.price < 20 THEN 'Budget'
                WHEN b.price < 50 THEN 'Standard'
                ELSE 'Premium'
            END as priceCategory
        FROM Book b
    """)
    fun categorizeBooksByPrice(): List<Array<Any>>
    
    // JPQL with collection operations
    @Query("""
        SELECT b FROM Book b 
        WHERE :category MEMBER OF b.categories
    """)
    fun findBooksInCategory(@Param("category") category: String): List<Book>
    
    // JPQL with IS EMPTY check
    @Query("SELECT b FROM Book b WHERE b.reviews IS EMPTY")
    fun findBooksWithoutReviews(): List<Book>
}

// DTO for projection
data class BookSummary(
    val id: Long?,
    val isbn: String,
    val title: String,
    val authorName: String,
    val price: BigDecimal
)
```

### Advanced JPQL Features

```kotlin
interface AdvancedBookRepository : JpaRepository<Book, Long> {
    
    // Using TRIM, UPPER, LOWER functions
    @Query("""
        SELECT b FROM Book b 
        WHERE UPPER(TRIM(b.title)) LIKE UPPER(CONCAT('%', :keyword, '%'))
    """)
    fun searchBooksIgnoreCase(@Param("keyword") keyword: String): List<Book>
    
    // Using temporal functions
    @Query("""
        SELECT b FROM Book b 
        WHERE YEAR(b.publicationDate) = :year 
        AND MONTH(b.publicationDate) = :month
    """)
    fun findBooksPublishedInMonth(
        @Param("year") year: Int,
        @Param("month") month: Int
    ): List<Book>
    
    // Using EXISTS
    @Query("""
        SELECT a FROM Author a 
        WHERE EXISTS (
            SELECT b FROM Book b 
            WHERE b.author = a 
            AND b.price > :minPrice
        )
    """)
    fun findAuthorsWithExpensiveBooks(@Param("minPrice") minPrice: BigDecimal): List<Author>
    
    // Using ALL, ANY, SOME
    @Query("""
        SELECT b FROM Book b 
        WHERE b.price < ALL (
            SELECT b2.price FROM Book b2 
            WHERE b2.author.id = :authorId
        )
    """)
    fun findBooksCheaperThanAllByAuthor(@Param("authorId") authorId: Long): List<Book>
}
```

## 8.3 Query Methods

Spring Data JPA's query methods allow you to define queries by simply declaring method names following naming conventions.

### Method Name Query Creation

```kotlin
interface BookQueryMethodRepository : JpaRepository<Book, Long> {
    
    // Simple property expressions
    fun findByTitle(title: String): List<Book>
    fun findByIsbn(isbn: String): Book?
    fun findByTitleContaining(keyword: String): List<Book>
    fun findByTitleStartingWith(prefix: String): List<Book>
    fun findByPriceLessThan(price: BigDecimal): List<Book>
    fun findByPriceBetween(minPrice: BigDecimal, maxPrice: BigDecimal): List<Book>
    
    // Multiple properties with logical operators
    fun findByTitleAndAuthorLastName(title: String, lastName: String): List<Book>
    fun findByTitleOrIsbn(title: String, isbn: String): List<Book>
    fun findByStatusAndPriceLessThan(status: BookStatus, price: BigDecimal): List<Book>
    
    // Nested properties
    fun findByAuthorFirstName(firstName: String): List<Book>
    fun findByAuthorEmailIsNull(): List<Book>
    fun findByPublisherCountry(country: String): List<Book>
    
    // Collection operations
    fun findByCategoriesContaining(category: String): List<Book>
    fun findByReviewsIsEmpty(): List<Book>
    fun findByReviewsIsNotEmpty(): List<Book>
    
    // Date comparisons
    fun findByPublicationDateBefore(date: LocalDate): List<Book>
    fun findByPublicationDateAfter(date: LocalDate): List<Book>
    fun findByCreatedAtBetween(start: Instant, end: Instant): List<Book>
    
    // Ordering
    fun findByStatusOrderByPriceAsc(status: BookStatus): List<Book>
    fun findByAuthorIdOrderByPublicationDateDesc(authorId: Long): List<Book>
    
    // Limiting results
    fun findTop3ByOrderByPriceDesc(): List<Book>
    fun findFirst5ByStatusOrderByCreatedAtDesc(status: BookStatus): List<Book>
    
    // Distinct
    fun findDistinctByAuthorLastName(lastName: String): List<Book>
    
    // Counting and existence
    fun countByStatus(status: BookStatus): Long
    fun existsByIsbn(isbn: String): Boolean
    
    // Delete operations
    fun deleteByStatus(status: BookStatus): Long
    fun removeByPublicationDateBefore(date: LocalDate): List<Book>
}
```

### Advanced Query Method Features

```kotlin
interface AdvancedQueryMethodRepository : JpaRepository<Book, Long> {
    
    // Using Optional for null safety
    fun findOptionalByIsbn(isbn: String): Optional<Book>
    
    // Returning Stream for large datasets
    @Query("SELECT b FROM Book b")
    fun streamAllBooks(): Stream<Book>
    
    // Async queries with CompletableFuture
    @Async
    fun findByTitleContainingAsync(keyword: String): CompletableFuture<List<Book>>
    
    // Using Slice for pagination without count query
    fun findByStatus(status: BookStatus, pageable: Pageable): Slice<Book>
    
    // Custom return types with projections
    fun findByStatusAndPriceLessThan(
        status: BookStatus,
        price: BigDecimal
    ): List<BookProjection>
    
    // Dynamic projections
    fun <T> findByIsbn(isbn: String, type: Class<T>): T?
}

// Projection interface
interface BookProjection {
    fun getTitle(): String
    fun getIsbn(): String
    fun getAuthor(): AuthorProjection
    
    interface AuthorProjection {
        fun getFirstName(): String
        fun getLastName(): String
    }
}

// Closed projection with computed property
interface BookWithRating {
    fun getTitle(): String
    fun getReviews(): List<Review>
    
    @Value("#{target.reviews.size() > 0 ? target.reviews.stream().mapToInt(r -> r.rating).average().orElse(0.0) : 0.0}")
    fun getAverageRating(): Double
}
```

## 8.4 Sorting and Paging

Sorting and pagination are essential for handling large datasets efficiently.

### Basic Sorting and Paging

```kotlin
@RestController
@RequestMapping("/api/books")
class BookController(
    private val bookRepository: BookRepository
) {
    
    // Simple pagination
    @GetMapping
    fun getAllBooks(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int
    ): Page<Book> {
        val pageable = PageRequest.of(page, size)
        return bookRepository.findAll(pageable)
    }
    
    // Pagination with sorting
    @GetMapping("/sorted")
    fun getBooksSorted(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int,
        @RequestParam(defaultValue = "title") sortBy: String,
        @RequestParam(defaultValue = "ASC") direction: Sort.Direction
    ): Page<Book> {
        val pageable = PageRequest.of(page, size, Sort.by(direction, sortBy))
        return bookRepository.findAll(pageable)
    }
    
    // Multiple sort criteria
    @GetMapping("/multi-sorted")
    fun getBooksMultiSorted(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int
    ): Page<Book> {
        val sort = Sort.by(
            Sort.Order.asc("status"),
            Sort.Order.desc("price"),
            Sort.Order.asc("title")
        )
        val pageable = PageRequest.of(page, size, sort)
        return bookRepository.findAll(pageable)
    }
    
    // Dynamic sorting with type safety
    @GetMapping("/dynamic-sorted")
    fun getBooksDynamicSorted(
        @PageableDefault(size = 20, sort = ["title"], direction = Sort.Direction.ASC)
        pageable: Pageable
    ): Page<Book> {
        return bookRepository.findAll(pageable)
    }
}
```

### Advanced Pagination Patterns

```kotlin
@Service
class BookPaginationService(
    private val bookRepository: BookRepository
) {
    
    // Custom page response with metadata
    fun getPagedBooks(pageable: Pageable): PagedResponse<BookDTO> {
        val page = bookRepository.findAll(pageable)
        
        return PagedResponse(
            content = page.content.map { it.toDTO() },
            pageNumber = page.number,
            pageSize = page.size,
            totalElements = page.totalElements,
            totalPages = page.totalPages,
            first = page.isFirst,
            last = page.isLast,
            hasNext = page.hasNext(),
            hasPrevious = page.hasPrevious()
        )
    }
    
    // Slice-based pagination (no count query)
    fun getSlicedBooks(pageable: Pageable): SliceResponse<BookDTO> {
        val slice = bookRepository.findSliceBy(pageable)
        
        return SliceResponse(
            content = slice.content.map { it.toDTO() },
            pageNumber = slice.number,
            pageSize = slice.size,
            hasNext = slice.hasNext(),
            hasPrevious = slice.hasPrevious()
        )
    }
    
    // Cursor-based pagination for better performance
    fun getCursorBasedBooks(
        cursor: Long? = null,
        limit: Int = 20
    ): CursorResponse<BookDTO> {
        val books = if (cursor == null) {
            bookRepository.findTopNByOrderByIdAsc(limit + 1)
        } else {
            bookRepository.findByIdGreaterThanOrderByIdAsc(cursor, PageRequest.of(0, limit + 1))
        }
        
        val hasMore = books.size > limit
        val items = if (hasMore) books.dropLast(1) else books
        val nextCursor = if (hasMore) items.lastOrNull()?.id else null
        
        return CursorResponse(
            items = items.map { it.toDTO() },
            nextCursor = nextCursor,
            hasMore = hasMore
        )
    }
}

// Response DTOs
data class PagedResponse<T>(
    val content: List<T>,
    val pageNumber: Int,
    val pageSize: Int,
    val totalElements: Long,
    val totalPages: Int,
    val first: Boolean,
    val last: Boolean,
    val hasNext: Boolean,
    val hasPrevious: Boolean
)

data class SliceResponse<T>(
    val content: List<T>,
    val pageNumber: Int,
    val pageSize: Int,
    val hasNext: Boolean,
    val hasPrevious: Boolean
)

data class CursorResponse<T>(
    val items: List<T>,
    val nextCursor: Long?,
    val hasMore: Boolean
)
```

## 8.5 @Query Annotation

The @Query annotation provides flexibility for complex queries while maintaining type safety.

### Native Queries

```kotlin
interface NativeQueryRepository : JpaRepository<Book, Long> {
    
    // Basic native query
    @Query(
        value = "SELECT * FROM books WHERE price < :maxPrice",
        nativeQuery = true
    )
    fun findCheapBooksNative(@Param("maxPrice") maxPrice: BigDecimal): List<Book>
    
    // Native query with complex SQL
    @Query(
        value = """
            WITH book_stats AS (
                SELECT 
                    author_id,
                    AVG(price) as avg_price,
                    COUNT(*) as book_count
                FROM books
                GROUP BY author_id
            )
            SELECT b.* 
            FROM books b
            JOIN book_stats bs ON b.author_id = bs.author_id
            WHERE b.price > bs.avg_price
            AND bs.book_count > 2
        """,
        nativeQuery = true
    )
    fun findBooksAboveAuthorAverageWithMinBooks(): List<Book>
    
    // Native query with pagination
    @Query(
        value = """
            SELECT * FROM books b
            WHERE b.publication_date >= :fromDate
            ORDER BY b.price DESC
        """,
        countQuery = "SELECT COUNT(*) FROM books WHERE publication_date >= :fromDate",
        nativeQuery = true
    )
    fun findRecentBooksNative(
        @Param("fromDate") fromDate: LocalDate,
        pageable: Pageable
    ): Page<Book>
    
    // Native query with result mapping
    @Query(
        value = """
            SELECT 
                a.id as author_id,
                a.first_name || ' ' || a.last_name as author_name,
                COUNT(b.id) as book_count,
                AVG(b.price) as avg_price,
                MAX(b.price) as max_price
            FROM authors a
            LEFT JOIN books b ON a.id = b.author_id
            GROUP BY a.id, a.first_name, a.last_name
        """,
        nativeQuery = true
    )
    fun getAuthorStatisticsNative(): List<AuthorStats>
}

// DTO for native query results
interface AuthorStats {
    fun getAuthorId(): Long
    fun getAuthorName(): String
    fun getBookCount(): Int
    fun getAvgPrice(): BigDecimal?
    fun getMaxPrice(): BigDecimal?
}
```

### Modifying Queries

```kotlin
interface ModifyingQueryRepository : JpaRepository<Book, Long> {
    
    // Update query
    @Modifying
    @Query("UPDATE Book b SET b.status = :status WHERE b.stockQuantity = 0")
    fun updateOutOfStockBooks(@Param("status") status: BookStatus): Int
    
    // Bulk price update
    @Modifying
    @Query("""
        UPDATE Book b 
        SET b.price = b.price * :multiplier 
        WHERE b.publisher.id = :publisherId
    """)
    fun adjustPricesByPublisher(
        @Param("publisherId") publisherId: Long,
        @Param("multiplier") multiplier: BigDecimal
    ): Int
    
    // Delete query
    @Modifying
    @Query("DELETE FROM Book b WHERE b.status = :status AND b.createdAt < :before")
    fun deleteOldBooksByStatus(
        @Param("status") status: BookStatus,
        @Param("before") before: Instant
    ): Int
    
    // Native update with returning
    @Modifying
    @Query(
        value = """
            UPDATE books 
            SET stock_quantity = stock_quantity - :quantity,
                updated_at = CURRENT_TIMESTAMP
            WHERE id = :bookId 
            AND stock_quantity >= :quantity
            RETURNING stock_quantity
        """,
        nativeQuery = true
    )
    fun decrementStock(
        @Param("bookId") bookId: Long,
        @Param("quantity") quantity: Int
    ): Int?
}

// Service using modifying queries
@Service
@Transactional
class BookInventoryService(
    private val modifyingRepository: ModifyingQueryRepository
) {
    
    fun markOutOfStockBooks(): Int {
        return modifyingRepository.updateOutOfStockBooks(BookStatus.OUT_OF_STOCK)
    }
    
    fun applyPublisherDiscount(publisherId: Long, discountPercent: Int): Int {
        val multiplier = BigDecimal.ONE - (BigDecimal(discountPercent) / BigDecimal(100))
        return modifyingRepository.adjustPricesByPublisher(publisherId, multiplier)
    }
    
    @Transactional(isolation = Isolation.READ_COMMITTED)
    fun purchaseBook(bookId: Long, quantity: Int): Boolean {
        val remainingStock = modifyingRepository.decrementStock(bookId, quantity)
        return remainingStock != null
    }
}
```

## 8.6 Querydsl

Querydsl provides a type-safe way to construct dynamic queries programmatically.

### Querydsl Configuration

```kotlin
// Configuration class for Querydsl
@Configuration
class QuerydslConfiguration {
    
    @Bean
    fun querydslJpaPredicateExecutor(): JPAQueryFactory {
        return JPAQueryFactory(entityManager())
    }
    
    @PersistenceContext
    private lateinit var entityManager: EntityManager
    
    private fun entityManager(): EntityManager = entityManager
}
```

### Using Querydsl

```kotlin
// Repository with Querydsl support
interface BookQuerydslRepository : JpaRepository<Book, Long>, QuerydslPredicateExecutor<Book>

// Custom repository with Querydsl
@Repository
class BookQuerydslCustomRepository(
    private val queryFactory: JPAQueryFactory
) {
    
    fun findBooksWithDynamicFilters(
        title: String? = null,
        minPrice: BigDecimal? = null,
        maxPrice: BigDecimal? = null,
        status: BookStatus? = null,
        authorName: String? = null,
        categories: List<String>? = null
    ): List<Book> {
        val book = QBook.book
        val author = QAuthor.author
        
        val query = queryFactory
            .selectFrom(book)
            .leftJoin(book.author, author).fetchJoin()
        
        // Dynamic where conditions
        val predicates = mutableListOf<BooleanExpression>()
        
        title?.let { 
            predicates.add(book.title.containsIgnoreCase(it))
        }
        
        minPrice?.let {
            predicates.add(book.price.goe(it))
        }
        
        maxPrice?.let {
            predicates.add(book.price.loe(it))
        }
        
        status?.let {
            predicates.add(book.status.eq(it))
        }
        
        authorName?.let {
            predicates.add(
                author.firstName.containsIgnoreCase(it)
                    .or(author.lastName.containsIgnoreCase(it))
            )
        }
        
        categories?.let { cats ->
            predicates.add(book.categories.any().`in`(cats))
        }
        
        if (predicates.isNotEmpty()) {
            query.where(*predicates.toTypedArray())
        }
        
        return query.fetch()
    }
    
    // Complex aggregation with Querydsl
    fun getBookStatisticsByAuthor(): List<AuthorBookStats> {
        val book = QBook.book
        val author = QAuthor.author
        
        return queryFactory
            .select(
                Projections.constructor(
                    AuthorBookStats::class.java,
                    author.id,
                    author.firstName.concat(" ").concat(author.lastName),
                    book.count(),
                    book.price.avg(),
                    book.price.min(),
                    book.price.max()
                )
            )
            .from(book)
            .join(book.author, author)
            .groupBy(author.id, author.firstName, author.lastName)
            .having(book.count().gt(0))
            .fetch()
    }
    
    // Subquery example
    fun findBooksWithAboveAverageReviews(): List<Book> {
        val book = QBook.book
        val review = QReview.review
        
        val avgRating = JPAExpressions
            .select(review.rating.avg())
            .from(review)
        
        return queryFactory
            .selectFrom(book)
            .where(
                JPAExpressions
                    .select(review.rating.avg())
                    .from(review)
                    .where(review.book.eq(book))
                    .gt(avgRating)
            )
            .fetch()
    }
    
    // Dynamic sorting
    fun findBooksWithDynamicSort(
        sortField: String,
        sortDirection: Sort.Direction
    ): List<Book> {
        val book = QBook.book
        
        val orderSpecifier = when (sortField) {
            "title" -> if (sortDirection == Sort.Direction.ASC) 
                book.title.asc() else book.title.desc()
            "price" -> if (sortDirection == Sort.Direction.ASC)
                book.price.asc() else book.price.desc()
            "publicationDate" -> if (sortDirection == Sort.Direction.ASC)
                book.publicationDate.asc() else book.publicationDate.desc()
            else -> book.id.asc()
        }
        
        return queryFactory
            .selectFrom(book)
            .orderBy(orderSpecifier)
            .fetch()
    }
}

data class AuthorBookStats(
    val authorId: Long?,
    val authorName: String,
    val bookCount: Long,
    val avgPrice: Double?,
    val minPrice: BigDecimal?,
    val maxPrice: BigDecimal?
)
```

## 8.7 JPA Auditing

JPA Auditing automatically tracks entity changes, recording who made changes and when.

### Auditing Configuration

```kotlin
// Enable JPA Auditing
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
class JpaAuditingConfiguration {
    
    @Bean
    fun auditorProvider(): AuditorAware<String> {
        return SpringSecurityAuditorAware()
    }
}

// Auditor provider implementation
class SpringSecurityAuditorAware : AuditorAware<String> {
    override fun getCurrentAuditor(): Optional<String> {
        // In a real application, get from SecurityContext
        return Optional.of("system")
        
        // With Spring Security:
        // return Optional.ofNullable(SecurityContextHolder.getContext())
        //     .map { it.authentication }
        //     .filter { it.isAuthenticated }
        //     .map { it.name }
    }
}
```

### Audited Entities

```kotlin
// Base audit entity
@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
abstract class AuditableEntity {
    
    @CreatedDate
    @Column(nullable = false, updatable = false)
    var createdDate: Instant? = null
    
    @LastModifiedDate
    @Column(nullable = false)
    var lastModifiedDate: Instant? = null
    
    @CreatedBy
    @Column(updatable = false)
    var createdBy: String? = null
    
    @LastModifiedBy
    var lastModifiedBy: String? = null
}

// Entity extending auditable base
@Entity
@Table(name = "orders")
class Order(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false, unique = true)
    var orderNumber: String,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    var customer: Customer,
    
    @Enumerated(EnumType.STRING)
    var status: OrderStatus = OrderStatus.PENDING,
    
    @Column(nullable = false)
    var totalAmount: BigDecimal,
    
    @OneToMany(mappedBy = "order", cascade = [CascadeType.ALL])
    var items: MutableList<OrderItem> = mutableListOf()
) : AuditableEntity()

// Custom audit listener
@Component
class CustomAuditListener {
    
    @PrePersist
    fun prePersist(entity: Any) {
        println("Before persisting: $entity")
        
        if (entity is AuditableEntity) {
            // Custom logic before persisting
            entity.createdDate = Instant.now()
            entity.createdBy = getCurrentUser()
        }
    }
    
    @PostPersist
    fun postPersist(entity: Any) {
        println("After persisting: $entity")
        // Send notifications, update cache, etc.
    }
    
    @PreUpdate
    fun preUpdate(entity: Any) {
        if (entity is AuditableEntity) {
            entity.lastModifiedDate = Instant.now()
            entity.lastModifiedBy = getCurrentUser()
        }
    }
    
    @PreRemove
    fun preRemove(entity: Any) {
        println("Before removing: $entity")
        // Archive data, cascade deletes, etc.
    }
    
    private fun getCurrentUser(): String {
        // Get from security context or return default
        return "system"
    }
}
```

### Audit History Tracking

```kotlin
// Entity for tracking audit history
@Entity
@Table(name = "audit_log")
class AuditLog(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var entityName: String,
    
    @Column(nullable = false)
    var entityId: String,
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    var action: AuditAction,
    
    @Column(columnDefinition = "TEXT")
    var oldValue: String? = null,
    
    @Column(columnDefinition = "TEXT")
    var newValue: String? = null,
    
    @Column(nullable = false)
    var modifiedBy: String,
    
    @Column(nullable = false)
    var modifiedDate: Instant = Instant.now()
)

enum class AuditAction {
    CREATE, UPDATE, DELETE
}

// Service for audit logging
@Service
class AuditLogService(
    private val auditLogRepository: AuditLogRepository,
    private val objectMapper: ObjectMapper
) {
    
    fun logCreate(entity: Any, entityId: String) {
        val auditLog = AuditLog(
            entityName = entity::class.simpleName ?: "Unknown",
            entityId = entityId,
            action = AuditAction.CREATE,
            newValue = objectMapper.writeValueAsString(entity),
            modifiedBy = getCurrentUser(),
            modifiedDate = Instant.now()
        )
        auditLogRepository.save(auditLog)
    }
    
    fun logUpdate(entityName: String, entityId: String, oldValue: Any?, newValue: Any?) {
        val auditLog = AuditLog(
            entityName = entityName,
            entityId = entityId,
            action = AuditAction.UPDATE,
            oldValue = oldValue?.let { objectMapper.writeValueAsString(it) },
            newValue = newValue?.let { objectMapper.writeValueAsString(it) },
            modifiedBy = getCurrentUser(),
            modifiedDate = Instant.now()
        )
        auditLogRepository.save(auditLog)
    }
    
    fun logDelete(entity: Any, entityId: String) {
        val auditLog = AuditLog(
            entityName = entity::class.simpleName ?: "Unknown",
            entityId = entityId,
            action = AuditAction.DELETE,
            oldValue = objectMapper.writeValueAsString(entity),
            modifiedBy = getCurrentUser(),
            modifiedDate = Instant.now()
        )
        auditLogRepository.save(auditLog)
    }
    
    private fun getCurrentUser(): String = "system" // Get from security context
}
```

## 8.8 Summary

In this chapter, we've explored the advanced features of Spring Data JPA that make it a powerful tool for database access in Spring Boot applications with Kotlin:

- **JPQL**: We learned how to write database-independent queries using JPQL, including joins, aggregations, subqueries, and projections
- **Query Methods**: We discovered how Spring Data JPA's method naming conventions can generate queries automatically, reducing boilerplate code
- **Sorting and Paging**: We implemented efficient pagination strategies including traditional pagination, slice-based pagination, and cursor-based pagination for large datasets
- **@Query Annotation**: We explored both JPQL and native SQL queries, including modifying queries for bulk updates and deletes
- **Querydsl**: We saw how to build type-safe, dynamic queries programmatically, which is especially useful for complex search functionality
- **JPA Auditing**: We implemented automatic tracking of entity changes, including who made changes and when, plus custom audit logging

These features, when combined with Kotlin's expressiveness and null safety, provide a robust foundation for building data access layers that are both powerful and maintainable. The key is choosing the right tool for each use case: simple query methods for straightforward queries, JPQL for complex but static queries, Querydsl for dynamic queries, and native SQL when you need database-specific features.

In the next chapter, we'll dive into relationship mapping, exploring how to model complex domain relationships with JPA while avoiding common pitfalls like the N+1 problem.