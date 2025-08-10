---
layout: default
parent: Spring Boot with Kotlin (2025)
nav_exclude: true
---
# Chapter 10: Validation and Exception Handling
- TOC
{:toc}

Building robust applications requires more than just functional code—it demands comprehensive validation and exceptional exception handling. In this chapter, we'll explore how to implement effective validation strategies using Kotlin with Spring Boot and Hibernate Validator. We'll cover everything from basic bean validation to sophisticated custom validators, and we'll dive deep into creating a robust exception handling framework that provides meaningful feedback to API consumers.

Validation in Kotlin presents unique opportunities and challenges. Kotlin's null safety, data classes, and extension functions allow us to create more expressive and type-safe validation code than traditional Java approaches. We'll explore how to leverage these language features while working within the Spring Boot ecosystem.

Exception handling is equally critical. A well-designed exception handling strategy not only prevents crashes but also provides valuable debugging information and user-friendly error messages. We'll build a comprehensive exception handling framework that handles everything from validation errors to database constraints, service failures, and external API errors.

## 10.1 Validation Strategies

Effective validation is multi-layered, occurring at different points in your application stack. Let's explore the various validation strategies and when to apply each one.

### Input Validation Strategy

Input validation is your first line of defense against invalid data. In Spring Boot applications, this typically happens at the controller layer:

```kotlin
// Comprehensive input validation strategy
@RestController
@RequestMapping("/api/v1/users")
@Validated // Enable method-level validation
class UserController(
    private val userService: UserService
) {

    @PostMapping
    fun createUser(
        @Valid @RequestBody createUserRequest: CreateUserRequest
    ): ResponseEntity<UserResponse> {
        val user = userService.createUser(createUserRequest)
        return ResponseEntity.status(HttpStatus.CREATED).body(user)
    }

    @PutMapping("/{id}")
    fun updateUser(
        @PathVariable @Positive id: Long,
        @Valid @RequestBody updateUserRequest: UpdateUserRequest
    ): ResponseEntity<UserResponse> {
        val user = userService.updateUser(id, updateUserRequest)
        return ResponseEntity.ok(user)
    }

    @GetMapping
    fun getUsers(
        @RequestParam(defaultValue = "0") @Min(0) page: Int,
        @RequestParam(defaultValue = "20") @Min(1) @Max(100) size: Int,
        @RequestParam @Pattern(regexp = "^[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", message = "Invalid email format")
        email: String?
    ): ResponseEntity<Page<UserResponse>> {
        val users = userService.getUsers(page, size, email)
        return ResponseEntity.ok(users)
    }
}

// Data Transfer Objects with comprehensive validation
data class CreateUserRequest(
    @field:NotBlank(message = "Username is required")
    @field:Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    @field:Pattern(
        regexp = "^[a-zA-Z0-9_-]+$",
        message = "Username can only contain letters, numbers, underscores, and hyphens"
    )
    val username: String,

    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Email must be valid")
    val email: String,

    @field:NotBlank(message = "Password is required")
    @field:Size(min = 8, max = 100, message = "Password must be between 8 and 100 characters")
    @field:Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]+$",
        message = "Password must contain at least one lowercase letter, one uppercase letter, one digit, and one special character"
    )
    val password: String,

    @field:NotBlank(message = "First name is required")
    @field:Size(max = 50, message = "First name cannot exceed 50 characters")
    val firstName: String,

    @field:NotBlank(message = "Last name is required")
    @field:Size(max = 50, message = "Last name cannot exceed 50 characters")
    val lastName: String,

    @field:Valid
    val profile: UserProfileRequest?,

    @field:Size(max = 10, message = "Cannot have more than 10 roles")
    val roles: Set<@NotBlank @Size(max = 20) String> = emptySet()
)

data class UserProfileRequest(
    @field:Past(message = "Birth date must be in the past")
    val birthDate: LocalDate?,

    @field:Size(max = 200, message = "Bio cannot exceed 200 characters")
    val bio: String?,

    @field:URL(message = "Website must be a valid URL")
    val website: String?,

    @field:Pattern(
        regexp = "^\\+?[1-9]\\d{1,14}$",
        message = "Phone number must be in international format"
    )
    val phoneNumber: String?,

    @field:Valid
    val address: AddressRequest?
)

data class AddressRequest(
    @field:NotBlank(message = "Street is required")
    @field:Size(max = 100, message = "Street cannot exceed 100 characters")
    val street: String,

    @field:NotBlank(message = "City is required")
    @field:Size(max = 50, message = "City cannot exceed 50 characters")
    val city: String,

    @field:NotBlank(message = "State is required")
    @field:Size(min = 2, max = 50, message = "State must be between 2 and 50 characters")
    val state: String,

    @field:NotBlank(message = "Postal code is required")
    @field:Pattern(
        regexp = "^\\d{5}(-\\d{4})?$",
        message = "Postal code must be in format 12345 or 12345-6789"
    )
    val postalCode: String,

    @field:NotBlank(message = "Country is required")
    @field:Size(min = 2, max = 2, message = "Country must be 2-letter ISO code")
    val country: String
)

// Update request with partial validation
data class UpdateUserRequest(
    @field:Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    @field:Pattern(
        regexp = "^[a-zA-Z0-9_-]+$",
        message = "Username can only contain letters, numbers, underscores, and hyphens"
    )
    val username: String? = null,

    @field:Email(message = "Email must be valid")
    val email: String? = null,

    @field:Size(max = 50, message = "First name cannot exceed 50 characters")
    val firstName: String? = null,

    @field:Size(max = 50, message = "Last name cannot exceed 50 characters")
    val lastName: String? = null,

    @field:Valid
    val profile: UserProfileRequest? = null
) {
    // Validation method to ensure at least one field is provided for update
    fun hasAtLeastOneField(): Boolean {
        return username != null || email != null || firstName != null ||
               lastName != null || profile != null
    }
}
```

### Business Logic Validation Strategy

While input validation handles format and basic constraints, business logic validation ensures that operations make sense within your domain:

```kotlin
// Business logic validation service
@Service
@Transactional(readOnly = true)
class UserValidationService(
    private val userRepository: UserRepository,
    private val roleService: RoleService
) {

    /**
     * Validates business rules for user creation.
     * This goes beyond simple format validation to check business constraints.
     */
    fun validateUserCreation(request: CreateUserRequest): ValidationResult {
        val errors = mutableListOf<ValidationError>()

        // Check username uniqueness
        if (userRepository.existsByUsername(request.username)) {
            errors.add(ValidationError(
                field = "username",
                message = "Username '${request.username}' is already taken",
                code = "USERNAME_ALREADY_EXISTS"
            ))
        }

        // Check email uniqueness
        if (userRepository.existsByEmail(request.email)) {
            errors.add(ValidationError(
                field = "email",
                message = "Email '${request.email}' is already registered",
                code = "EMAIL_ALREADY_EXISTS"
            ))
        }

        // Validate roles exist and user has permission to assign them
        request.roles.forEach { roleName ->
            if (!roleService.roleExists(roleName)) {
                errors.add(ValidationError(
                    field = "roles",
                    message = "Role '$roleName' does not exist",
                    code = "INVALID_ROLE"
                ))
            }
        }

        // Business rule: Age validation if birthdate provided
        request.profile?.birthDate?.let { birthDate ->
            val age = Period.between(birthDate, LocalDate.now()).years
            if (age < 13) {
                errors.add(ValidationError(
                    field = "profile.birthDate",
                    message = "User must be at least 13 years old",
                    code = "MINIMUM_AGE_REQUIRED"
                ))
            }
        }

        return ValidationResult(
            isValid = errors.isEmpty(),
            errors = errors
        )
    }

    /**
     * Validates business rules for user updates.
     */
    fun validateUserUpdate(userId: Long, request: UpdateUserRequest): ValidationResult {
        val errors = mutableListOf<ValidationError>()
        val existingUser = userRepository.findById(userId).orElse(null)
            ?: return ValidationResult(
                isValid = false,
                errors = listOf(ValidationError(
                    field = "id",
                    message = "User with ID $userId not found",
                    code = "USER_NOT_FOUND"
                ))
            )

        // Validate at least one field is provided
        if (!request.hasAtLeastOneField()) {
            errors.add(ValidationError(
                field = "request",
                message = "At least one field must be provided for update",
                code = "NO_FIELDS_TO_UPDATE"
            ))
        }

        // Check username uniqueness (if changing)
        request.username?.let { newUsername ->
            if (newUsername != existingUser.username && userRepository.existsByUsername(newUsername)) {
                errors.add(ValidationError(
                    field = "username",
                    message = "Username '$newUsername' is already taken",
                    code = "USERNAME_ALREADY_EXISTS"
                ))
            }
        }

        // Check email uniqueness (if changing)
        request.email?.let { newEmail ->
            if (newEmail != existingUser.email && userRepository.existsByEmail(newEmail)) {
                errors.add(ValidationError(
                    field = "email",
                    message = "Email '$newEmail' is already registered",
                    code = "EMAIL_ALREADY_EXISTS"
                ))
            }
        }

        return ValidationResult(
            isValid = errors.isEmpty(),
            errors = errors
        )
    }

    /**
     * Validates that a user can be deleted.
     */
    fun validateUserDeletion(userId: Long): ValidationResult {
        val errors = mutableListOf<ValidationError>()
        val user = userRepository.findById(userId).orElse(null)
            ?: return ValidationResult(
                isValid = false,
                errors = listOf(ValidationError(
                    field = "id",
                    message = "User with ID $userId not found",
                    code = "USER_NOT_FOUND"
                ))
            )

        // Business rule: Cannot delete users with active orders
        if (userRepository.hasActiveOrders(userId)) {
            errors.add(ValidationError(
                field = "id",
                message = "Cannot delete user with active orders",
                code = "USER_HAS_ACTIVE_ORDERS"
            ))
        }

        // Business rule: Cannot delete admin users if they're the last admin
        if (user.hasRole("ADMIN") && userRepository.countAdminUsers() <= 1) {
            errors.add(ValidationError(
                field = "id",
                message = "Cannot delete the last admin user",
                code = "LAST_ADMIN_USER"
            ))
        }

        return ValidationResult(
            isValid = errors.isEmpty(),
            errors = errors
        )
    }
}

// Validation result classes
data class ValidationResult(
    val isValid: Boolean,
    val errors: List<ValidationError>
) {
    fun throwIfInvalid() {
        if (!isValid) {
            throw BusinessValidationException(errors)
        }
    }
}

data class ValidationError(
    val field: String,
    val message: String,
    val code: String,
    val rejectedValue: Any? = null
)

// Custom exceptions for validation failures
class BusinessValidationException(
    val validationErrors: List<ValidationError>
) : RuntimeException("Business validation failed") {

    override val message: String
        get() = validationErrors.joinToString("; ") { "${it.field}: ${it.message}" }
}
```

### Database Constraint Validation Strategy

Database constraints provide the final layer of validation and ensure data integrity even if application-level validation fails:

```kotlin
// Entity with comprehensive database constraints
@Entity
@Table(
    name = "users",
    uniqueConstraints = [
        UniqueConstraint(name = "uk_users_username", columnNames = ["username"]),
        UniqueConstraint(name = "uk_users_email", columnNames = ["email"])
    ],
    indexes = [
        Index(name = "idx_users_email", columnList = "email"),
        Index(name = "idx_users_created_at", columnList = "created_at")
    ]
)
class User : BaseEntity() {

    @Column(name = "username", nullable = false, length = 50)
    var username: String = ""
        protected set

    @Column(name = "email", nullable = false, length = 255)
    var email: String = ""
        protected set

    @Column(name = "password_hash", nullable = false, length = 255)
    var passwordHash: String = ""
        protected set

    @Column(name = "first_name", nullable = false, length = 50)
    var firstName: String = ""
        protected set

    @Column(name = "last_name", nullable = false, length = 50)
    var lastName: String = ""
        protected set

    @Column(name = "active", nullable = false)
    var active: Boolean = true
        protected set

    @Column(name = "email_verified", nullable = false)
    var emailVerified: Boolean = false
        protected set

    @Column(name = "last_login")
    var lastLogin: LocalDateTime? = null
        protected set

    // Relationships with proper constraints
    @OneToOne(cascade = [CascadeType.ALL], fetch = FetchType.LAZY, optional = true)
    @JoinColumn(name = "profile_id", referencedColumnName = "id")
    var profile: UserProfile? = null
        protected set

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns = [JoinColumn(name = "user_id")],
        inverseJoinColumns = [JoinColumn(name = "role_id")]
    )
    var roles: MutableSet<Role> = mutableSetOf()
        protected set

    // Factory method with validation
    companion object {
        fun create(
            username: String,
            email: String,
            passwordHash: String,
            firstName: String,
            lastName: String
        ): User {
            require(username.isNotBlank()) { "Username cannot be blank" }
            require(email.isNotBlank()) { "Email cannot be blank" }
            require(passwordHash.isNotBlank()) { "Password hash cannot be blank" }
            require(firstName.isNotBlank()) { "First name cannot be blank" }
            require(lastName.isNotBlank()) { "Last name cannot be blank" }

            return User().apply {
                this.username = username
                this.email = email
                this.passwordHash = passwordHash
                this.firstName = firstName
                this.lastName = lastName
            }
        }
    }

    // Business methods with validation
    fun updateEmail(newEmail: String) {
        require(newEmail.isNotBlank()) { "Email cannot be blank" }
        require(newEmail.contains("@")) { "Email must be valid" }

        this.email = newEmail
        this.emailVerified = false // Reset verification when email changes
    }

    fun addRole(role: Role) {
        roles.add(role)
    }

    fun removeRole(role: Role) {
        roles.remove(role)
    }

    fun hasRole(roleName: String): Boolean {
        return roles.any { it.name == roleName }
    }
}

// Database constraint exception handler
@Component
class DatabaseConstraintExceptionHandler {

    fun handleConstraintViolation(ex: DataIntegrityViolationException): ValidationResult {
        val errors = mutableListOf<ValidationError>()

        val rootCause = ex.rootCause
        if (rootCause is SQLIntegrityConstraintViolationException) {
            val constraintName = extractConstraintName(rootCause.message ?: "")

            when {
                constraintName.contains("uk_users_username") -> {
                    errors.add(ValidationError(
                        field = "username",
                        message = "Username is already taken",
                        code = "USERNAME_ALREADY_EXISTS"
                    ))
                }
                constraintName.contains("uk_users_email") -> {
                    errors.add(ValidationError(
                        field = "email",
                        message = "Email is already registered",
                        code = "EMAIL_ALREADY_EXISTS"
                    ))
                }
                else -> {
                    errors.add(ValidationError(
                        field = "database",
                        message = "Database constraint violation",
                        code = "CONSTRAINT_VIOLATION"
                    ))
                }
            }
        }

        return ValidationResult(
            isValid = false,
            errors = errors
        )
    }

    private fun extractConstraintName(message: String): String {
        // Extract constraint name from database error message
        // This is database-specific implementation
        return when {
            message.contains("Duplicate entry") && message.contains("uk_users_username") -> "uk_users_username"
            message.contains("Duplicate entry") && message.contains("uk_users_email") -> "uk_users_email"
            else -> message
        }
    }
}
```

## 10.2 Hibernate Validator in Kotlin

Hibernate Validator is the reference implementation of Bean Validation and integrates seamlessly with Spring Boot. Let's explore how to use it effectively in Kotlin applications.

### Basic Hibernate Validator Setup

First, let's ensure proper configuration and dependencies:

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-validation")
    // This includes hibernate-validator automatically
}

// Validation configuration
@Configuration
class ValidationConfig {

    @Bean
    fun validator(): Validator {
        return ValidatorBuilder.create()
            .with(KotlinValidatorConfiguration())
            .buildValidatorFactory()
            .validator
    }

    @Bean
    fun methodValidationPostProcessor(): MethodValidationPostProcessor {
        val processor = MethodValidationPostProcessor()
        processor.setValidator(validator())
        return processor
    }
}

// Kotlin-specific validator configuration
class KotlinValidatorConfiguration : ValidatorConfiguration<KotlinValidatorConfiguration> {

    override fun addMapping(stream: InputStream?): KotlinValidatorConfiguration = this
    override fun addProperty(name: String?, value: String?): KotlinValidatorConfiguration = this
    override fun messageInterpolator(interpolator: MessageInterpolator?): KotlinValidatorConfiguration = this
    override fun traversableResolver(resolver: TraversableResolver?): KotlinValidatorConfiguration = this
    override fun constraintValidatorFactory(constraintValidatorFactory: ConstraintValidatorFactory?): KotlinValidatorConfiguration = this
    override fun parameterNameProvider(parameterNameProvider: ParameterNameProvider?): KotlinValidatorConfiguration = this
    override fun clockProvider(clockProvider: ClockProvider?): KotlinValidatorConfiguration = this
    override fun addValueExtractor(extractor: ValueExtractor<*>?): KotlinValidatorConfiguration = this

    override fun buildValidatorFactory(): ValidatorFactory {
        return Validation.byProvider(HibernateValidator::class.java)
            .configure()
            .parameterNameProvider(ReflectionParameterNameProvider())
            .buildValidatorFactory()
    }
}
```

### Advanced Validation Annotations in Kotlin

Kotlin's type system and language features work excellently with Hibernate Validator:

```kotlin
// Advanced validation examples leveraging Kotlin features
data class ProductRequest(
    @field:NotBlank(message = "Product name is required")
    @field:Size(min = 2, max = 100, message = "Product name must be between 2 and 100 characters")
    val name: String,

    @field:NotBlank(message = "Description is required")
    @field:Size(min = 10, max = 1000, message = "Description must be between 10 and 1000 characters")
    val description: String,

    @field:NotNull(message = "Price is required")
    @field:DecimalMin(value = "0.01", message = "Price must be greater than 0")
    @field:DecimalMax(value = "999999.99", message = "Price must be less than 1,000,000")
    @field:Digits(integer = 6, fraction = 2, message = "Price must have at most 6 digits and 2 decimal places")
    val price: BigDecimal,

    @field:NotEmpty(message = "At least one category is required")
    @field:Size(max = 5, message = "Cannot have more than 5 categories")
    val categories: List<@NotBlank @Size(max = 50) String>,

    @field:Valid
    @field:NotNull(message = "Dimensions are required")
    val dimensions: ProductDimensions,

    @field:Valid
    val images: List<ProductImage>? = null,

    @field:Future(message = "Launch date must be in the future")
    val launchDate: LocalDateTime? = null,

    // Using Kotlin's sealed class for type-safe status
    val status: ProductStatus = ProductStatus.DRAFT
)

data class ProductDimensions(
    @field:Positive(message = "Length must be positive")
    @field:DecimalMax(value = "1000.0", message = "Length cannot exceed 1000 cm")
    val length: BigDecimal,

    @field:Positive(message = "Width must be positive")
    @field:DecimalMax(value = "1000.0", message = "Width cannot exceed 1000 cm")
    val width: BigDecimal,

    @field:Positive(message = "Height must be positive")
    @field:DecimalMax(value = "1000.0", message = "Height cannot exceed 1000 cm")
    val height: BigDecimal,

    @field:NotNull(message = "Unit is required")
    val unit: DimensionUnit
) {
    // Computed property for volume
    val volume: BigDecimal
        get() = length * width * height

    // Validation method
    @AssertTrue(message = "Product dimensions result in impractical volume")
    fun isVolumeReasonable(): Boolean {
        return volume <= BigDecimal("1000000") // 1 cubic meter in cubic cm
    }
}

data class ProductImage(
    @field:NotBlank(message = "Image URL is required")
    @field:URL(message = "Image URL must be valid")
    val url: String,

    @field:NotBlank(message = "Alt text is required for accessibility")
    @field:Size(max = 200, message = "Alt text cannot exceed 200 characters")
    val altText: String,

    @field:PositiveOrZero(message = "Display order cannot be negative")
    val displayOrder: Int = 0
)

// Kotlin sealed class for type-safe status
sealed class ProductStatus {
    object DRAFT : ProductStatus()
    object PENDING_REVIEW : ProductStatus()
    object APPROVED : ProductStatus()
    object PUBLISHED : ProductStatus()
    object DISCONTINUED : ProductStatus()

    override fun toString(): String = this::class.simpleName ?: "UNKNOWN"
}

enum class DimensionUnit {
    CM, INCH, M, FT
}

// Group validation for different validation scenarios
interface BasicValidation
interface DetailedValidation : BasicValidation
interface PublishingValidation : DetailedValidation

data class ProductPublishRequest(
    @field:Valid
    val product: ProductRequest,

    @field:NotBlank(message = "Publishing reason is required", groups = [PublishingValidation::class])
    @field:Size(max = 500, message = "Publishing reason cannot exceed 500 characters", groups = [PublishingValidation::class])
    val publishingReason: String? = null,

    @field:NotNull(message = "Reviewer is required for publishing", groups = [PublishingValidation::class])
    val reviewedBy: Long? = null,

    @field:AssertTrue(message = "Product must be approved before publishing", groups = [PublishingValidation::class])
    fun isProductApproved(): Boolean = product.status == ProductStatus.APPROVED
)
```

### Programmatic Validation in Services

Sometimes you need to perform validation programmatically in your service layer:

```kotlin
@Service
@Transactional
class ProductValidationService(
    private val validator: Validator,
    private val productRepository: ProductRepository
) {

    /**
     * Validates a product using different validation groups based on the operation.
     */
    fun validateProduct(request: ProductRequest, operation: ValidationOperation): ValidationResult {
        val groups = when (operation) {
            ValidationOperation.CREATE -> arrayOf(BasicValidation::class.java)
            ValidationOperation.UPDATE -> arrayOf(DetailedValidation::class.java)
            ValidationOperation.PUBLISH -> arrayOf(PublishingValidation::class.java)
        }

        val violations = validator.validate(request, *groups)

        val errors = violations.map { violation ->
            ValidationError(
                field = violation.propertyPath.toString(),
                message = violation.message,
                code = violation.constraintDescriptor.annotation.annotationClass.simpleName ?: "VALIDATION_ERROR",
                rejectedValue = violation.invalidValue
            )
        }.toMutableList()

        // Add custom business logic validation
        errors.addAll(validateBusinessRules(request, operation))

        return ValidationResult(
            isValid = errors.isEmpty(),
            errors = errors
        )
    }

    /**
     * Validates individual fields programmatically.
     */
    fun validateField(obj: Any, fieldName: String): List<ValidationError> {
        val violations = validator.validateProperty(obj, fieldName)

        return violations.map { violation ->
            ValidationError(
                field = violation.propertyPath.toString(),
                message = violation.message,
                code = violation.constraintDescriptor.annotation.annotationClass.simpleName ?: "VALIDATION_ERROR",
                rejectedValue = violation.invalidValue
            )
        }
    }

    /**
     * Validates a value against specific constraints.
     */
    fun <T> validateValue(beanType: Class<T>, fieldName: String, value: Any?): List<ValidationError> {
        val violations = validator.validateValue(beanType, fieldName, value)

        return violations.map { violation ->
            ValidationError(
                field = violation.propertyPath.toString(),
                message = violation.message,
                code = violation.constraintDescriptor.annotation.annotationClass.simpleName ?: "VALIDATION_ERROR",
                rejectedValue = violation.invalidValue
            )
        }
    }

    private fun validateBusinessRules(request: ProductRequest, operation: ValidationOperation): List<ValidationError> {
        val errors = mutableListOf<ValidationError>()

        when (operation) {
            ValidationOperation.CREATE -> {
                // Check for duplicate product names
                if (productRepository.existsByName(request.name)) {
                    errors.add(ValidationError(
                        field = "name",
                        message = "Product with name '${request.name}' already exists",
                        code = "DUPLICATE_PRODUCT_NAME"
                    ))
                }
            }

            ValidationOperation.PUBLISH -> {
                // Additional validation for publishing
                if (request.images.isNullOrEmpty()) {
                    errors.add(ValidationError(
                        field = "images",
                        message = "At least one product image is required for publishing",
                        code = "IMAGES_REQUIRED_FOR_PUBLISHING"
                    ))
                }

                if (request.launchDate == null) {
                    errors.add(ValidationError(
                        field = "launchDate",
                        message = "Launch date is required for publishing",
                        code = "LAUNCH_DATE_REQUIRED_FOR_PUBLISHING"
                    ))
                }
            }

            else -> { /* No additional validation */ }
        }

        return errors
    }
}

enum class ValidationOperation {
    CREATE, UPDATE, PUBLISH
}

// Extension function for easy validation
fun <T> Validator.validateKotlin(obj: T, vararg groups: Class<*>): ValidationResult {
    val violations = this.validate(obj, *groups)

    val errors = violations.map { violation ->
        ValidationError(
            field = violation.propertyPath.toString(),
            message = violation.message,
            code = violation.constraintDescriptor.annotation.annotationClass.simpleName ?: "VALIDATION_ERROR",
            rejectedValue = violation.invalidValue
        )
    }

    return ValidationResult(
        isValid = errors.isEmpty(),
        errors = errors
    )
}
```

## 10.3 Spring Boot Integration with Validation

Spring Boot provides excellent integration with validation frameworks. Let's explore how to configure and customize this integration effectively.

### Global Validation Configuration

```kotlin
@Configuration
@EnableConfigurationProperties(ValidationProperties::class)
class ValidationConfiguration(
    private val validationProperties: ValidationProperties
) {

    @Bean
    @Primary
    fun validator(messageSource: MessageSource): LocalValidatorFactoryBean {
        val validator = LocalValidatorFactoryBean()
        validator.setValidationMessageSource(messageSource)
        validator.setParameterNameDiscoverer(DefaultParameterNameDiscoverer())

        // Configure Hibernate Validator specific settings
        validator.setConfigurationInitializer { configuration ->
            configuration
                .messageInterpolator(
                    ResourceBundleMessageInterpolator(
                        PlatformResourceBundleLocator("ValidationMessages")
                    )
                )
                .clockProvider(DefaultClockProvider.INSTANCE)
                .addValueExtractor(OptionalValueExtractor())
        }

        return validator
    }

    @Bean
    fun methodValidationPostProcessor(validator: LocalValidatorFactoryBean): MethodValidationPostProcessor {
        val processor = MethodValidationPostProcessor()
        processor.setValidator(validator)
        processor.setProxyTargetClass(true)
        return processor
    }

    @Bean
    @ConditionalOnMissingBean
    fun validationService(validator: Validator): ValidationService {
        return ValidationService(validator)
    }
}

// Custom validation properties
@ConfigurationProperties(prefix = "app.validation")
data class ValidationProperties(
    val failFast: Boolean = false,
    val maxViolations: Int = 100,
    val enableMethodValidation: Boolean = true,
    val customMessages: Map<String, String> = emptyMap()
)

// Custom value extractor for Kotlin's nullable types
class OptionalValueExtractor : ValueExtractor<Optional<*>> {

    override fun extractValues(originalValue: Optional<*>, receiver: ValueExtractor.ValueReceiver) {
        if (originalValue.isPresent) {
            receiver.value(null, originalValue.get())
        }
    }
}
```

### Controller-Level Validation Integration

```kotlin
// Enhanced controller with comprehensive validation
@RestController
@RequestMapping("/api/v1/products")
@Validated
class ProductController(
    private val productService: ProductService,
    private val validationService: ValidationService
) {

    @PostMapping
    fun createProduct(
        @Valid @RequestBody request: ProductRequest,
        bindingResult: BindingResult
    ): ResponseEntity<*> {

        // Manual validation for complex scenarios
        val validationResult = validationService.validate(request, ValidationOperation.CREATE)
        if (!validationResult.isValid) {
            return ResponseEntity.badRequest().body(
                ErrorResponse(
                    message = "Validation failed",
                    errors = validationResult.errors
                )
            )
        }

        val product = productService.createProduct(request)
        return ResponseEntity.status(HttpStatus.CREATED).body(product)
    }

    @PutMapping("/{id}")
    fun updateProduct(
        @PathVariable @Positive id: Long,
        @Valid @RequestBody request: ProductRequest
    ): ResponseEntity<ProductResponse> {
        val product = productService.updateProduct(id, request)
        return ResponseEntity.ok(product)
    }

    // Method-level validation for complex parameters
    @GetMapping("/search")
    fun searchProducts(
        @RequestParam(required = false)
        @Size(min = 2, max = 100, message = "Search term must be between 2 and 100 characters")
        query: String?,

        @RequestParam(defaultValue = "0")
        @Min(value = 0, message = "Page number cannot be negative")
        page: Int,

        @RequestParam(defaultValue = "20")
        @Min(value = 1, message = "Page size must be at least 1")
        @Max(value = 100, message = "Page size cannot exceed 100")
        size: Int,

        @RequestParam(required = false)
        @DecimalMin(value = "0.01", message = "Minimum price must be greater than 0")
        minPrice: BigDecimal?,

        @RequestParam(required = false)
        @DecimalMax(value = "999999.99", message = "Maximum price must be less than 1,000,000")
        maxPrice: BigDecimal?,

        @RequestParam(required = false)
        categories: List<@NotBlank @Size(max = 50) String>?
    ): ResponseEntity<Page<ProductResponse>> {

        // Custom validation for parameter relationships
        if (minPrice != null && maxPrice != null && minPrice > maxPrice) {
            throw IllegalArgumentException("Minimum price cannot be greater than maximum price")
        }

        val products = productService.searchProducts(query, page, size, minPrice, maxPrice, categories)
        return ResponseEntity.ok(products)
    }

    // Batch operations with validation
    @PostMapping("/batch")
    fun createProductsBatch(
        @Valid @RequestBody requests: List<@Valid ProductRequest>
    ): ResponseEntity<BatchResponse<ProductResponse>> {

        if (requests.size > 100) {
            throw IllegalArgumentException("Cannot process more than 100 products in a single batch")
        }

        val results = productService.createProductsBatch(requests)
        return ResponseEntity.ok(results)
    }

    // Publishing with group validation
    @PostMapping("/{id}/publish")
    fun publishProduct(
        @PathVariable @Positive id: Long,
        @RequestBody @Validated(PublishingValidation::class) request: ProductPublishRequest
    ): ResponseEntity<ProductResponse> {
        val product = productService.publishProduct(id, request)
        return ResponseEntity.ok(product)
    }
}

// Batch response for bulk operations
data class BatchResponse<T>(
    val successful: List<T>,
    val failed: List<BatchError>,
    val totalProcessed: Int,
    val successCount: Int,
    val failureCount: Int
) {
    companion object {
        fun <T> create(successful: List<T>, failed: List<BatchError>): BatchResponse<T> {
            return BatchResponse(
                successful = successful,
                failed = failed,
                totalProcessed = successful.size + failed.size,
                successCount = successful.size,
                failureCount = failed.size
            )
        }
    }
}

data class BatchError(
    val index: Int,
    val errors: List<ValidationError>,
    val originalRequest: Any
)
```

### Service-Level Validation Integration

```kotlin
@Service
@Transactional
class ProductService(
    private val productRepository: ProductRepository,
    private val validationService: ValidationService,
    private val productMapper: ProductMapper
) {

    /**
     * Creates a product with comprehensive validation.
     */
    fun createProduct(request: ProductRequest): ProductResponse {
        // Validate using service-level validation
        val validationResult = validationService.validate(request, ValidationOperation.CREATE)
        validationResult.throwIfInvalid()

        val product = productMapper.toEntity(request)
        val savedProduct = productRepository.save(product)

        return productMapper.toResponse(savedProduct)
    }

    /**
     * Updates a product with partial validation.
     */
    fun updateProduct(id: Long, request: ProductRequest): ProductResponse {
        val existingProduct = productRepository.findById(id).orElse(null)
            ?: throw ProductNotFoundException("Product not found with ID: $id")

        // Validate update operation
        val validationResult = validationService.validate(request, ValidationOperation.UPDATE)
        validationResult.throwIfInvalid()

        val updatedProduct = productMapper.updateEntity(existingProduct, request)
        val savedProduct = productRepository.save(updatedProduct)

        return productMapper.toResponse(savedProduct)
    }

    /**
     * Batch creation with individual validation tracking.
     */
    fun createProductsBatch(requests: List<ProductRequest>): BatchResponse<ProductResponse> {
        val successful = mutableListOf<ProductResponse>()
        val failed = mutableListOf<BatchError>()

        requests.forEachIndexed { index, request ->
            try {
                val validationResult = validationService.validate(request, ValidationOperation.CREATE)

                if (validationResult.isValid) {
                    val product = createProduct(request)
                    successful.add(product)
                } else {
                    failed.add(BatchError(
                        index = index,
                        errors = validationResult.errors,
                        originalRequest = request
                    ))
                }
            } catch (ex: Exception) {
                failed.add(BatchError(
                    index = index,
                    errors = listOf(ValidationError(
                        field = "general",
                        message = ex.message ?: "Unknown error occurred",
                        code = "PROCESSING_ERROR"
                    )),
                    originalRequest = request
                ))
            }
        }

        return BatchResponse.create(successful, failed)
    }

    /**
     * Publishes a product with strict validation.
     */
    fun publishProduct(id: Long, request: ProductPublishRequest): ProductResponse {
        val product = productRepository.findById(id).orElse(null)
            ?: throw ProductNotFoundException("Product not found with ID: $id")

        // Validate publishing requirements
        val validationResult = validationService.validate(
            request.product,
            ValidationOperation.PUBLISH
        )
        validationResult.throwIfInvalid()

        // Additional business validation for publishing
        if (product.status != ProductStatus.APPROVED) {
            throw IllegalStateException("Product must be approved before publishing")
        }

        product.status = ProductStatus.PUBLISHED
        product.publishedAt = LocalDateTime.now()
        product.publishedBy = request.reviewedBy

        val savedProduct = productRepository.save(product)
        return productMapper.toResponse(savedProduct)
    }
}

// Validation service wrapper
@Component
class ValidationService(private val validator: Validator) {

    fun <T> validate(obj: T, operation: ValidationOperation? = null): ValidationResult {
        val groups = when (operation) {
            ValidationOperation.CREATE -> arrayOf(BasicValidation::class.java)
            ValidationOperation.UPDATE -> arrayOf(DetailedValidation::class.java)
            ValidationOperation.PUBLISH -> arrayOf(PublishingValidation::class.java)
            null -> arrayOf<Class<*>>()
        }

        return validator.validateKotlin(obj, *groups)
    }

    fun <T> validateProperty(obj: T, propertyName: String): ValidationResult {
        val violations = validator.validateProperty(obj, propertyName)

        val errors = violations.map { violation ->
            ValidationError(
                field = violation.propertyPath.toString(),
                message = violation.message,
                code = violation.constraintDescriptor.annotation.annotationClass.simpleName ?: "VALIDATION_ERROR",
                rejectedValue = violation.invalidValue
            )
        }

        return ValidationResult(
            isValid = errors.isEmpty(),
            errors = errors
        )
    }
}
```

## 10.4 Custom Validation

Sometimes built-in validation annotations aren't sufficient for your business requirements. Let's explore how to create custom validation annotations and validators in Kotlin.

### Creating Custom Validation Annotations

```kotlin
// Custom validation annotation for password strength
@Target(AnnotationTarget.FIELD, AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [PasswordStrengthValidator::class])
@MustBeDocumented
annotation class ValidPassword(
    val message: String = "Password does not meet strength requirements",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = [],
    val minLength: Int = 8,
    val requireUppercase: Boolean = true,
    val requireLowercase: Boolean = true,
    val requireDigit: Boolean = true,
    val requireSpecialChar: Boolean = true,
    val maxRepeatingChars: Int = 3
)

// Password strength validator
class PasswordStrengthValidator : ConstraintValidator<ValidPassword, String> {

    private var minLength: Int = 8
    private var requireUppercase: Boolean = true
    private var requireLowercase: Boolean = true
    private var requireDigit: Boolean = true
    private var requireSpecialChar: Boolean = true
    private var maxRepeatingChars: Int = 3

    override fun initialize(annotation: ValidPassword) {
        minLength = annotation.minLength
        requireUppercase = annotation.requireUppercase
        requireLowercase = annotation.requireLowercase
        requireDigit = annotation.requireDigit
        requireSpecialChar = annotation.requireSpecialChar
        maxRepeatingChars = annotation.maxRepeatingChars
    }

    override fun isValid(value: String?, context: ConstraintValidatorContext): Boolean {
        if (value == null) return false

        val violations = mutableListOf<String>()

        // Check minimum length
        if (value.length < minLength) {
            violations.add("must be at least $minLength characters long")
        }

        // Check character requirements
        if (requireUppercase && !value.any { it.isUpperCase() }) {
            violations.add("must contain at least one uppercase letter")
        }

        if (requireLowercase && !value.any { it.isLowerCase() }) {
            violations.add("must contain at least one lowercase letter")
        }

        if (requireDigit && !value.any { it.isDigit() }) {
            violations.add("must contain at least one digit")
        }

        if (requireSpecialChar && !value.any { !it.isLetterOrDigit() }) {
            violations.add("must contain at least one special character")
        }

        // Check for too many repeating characters
        if (hasExcessiveRepeating(value, maxRepeatingChars)) {
            violations.add("cannot have more than $maxRepeatingChars consecutive repeating characters")
        }

        // If there are violations, customize the error message
        if (violations.isNotEmpty()) {
            context.disableDefaultConstraintViolation()
            context.buildConstraintViolationWithTemplate(
                "Password ${violations.joinToString(", ")}"
            ).addConstraintViolation()
            return false
        }

        return true
    }

    private fun hasExcessiveRepeating(value: String, maxRepeating: Int): Boolean {
        var count = 1
        for (i in 1 until value.length) {
            if (value[i] == value[i - 1]) {
                count++
                if (count > maxRepeating) return true
            } else {
                count = 1
            }
        }
        return false
    }
}

// Custom validation for date ranges
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [DateRangeValidator::class])
@MustBeDocumented
annotation class ValidDateRange(
    val message: String = "End date must be after start date",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = [],
    val startDateField: String = "startDate",
    val endDateField: String = "endDate",
    val allowSameDate: Boolean = false
)

// Date range validator
class DateRangeValidator : ConstraintValidator<ValidDateRange, Any> {

    private var startDateField: String = ""
    private var endDateField: String = ""
    private var allowSameDate: Boolean = false

    override fun initialize(annotation: ValidDateRange) {
        startDateField = annotation.startDateField
        endDateField = annotation.endDateField
        allowSameDate = annotation.allowSameDate
    }

    override fun isValid(obj: Any?, context: ConstraintValidatorContext): Boolean {
        if (obj == null) return true

        try {
            val startDate = getFieldValue(obj, startDateField) as? LocalDate
            val endDate = getFieldValue(obj, endDateField) as? LocalDate

            if (startDate == null || endDate == null) return true

            return if (allowSameDate) {
                !endDate.isBefore(startDate)
            } else {
                endDate.isAfter(startDate)
            }

        } catch (ex: Exception) {
            return false
        }
    }

    private fun getFieldValue(obj: Any, fieldName: String): Any? {
        val field = obj.javaClass.getDeclaredField(fieldName)
        field.isAccessible = true
        return field.get(obj)
    }
}

// Custom validation for unique collection elements
@Target(AnnotationTarget.FIELD, AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [UniqueElementsValidator::class])
@MustBeDocumented
annotation class UniqueElements(
    val message: String = "Collection must contain unique elements",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = [],
    val property: String = "" // Property name to check uniqueness on
)

// Unique elements validator
class UniqueElementsValidator : ConstraintValidator<UniqueElements, Collection<*>> {

    private var property: String = ""

    override fun initialize(annotation: UniqueElements) {
        property = annotation.property
    }

    override fun isValid(collection: Collection<*>?, context: ConstraintValidatorContext): Boolean {
        if (collection.isNullOrEmpty()) return true

        return if (property.isBlank()) {
            // Check uniqueness of elements themselves
            collection.size == collection.toSet().size
        } else {
            // Check uniqueness of specified property
            val propertyValues = collection.mapNotNull { element ->
                getPropertyValue(element, property)
            }
            propertyValues.size == propertyValues.toSet().size
        }
    }

    private fun getPropertyValue(obj: Any?, propertyName: String): Any? {
        if (obj == null) return null

        try {
            val field = obj.javaClass.getDeclaredField(propertyName)
            field.isAccessible = true
            return field.get(obj)
        } catch (ex: Exception) {
            return null
        }
    }
}
```

### Using Custom Validation Annotations

```kotlin
// Data classes using custom validation annotations
data class CreateAccountRequest(
    @field:NotBlank(message = "Username is required")
    @field:Size(min = 3, max = 30, message = "Username must be between 3 and 30 characters")
    val username: String,

    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Email must be valid")
    val email: String,

    @field:NotBlank(message = "Password is required")
    @field:ValidPassword(
        minLength = 10,
        requireUppercase = true,
        requireLowercase = true,
        requireDigit = true,
        requireSpecialChar = true,
        maxRepeatingChars = 2
    )
    val password: String,

    @field:NotBlank(message = "Password confirmation is required")
    val passwordConfirmation: String
) {
    // Class-level validation to ensure passwords match
    @AssertTrue(message = "Password and confirmation must match")
    fun isPasswordConfirmed(): Boolean {
        return password == passwordConfirmation
    }
}

@ValidDateRange(
    startDateField = "startDate",
    endDateField = "endDate",
    allowSameDate = false,
    message = "End date must be after start date"
)
data class EventRequest(
    @field:NotBlank(message = "Event name is required")
    val name: String,

    @field:NotNull(message = "Start date is required")
    @field:Future(message = "Start date must be in the future")
    val startDate: LocalDate,

    @field:NotNull(message = "End date is required")
    val endDate: LocalDate,

    @field:UniqueElements(
        property = "email",
        message = "Each attendee email must be unique"
    )
    val attendees: List<AttendeeRequest> = emptyList()
)

data class AttendeeRequest(
    @field:NotBlank(message = "Attendee name is required")
    val name: String,

    @field:NotBlank(message = "Attendee email is required")
    @field:Email(message = "Attendee email must be valid")
    val email: String
)

// Product with multiple custom validations
data class AdvancedProductRequest(
    @field:NotBlank(message = "Product name is required")
    val name: String,

    @field:NotNull(message = "Price is required")
    @field:Positive(message = "Price must be positive")
    val price: BigDecimal,

    @field:UniqueElements(message = "Product tags must be unique")
    val tags: List<@NotBlank @Size(max = 20) String> = emptyList(),

    @field:Valid
    val variants: List<ProductVariant> = emptyList()
)

data class ProductVariant(
    @field:NotBlank(message = "Variant name is required")
    val name: String,

    @field:NotNull(message = "Variant price is required")
    @field:Positive(message = "Variant price must be positive")
    val price: BigDecimal,

    @field:PositiveOrZero(message = "Stock quantity cannot be negative")
    val stockQuantity: Int = 0
)
```

### Complex Custom Validators

For more complex validation scenarios, you might need validators that interact with the application context:

```kotlin
// Context-aware validation annotation
@Target(AnnotationTarget.FIELD, AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [UniqueUsernameValidator::class])
@MustBeDocumented
annotation class UniqueUsername(
    val message: String = "Username is already taken",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = [],
    val ignoreCase: Boolean = true
)

// Context-aware validator that uses repository
@Component
class UniqueUsernameValidator(
    private val userRepository: UserRepository
) : ConstraintValidator<UniqueUsername, String> {

    private var ignoreCase: Boolean = true

    override fun initialize(annotation: UniqueUsername) {
        ignoreCase = annotation.ignoreCase
    }

    override fun isValid(username: String?, context: ConstraintValidatorContext): Boolean {
        if (username.isNullOrBlank()) return true

        return if (ignoreCase) {
            !userRepository.existsByUsernameIgnoreCase(username)
        } else {
            !userRepository.existsByUsername(username)
        }
    }
}

// Complex business rule validator
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [OrderValidationValidator::class])
@MustBeDocumented
annotation class ValidOrder(
    val message: String = "Order validation failed",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

@Component
class OrderValidationValidator(
    private val productService: ProductService,
    private val inventoryService: InventoryService,
    private val userService: UserService
) : ConstraintValidator<ValidOrder, OrderRequest> {

    override fun isValid(order: OrderRequest?, context: ConstraintValidatorContext): Boolean {
        if (order == null) return true

        context.disableDefaultConstraintViolation()
        var isValid = true

        // Validate customer exists and is active
        if (!userService.isActiveUser(order.customerId)) {
            context.buildConstraintViolationWithTemplate(
                "Customer with ID ${order.customerId} is not active"
            ).addPropertyNode("customerId").addConstraintViolation()
            isValid = false
        }

        // Validate each order item
        order.items.forEachIndexed { index, item ->
            // Check product exists
            if (!productService.productExists(item.productId)) {
                context.buildConstraintViolationWithTemplate(
                    "Product with ID ${item.productId} does not exist"
                ).addPropertyNode("items[$index].productId").addConstraintViolation()
                isValid = false
            }

            // Check inventory availability
            if (!inventoryService.isAvailable(item.productId, item.quantity)) {
                context.buildConstraintViolationWithTemplate(
                    "Insufficient inventory for product ${item.productId}"
                ).addPropertyNode("items[$index].quantity").addConstraintViolation()
                isValid = false
            }
        }

        // Validate total amount
        val calculatedTotal = order.items.sumOf { item ->
            productService.getPrice(item.productId) * item.quantity.toBigDecimal()
        }

        if (order.totalAmount != calculatedTotal) {
            context.buildConstraintViolationWithTemplate(
                "Total amount ${order.totalAmount} does not match calculated total $calculatedTotal"
            ).addPropertyNode("totalAmount").addConstraintViolation()
            isValid = false
        }

        return isValid
    }
}

@ValidOrder
data class OrderRequest(
    @field:NotNull(message = "Customer ID is required")
    @field:Positive(message = "Customer ID must be positive")
    val customerId: Long,

    @field:NotEmpty(message = "Order must contain at least one item")
    @field:Size(max = 50, message = "Order cannot contain more than 50 items")
    val items: List<@Valid OrderItemRequest>,

    @field:NotNull(message = "Total amount is required")
    @field:Positive(message = "Total amount must be positive")
    val totalAmount: BigDecimal,

    @field:Valid
    val shippingAddress: AddressRequest,

    @field:Valid
    val billingAddress: AddressRequest? = null
)

data class OrderItemRequest(
    @field:NotNull(message = "Product ID is required")
    @field:Positive(message = "Product ID must be positive")
    val productId: Long,

    @field:NotNull(message = "Quantity is required")
    @field:Min(value = 1, message = "Quantity must be at least 1")
    @field:Max(value = 100, message = "Quantity cannot exceed 100")
    val quantity: Int
)
```

## 10.5 Exception Handling and Custom Exceptions

Robust exception handling is crucial for providing meaningful feedback to API consumers and maintaining application stability. Let's build a comprehensive exception handling framework.

### Custom Exception Hierarchy

```kotlin
// Base exception classes
abstract class ApplicationException(
    message: String,
    cause: Throwable? = null,
    val errorCode: String,
    val httpStatus: HttpStatus = HttpStatus.INTERNAL_SERVER_ERROR
) : RuntimeException(message, cause)

// Domain-specific exceptions
class ValidationException(
    message: String = "Validation failed",
    val errors: List<ValidationError>,
    cause: Throwable? = null
) : ApplicationException(
    message = message,
    cause = cause,
    errorCode = "VALIDATION_ERROR",
    httpStatus = HttpStatus.BAD_REQUEST
)

class ResourceNotFoundException(
    resource: String,
    identifier: Any,
    cause: Throwable? = null
) : ApplicationException(
    message = "$resource not found with identifier: $identifier",
    cause = cause,
    errorCode = "RESOURCE_NOT_FOUND",
    httpStatus = HttpStatus.NOT_FOUND
)

class BusinessRuleException(
    message: String,
    val ruleCode: String,
    cause: Throwable? = null
) : ApplicationException(
    message = message,
    cause = cause,
    errorCode = ruleCode,
    httpStatus = HttpStatus.UNPROCESSABLE_ENTITY
)

class DuplicateResourceException(
    resource: String,
    field: String,
    value: Any,
    cause: Throwable? = null
) : ApplicationException(
    message = "$resource with $field '$value' already exists",
    cause = cause,
    errorCode = "DUPLICATE_RESOURCE",
    httpStatus = HttpStatus.CONFLICT
)

class InsufficientPermissionException(
    action: String,
    resource: String,
    cause: Throwable? = null
) : ApplicationException(
    message = "Insufficient permission to $action $resource",
    cause = cause,
    errorCode = "INSUFFICIENT_PERMISSION",
    httpStatus = HttpStatus.FORBIDDEN
)

class ExternalServiceException(
    service: String,
    operation: String,
    message: String,
    cause: Throwable? = null
) : ApplicationException(
    message = "External service '$service' failed during '$operation': $message",
    cause = cause,
    errorCode = "EXTERNAL_SERVICE_ERROR",
    httpStatus = HttpStatus.BAD_GATEWAY
)

// Specific domain exceptions
class UserNotFoundException(userId: Long) : ResourceNotFoundException("User", userId)
class ProductNotFoundException(productId: Long) : ResourceNotFoundException("Product", productId)
class OrderNotFoundException(orderId: Long) : ResourceNotFoundException("Order", orderId)

class UsernameAlreadyExistsException(username: String) :
    DuplicateResourceException("User", "username", username)

class EmailAlreadyExistsException(email: String) :
    DuplicateResourceException("User", "email", email)

class InsufficientStockException(
    productId: Long,
    requested: Int,
    available: Int
) : BusinessRuleException(
    message = "Insufficient stock for product $productId. Requested: $requested, Available: $available",
    ruleCode = "INSUFFICIENT_STOCK"
)

class AccountNotActiveException(userId: Long) : BusinessRuleException(
    message = "User account $userId is not active",
    ruleCode = "ACCOUNT_NOT_ACTIVE"
)
```

### Global Exception Handler

```kotlin
@RestControllerAdvice
@Order(Ordered.HIGHEST_PRECEDENCE)
class GlobalExceptionHandler {

    private val logger = LoggerFactory.getLogger(GlobalExceptionHandler::class.java)

    /**
     * Handles validation exceptions from @Valid annotations.
     */
    @ExceptionHandler(MethodArgumentNotValidException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleValidationException(ex: MethodArgumentNotValidException): ErrorResponse {
        logger.debug("Validation error occurred", ex)

        val errors = ex.bindingResult.fieldErrors.map { fieldError ->
            ValidationError(
                field = fieldError.field,
                message = fieldError.defaultMessage ?: "Invalid value",
                code = fieldError.code ?: "VALIDATION_ERROR",
                rejectedValue = fieldError.rejectedValue
            )
        }.plus(
            ex.bindingResult.globalErrors.map { globalError ->
                ValidationError(
                    field = globalError.objectName,
                    message = globalError.defaultMessage ?: "Invalid object",
                    code = globalError.code ?: "VALIDATION_ERROR"
                )
            }
        )

        return ErrorResponse(
            message = "Validation failed",
            errorCode = "VALIDATION_ERROR",
            details = errors,
            timestamp = Instant.now()
        )
    }

    /**
     * Handles constraint violation exceptions from method-level validation.
     */
    @ExceptionHandler(ConstraintViolationException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleConstraintViolationException(ex: ConstraintViolationException): ErrorResponse {
        logger.debug("Constraint violation occurred", ex)

        val errors = ex.constraintViolations.map { violation ->
            ValidationError(
                field = violation.propertyPath.toString(),
                message = violation.message,
                code = violation.constraintDescriptor.annotation.annotationClass.simpleName ?: "CONSTRAINT_VIOLATION",
                rejectedValue = violation.invalidValue
            )
        }

        return ErrorResponse(
            message = "Constraint violation",
            errorCode = "CONSTRAINT_VIOLATION",
            details = errors,
            timestamp = Instant.now()
        )
    }

    /**
     * Handles custom application exceptions.
     */
    @ExceptionHandler(ApplicationException::class)
    fun handleApplicationException(ex: ApplicationException): ResponseEntity<ErrorResponse> {
        when (ex.httpStatus.series()) {
            HttpStatus.Series.CLIENT_ERROR -> logger.warn("Client error occurred: {}", ex.message, ex)
            HttpStatus.Series.SERVER_ERROR -> logger.error("Server error occurred: {}", ex.message, ex)
            else -> logger.info("Application error occurred: {}", ex.message, ex)
        }

        val errorResponse = when (ex) {
            is ValidationException -> ErrorResponse(
                message = ex.message,
                errorCode = ex.errorCode,
                details = ex.errors,
                timestamp = Instant.now()
            )
            else -> ErrorResponse(
                message = ex.message,
                errorCode = ex.errorCode,
                timestamp = Instant.now()
            )
        }

        return ResponseEntity.status(ex.httpStatus).body(errorResponse)
    }

    /**
     * Handles database constraint violations.
     */
    @ExceptionHandler(DataIntegrityViolationException::class)
    @ResponseStatus(HttpStatus.CONFLICT)
    fun handleDataIntegrityViolationException(ex: DataIntegrityViolationException): ErrorResponse {
        logger.warn("Database constraint violation occurred", ex)

        val errorDetails = parseDataIntegrityViolation(ex)

        return ErrorResponse(
            message = errorDetails.message,
            errorCode = errorDetails.code,
            details = listOf(ValidationError(
                field = errorDetails.field,
                message = errorDetails.message,
                code = errorDetails.code
            )),
            timestamp = Instant.now()
        )
    }

    /**
     * Handles resource access exceptions (e.g., file not found, database connection issues).
     */
    @ExceptionHandler(DataAccessException::class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    fun handleDataAccessException(ex: DataAccessException): ErrorResponse {
        logger.error("Data access error occurred", ex)

        return ErrorResponse(
            message = "A database error occurred",
            errorCode = "DATABASE_ERROR",
            timestamp = Instant.now()
        )
    }

    /**
     * Handles HTTP message conversion errors.
     */
    @ExceptionHandler(HttpMessageNotReadableException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleHttpMessageNotReadableException(ex: HttpMessageNotReadableException): ErrorResponse {
        logger.debug("HTTP message not readable", ex)

        val message = when {
            ex.message?.contains("JSON parse error") == true -> "Invalid JSON format"
            ex.message?.contains("Required request body is missing") == true -> "Request body is required"
            else -> "Invalid request format"
        }

        return ErrorResponse(
            message = message,
            errorCode = "INVALID_REQUEST_FORMAT",
            timestamp = Instant.now()
        )
    }

    /**
     * Handles method argument type mismatches.
     */
    @ExceptionHandler(MethodArgumentTypeMismatchException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleTypeMismatchException(ex: MethodArgumentTypeMismatchException): ErrorResponse {
        logger.debug("Method argument type mismatch", ex)

        val error = ValidationError(
            field = ex.name,
            message = "Invalid value '${ex.value}' for parameter '${ex.name}'. Expected type: ${ex.requiredType?.simpleName}",
            code = "TYPE_MISMATCH",
            rejectedValue = ex.value
        )

        return ErrorResponse(
            message = "Invalid parameter type",
            errorCode = "TYPE_MISMATCH",
            details = listOf(error),
            timestamp = Instant.now()
        )
    }

    /**
     * Handles missing required parameters.
     */
    @ExceptionHandler(MissingServletRequestParameterException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleMissingParameterException(ex: MissingServletRequestParameterException): ErrorResponse {
        logger.debug("Missing required parameter", ex)

        val error = ValidationError(
            field = ex.parameterName,
            message = "Required parameter '${ex.parameterName}' is missing",
            code = "MISSING_PARAMETER"
        )

        return ErrorResponse(
            message = "Missing required parameter",
            errorCode = "MISSING_PARAMETER",
            details = listOf(error),
            timestamp = Instant.now()
        )
    }

    /**
     * Handles authentication exceptions.
     */
    @ExceptionHandler(AuthenticationException::class)
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    fun handleAuthenticationException(ex: AuthenticationException): ErrorResponse {
        logger.warn("Authentication failed: {}", ex.message)

        return ErrorResponse(
            message = "Authentication failed",
            errorCode = "AUTHENTICATION_FAILED",
            timestamp = Instant.now()
        )
    }

    /**
     * Handles access denied exceptions.
     */
    @ExceptionHandler(AccessDeniedException::class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    fun handleAccessDeniedException(ex: AccessDeniedException): ErrorResponse {
        logger.warn("Access denied: {}", ex.message)

        return ErrorResponse(
            message = "Access denied",
            errorCode = "ACCESS_DENIED",
            timestamp = Instant.now()
        )
    }

    /**
     * Handles all other unexpected exceptions.
     */
    @ExceptionHandler(Exception::class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    fun handleGenericException(ex: Exception): ErrorResponse {
        logger.error("Unexpected error occurred", ex)

        return ErrorResponse(
            message = "An unexpected error occurred",
            errorCode = "INTERNAL_SERVER_ERROR",
            timestamp = Instant.now()
        )
    }

    private fun parseDataIntegrityViolation(ex: DataIntegrityViolationException): ErrorDetails {
        val message = ex.rootCause?.message ?: ex.message ?: "Database constraint violation"

        return when {
            message.contains("uk_users_username") -> ErrorDetails(
                field = "username",
                message = "Username is already taken",
                code = "USERNAME_ALREADY_EXISTS"
            )
            message.contains("uk_users_email") -> ErrorDetails(
                field = "email",
                message = "Email is already registered",
                code = "EMAIL_ALREADY_EXISTS"
            )
            message.contains("foreign key constraint") -> ErrorDetails(
                field = "reference",
                message = "Referenced entity does not exist",
                code = "INVALID_REFERENCE"
            )
            else -> ErrorDetails(
                field = "database",
                message = "Database constraint violation",
                code = "CONSTRAINT_VIOLATION"
            )
        }
    }

    private data class ErrorDetails(
        val field: String,
        val message: String,
        val code: String
    )
}

// Error response model
data class ErrorResponse(
    val message: String,
    val errorCode: String,
    val details: List<ValidationError>? = null,
    val timestamp: Instant,
    val path: String? = null
) {
    companion object {
        fun simple(message: String, errorCode: String): ErrorResponse {
            return ErrorResponse(
                message = message,
                errorCode = errorCode,
                timestamp = Instant.now()
            )
        }
    }
}
```

### Service-Level Exception Handling

```kotlin
@Service
@Transactional
class UserService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder,
    private val validationService: ValidationService
) {

    private val logger = LoggerFactory.getLogger(UserService::class.java)

    /**
     * Creates a user with comprehensive error handling.
     */
    fun createUser(request: CreateUserRequest): UserResponse {
        logger.info("Creating user with username: {}", request.username)

        try {
            // Validate request
            val validationResult = validationService.validate(request, ValidationOperation.CREATE)
            if (!validationResult.isValid) {
                throw ValidationException("User creation validation failed", validationResult.errors)
            }

            // Check for existing username
            if (userRepository.existsByUsername(request.username)) {
                throw UsernameAlreadyExistsException(request.username)
            }

            // Check for existing email
            if (userRepository.existsByEmail(request.email)) {
                throw EmailAlreadyExistsException(request.email)
            }

            // Create user entity
            val user = User.create(
                username = request.username,
                email = request.email,
                passwordHash = passwordEncoder.encode(request.password),
                firstName = request.firstName,
                lastName = request.lastName
            )

            // Add profile if provided
            request.profile?.let { profileRequest ->
                user.profile = UserProfile.create(
                    user = user,
                    birthDate = profileRequest.birthDate,
                    bio = profileRequest.bio,
                    website = profileRequest.website,
                    phoneNumber = profileRequest.phoneNumber
                )
            }

            val savedUser = userRepository.save(user)
            logger.info("User created successfully with ID: {}", savedUser.id)

            return UserResponse.from(savedUser)

        } catch (ex: DataIntegrityViolationException) {
            logger.warn("Database constraint violation during user creation", ex)

            // Convert database exceptions to application exceptions
            when {
                ex.message?.contains("uk_users_username") == true ->
                    throw UsernameAlreadyExistsException(request.username)
                ex.message?.contains("uk_users_email") == true ->
                    throw EmailAlreadyExistsException(request.email)
                else -> throw BusinessRuleException("User creation failed due to database constraint", "DATABASE_CONSTRAINT", ex)
            }

        } catch (ex: ApplicationException) {
            // Re-throw application exceptions
            throw ex

        } catch (ex: Exception) {
            logger.error("Unexpected error during user creation", ex)
            throw ApplicationException(
                message = "User creation failed due to unexpected error",
                cause = ex,
                errorCode = "USER_CREATION_FAILED",
                httpStatus = HttpStatus.INTERNAL_SERVER_ERROR
            )
        }
    }

    /**
     * Updates a user with proper error handling.
     */
    fun updateUser(userId: Long, request: UpdateUserRequest): UserResponse {
        logger.info("Updating user with ID: {}", userId)

        try {
            val existingUser = userRepository.findById(userId).orElse(null)
                ?: throw UserNotFoundException(userId)

            // Validate update request
            val validationResult = validationService.validateUserUpdate(userId, request)
            if (!validationResult.isValid) {
                throw ValidationException("User update validation failed", validationResult.errors)
            }

            // Apply updates
            request.username?.let { username ->
                if (username != existingUser.username && userRepository.existsByUsername(username)) {
                    throw UsernameAlreadyExistsException(username)
                }
                existingUser.username = username
            }

            request.email?.let { email ->
                if (email != existingUser.email) {
                    if (userRepository.existsByEmail(email)) {
                        throw EmailAlreadyExistsException(email)
                    }
                    existingUser.updateEmail(email)
                }
            }

            request.firstName?.let { existingUser.firstName = it }
            request.lastName?.let { existingUser.lastName = it }

            val savedUser = userRepository.save(existingUser)
            logger.info("User updated successfully: {}", savedUser.id)

            return UserResponse.from(savedUser)

        } catch (ex: ApplicationException) {
            throw ex
        } catch (ex: Exception) {
            logger.error("Unexpected error during user update", ex)
            throw ApplicationException(
                message = "User update failed due to unexpected error",
                cause = ex,
                errorCode = "USER_UPDATE_FAILED",
                httpStatus = HttpStatus.INTERNAL_SERVER_ERROR
            )
        }
    }

    /**
     * Deletes a user with business rule validation.
     */
    fun deleteUser(userId: Long) {
        logger.info("Deleting user with ID: {}", userId)

        try {
            val user = userRepository.findById(userId).orElse(null)
                ?: throw UserNotFoundException(userId)

            // Validate deletion is allowed
            val validationResult = validationService.validateUserDeletion(userId)
            if (!validationResult.isValid) {
                throw ValidationException("User deletion validation failed", validationResult.errors)
            }

            userRepository.delete(user)
            logger.info("User deleted successfully: {}", userId)

        } catch (ex: ApplicationException) {
            throw ex
        } catch (ex: Exception) {
            logger.error("Unexpected error during user deletion", ex)
            throw ApplicationException(
                message = "User deletion failed due to unexpected error",
                cause = ex,
                errorCode = "USER_DELETION_FAILED",
                httpStatus = HttpStatus.INTERNAL_SERVER_ERROR
            )
        }
    }
}

// Response model
data class UserResponse(
    val id: Long,
    val username: String,
    val email: String,
    val firstName: String,
    val lastName: String,
    val active: Boolean,
    val emailVerified: Boolean,
    val profile: UserProfileResponse?,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
) {
    companion object {
        fun from(user: User): UserResponse {
            return UserResponse(
                id = user.id,
                username = user.username,
                email = user.email,
                firstName = user.firstName,
                lastName = user.lastName,
                active = user.active,
                emailVerified = user.emailVerified,
                profile = user.profile?.let { UserProfileResponse.from(it) },
                createdAt = user.createdAt!!,
                updatedAt = user.updatedAt!!
            )
        }
    }
}

data class UserProfileResponse(
    val birthDate: LocalDate?,
    val bio: String?,
    val website: String?,
    val phoneNumber: String?
) {
    companion object {
        fun from(profile: UserProfile): UserProfileResponse {
            return UserProfileResponse(
                birthDate = profile.birthDate,
                bio = profile.bio,
                website = profile.website,
                phoneNumber = profile.phoneNumber
            )
        }
    }
}
```

## 10.6 Summary

In this comprehensive chapter, we've built a robust validation and exception handling framework that leverages Kotlin's language features while integrating seamlessly with Spring Boot and Hibernate Validator.

**Validation Strategies** form the foundation of data integrity in our applications. We explored three key layers:
- Input validation at the controller level using standard annotations
- Business logic validation in the service layer for domain-specific rules
- Database constraint validation as the final safety net

Each layer serves a specific purpose and together they provide comprehensive protection against invalid data entering your system.

**Hibernate Validator integration** with Kotlin shows how to effectively use Bean Validation in a type-safe manner. We covered:
- Proper configuration for Kotlin projects
- Advanced validation scenarios using groups and conditional validation
- Programmatic validation for complex business logic scenarios
- Leveraging Kotlin's language features like data classes and null safety

**Spring Boot's validation integration** provides seamless framework support through:
- Automatic validation of request bodies and parameters
- Method-level validation for service boundaries
- Customizable validation behavior and error messages
- Batch validation for bulk operations

**Custom validation** capabilities enable domain-specific validation rules:
- Creating custom annotation-based validators
- Context-aware validation using Spring beans
- Complex multi-field validation scenarios
- Property-based validation for collections and nested objects

**Exception handling** provides the critical safety net that ensures graceful failure handling:
- Hierarchical custom exception design that reflects domain concepts
- Comprehensive global exception handler covering all error scenarios
- Meaningful error responses that help API consumers understand and fix issues
- Proper logging and monitoring integration

The patterns and techniques demonstrated in this chapter create several key benefits:

1. **Type Safety**: Kotlin's null safety and type system prevent many validation errors at compile time
2. **Expressiveness**: Custom validators and exception types make business rules explicit in code
3. **Consistency**: Standardized error responses and validation patterns across the entire application
4. **Maintainability**: Clear separation between validation layers and comprehensive error handling
5. **Developer Experience**: Meaningful error messages and proper HTTP status codes

Key principles to remember when implementing validation and exception handling:

- **Validate early and often** - catch errors as close to the source as possible
- **Be specific in error messages** - help users understand exactly what went wrong
- **Use appropriate HTTP status codes** - follow REST conventions for error responses
- **Log appropriately** - log errors for debugging but don't expose sensitive information
- **Fail fast** - don't continue processing when validation fails

The validation and exception handling framework we've built provides a solid foundation for building robust, production-ready applications. In the next chapter, we'll explore Spring Boot Actuator, which builds upon these error handling concepts to provide comprehensive application monitoring and health checks.

The investment in comprehensive validation and exception handling pays dividends in application reliability, maintainability, and user experience. Well-designed validation prevents data corruption, while thoughtful exception handling ensures graceful degradation and meaningful feedback when things go wrong.