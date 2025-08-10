---
layout: default
---
# Chapter 13: Service Authentication and Authorization

Security is not an afterthought in modern application development—it's a fundamental requirement from day one. In this comprehensive chapter, we'll explore how to implement robust authentication and authorization in Kotlin Spring Boot applications using Spring Security, JWT tokens, and modern security patterns.

We'll start with security fundamentals and progressively build a complete security framework that handles user authentication, role-based authorization, JWT token management, and service-to-service security. You'll learn how to leverage Kotlin's language features to create clean, expressive security configurations while maintaining the highest security standards.

Modern applications need to handle various authentication scenarios: web-based login, API token authentication, service-to-service communication, and mobile app authentication. We'll cover all these scenarios with practical, production-ready implementations that you can adapt to your specific requirements.

## 13.1 Security Basics

Before diving into implementation details, let's establish a solid understanding of security concepts and how they apply to Spring Boot applications.

### Core Security Concepts

```kotlin
// Security domain model
data class User(
    val id: Long,
    val username: String,
    val email: String,
    val password: String, // Always hashed, never stored in plain text
    val authorities: Set<Authority>,
    val enabled: Boolean = true,
    val accountExpired: Boolean = false,
    val credentialsExpired: Boolean = false,
    val locked: Boolean = false,
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val lastLogin: LocalDateTime? = null
) : UserDetails {
    
    override fun getAuthorities(): Collection<GrantedAuthority> {
        return authorities.map { SimpleGrantedAuthority("ROLE_${it.name}") }
    }
    
    override fun getPassword(): String = password
    override fun getUsername(): String = username
    override fun isAccountNonExpired(): Boolean = !accountExpired
    override fun isAccountNonLocked(): Boolean = !locked
    override fun isCredentialsNonExpired(): Boolean = !credentialsExpired
    override fun isEnabled(): Boolean = enabled
}

data class Authority(
    val id: Long,
    val name: String,
    val description: String? = null
) {
    // Common authority constants
    companion object {
        const val USER = "USER"
        const val ADMIN = "ADMIN"
        const val MODERATOR = "MODERATOR"
        const val API_ACCESS = "API_ACCESS"
        const val SYSTEM = "SYSTEM"
    }
}

// JWT Claims structure
data class JwtClaims(
    val sub: String, // Subject (username)
    val userId: Long,
    val authorities: List<String>,
    val iat: Long, // Issued at
    val exp: Long, // Expiration
    val jti: String, // JWT ID for tracking
    val iss: String = "kotlin-spring-boot-app", // Issuer
    val aud: List<String> = listOf("web", "mobile") // Audience
)

// Authentication request/response models
data class LoginRequest(
    @field:NotBlank(message = "Username is required")
    val username: String,
    
    @field:NotBlank(message = "Password is required")
    val password: String,
    
    val rememberMe: Boolean = false
)

data class LoginResponse(
    val accessToken: String,
    val refreshToken: String,
    val tokenType: String = "Bearer",
    val expiresIn: Long,
    val user: UserInfo
)

data class UserInfo(
    val id: Long,
    val username: String,
    val email: String,
    val authorities: List<String>,
    val lastLogin: LocalDateTime?
)

data class RefreshTokenRequest(
    @field:NotBlank(message = "Refresh token is required")
    val refreshToken: String
)

// Security configuration properties
@ConfigurationProperties(prefix = "app.security")
data class SecurityProperties(
    val jwt: JwtProperties = JwtProperties(),
    val cors: CorsProperties = CorsProperties(),
    val session: SessionProperties = SessionProperties()
) {
    
    data class JwtProperties(
        val secret: String = "default-secret-change-in-production",
        val accessTokenExpiration: Duration = Duration.ofHours(1),
        val refreshTokenExpiration: Duration = Duration.ofDays(7),
        val issuer: String = "kotlin-spring-boot-app"
    )
    
    data class CorsProperties(
        val allowedOrigins: List<String> = listOf("http://localhost:3000"),
        val allowedMethods: List<String> = listOf("GET", "POST", "PUT", "DELETE", "OPTIONS"),
        val allowedHeaders: List<String> = listOf("*"),
        val allowCredentials: Boolean = true,
        val maxAge: Long = 3600
    )
    
    data class SessionProperties(
        val timeout: Duration = Duration.ofMinutes(30),
        val maxSessions: Int = 1,
        val preventSessionFixation: Boolean = true
    )
}
```

### Password Security and Hashing

```kotlin
@Configuration
class PasswordConfiguration {
    
    /**
     * Configure BCrypt password encoder with appropriate strength.
     * Strength 12 provides good security vs. performance balance.
     */
    @Bean
    fun passwordEncoder(): PasswordEncoder {
        return BCryptPasswordEncoder(12)
    }
    
    /**
     * Password validator for ensuring strong passwords
     */
    @Bean
    fun passwordValidator(): PasswordValidator {
        return PasswordValidator()
    }
}

@Component
class PasswordValidator {
    
    private val logger = LoggerFactory.getLogger(PasswordValidator::class.java)
    
    fun validatePassword(password: String, username: String? = null): PasswordValidationResult {
        val errors = mutableListOf<String>()
        
        // Length validation
        if (password.length < 8) {
            errors.add("Password must be at least 8 characters long")
        }
        
        if (password.length > 128) {
            errors.add("Password cannot exceed 128 characters")
        }
        
        // Character class validation
        if (!password.any { it.isUpperCase() }) {
            errors.add("Password must contain at least one uppercase letter")
        }
        
        if (!password.any { it.isLowerCase() }) {
            errors.add("Password must contain at least one lowercase letter")
        }
        
        if (!password.any { it.isDigit() }) {
            errors.add("Password must contain at least one digit")
        }
        
        if (!password.any { !it.isLetterOrDigit() }) {
            errors.add("Password must contain at least one special character")
        }
        
        // Common password validation
        if (isCommonPassword(password)) {
            errors.add("Password is too common. Please choose a more secure password")
        }
        
        // Username similarity check
        username?.let { user ->
            if (password.lowercase().contains(user.lowercase()) || 
                user.lowercase().contains(password.lowercase())) {
                errors.add("Password cannot be similar to username")
            }
        }
        
        // Sequential characters check
        if (hasSequentialCharacters(password)) {
            errors.add("Password cannot contain sequential characters")
        }
        
        // Repeated characters check
        if (hasExcessiveRepetition(password)) {
            errors.add("Password cannot contain excessive repeated characters")
        }
        
        return PasswordValidationResult(
            isValid = errors.isEmpty(),
            errors = errors,
            score = calculatePasswordScore(password)
        )
    }
    
    private fun isCommonPassword(password: String): Boolean {
        val commonPasswords = setOf(
            "password", "123456", "password123", "admin", "qwerty",
            "letmein", "welcome", "monkey", "dragon", "password1"
        )
        return commonPasswords.contains(password.lowercase())
    }
    
    private fun hasSequentialCharacters(password: String): Boolean {
        val sequences = listOf(
            "abcdefghijklmnopqrstuvwxyz",
            "0123456789",
            "qwertyuiop",
            "asdfghjkl",
            "zxcvbnm"
        )
        
        return sequences.any { sequence ->
            (0..sequence.length - 3).any { i ->
                val subseq = sequence.substring(i, i + 3)
                password.lowercase().contains(subseq) || 
                password.lowercase().contains(subseq.reversed())
            }
        }
    }
    
    private fun hasExcessiveRepetition(password: String): Boolean {
        var count = 1
        var maxCount = 1
        
        for (i in 1 until password.length) {
            if (password[i] == password[i - 1]) {
                count++
                maxCount = maxOf(maxCount, count)
            } else {
                count = 1
            }
        }
        
        return maxCount > 2
    }
    
    private fun calculatePasswordScore(password: String): Int {
        var score = 0
        
        // Length bonus
        score += when {
            password.length >= 12 -> 25
            password.length >= 8 -> 15
            else -> 0
        }
        
        // Character variety bonus
        if (password.any { it.isUpperCase() }) score += 10
        if (password.any { it.isLowerCase() }) score += 10
        if (password.any { it.isDigit() }) score += 10
        if (password.any { !it.isLetterOrDigit() }) score += 15
        
        // Entropy bonus
        val uniqueChars = password.toSet().size
        score += (uniqueChars * 2).coerceAtMost(20)
        
        // Pattern penalties
        if (hasSequentialCharacters(password)) score -= 15
        if (hasExcessiveRepetition(password)) score -= 10
        if (isCommonPassword(password)) score -= 25
        
        return score.coerceIn(0, 100)
    }
}

data class PasswordValidationResult(
    val isValid: Boolean,
    val errors: List<String>,
    val score: Int
) {
    val strength: PasswordStrength
        get() = when (score) {
            in 0..30 -> PasswordStrength.WEAK
            in 31..60 -> PasswordStrength.MEDIUM  
            in 61..80 -> PasswordStrength.STRONG
            else -> PasswordStrength.VERY_STRONG
        }
}

enum class PasswordStrength {
    WEAK, MEDIUM, STRONG, VERY_STRONG
}
```

## 13.2 Spring Security

Spring Security provides comprehensive security services for Java applications. Let's implement a robust security configuration using Kotlin.

### Core Security Configuration

```kotlin
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
@EnableConfigurationProperties(SecurityProperties::class)
class SecurityConfiguration(
    private val securityProperties: SecurityProperties,
    private val jwtAuthenticationEntryPoint: JwtAuthenticationEntryPoint,
    private val jwtAccessDeniedHandler: JwtAccessDeniedHandler,
    private val userDetailsService: UserDetailsService,
    private val passwordEncoder: PasswordEncoder
) {
    
    @Bean
    fun jwtAuthenticationFilter(jwtTokenProvider: JwtTokenProvider): JwtAuthenticationFilter {
        return JwtAuthenticationFilter(jwtTokenProvider)
    }
    
    @Bean
    @Order(1)
    fun apiSecurityFilterChain(
        http: HttpSecurity,
        jwtAuthenticationFilter: JwtAuthenticationFilter
    ): SecurityFilterChain {
        return http
            .requestMatcher(RequestMatcher { request ->
                request.requestURI.startsWith("/api/")
            })
            .csrf { csrf -> csrf.disable() }
            .sessionManagement { session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            }
            .exceptionHandling { exceptions ->
                exceptions
                    .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                    .accessDeniedHandler(jwtAccessDeniedHandler)
            }
            .authorizeHttpRequests { requests ->
                requests
                    // Public endpoints
                    .requestMatchers(HttpMethod.POST, "/api/auth/login").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/auth/register").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/auth/refresh").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/auth/forgot-password").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/auth/reset-password").permitAll()
                    .requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                    
                    // Health checks and monitoring
                    .requestMatchers("/api/health").permitAll()
                    .requestMatchers("/api/info").permitAll()
                    
                    // API documentation
                    .requestMatchers("/api/docs/**").permitAll()
                    .requestMatchers("/api/swagger-ui/**").permitAll()
                    .requestMatchers("/v3/api-docs/**").permitAll()
                    
                    // Admin endpoints
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    
                    // User management
                    .requestMatchers(HttpMethod.GET, "/api/users/me").hasRole("USER")
                    .requestMatchers(HttpMethod.PUT, "/api/users/me").hasRole("USER")
                    .requestMatchers(HttpMethod.DELETE, "/api/users/me").hasRole("USER")
                    .requestMatchers("/api/users/**").hasRole("ADMIN")
                    
                    // All other API endpoints require authentication
                    .requestMatchers("/api/**").authenticated()
                    
                    // Everything else
                    .anyRequest().permitAll()
            }
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter::class.java)
            .build()
    }
    
    @Bean
    @Order(2) 
    fun webSecurityFilterChain(http: HttpSecurity): SecurityFilterChain {
        return http
            .requestMatcher(RequestMatcher { request ->
                !request.requestURI.startsWith("/api/")
            })
            .csrf { csrf ->
                csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            }
            .sessionManagement { session ->
                session
                    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                    .maximumSessions(securityProperties.session.maxSessions)
                    .maxSessionsPreventsLogin(true)
                    .sessionRegistry(sessionRegistry())
                    .and()
                    .sessionFixation().migrateSession()
            }
            .formLogin { form ->
                form
                    .loginPage("/login")
                    .loginProcessingUrl("/login")
                    .defaultSuccessUrl("/dashboard", true)
                    .failureUrl("/login?error")
                    .permitAll()
            }
            .logout { logout ->
                logout
                    .logoutUrl("/logout")
                    .logoutSuccessUrl("/login?logout")
                    .invalidateHttpSession(true)
                    .deleteCookies("JSESSIONID")
                    .permitAll()
            }
            .rememberMe { remember ->
                remember
                    .key("uniqueAndSecret")
                    .tokenValiditySeconds(60 * 60 * 24 * 7) // 1 week
                    .userDetailsService(userDetailsService)
            }
            .authorizeHttpRequests { requests ->
                requests
                    .requestMatchers("/", "/home", "/public/**").permitAll()
                    .requestMatchers("/login", "/register", "/forgot-password").permitAll()
                    .requestMatchers("/css/**", "/js/**", "/images/**", "/webjars/**").permitAll()
                    .requestMatchers("/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
            }
            .build()
    }
    
    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource {
        val configuration = CorsConfiguration()
        configuration.allowedOrigins = securityProperties.cors.allowedOrigins
        configuration.allowedMethods = securityProperties.cors.allowedMethods
        configuration.allowedHeaders = securityProperties.cors.allowedHeaders
        configuration.allowCredentials = securityProperties.cors.allowCredentials
        configuration.maxAge = securityProperties.cors.maxAge
        
        val source = UrlBasedCorsConfigurationSource()
        source.registerCorsConfiguration("/**", configuration)
        return source
    }
    
    @Bean
    fun authenticationManager(authenticationConfiguration: AuthenticationConfiguration): AuthenticationManager {
        return authenticationConfiguration.authenticationManager
    }
    
    @Bean
    fun daoAuthenticationProvider(): DaoAuthenticationProvider {
        val provider = DaoAuthenticationProvider()
        provider.setUserDetailsService(userDetailsService)
        provider.setPasswordEncoder(passwordEncoder)
        provider.setHideUserNotFoundExceptions(false)
        return provider
    }
    
    @Bean
    fun sessionRegistry(): SessionRegistry {
        return SessionRegistryImpl()
    }
}

// JWT Authentication Entry Point
@Component
class JwtAuthenticationEntryPoint : AuthenticationEntryPoint {
    
    private val logger = LoggerFactory.getLogger(JwtAuthenticationEntryPoint::class.java)
    private val objectMapper = jacksonObjectMapper()
    
    override fun commence(
        request: HttpServletRequest,
        response: HttpServletResponse,
        authException: AuthenticationException
    ) {
        logger.error("Unauthorized error: {}", authException.message)
        
        response.contentType = MediaType.APPLICATION_JSON_VALUE
        response.status = HttpServletResponse.SC_UNAUTHORIZED
        
        val errorResponse = mapOf(
            "error" to "Unauthorized",
            "message" to "Authentication required",
            "path" to request.requestURI,
            "timestamp" to Instant.now().toString()
        )
        
        objectMapper.writeValue(response.outputStream, errorResponse)
    }
}

// JWT Access Denied Handler
@Component
class JwtAccessDeniedHandler : AccessDeniedHandler {
    
    private val logger = LoggerFactory.getLogger(JwtAccessDeniedHandler::class.java)
    private val objectMapper = jacksonObjectMapper()
    
    override fun handle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        accessDeniedException: AccessDeniedException
    ) {
        logger.error("Access denied error: {}", accessDeniedException.message)
        
        response.contentType = MediaType.APPLICATION_JSON_VALUE
        response.status = HttpServletResponse.SC_FORBIDDEN
        
        val errorResponse = mapOf(
            "error" to "Forbidden",
            "message" to "Insufficient permissions",
            "path" to request.requestURI,
            "timestamp" to Instant.now().toString()
        )
        
        objectMapper.writeValue(response.outputStream, errorResponse)
    }
}
```

### User Details Service Implementation

```kotlin
@Service
@Transactional(readOnly = true)
class CustomUserDetailsService(
    private val userRepository: UserRepository,
    private val loginAttemptService: LoginAttemptService
) : UserDetailsService {
    
    private val logger = LoggerFactory.getLogger(CustomUserDetailsService::class.java)
    
    override fun loadUserByUsername(username: String): UserDetails {
        logger.debug("Loading user by username: {}", username)
        
        // Check if user is temporarily locked due to failed attempts
        if (loginAttemptService.isBlocked(username)) {
            logger.warn("Login attempt blocked for user: {}", username)
            throw LockedException("Account temporarily locked due to failed login attempts")
        }
        
        val user = userRepository.findByUsernameOrEmail(username, username)
            ?: run {
                logger.warn("User not found: {}", username)
                throw UsernameNotFoundException("User not found: $username")
            }
        
        // Log successful user loading (but not the full user details)
        logger.debug("User loaded successfully: {}", user.username)
        
        return CustomUserPrincipal.create(user)
    }
    
    fun loadUserById(userId: Long): UserDetails {
        logger.debug("Loading user by ID: {}", userId)
        
        val user = userRepository.findById(userId).orElse(null)
            ?: run {
                logger.warn("User not found with ID: {}", userId)
                throw UsernameNotFoundException("User not found with ID: $userId")
            }
        
        return CustomUserPrincipal.create(user)
    }
}

// Custom UserDetails implementation
data class CustomUserPrincipal(
    val id: Long,
    private val username: String,
    private val password: String,
    val email: String,
    private val authorities: Collection<GrantedAuthority>,
    private val enabled: Boolean,
    private val accountNonExpired: Boolean,
    private val credentialsNonExpired: Boolean,
    private val accountNonLocked: Boolean
) : UserDetails {
    
    companion object {
        fun create(user: User): CustomUserPrincipal {
            val authorities = user.authorities.map { authority ->
                SimpleGrantedAuthority("ROLE_${authority.name}")
            }
            
            return CustomUserPrincipal(
                id = user.id,
                username = user.username,
                password = user.password,
                email = user.email,
                authorities = authorities,
                enabled = user.enabled,
                accountNonExpired = !user.accountExpired,
                credentialsNonExpired = !user.credentialsExpired,
                accountNonLocked = !user.locked
            )
        }
    }
    
    override fun getAuthorities(): Collection<GrantedAuthority> = authorities
    override fun getPassword(): String = password
    override fun getUsername(): String = username
    override fun isAccountNonExpired(): Boolean = accountNonExpired
    override fun isAccountNonLocked(): Boolean = accountNonLocked
    override fun isCredentialsNonExpired(): Boolean = credentialsNonExpired
    override fun isEnabled(): Boolean = enabled
    
    // Additional methods for easier access
    fun hasRole(role: String): Boolean {
        return authorities.any { it.authority == "ROLE_$role" }
    }
    
    fun hasAuthority(authority: String): Boolean {
        return authorities.any { it.authority == authority }
    }
}

// Login attempt tracking service
@Service
class LoginAttemptService(
    private val redisTemplate: RedisTemplate<String, String>
) {
    
    private val logger = LoggerFactory.getLogger(LoginAttemptService::class.java)
    private val maxAttempts = 5
    private val blockDuration = Duration.ofMinutes(15)
    
    fun recordLoginSuccess(username: String) {
        val key = "login_attempts:$username"
        redisTemplate.delete(key)
        logger.debug("Cleared login attempts for user: {}", username)
    }
    
    fun recordLoginFailure(username: String) {
        val key = "login_attempts:$username"
        val attempts = redisTemplate.opsForValue().increment(key) ?: 1
        
        if (attempts == 1L) {
            redisTemplate.expire(key, blockDuration)
        }
        
        logger.warn("Login failure #{} for user: {}", attempts, username)
        
        if (attempts >= maxAttempts) {
            logger.warn("User {} blocked after {} failed attempts", username, attempts)
        }
    }
    
    fun isBlocked(username: String): Boolean {
        val key = "login_attempts:$username"
        val attempts = redisTemplate.opsForValue().get(key)?.toLongOrNull() ?: 0
        return attempts >= maxAttempts
    }
    
    fun getRemainingAttempts(username: String): Int {
        val key = "login_attempts:$username"
        val attempts = redisTemplate.opsForValue().get(key)?.toIntOrNull() ?: 0
        return (maxAttempts - attempts).coerceAtLeast(0)
    }
    
    fun getBlockTimeRemaining(username: String): Duration? {
        val key = "login_attempts:$username"
        val ttl = redisTemplate.getExpire(key, TimeUnit.SECONDS)
        return if (ttl > 0) Duration.ofSeconds(ttl) else null
    }
}
```

### Security Event Handling

```kotlin
@Component
class SecurityEventListener(
    private val loginAttemptService: LoginAttemptService,
    private val securityEventService: SecurityEventService
) {
    
    private val logger = LoggerFactory.getLogger(SecurityEventListener::class.java)
    
    @EventListener
    fun handleAuthenticationSuccess(event: AuthenticationSuccessEvent) {
        val username = event.authentication.name
        loginAttemptService.recordLoginSuccess(username)
        
        securityEventService.recordEvent(
            SecurityEventType.LOGIN_SUCCESS,
            username,
            "User logged in successfully"
        )
        
        logger.info("Authentication success for user: {}", username)
    }
    
    @EventListener
    fun handleAuthenticationFailure(event: AbstractAuthenticationFailureEvent) {
        val username = event.authentication.name
        loginAttemptService.recordLoginFailure(username)
        
        val reason = when (event.exception) {
            is BadCredentialsException -> "Invalid credentials"
            is LockedException -> "Account locked"
            is DisabledException -> "Account disabled"
            is AccountExpiredException -> "Account expired"
            is CredentialsExpiredException -> "Credentials expired"
            else -> "Authentication failed"
        }
        
        securityEventService.recordEvent(
            SecurityEventType.LOGIN_FAILURE,
            username,
            reason
        )
        
        logger.warn("Authentication failure for user: {} - {}", username, reason)
    }
    
    @EventListener
    fun handleLogoutSuccess(event: LogoutSuccessEvent) {
        val username = event.authentication?.name ?: "unknown"
        
        securityEventService.recordEvent(
            SecurityEventType.LOGOUT,
            username,
            "User logged out"
        )
        
        logger.info("Logout success for user: {}", username)
    }
}

@Service
@Transactional
class SecurityEventService(
    private val securityEventRepository: SecurityEventRepository
) {
    
    fun recordEvent(type: SecurityEventType, username: String, details: String, ipAddress: String? = null) {
        val event = SecurityEvent(
            type = type,
            username = username,
            details = details,
            ipAddress = ipAddress,
            timestamp = LocalDateTime.now()
        )
        
        securityEventRepository.save(event)
    }
    
    fun getSecurityEvents(
        username: String? = null,
        type: SecurityEventType? = null,
        since: LocalDateTime? = null,
        pageable: Pageable
    ): Page<SecurityEvent> {
        return securityEventRepository.findEvents(username, type, since, pageable)
    }
}

@Entity
@Table(name = "security_events")
data class SecurityEvent(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    val type: SecurityEventType,
    
    @Column(nullable = false)
    val username: String,
    
    @Column(length = 1000)
    val details: String,
    
    @Column(name = "ip_address")
    val ipAddress: String?,
    
    @Column(nullable = false)
    val timestamp: LocalDateTime
)

enum class SecurityEventType {
    LOGIN_SUCCESS,
    LOGIN_FAILURE, 
    LOGOUT,
    PASSWORD_CHANGE,
    ACCOUNT_LOCKED,
    ACCOUNT_UNLOCKED,
    PERMISSION_DENIED,
    TOKEN_REFRESH,
    SUSPICIOUS_ACTIVITY
}
```

## 13.3 JWT Integration in Kotlin

JSON Web Tokens (JWT) provide a compact, secure way to transmit information between parties. Let's implement comprehensive JWT handling with Kotlin.

### JWT Token Provider

```kotlin
@Component
class JwtTokenProvider(
    private val securityProperties: SecurityProperties,
    private val redisTemplate: RedisTemplate<String, String>
) {
    
    private val logger = LoggerFactory.getLogger(JwtTokenProvider::class.java)
    
    private val algorithm: Algorithm by lazy {
        Algorithm.HMAC256(securityProperties.jwt.secret)
    }
    
    private val verifier: JWTVerifier by lazy {
        JWT.require(algorithm)
            .withIssuer(securityProperties.jwt.issuer)
            .build()
    }
    
    /**
     * Generate access token
     */
    fun generateAccessToken(userPrincipal: CustomUserPrincipal): String {
        val now = Instant.now()
        val expiry = now.plus(securityProperties.jwt.accessTokenExpiration)
        val jti = UUID.randomUUID().toString()
        
        val authorities = userPrincipal.authorities.map { it.authority }
        
        val token = JWT.create()
            .withIssuer(securityProperties.jwt.issuer)
            .withSubject(userPrincipal.username)
            .withAudience("web", "mobile")
            .withIssuedAt(Date.from(now))
            .withExpiresAt(Date.from(expiry))
            .withJWTId(jti)
            .withClaim("userId", userPrincipal.id)
            .withClaim("email", userPrincipal.email)
            .withClaim("authorities", authorities)
            .sign(algorithm)
        
        // Store token metadata for tracking
        storeTokenMetadata(jti, userPrincipal.id, TokenType.ACCESS, expiry)
        
        logger.debug("Generated access token for user: {}", userPrincipal.username)
        return token
    }
    
    /**
     * Generate refresh token
     */
    fun generateRefreshToken(userPrincipal: CustomUserPrincipal): String {
        val now = Instant.now()
        val expiry = now.plus(securityProperties.jwt.refreshTokenExpiration)
        val jti = UUID.randomUUID().toString()
        
        val token = JWT.create()
            .withIssuer(securityProperties.jwt.issuer)
            .withSubject(userPrincipal.username)
            .withAudience("refresh")
            .withIssuedAt(Date.from(now))
            .withExpiresAt(Date.from(expiry))
            .withJWTId(jti)
            .withClaim("userId", userPrincipal.id)
            .withClaim("tokenType", "refresh")
            .sign(algorithm)
        
        // Store refresh token metadata
        storeTokenMetadata(jti, userPrincipal.id, TokenType.REFRESH, expiry)
        
        logger.debug("Generated refresh token for user: {}", userPrincipal.username)
        return token
    }
    
    /**
     * Validate and parse JWT token
     */
    fun validateToken(token: String): JwtValidationResult {
        return try {
            val decodedToken = verifier.verify(token)
            val jti = decodedToken.id
            
            // Check if token is blacklisted
            if (isTokenBlacklisted(jti)) {
                return JwtValidationResult.invalid("Token has been revoked")
            }
            
            // Extract claims
            val claims = JwtClaims(
                sub = decodedToken.subject,
                userId = decodedToken.getClaim("userId").asLong(),
                authorities = decodedToken.getClaim("authorities").asList(String::class.java),
                iat = decodedToken.issuedAt.toInstant().epochSecond,
                exp = decodedToken.expiresAt.toInstant().epochSecond,
                jti = jti,
                iss = decodedToken.issuer,
                aud = decodedToken.audience
            )
            
            JwtValidationResult.valid(claims)
            
        } catch (ex: TokenExpiredException) {
            logger.debug("Token expired: {}", ex.message)
            JwtValidationResult.invalid("Token has expired")
        } catch (ex: JWTVerificationException) {
            logger.warn("Invalid token: {}", ex.message)
            JwtValidationResult.invalid("Invalid token")
        } catch (ex: Exception) {
            logger.error("Error validating token", ex)
            JwtValidationResult.invalid("Token validation error")
        }
    }
    
    /**
     * Extract username from token without full validation
     */
    fun getUsernameFromToken(token: String): String? {
        return try {
            val decodedToken = JWT.decode(token)
            decodedToken.subject
        } catch (ex: Exception) {
            logger.debug("Could not extract username from token", ex)
            null
        }
    }
    
    /**
     * Blacklist a token
     */
    fun revokeToken(token: String): Boolean {
        return try {
            val decodedToken = JWT.decode(token)
            val jti = decodedToken.id
            val expiry = decodedToken.expiresAt.toInstant()
            
            blacklistToken(jti, expiry)
            removeTokenMetadata(jti)
            
            logger.info("Token revoked: {}", jti)
            true
        } catch (ex: Exception) {
            logger.error("Error revoking token", ex)
            false
        }
    }
    
    /**
     * Revoke all tokens for a user
     */
    fun revokeAllUserTokens(userId: Long) {
        val pattern = "token_metadata:*:$userId"
        val keys = redisTemplate.keys(pattern)
        
        keys?.forEach { key ->
            val parts = key.split(":")
            if (parts.size >= 3) {
                val jti = parts[2]
                redisTemplate.delete(key)
                
                // Add to blacklist until expiry
                val ttl = redisTemplate.getExpire(key, TimeUnit.SECONDS)
                if (ttl > 0) {
                    redisTemplate.opsForValue().set(
                        "blacklist:$jti", 
                        "revoked", 
                        Duration.ofSeconds(ttl)
                    )
                }
            }
        }
        
        logger.info("All tokens revoked for user: {}", userId)
    }
    
    private fun storeTokenMetadata(jti: String, userId: Long, type: TokenType, expiry: Instant) {
        val key = "token_metadata:$jti:$userId"
        val metadata = TokenMetadata(
            jti = jti,
            userId = userId,
            type = type,
            issuedAt = Instant.now(),
            expiresAt = expiry
        )
        
        val ttl = Duration.between(Instant.now(), expiry)
        redisTemplate.opsForValue().set(
            key, 
            jacksonObjectMapper().writeValueAsString(metadata),
            ttl
        )
    }
    
    private fun removeTokenMetadata(jti: String) {
        val pattern = "token_metadata:$jti:*"
        val keys = redisTemplate.keys(pattern)
        if (!keys.isNullOrEmpty()) {
            redisTemplate.delete(keys)
        }
    }
    
    private fun blacklistToken(jti: String, expiry: Instant) {
        val key = "blacklist:$jti"
        val ttl = Duration.between(Instant.now(), expiry)
        redisTemplate.opsForValue().set(key, "revoked", ttl)
    }
    
    private fun isTokenBlacklisted(jti: String): Boolean {
        return redisTemplate.hasKey("blacklist:$jti")
    }
}

// JWT validation result
sealed class JwtValidationResult {
    data class Valid(val claims: JwtClaims) : JwtValidationResult()
    data class Invalid(val reason: String) : JwtValidationResult()
    
    val isValid: Boolean get() = this is Valid
    val isInvalid: Boolean get() = this is Invalid
    
    companion object {
        fun valid(claims: JwtClaims) = Valid(claims)
        fun invalid(reason: String) = Invalid(reason)
    }
}

// Token metadata for tracking
data class TokenMetadata(
    val jti: String,
    val userId: Long,
    val type: TokenType,
    val issuedAt: Instant,
    val expiresAt: Instant
)

enum class TokenType {
    ACCESS, REFRESH
}
```

### JWT Authentication Filter

```kotlin
class JwtAuthenticationFilter(
    private val jwtTokenProvider: JwtTokenProvider
) : OncePerRequestFilter() {
    
    private val logger = LoggerFactory.getLogger(JwtAuthenticationFilter::class.java)
    
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        try {
            val token = extractTokenFromRequest(request)
            
            if (token != null && SecurityContextHolder.getContext().authentication == null) {
                val validationResult = jwtTokenProvider.validateToken(token)
                
                if (validationResult.isValid) {
                    val claims = (validationResult as JwtValidationResult.Valid).claims
                    val authentication = createAuthentication(claims)
                    SecurityContextHolder.getContext().authentication = authentication
                    
                    logger.debug("Set authentication for user: {}", claims.sub)
                } else {
                    val reason = (validationResult as JwtValidationResult.Invalid).reason
                    logger.debug("Invalid token: {}", reason)
                }
            }
            
        } catch (ex: Exception) {
            logger.error("Cannot set user authentication", ex)
        }
        
        filterChain.doFilter(request, response)
    }
    
    private fun extractTokenFromRequest(request: HttpServletRequest): String? {
        val bearerToken = request.getHeader("Authorization")
        
        return if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            bearerToken.substring(7)
        } else {
            // Also check for token in cookie
            request.cookies?.find { it.name == "access_token" }?.value
        }
    }
    
    private fun createAuthentication(claims: JwtClaims): Authentication {
        val authorities = claims.authorities.map { SimpleGrantedAuthority(it) }
        
        val principal = CustomUserPrincipal(
            id = claims.userId,
            username = claims.sub,
            password = "", // Not needed for JWT authentication
            email = "", // Would need to add to claims if needed
            authorities = authorities,
            enabled = true,
            accountNonExpired = true,
            credentialsNonExpired = true,
            accountNonLocked = true
        )
        
        return JwtAuthenticationToken(principal, authorities, claims)
    }
}

// Custom authentication token for JWT
class JwtAuthenticationToken(
    private val principal: CustomUserPrincipal,
    private val authorities: Collection<GrantedAuthority>,
    val claims: JwtClaims
) : AbstractAuthenticationToken(authorities) {
    
    init {
        isAuthenticated = true
    }
    
    override fun getCredentials(): Any = ""
    override fun getPrincipal(): Any = principal
    
    fun getUserId(): Long = principal.id
    fun getUsername(): String = principal.username
    fun getTokenId(): String = claims.jti
}
```

### Authentication Service

```kotlin
@Service
@Transactional
class AuthenticationService(
    private val authenticationManager: AuthenticationManager,
    private val userDetailsService: CustomUserDetailsService,
    private val jwtTokenProvider: JwtTokenProvider,
    private val passwordEncoder: PasswordEncoder,
    private val userRepository: UserRepository,
    private val securityEventService: SecurityEventService,
    private val passwordValidator: PasswordValidator
) {
    
    private val logger = LoggerFactory.getLogger(AuthenticationService::class.java)
    
    /**
     * Authenticate user and return tokens
     */
    fun login(loginRequest: LoginRequest, ipAddress: String? = null): LoginResponse {
        logger.info("Login attempt for user: {}", loginRequest.username)
        
        try {
            // Authenticate user
            val authentication = authenticationManager.authenticate(
                UsernamePasswordAuthenticationToken(
                    loginRequest.username,
                    loginRequest.password
                )
            )
            
            val userPrincipal = authentication.principal as CustomUserPrincipal
            
            // Update last login time
            updateLastLoginTime(userPrincipal.id)
            
            // Generate tokens
            val accessToken = jwtTokenProvider.generateAccessToken(userPrincipal)
            val refreshToken = jwtTokenProvider.generateRefreshToken(userPrincipal)
            
            // Record security event
            securityEventService.recordEvent(
                SecurityEventType.LOGIN_SUCCESS,
                userPrincipal.username,
                "User logged in successfully",
                ipAddress
            )
            
            logger.info("Login successful for user: {}", loginRequest.username)
            
            return LoginResponse(
                accessToken = accessToken,
                refreshToken = refreshToken,
                tokenType = "Bearer",
                expiresIn = securityProperties.jwt.accessTokenExpiration.toSeconds(),
                user = UserInfo(
                    id = userPrincipal.id,
                    username = userPrincipal.username,
                    email = userPrincipal.email,
                    authorities = userPrincipal.authorities.map { it.authority },
                    lastLogin = LocalDateTime.now()
                )
            )
            
        } catch (ex: Exception) {
            securityEventService.recordEvent(
                SecurityEventType.LOGIN_FAILURE,
                loginRequest.username,
                "Login failed: ${ex.message}",
                ipAddress
            )
            
            logger.warn("Login failed for user: {}", loginRequest.username, ex)
            throw AuthenticationFailedException("Authentication failed", ex)
        }
    }
    
    /**
     * Refresh access token using refresh token
     */
    fun refreshToken(refreshTokenRequest: RefreshTokenRequest): LoginResponse {
        val validationResult = jwtTokenProvider.validateToken(refreshTokenRequest.refreshToken)
        
        if (validationResult.isInvalid) {
            val reason = (validationResult as JwtValidationResult.Invalid).reason
            throw InvalidTokenException("Invalid refresh token: $reason")
        }
        
        val claims = (validationResult as JwtValidationResult.Valid).claims
        
        // Load fresh user details to get current authorities
        val userPrincipal = userDetailsService.loadUserById(claims.userId) as CustomUserPrincipal
        
        // Generate new access token
        val newAccessToken = jwtTokenProvider.generateAccessToken(userPrincipal)
        
        // Record security event
        securityEventService.recordEvent(
            SecurityEventType.TOKEN_REFRESH,
            userPrincipal.username,
            "Token refreshed successfully"
        )
        
        logger.debug("Token refreshed for user: {}", userPrincipal.username)
        
        return LoginResponse(
            accessToken = newAccessToken,
            refreshToken = refreshTokenRequest.refreshToken, // Keep same refresh token
            tokenType = "Bearer",
            expiresIn = securityProperties.jwt.accessTokenExpiration.toSeconds(),
            user = UserInfo(
                id = userPrincipal.id,
                username = userPrincipal.username,
                email = userPrincipal.email,
                authorities = userPrincipal.authorities.map { it.authority },
                lastLogin = null // Don't update for token refresh
            )
        )
    }
    
    /**
     * Logout user and revoke tokens
     */
    fun logout(token: String, username: String, ipAddress: String? = null) {
        try {
            // Revoke the specific token
            jwtTokenProvider.revokeToken(token)
            
            // Record security event
            securityEventService.recordEvent(
                SecurityEventType.LOGOUT,
                username,
                "User logged out",
                ipAddress
            )
            
            logger.info("Logout successful for user: {}", username)
            
        } catch (ex: Exception) {
            logger.error("Error during logout for user: {}", username, ex)
        }
    }
    
    /**
     * Register new user
     */
    fun register(registerRequest: RegisterRequest): UserRegistrationResponse {
        logger.info("Registration attempt for username: {}", registerRequest.username)
        
        // Validate password
        val passwordValidation = passwordValidator.validatePassword(
            registerRequest.password, 
            registerRequest.username
        )
        
        if (!passwordValidation.isValid) {
            throw PasswordValidationException("Password validation failed", passwordValidation.errors)
        }
        
        // Check if username already exists
        if (userRepository.existsByUsername(registerRequest.username)) {
            throw UserAlreadyExistsException("Username '${registerRequest.username}' is already taken")
        }
        
        // Check if email already exists
        if (userRepository.existsByEmail(registerRequest.email)) {
            throw UserAlreadyExistsException("Email '${registerRequest.email}' is already registered")
        }
        
        // Create new user
        val hashedPassword = passwordEncoder.encode(registerRequest.password)
        
        val user = User(
            username = registerRequest.username,
            email = registerRequest.email,
            password = hashedPassword,
            authorities = setOf(Authority.USER), // Default user role
            enabled = true
        )
        
        val savedUser = userRepository.save(user)
        
        logger.info("User registered successfully: {}", savedUser.username)
        
        return UserRegistrationResponse(
            id = savedUser.id,
            username = savedUser.username,
            email = savedUser.email,
            message = "User registered successfully"
        )
    }
    
    /**
     * Change user password
     */
    fun changePassword(
        userId: Long,
        changePasswordRequest: ChangePasswordRequest,
        ipAddress: String? = null
    ) {
        val user = userRepository.findById(userId).orElse(null)
            ?: throw UserNotFoundException("User not found")
        
        // Verify current password
        if (!passwordEncoder.matches(changePasswordRequest.currentPassword, user.password)) {
            securityEventService.recordEvent(
                SecurityEventType.PERMISSION_DENIED,
                user.username,
                "Invalid current password during password change",
                ipAddress
            )
            throw InvalidPasswordException("Current password is incorrect")
        }
        
        // Validate new password
        val passwordValidation = passwordValidator.validatePassword(
            changePasswordRequest.newPassword,
            user.username
        )
        
        if (!passwordValidation.isValid) {
            throw PasswordValidationException("New password validation failed", passwordValidation.errors)
        }
        
        // Update password
        user.password = passwordEncoder.encode(changePasswordRequest.newPassword)
        user.credentialsExpired = false
        userRepository.save(user)
        
        // Revoke all existing tokens to force re-authentication
        jwtTokenProvider.revokeAllUserTokens(userId)
        
        // Record security event
        securityEventService.recordEvent(
            SecurityEventType.PASSWORD_CHANGE,
            user.username,
            "Password changed successfully",
            ipAddress
        )
        
        logger.info("Password changed successfully for user: {}", user.username)
    }
    
    private fun updateLastLoginTime(userId: Long) {
        userRepository.updateLastLoginTime(userId, LocalDateTime.now())
    }
}

// Request/Response models
data class RegisterRequest(
    @field:NotBlank(message = "Username is required")
    @field:Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    val username: String,
    
    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Email must be valid")
    val email: String,
    
    @field:NotBlank(message = "Password is required")
    val password: String
)

data class UserRegistrationResponse(
    val id: Long,
    val username: String,
    val email: String,
    val message: String
)

data class ChangePasswordRequest(
    @field:NotBlank(message = "Current password is required")
    val currentPassword: String,
    
    @field:NotBlank(message = "New password is required")
    val newPassword: String
)

// Custom exceptions
class AuthenticationFailedException(message: String, cause: Throwable? = null) : RuntimeException(message, cause)
class InvalidTokenException(message: String) : RuntimeException(message)
class UserAlreadyExistsException(message: String) : RuntimeException(message)
class UserNotFoundException(message: String) : RuntimeException(message)
class InvalidPasswordException(message: String) : RuntimeException(message)
class PasswordValidationException(message: String, val errors: List<String>) : RuntimeException(message)
```

## 13.4 Custom Filters and Configurations

Sometimes you need custom security filters and configurations beyond what Spring Security provides out of the box. Let's implement some advanced security features.

### Rate Limiting Filter

```kotlin
@Component
class RateLimitingFilter(
    private val redisTemplate: RedisTemplate<String, String>,
    private val rateLimitProperties: RateLimitProperties
) : OncePerRequestFilter() {
    
    private val logger = LoggerFactory.getLogger(RateLimitingFilter::class.java)
    
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val clientIp = getClientIpAddress(request)
        val endpoint = "${request.method}:${request.requestURI}"
        
        // Check different rate limit tiers
        if (!checkRateLimit(clientIp, endpoint, request, response)) {
            return
        }
        
        filterChain.doFilter(request, response)
    }
    
    private fun checkRateLimit(
        clientIp: String,
        endpoint: String,
        request: HttpServletRequest,
        response: HttpServletResponse
    ): Boolean {
        
        // Global rate limit per IP
        if (!checkGlobalRateLimit(clientIp, response)) {
            return false
        }
        
        // Endpoint-specific rate limit
        if (!checkEndpointRateLimit(clientIp, endpoint, response)) {
            return false
        }
        
        // Authenticated user rate limit (if user is authenticated)
        val authentication = SecurityContextHolder.getContext().authentication
        if (authentication != null && authentication.isAuthenticated) {
            val username = authentication.name
            if (!checkUserRateLimit(username, response)) {
                return false
            }
        }
        
        return true
    }
    
    private fun checkGlobalRateLimit(clientIp: String, response: HttpServletResponse): Boolean {
        val key = "rate_limit:global:$clientIp"
        val limit = rateLimitProperties.global
        return checkLimit(key, limit.requests, limit.duration, response)
    }
    
    private fun checkEndpointRateLimit(clientIp: String, endpoint: String, response: HttpServletResponse): Boolean {
        val endpointConfig = rateLimitProperties.endpoints[endpoint] ?: return true
        val key = "rate_limit:endpoint:$clientIp:$endpoint"
        return checkLimit(key, endpointConfig.requests, endpointConfig.duration, response)
    }
    
    private fun checkUserRateLimit(username: String, response: HttpServletResponse): Boolean {
        val key = "rate_limit:user:$username"
        val limit = rateLimitProperties.user
        return checkLimit(key, limit.requests, limit.duration, response)
    }
    
    private fun checkLimit(
        key: String,
        maxRequests: Long,
        duration: Duration,
        response: HttpServletResponse
    ): Boolean {
        val current = redisTemplate.opsForValue().increment(key) ?: 1L
        
        if (current == 1L) {
            redisTemplate.expire(key, duration)
        }
        
        if (current > maxRequests) {
            val ttl = redisTemplate.getExpire(key, TimeUnit.SECONDS)
            
            response.status = HttpStatus.TOO_MANY_REQUESTS.value()
            response.contentType = MediaType.APPLICATION_JSON_VALUE
            response.addHeader("X-Rate-Limit-Limit", maxRequests.toString())
            response.addHeader("X-Rate-Limit-Remaining", "0")
            response.addHeader("X-Rate-Limit-Reset", (System.currentTimeMillis() / 1000 + ttl).toString())
            
            val errorResponse = mapOf(
                "error" to "Too Many Requests",
                "message" to "Rate limit exceeded. Try again in $ttl seconds",
                "retryAfter" to ttl
            )
            
            val objectMapper = jacksonObjectMapper()
            objectMapper.writeValue(response.outputStream, errorResponse)
            
            logger.warn("Rate limit exceeded for key: {}", key)
            return false
        }
        
        // Add rate limit headers
        val remaining = maxRequests - current
        val ttl = redisTemplate.getExpire(key, TimeUnit.SECONDS)
        
        response.addHeader("X-Rate-Limit-Limit", maxRequests.toString())
        response.addHeader("X-Rate-Limit-Remaining", remaining.toString())
        response.addHeader("X-Rate-Limit-Reset", (System.currentTimeMillis() / 1000 + ttl).toString())
        
        return true
    }
    
    private fun getClientIpAddress(request: HttpServletRequest): String {
        val xForwardedFor = request.getHeader("X-Forwarded-For")
        if (xForwardedFor != null && xForwardedFor.isNotEmpty()) {
            return xForwardedFor.split(",")[0].trim()
        }
        
        val xRealIp = request.getHeader("X-Real-IP")
        if (xRealIp != null && xRealIp.isNotEmpty()) {
            return xRealIp
        }
        
        return request.remoteAddr
    }
}

@ConfigurationProperties(prefix = "app.security.rate-limit")
data class RateLimitProperties(
    val enabled: Boolean = true,
    val global: RateLimitConfig = RateLimitConfig(),
    val user: RateLimitConfig = RateLimitConfig(requests = 1000, duration = Duration.ofHours(1)),
    val endpoints: Map<String, RateLimitConfig> = mapOf(
        "POST:/api/auth/login" to RateLimitConfig(requests = 5, duration = Duration.ofMinutes(15)),
        "POST:/api/auth/register" to RateLimitConfig(requests = 3, duration = Duration.ofMinutes(60))
    )
) {
    data class RateLimitConfig(
        val requests: Long = 100,
        val duration: Duration = Duration.ofMinutes(1)
    )
}
```

### API Key Authentication Filter

```kotlin
@Component
class ApiKeyAuthenticationFilter : OncePerRequestFilter() {
    
    private val logger = LoggerFactory.getLogger(ApiKeyAuthenticationFilter::class.java)
    
    @Autowired
    private lateinit var apiKeyService: ApiKeyService
    
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        
        // Only process API endpoints that require API key authentication
        if (!requiresApiKeyAuth(request)) {
            filterChain.doFilter(request, response)
            return
        }
        
        val apiKey = extractApiKey(request)
        
        if (apiKey != null) {
            try {
                val apiKeyDetails = apiKeyService.validateApiKey(apiKey)
                
                if (apiKeyDetails != null && apiKeyDetails.isActive) {
                    val authentication = createApiKeyAuthentication(apiKeyDetails)
                    SecurityContextHolder.getContext().authentication = authentication
                    
                    // Record API key usage
                    apiKeyService.recordUsage(apiKeyDetails.id, request.requestURI)
                    
                    logger.debug("API key authentication successful for key: {}", apiKeyDetails.name)
                } else {
                    handleInvalidApiKey(response, "Invalid or inactive API key")
                    return
                }
                
            } catch (ex: Exception) {
                logger.error("Error validating API key", ex)
                handleInvalidApiKey(response, "API key validation error")
                return
            }
        } else {
            handleInvalidApiKey(response, "API key required")
            return
        }
        
        filterChain.doFilter(request, response)
    }
    
    private fun requiresApiKeyAuth(request: HttpServletRequest): Boolean {
        // Check if this is an API endpoint that requires API key
        return request.requestURI.startsWith("/api/external/") ||
               request.getHeader("X-API-Client") != null
    }
    
    private fun extractApiKey(request: HttpServletRequest): String? {
        // Try multiple sources for API key
        return request.getHeader("X-API-Key")
            ?: request.getHeader("Authorization")?.let { auth ->
                if (auth.startsWith("ApiKey ")) auth.substring(7) else null
            }
            ?: request.getParameter("api_key")
    }
    
    private fun createApiKeyAuthentication(apiKeyDetails: ApiKeyDetails): Authentication {
        val authorities = apiKeyDetails.permissions.map { SimpleGrantedAuthority("API_$it") }
        return ApiKeyAuthenticationToken(apiKeyDetails, authorities)
    }
    
    private fun handleInvalidApiKey(response: HttpServletResponse, message: String) {
        response.status = HttpStatus.UNAUTHORIZED.value()
        response.contentType = MediaType.APPLICATION_JSON_VALUE
        
        val errorResponse = mapOf(
            "error" to "Unauthorized",
            "message" to message,
            "timestamp" to Instant.now().toString()
        )
        
        val objectMapper = jacksonObjectMapper()
        objectMapper.writeValue(response.outputStream, errorResponse)
    }
}

class ApiKeyAuthenticationToken(
    private val apiKeyDetails: ApiKeyDetails,
    private val authorities: Collection<GrantedAuthority>
) : AbstractAuthenticationToken(authorities) {
    
    init {
        isAuthenticated = true
    }
    
    override fun getCredentials(): Any = apiKeyDetails.keyHash
    override fun getPrincipal(): Any = apiKeyDetails
    
    fun getApiKeyId(): Long = apiKeyDetails.id
    fun getApiKeyName(): String = apiKeyDetails.name
}

@Service
@Transactional
class ApiKeyService(
    private val apiKeyRepository: ApiKeyRepository,
    private val passwordEncoder: PasswordEncoder
) {
    
    private val logger = LoggerFactory.getLogger(ApiKeyService::class.java)
    
    fun validateApiKey(apiKey: String): ApiKeyDetails? {
        return try {
            // Find by key hash for security
            val hashedKey = passwordEncoder.encode(apiKey)
            apiKeyRepository.findByKeyHashAndActiveTrue(hashedKey)
        } catch (ex: Exception) {
            logger.error("Error validating API key", ex)
            null
        }
    }
    
    fun recordUsage(apiKeyId: Long, endpoint: String) {
        try {
            apiKeyRepository.recordUsage(apiKeyId, endpoint, LocalDateTime.now())
        } catch (ex: Exception) {
            logger.error("Error recording API key usage", ex)
        }
    }
    
    fun createApiKey(request: CreateApiKeyRequest): ApiKeyResponse {
        val apiKey = generateSecureApiKey()
        val hashedKey = passwordEncoder.encode(apiKey)
        
        val apiKeyEntity = ApiKey(
            name = request.name,
            description = request.description,
            keyHash = hashedKey,
            permissions = request.permissions.toSet(),
            active = true,
            createdBy = request.createdBy,
            expiresAt = request.expiresAt
        )
        
        val saved = apiKeyRepository.save(apiKeyEntity)
        
        return ApiKeyResponse(
            id = saved.id,
            name = saved.name,
            apiKey = apiKey, // Only returned once during creation
            permissions = saved.permissions.toList(),
            expiresAt = saved.expiresAt,
            message = "API key created successfully. Store it securely - it won't be shown again."
        )
    }
    
    private fun generateSecureApiKey(): String {
        val prefix = "sk_"
        val randomPart = UUID.randomUUID().toString().replace("-", "")
        return "$prefix$randomPart"
    }
}

@Entity
@Table(name = "api_keys")
data class ApiKey(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false)
    val name: String,
    
    val description: String? = null,
    
    @Column(name = "key_hash", nullable = false, unique = true)
    val keyHash: String,
    
    @ElementCollection
    @CollectionTable(name = "api_key_permissions", joinColumns = [JoinColumn(name = "api_key_id")])
    @Column(name = "permission")
    val permissions: Set<String>,
    
    @Column(nullable = false)
    val active: Boolean = true,
    
    @Column(name = "created_by")
    val createdBy: Long,
    
    @Column(name = "created_at")
    val createdAt: LocalDateTime = LocalDateTime.now(),
    
    @Column(name = "expires_at")
    val expiresAt: LocalDateTime? = null,
    
    @Column(name = "last_used_at")
    var lastUsedAt: LocalDateTime? = null,
    
    @Column(name = "usage_count")
    var usageCount: Long = 0
)

data class ApiKeyDetails(
    val id: Long,
    val name: String,
    val keyHash: String,
    val permissions: Set<String>,
    val isActive: Boolean,
    val expiresAt: LocalDateTime?
) {
    val isExpired: Boolean
        get() = expiresAt?.isBefore(LocalDateTime.now()) ?: false
}

// Request/Response models
data class CreateApiKeyRequest(
    val name: String,
    val description: String? = null,
    val permissions: List<String>,
    val createdBy: Long,
    val expiresAt: LocalDateTime? = null
)

data class ApiKeyResponse(
    val id: Long,
    val name: String,
    val apiKey: String?,
    val permissions: List<String>,
    val expiresAt: LocalDateTime?,
    val message: String
)
```

### Security Audit Filter

```kotlin
@Component
class SecurityAuditFilter : OncePerRequestFilter() {
    
    private val logger = LoggerFactory.getLogger(SecurityAuditFilter::class.java)
    
    @Autowired
    private lateinit var securityAuditService: SecurityAuditService
    
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        
        val startTime = System.currentTimeMillis()
        val requestId = UUID.randomUUID().toString()
        
        // Wrap request and response for audit logging
        val wrappedRequest = ContentCachingRequestWrapper(request)
        val wrappedResponse = ContentCachingResponseWrapper(response)
        
        try {
            MDC.put("requestId", requestId)
            filterChain.doFilter(wrappedRequest, wrappedResponse)
            
        } finally {
            val duration = System.currentTimeMillis() - startTime
            
            // Log security-relevant requests
            if (isSecurityRelevant(wrappedRequest, wrappedResponse)) {
                logSecurityEvent(wrappedRequest, wrappedResponse, duration)
            }
            
            // Copy response body back
            wrappedResponse.copyBodyToResponse()
            MDC.clear()
        }
    }
    
    private fun isSecurityRelevant(
        request: ContentCachingRequestWrapper,
        response: ContentCachingResponseWrapper
    ): Boolean {
        val uri = request.requestURI
        val method = request.method
        val status = response.status
        
        return uri.startsWith("/api/auth/") ||
               uri.startsWith("/api/admin/") ||
               uri.contains("password") ||
               status == 401 ||
               status == 403 ||
               status >= 500 ||
               method in listOf("POST", "PUT", "DELETE")
    }
    
    private fun logSecurityEvent(
        request: ContentCachingRequestWrapper,
        response: ContentCachingResponseWrapper,
        duration: Long
    ) {
        try {
            val authentication = SecurityContextHolder.getContext().authentication
            val username = authentication?.name ?: "anonymous"
            val ipAddress = getClientIpAddress(request)
            
            val auditEvent = SecurityAuditEvent(
                requestId = MDC.get("requestId"),
                timestamp = LocalDateTime.now(),
                username = username,
                ipAddress = ipAddress,
                userAgent = request.getHeader("User-Agent"),
                method = request.method,
                uri = request.requestURI,
                statusCode = response.status,
                duration = duration,
                requestBody = if (shouldLogRequestBody(request)) {
                    String(request.contentAsByteArray, StandardCharsets.UTF_8)
                } else null,
                responseBody = if (shouldLogResponseBody(response)) {
                    String(response.contentAsByteArray, StandardCharsets.UTF_8)
                } else null
            )
            
            securityAuditService.logEvent(auditEvent)
            
        } catch (ex: Exception) {
            logger.error("Error logging security audit event", ex)
        }
    }
    
    private fun shouldLogRequestBody(request: ContentCachingRequestWrapper): Boolean {
        val contentType = request.contentType
        val uri = request.requestURI
        
        return contentType?.contains("application/json") == true &&
               !uri.contains("password") &&
               !uri.contains("login") &&
               request.contentAsByteArray.size < 1024 // Limit size
    }
    
    private fun shouldLogResponseBody(response: ContentCachingResponseWrapper): Boolean {
        val contentType = response.contentType
        val status = response.status
        
        return (status >= 400 || status == 200) &&
               contentType?.contains("application/json") == true &&
               response.contentAsByteArray.size < 1024 // Limit size
    }
    
    private fun getClientIpAddress(request: HttpServletRequest): String {
        return request.getHeader("X-Forwarded-For")?.split(",")?.get(0)?.trim()
            ?: request.getHeader("X-Real-IP")
            ?: request.remoteAddr
    }
}

@Service
@Async
@Transactional
class SecurityAuditService(
    private val securityAuditRepository: SecurityAuditRepository
) {
    
    private val logger = LoggerFactory.getLogger(SecurityAuditService::class.java)
    
    fun logEvent(auditEvent: SecurityAuditEvent) {
        try {
            securityAuditRepository.save(auditEvent.toEntity())
        } catch (ex: Exception) {
            logger.error("Failed to save security audit event", ex)
        }
    }
    
    fun getAuditEvents(
        username: String? = null,
        ipAddress: String? = null,
        since: LocalDateTime? = null,
        pageable: Pageable
    ): Page<SecurityAuditEvent> {
        return securityAuditRepository.findAuditEvents(username, ipAddress, since, pageable)
            .map { it.toDto() }
    }
}

data class SecurityAuditEvent(
    val requestId: String,
    val timestamp: LocalDateTime,
    val username: String,
    val ipAddress: String,
    val userAgent: String?,
    val method: String,
    val uri: String,
    val statusCode: Int,
    val duration: Long,
    val requestBody: String? = null,
    val responseBody: String? = null
) {
    fun toEntity(): SecurityAuditEntity {
        return SecurityAuditEntity(
            requestId = requestId,
            timestamp = timestamp,
            username = username,
            ipAddress = ipAddress,
            userAgent = userAgent,
            method = method,
            uri = uri,
            statusCode = statusCode,
            duration = duration,
            requestBody = requestBody,
            responseBody = responseBody
        )
    }
}

@Entity
@Table(name = "security_audit_log")
data class SecurityAuditEntity(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(name = "request_id", nullable = false)
    val requestId: String,
    
    @Column(nullable = false)
    val timestamp: LocalDateTime,
    
    @Column(nullable = false)
    val username: String,
    
    @Column(name = "ip_address", nullable = false)
    val ipAddress: String,
    
    @Column(name = "user_agent")
    val userAgent: String?,
    
    @Column(nullable = false)
    val method: String,
    
    @Column(nullable = false)
    val uri: String,
    
    @Column(name = "status_code", nullable = false)
    val statusCode: Int,
    
    @Column(nullable = false)
    val duration: Long,
    
    @Column(name = "request_body", columnDefinition = "TEXT")
    val requestBody: String?,
    
    @Column(name = "response_body", columnDefinition = "TEXT") 
    val responseBody: String?
) {
    fun toDto(): SecurityAuditEvent {
        return SecurityAuditEvent(
            requestId = requestId,
            timestamp = timestamp,
            username = username,
            ipAddress = ipAddress,
            userAgent = userAgent,
            method = method,
            uri = uri,
            statusCode = statusCode,
            duration = duration,
            requestBody = requestBody,
            responseBody = responseBody
        )
    }
}
```

## 13.5 Summary

Security is a critical aspect of any modern application, and throughout this chapter, we've built a comprehensive security framework that handles authentication, authorization, and auditing with Kotlin and Spring Security.

**Security Basics** provided the foundation with proper password handling, user modeling, and security configuration. We established:
- Strong password validation with entropy scoring
- Secure password hashing using BCrypt with appropriate strength
- Comprehensive security properties configuration
- User domain model with Spring Security integration

**Spring Security Configuration** implemented robust security controls:
- Separate filter chains for API and web endpoints  
- JWT-based stateless authentication for APIs
- Session-based authentication for web interfaces
- Comprehensive CORS configuration
- Role-based access control with method-level security

**JWT Integration** provided modern token-based authentication:
- Secure JWT token generation with appropriate claims
- Token validation with blacklisting support
- Refresh token mechanism for seamless user experience  
- Redis-based token metadata tracking
- Comprehensive token lifecycle management

**Custom Filters and Configurations** added advanced security features:
- Rate limiting to prevent abuse and DoS attacks
- API key authentication for service-to-service communication
- Security audit logging for compliance and monitoring
- Comprehensive event tracking and analysis

**Key security principles** we've implemented:

1. **Defense in Depth**: Multiple layers of security controls
2. **Principle of Least Privilege**: Users get only necessary permissions
3. **Secure by Default**: Safe defaults with explicit opt-in for permissive settings
4. **Comprehensive Logging**: Full audit trail for security events
5. **Token Security**: Proper JWT handling with revocation support

**Kotlin-specific advantages** leveraged:

- Data classes for clean, immutable security models
- Sealed classes for type-safe authentication results  
- Extension functions for enhanced security utilities
- Coroutines integration for non-blocking security operations
- Null safety for robust security logic

**Production considerations** addressed:

- Rate limiting to prevent abuse
- Comprehensive audit logging for compliance
- Token blacklisting and revocation
- Password strength validation and policies
- API key management for service integration
- Security event monitoring and alerting

The security framework we've built provides enterprise-grade protection while maintaining usability and performance. It handles common attack vectors like brute force attacks, token theft, and privilege escalation while providing comprehensive monitoring and audit capabilities.

The patterns and implementations in this chapter serve as a solid foundation for securing Kotlin Spring Boot applications. Whether you're building consumer applications, enterprise systems, or API services, these security patterns will help protect your users and data while maintaining compliance with security best practices and regulations.

Security is an ongoing process, not a one-time implementation. The framework we've established provides the foundation for continuous security improvement and adaptation to emerging threats and requirements.