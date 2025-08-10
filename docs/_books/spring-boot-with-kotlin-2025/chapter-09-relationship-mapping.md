# Chapter 09: Relationship Mapping

Modeling relationships between entities is one of the most critical aspects of building a data-driven application. Get it right, and your application will be performant and maintainable. Get it wrong, and you'll face the dreaded N+1 problem, lazy loading exceptions, and performance nightmares. In this chapter, we'll explore how to properly map relationships in JPA with Kotlin, understanding the trade-offs and best practices for each approach.

## 9.1 One-to-One / One-to-Many / Many-to-Many

Let's start by understanding the three fundamental relationship types and how to implement them correctly in Kotlin with JPA.

### One-to-One Relationships

One-to-one relationships are the simplest but often the most misused. Use them when one entity is genuinely associated with exactly one instance of another entity.

```kotlin
// Bidirectional One-to-One with shared primary key
@Entity
@Table(name = "users")
class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false, unique = true)
    var email: String,
    
    @Column(nullable = false)
    var username: String,
    
    // Owner side of the relationship
    @OneToOne(
        mappedBy = "user",
        cascade = [CascadeType.ALL],
        fetch = FetchType.LAZY,
        optional = false
    )
    @PrimaryKeyJoinColumn
    var profile: UserProfile? = null
) {
    fun setProfileBidirectional(profile: UserProfile) {
        this.profile = profile
        profile.user = this
    }
}

@Entity
@Table(name = "user_profiles")
class UserProfile(
    @Id
    var id: Long? = null,
    
    @Column(nullable = false)
    var firstName: String,
    
    @Column(nullable = false)
    var lastName: String,
    
    var bio: String? = null,
    
    var avatarUrl: String? = null,
    
    @OneToOne(fetch = FetchType.LAZY)
    @MapsId
    @JoinColumn(name = "id")
    var user: User? = null
)

// Alternative: One-to-One with foreign key
@Entity
@Table(name = "employees")
class Employee(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var employeeNumber: String,
    
    @Column(nullable = false)
    var name: String,
    
    @OneToOne(
        cascade = [CascadeType.ALL],
        fetch = FetchType.LAZY
    )
    @JoinColumn(
        name = "parking_spot_id",
        referencedColumnName = "id",
        unique = true  // Ensures one-to-one
    )
    var parkingSpot: ParkingSpot? = null
)

@Entity
@Table(name = "parking_spots")
class ParkingSpot(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false, unique = true)
    var spotNumber: String,
    
    @Column(nullable = false)
    var floor: Int,
    
    @OneToOne(
        mappedBy = "parkingSpot",
        fetch = FetchType.LAZY
    )
    var employee: Employee? = null
)
```

**Best Practices for One-to-One:**

1. **Always use LAZY fetching**: Even for one-to-one, eager fetching can cause performance issues
2. **Consider embedding instead**: If the related entity has no identity of its own, use `@Embeddable`
3. **Be careful with bidirectional mappings**: They can cause infinite loops in serialization

```kotlin
// Better approach using @Embeddable for value objects
@Entity
@Table(name = "customers")
class Customer(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var email: String,
    
    @Embedded
    var address: Address = Address()
)

@Embeddable
class Address(
    var street: String = "",
    var city: String = "",
    var state: String = "",
    var zipCode: String = "",
    var country: String = ""
)
```

### One-to-Many and Many-to-One Relationships

These are the most common relationships in domain models. One entity has a collection of related entities.

```kotlin
// Bidirectional One-to-Many / Many-to-One
@Entity
@Table(name = "departments")
class Department(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false, unique = true)
    var name: String,
    
    @Column(nullable = false)
    var code: String,
    
    // One-to-Many side
    @OneToMany(
        mappedBy = "department",
        cascade = [CascadeType.ALL],
        orphanRemoval = true,
        fetch = FetchType.LAZY
    )
    var employees: MutableSet<Employee> = mutableSetOf()
) {
    // Helper methods to maintain bidirectional relationship
    fun addEmployee(employee: Employee) {
        employees.add(employee)
        employee.department = this
    }
    
    fun removeEmployee(employee: Employee) {
        employees.remove(employee)
        employee.department = null
    }
    
    fun clearEmployees() {
        employees.forEach { it.department = null }
        employees.clear()
    }
}

@Entity
@Table(name = "employees")
class Employee(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var name: String,
    
    @Column(nullable = false, unique = true)
    var employeeId: String,
    
    var salary: BigDecimal,
    
    // Many-to-One side (owner of the relationship)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    var department: Department? = null
) {
    // Important for Set operations
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is Employee) return false
        return id != null && id == other.id
    }
    
    override fun hashCode(): Int {
        return javaClass.hashCode()
    }
}

// Unidirectional One-to-Many (using @JoinColumn)
@Entity
@Table(name = "orders")
class Order(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false, unique = true)
    var orderNumber: String,
    
    var orderDate: LocalDateTime = LocalDateTime.now(),
    
    // Unidirectional - Order owns the relationship
    @OneToMany(
        cascade = [CascadeType.ALL],
        orphanRemoval = true,
        fetch = FetchType.LAZY
    )
    @JoinColumn(name = "order_id")  // Foreign key in order_items table
    var items: MutableList<OrderItem> = mutableListOf()
) {
    fun addItem(item: OrderItem) {
        items.add(item)
    }
    
    fun removeItem(item: OrderItem) {
        items.remove(item)
    }
    
    fun getTotalAmount(): BigDecimal {
        return items.map { it.price * BigDecimal(it.quantity) }
            .fold(BigDecimal.ZERO, BigDecimal::add)
    }
}

@Entity
@Table(name = "order_items")
class OrderItem(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var productId: String,
    
    @Column(nullable = false)
    var productName: String,
    
    @Column(nullable = false)
    var quantity: Int,
    
    @Column(nullable = false, precision = 10, scale = 2)
    var price: BigDecimal
)
```

**Performance Optimization for One-to-Many:**

```kotlin
@Repository
interface DepartmentRepository : JpaRepository<Department, Long> {
    
    // Fetch with JOIN to avoid N+1 problem
    @Query("""
        SELECT DISTINCT d FROM Department d
        LEFT JOIN FETCH d.employees
        WHERE d.id IN :ids
    """)
    fun findByIdsWithEmployees(@Param("ids") ids: List<Long>): List<Department>
    
    // Using @EntityGraph for fetching strategy
    @EntityGraph(attributePaths = ["employees"])
    override fun findById(id: Long): Optional<Department>
    
    // Batch fetching configuration
    @Query("SELECT d FROM Department d")
    @QueryHints(QueryHint(name = "org.hibernate.fetchSize", value = "25"))
    fun findAllWithBatchSize(): List<Department>
}
```

### Many-to-Many Relationships

Many-to-many relationships require a join table and can be the most complex to manage properly.

```kotlin
// Bidirectional Many-to-Many
@Entity
@Table(name = "students")
class Student(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var name: String,
    
    @Column(nullable = false, unique = true)
    var studentId: String,
    
    @ManyToMany(
        cascade = [CascadeType.PERSIST, CascadeType.MERGE],
        fetch = FetchType.LAZY
    )
    @JoinTable(
        name = "student_courses",
        joinColumns = [JoinColumn(name = "student_id")],
        inverseJoinColumns = [JoinColumn(name = "course_id")]
    )
    var courses: MutableSet<Course> = mutableSetOf()
) {
    fun enrollInCourse(course: Course) {
        courses.add(course)
        course.students.add(this)
    }
    
    fun withdrawFromCourse(course: Course) {
        courses.remove(course)
        course.students.remove(this)
    }
    
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is Student) return false
        return id != null && id == other.id
    }
    
    override fun hashCode(): Int = javaClass.hashCode()
}

@Entity
@Table(name = "courses")
class Course(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false, unique = true)
    var courseCode: String,
    
    @Column(nullable = false)
    var title: String,
    
    var credits: Int,
    
    @ManyToMany(
        mappedBy = "courses",
        fetch = FetchType.LAZY
    )
    var students: MutableSet<Student> = mutableSetOf()
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is Course) return false
        return id != null && id == other.id
    }
    
    override fun hashCode(): Int = javaClass.hashCode()
}

// Many-to-Many with additional attributes (using association entity)
@Entity
@Table(name = "enrollments")
class Enrollment(
    @EmbeddedId
    var id: EnrollmentId = EnrollmentId(),
    
    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("studentId")
    @JoinColumn(name = "student_id")
    var student: Student,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("courseId")
    @JoinColumn(name = "course_id")
    var course: Course,
    
    @Column(nullable = false)
    var enrollmentDate: LocalDate = LocalDate.now(),
    
    var grade: String? = null,
    
    @Enumerated(EnumType.STRING)
    var status: EnrollmentStatus = EnrollmentStatus.ACTIVE
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is Enrollment) return false
        return id == other.id
    }
    
    override fun hashCode(): Int = id.hashCode()
}

@Embeddable
class EnrollmentId(
    var studentId: Long? = null,
    var courseId: Long? = null
) : Serializable {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is EnrollmentId) return false
        return studentId == other.studentId && courseId == other.courseId
    }
    
    override fun hashCode(): Int {
        return Objects.hash(studentId, courseId)
    }
}

enum class EnrollmentStatus {
    ACTIVE, COMPLETED, WITHDRAWN, FAILED
}
```

## 9.2 Mapping Directionality

Understanding when to use unidirectional vs bidirectional mappings is crucial for maintainable code.

### Unidirectional Mappings

Unidirectional mappings are simpler and should be your default choice unless you specifically need bidirectional access.

```kotlin
// Unidirectional Many-to-One (most common and recommended)
@Entity
@Table(name = "comments")
class Comment(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false, length = 1000)
    var content: String,
    
    @Column(nullable = false)
    var authorName: String,
    
    // Only Comment knows about Post
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    var post: Post,
    
    var createdAt: LocalDateTime = LocalDateTime.now()
)

@Entity
@Table(name = "posts")
class Post(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var title: String,
    
    @Column(columnDefinition = "TEXT")
    var content: String
    // No reference to comments - unidirectional
)

// To get comments for a post, use repository
@Repository
interface CommentRepository : JpaRepository<Comment, Long> {
    fun findByPostIdOrderByCreatedAtDesc(postId: Long): List<Comment>
    fun countByPostId(postId: Long): Long
}
```

### Bidirectional Mappings

Use bidirectional mappings when you frequently need to navigate the relationship from both sides.

```kotlin
// Bidirectional with proper helper methods
@Entity
@Table(name = "authors")
class Author(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var name: String,
    
    var biography: String? = null,
    
    @OneToMany(
        mappedBy = "author",
        cascade = [CascadeType.ALL],
        orphanRemoval = true
    )
    private val _books: MutableSet<Book> = mutableSetOf()
) {
    // Expose as immutable collection
    val books: Set<Book>
        get() = _books.toSet()
    
    // Helper methods maintain consistency
    fun addBook(book: Book) {
        _books.add(book)
        book.author = this
    }
    
    fun removeBook(book: Book) {
        _books.remove(book)
        book.author = null
    }
    
    fun removeAllBooks() {
        _books.forEach { it.author = null }
        _books.clear()
    }
}

@Entity
@Table(name = "books")
class Book(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var title: String,
    
    @Column(nullable = false, unique = true)
    var isbn: String,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    var author: Author? = null
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is Book) return false
        return isbn == other.isbn
    }
    
    override fun hashCode(): Int {
        return isbn.hashCode()
    }
}
```

### Choosing Directionality

```kotlin
// Decision guide for relationship directionality

// Use UNIDIRECTIONAL when:
// 1. The relationship is only navigated from one side
// 2. You want to reduce complexity
// 3. The "many" side doesn't need to know about the "one" side

// Example: Order -> OrderItems (unidirectional)
@Entity
class Order(
    @Id
    var id: Long? = null,
    
    @OneToMany(cascade = [CascadeType.ALL])
    @JoinColumn(name = "order_id")
    var items: List<OrderItem> = listOf()
)

// Use BIDIRECTIONAL when:
// 1. You frequently navigate from both sides
// 2. You need to maintain referential integrity in the domain
// 3. Business logic requires both sides to know about each other

// Example: Project <-> Task (bidirectional)
@Entity
class Project(
    @Id
    var id: Long? = null,
    
    @OneToMany(mappedBy = "project")
    private val _tasks: MutableSet<Task> = mutableSetOf()
) {
    val tasks: Set<Task> get() = _tasks
    
    fun addTask(task: Task) {
        _tasks.add(task)
        task.project = this
    }
}

@Entity
class Task(
    @Id
    var id: Long? = null,
    
    @ManyToOne
    var project: Project? = null
)
```

## 9.3 Cascade and Orphan Removal

Cascade operations and orphan removal are powerful features that can simplify your code but must be used carefully.

### Understanding Cascade Types

```kotlin
// Comprehensive cascade example
@Entity
@Table(name = "shopping_carts")
class ShoppingCart(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var sessionId: String,
    
    // CASCADE.ALL = PERSIST, MERGE, REMOVE, REFRESH, DETACH
    @OneToMany(
        cascade = [CascadeType.ALL],
        orphanRemoval = true,
        fetch = FetchType.LAZY
    )
    @JoinColumn(name = "cart_id")
    private val _items: MutableList<CartItem> = mutableListOf()
) {
    val items: List<CartItem>
        get() = _items.toList()
    
    fun addItem(product: Product, quantity: Int) {
        val existingItem = _items.find { it.productId == product.id }
        
        if (existingItem != null) {
            existingItem.quantity += quantity
        } else {
            _items.add(CartItem(
                productId = product.id!!,
                productName = product.name,
                price = product.price,
                quantity = quantity
            ))
        }
    }
    
    fun removeItem(productId: Long) {
        _items.removeIf { it.productId == productId }
    }
    
    fun clear() {
        _items.clear()
    }
    
    fun updateQuantity(productId: Long, newQuantity: Int) {
        if (newQuantity <= 0) {
            removeItem(productId)
        } else {
            _items.find { it.productId == productId }?.quantity = newQuantity
        }
    }
}

@Entity
@Table(name = "cart_items")
class CartItem(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var productId: Long,
    
    @Column(nullable = false)
    var productName: String,
    
    @Column(nullable = false)
    var price: BigDecimal,
    
    @Column(nullable = false)
    var quantity: Int
) {
    fun getSubtotal(): BigDecimal = price * BigDecimal(quantity)
}
```

### Selective Cascade Operations

```kotlin
// Different cascade strategies for different relationships
@Entity
@Table(name = "companies")
class Company(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var name: String,
    
    // Full cascade - departments are owned by company
    @OneToMany(
        mappedBy = "company",
        cascade = [CascadeType.ALL],
        orphanRemoval = true
    )
    var departments: MutableSet<Department> = mutableSetOf(),
    
    // Only PERSIST and MERGE - contacts can exist independently
    @OneToMany(
        cascade = [CascadeType.PERSIST, CascadeType.MERGE]
    )
    @JoinTable(
        name = "company_contacts",
        joinColumns = [JoinColumn(name = "company_id")],
        inverseJoinColumns = [JoinColumn(name = "contact_id")]
    )
    var contacts: MutableSet<Contact> = mutableSetOf(),
    
    // No cascade - documents are managed separately
    @OneToMany(mappedBy = "company")
    var documents: MutableSet<Document> = mutableSetOf()
)
```

### Orphan Removal

```kotlin
// Orphan removal example
@Entity
@Table(name = "invoices")
class Invoice(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false, unique = true)
    var invoiceNumber: String,
    
    var issueDate: LocalDate = LocalDate.now(),
    
    // orphanRemoval = true means items are deleted when removed from collection
    @OneToMany(
        cascade = [CascadeType.ALL],
        orphanRemoval = true,
        fetch = FetchType.LAZY
    )
    @JoinColumn(name = "invoice_id")
    private val _lineItems: MutableList<InvoiceLineItem> = mutableListOf()
) {
    val lineItems: List<InvoiceLineItem>
        get() = _lineItems.toList()
    
    fun addLineItem(item: InvoiceLineItem) {
        _lineItems.add(item)
    }
    
    fun removeLineItem(item: InvoiceLineItem) {
        _lineItems.remove(item)  // Item will be deleted from database
    }
    
    fun replaceLineItems(newItems: List<InvoiceLineItem>) {
        _lineItems.clear()  // All existing items will be deleted
        _lineItems.addAll(newItems)
    }
}

// Service demonstrating cascade and orphan removal
@Service
@Transactional
class InvoiceService(
    private val invoiceRepository: InvoiceRepository
) {
    
    fun createInvoice(request: CreateInvoiceRequest): Invoice {
        val invoice = Invoice(
            invoiceNumber = generateInvoiceNumber()
        )
        
        // Line items will be persisted automatically (CascadeType.PERSIST)
        request.items.forEach { itemRequest ->
            invoice.addLineItem(InvoiceLineItem(
                description = itemRequest.description,
                quantity = itemRequest.quantity,
                unitPrice = itemRequest.unitPrice
            ))
        }
        
        return invoiceRepository.save(invoice)
    }
    
    fun updateInvoice(id: Long, request: UpdateInvoiceRequest): Invoice {
        val invoice = invoiceRepository.findById(id)
            .orElseThrow { EntityNotFoundException("Invoice not found") }
        
        // This will delete all existing items and create new ones
        invoice.replaceLineItems(
            request.items.map { itemRequest ->
                InvoiceLineItem(
                    description = itemRequest.description,
                    quantity = itemRequest.quantity,
                    unitPrice = itemRequest.unitPrice
                )
            }
        )
        
        // Changes will be persisted automatically (CascadeType.MERGE)
        return invoice
    }
    
    fun deleteInvoice(id: Long) {
        // All line items will be deleted automatically (CascadeType.REMOVE)
        invoiceRepository.deleteById(id)
    }
}
```

### Advanced Cascade Patterns

```kotlin
// Custom cascade behavior with event listeners
@Entity
@EntityListeners(AuditingEntityListener::class)
@Table(name = "projects")
class Project(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var name: String,
    
    @Enumerated(EnumType.STRING)
    var status: ProjectStatus = ProjectStatus.PLANNING,
    
    @OneToMany(
        mappedBy = "project",
        cascade = [CascadeType.PERSIST, CascadeType.MERGE]
    )
    private val _tasks: MutableSet<Task> = mutableSetOf(),
    
    @OneToMany(
        mappedBy = "project",
        cascade = [CascadeType.ALL],
        orphanRemoval = true
    )
    private val _milestones: MutableSet<Milestone> = mutableSetOf()
) {
    @PreRemove
    private fun preRemove() {
        // Custom logic before removal
        if (_tasks.any { it.status != TaskStatus.COMPLETED }) {
            throw IllegalStateException("Cannot delete project with incomplete tasks")
        }
    }
    
    @PostPersist
    private fun postPersist() {
        // Automatically create initial milestone
        if (_milestones.isEmpty()) {
            _milestones.add(Milestone(
                name = "Project Kickoff",
                project = this,
                dueDate = LocalDate.now().plusDays(7)
            ))
        }
    }
}

// Cascade with inheritance
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "content_type")
abstract class Content(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    
    @Column(nullable = false)
    var title: String,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    var author: User? = null,
    
    @OneToMany(
        cascade = [CascadeType.ALL],
        orphanRemoval = true
    )
    @JoinColumn(name = "content_id")
    var attachments: MutableList<Attachment> = mutableListOf()
)

@Entity
@DiscriminatorValue("ARTICLE")
class Article(
    title: String,
    var body: String,
    var publishedAt: LocalDateTime? = null
) : Content(title = title)

@Entity
@DiscriminatorValue("VIDEO")
class Video(
    title: String,
    var url: String,
    var duration: Int
) : Content(title = title)
```

### Performance Considerations

```kotlin
// Avoiding cascade performance pitfalls
@Service
@Transactional
class RelationshipPerformanceService(
    private val entityManager: EntityManager,
    private val departmentRepository: DepartmentRepository
) {
    
    // Bad: Loading entire object graph
    fun inefficientUpdate(departmentId: Long, employeeName: String) {
        val department = departmentRepository.findById(departmentId).orElseThrow()
        // This loads ALL employees due to cascade
        department.employees.find { it.name == employeeName }?.salary = BigDecimal(100000)
    }
    
    // Good: Targeted update
    fun efficientUpdate(departmentId: Long, employeeName: String) {
        val query = entityManager.createQuery("""
            UPDATE Employee e 
            SET e.salary = :salary 
            WHERE e.department.id = :deptId 
            AND e.name = :name
        """)
        query.setParameter("salary", BigDecimal(100000))
        query.setParameter("deptId", departmentId)
        query.setParameter("name", employeeName)
        query.executeUpdate()
    }
    
    // Batch operations with cascade
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun batchCreateWithCascade(departments: List<Department>) {
        var count = 0
        departments.forEach { dept ->
            entityManager.persist(dept)  // Cascades to employees
            
            if (++count % 20 == 0) {
                entityManager.flush()
                entityManager.clear()
            }
        }
    }
}
```

## 9.4 Summary

In this chapter, we've explored the complexities of relationship mapping in JPA with Kotlin. Here are the key takeaways:

**Relationship Types:**
- **One-to-One**: Use sparingly, consider @Embeddable for value objects
- **One-to-Many/Many-to-One**: Most common, prefer unidirectional from the "many" side
- **Many-to-Many**: Consider using an association entity when you need additional attributes

**Directionality Best Practices:**
- Start with unidirectional relationships for simplicity
- Use bidirectional only when you frequently navigate from both sides
- Always maintain consistency with helper methods in bidirectional relationships

**Cascade and Orphan Removal:**
- Use CascadeType.ALL for true parent-child relationships where the child can't exist without the parent
- Be selective with cascade operations - not everything needs to cascade
- orphanRemoval = true is powerful but use it carefully to avoid accidental deletions

**Performance Considerations:**
- Always use LAZY fetching by default
- Use JOIN FETCH or @EntityGraph to avoid N+1 problems
- Consider the impact of cascade operations on large object graphs
- Implement equals() and hashCode() properly for entities used in collections

**Kotlin-Specific Tips:**
- Use data classes for @Embeddable value objects
- Leverage immutable collections with private mutable backing fields
- Use helper methods to maintain bidirectional consistency
- Be careful with nullable references in relationships

Remember, the goal is not to model every possible relationship in your database as JPA associations. Sometimes a simple foreign key with repository queries is cleaner and more performant than complex mappings. Choose the approach that makes your code most maintainable and performs well for your specific use case.

In the next chapter, we'll explore validation and exception handling to ensure our domain models maintain integrity and provide meaningful feedback when things go wrong.