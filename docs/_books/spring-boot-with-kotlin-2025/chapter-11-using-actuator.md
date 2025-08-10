---
layout: default
---
# Chapter 11: Using Actuator

Spring Boot Actuator is one of the most valuable features of the Spring Boot ecosystem, providing production-ready monitoring, metrics, and operational insights out of the box. In this chapter, we'll explore how to effectively use Actuator in Kotlin-based Spring Boot applications to gain deep visibility into your application's health, performance, and behavior.

Actuator transforms your application from a black box into an observable system that can be monitored, diagnosed, and managed in production environments. Whether you're debugging performance issues, monitoring resource usage, or ensuring your application is healthy, Actuator provides the tools you need.

We'll start with basic setup and configuration, explore the rich set of built-in endpoints, and then dive deep into customization options. By the end of this chapter, you'll understand how to leverage Actuator for comprehensive application observability and how to extend it with custom metrics and health checks tailored to your specific needs.

## 11.1 Dependency and Setup

Setting up Actuator in a Kotlin Spring Boot application is straightforward, but proper configuration is crucial for both security and functionality.

### Basic Dependency Configuration

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security") // Recommended for production
    
    // Optional: For enhanced metrics
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-registry-jmx")
    
    // Optional: For tracing (if using distributed tracing)
    implementation("io.micrometer:micrometer-tracing-bridge-brave")
    implementation("io.zipkin.reporter2:zipkin-reporter-brave")
    
    // Development and testing
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

### Basic Application Configuration

Let's start with a comprehensive Actuator configuration that balances functionality with security:

```kotlin
# application.yml
server:
  port: 8080
  
management:
  # Use a different port for actuator endpoints in production
  server:
    port: 8081
    ssl:
      enabled: false # Set to true in production with proper certificates
  
  # Configure which endpoints to expose
  endpoints:
    web:
      exposure:
        # Be very careful with 'include: "*"' in production
        include: health,info,metrics,env,beans,mappings,prometheus
      base-path: /actuator
      cors:
        allowed-origins: "*"
        allowed-methods: GET,POST
    jmx:
      exposure:
        include: "*"
  
  # Endpoint-specific configuration
  endpoint:
    health:
      show-details: when-authorized
      show-components: always
      probes:
        enabled: true
    info:
      enabled: true
    metrics:
      enabled: true
    env:
      show-values: when-authorized
    beans:
      enabled: true
    shutdown:
      enabled: false # Never enable in production without proper security
  
  # Metrics configuration
  metrics:
    enabled: true
    export:
      prometheus:
        enabled: true
        step: PT1M
      jmx:
        enabled: true
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5,0.95,0.99
      slo:
        http.server.requests: 10ms,50ms,100ms,200ms,500ms
    
  # Tracing configuration (if using distributed tracing)
  tracing:
    sampling:
      probability: 0.1 # 10% sampling rate

# Application-specific info
info:
  app:
    name: ${spring.application.name:My Kotlin Spring Boot App}
    description: A comprehensive Spring Boot application with Kotlin
    version: ${app.version:1.0.0}
    encoding: ${file.encoding:UTF-8}
    java:
      version: ${java.version}
  build:
    time: ${app.build.time:unknown}
    artifact: ${project.artifactId:app}
    group: ${project.groupId:com.example}
```

### Security Configuration for Actuator

Security is crucial when exposing Actuator endpoints. Here's a comprehensive security configuration:

```kotlin
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
class ActuatorSecurityConfiguration {
    
    @Bean
    @Order(1)
    fun actuatorSecurityFilterChain(http: HttpSecurity): SecurityFilterChain {
        return http
            .requestMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests { requests ->
                requests
                    // Public endpoints - no authentication required
                    .requestMatchers(EndpointRequest.to(HealthEndpoint::class.java)).permitAll()
                    .requestMatchers(EndpointRequest.to(InfoEndpoint::class.java)).permitAll()
                    
                    // Prometheus metrics - typically accessed by monitoring systems
                    .requestMatchers(EndpointRequest.to("prometheus")).hasRole("METRICS_READER")
                    
                    // Sensitive endpoints - require admin privileges
                    .requestMatchers(
                        EndpointRequest.to(
                            EnvironmentEndpoint::class.java,
                            BeansEndpoint::class.java,
                            ConfigurableEnvironmentEndpoint::class.java
                        )
                    ).hasRole("ACTUATOR_ADMIN")
                    
                    // All other actuator endpoints require monitoring role
                    .anyRequest().hasRole("ACTUATOR_USER")
            }
            .httpBasic { }
            .csrf { csrf -> csrf.disable() }
            .build()
    }
    
    @Bean
    @Order(2)
    fun applicationSecurityFilterChain(http: HttpSecurity): SecurityFilterChain {
        return http
            .requestMatcher(AnyRequestMatcher.INSTANCE)
            .authorizeHttpRequests { requests ->
                requests
                    .requestMatchers("/api/v1/public/**").permitAll()
                    .requestMatchers("/api/v1/auth/**").permitAll()
                    .anyRequest().authenticated()
            }
            .oauth2ResourceServer { oauth2 ->
                oauth2.jwt { }
            }
            .build()
    }
    
    @Bean
    fun actuatorUserDetailsService(): UserDetailsService {
        val actuatorUser = User.builder()
            .username("actuator-user")
            .password("{noop}actuator-password") // Use proper password encoding in production
            .roles("ACTUATOR_USER")
            .build()
        
        val metricsReader = User.builder()
            .username("metrics-reader")
            .password("{noop}metrics-password")
            .roles("METRICS_READER")
            .build()
        
        val actuatorAdmin = User.builder()
            .username("actuator-admin")
            .password("{noop}admin-password")
            .roles("ACTUATOR_ADMIN", "ACTUATOR_USER", "METRICS_READER")
            .build()
        
        return InMemoryUserDetailsManager(actuatorUser, metricsReader, actuatorAdmin)
    }
}
```

### Production-Ready Configuration

For production environments, you'll want more sophisticated configuration:

```kotlin
@Configuration
@EnableConfigurationProperties(ActuatorConfiguration.ActuatorProperties::class)
class ActuatorConfiguration(
    private val actuatorProperties: ActuatorProperties
) {
    
    @ConfigurationProperties(prefix = "app.actuator")
    data class ActuatorProperties(
        val enabled: Boolean = true,
        val security: SecurityProperties = SecurityProperties(),
        val metrics: MetricsProperties = MetricsProperties()
    ) {
        data class SecurityProperties(
            val enabled: Boolean = true,
            val allowedIps: List<String> = emptyList(),
            val requireSsl: Boolean = false
        )
        
        data class MetricsProperties(
            val export: ExportProperties = ExportProperties()
        ) {
            data class ExportProperties(
                val prometheus: PrometheusProperties = PrometheusProperties(),
                val jmx: JmxProperties = JmxProperties()
            ) {
                data class PrometheusProperties(
                    val enabled: Boolean = false,
                    val step: Duration = Duration.ofMinutes(1),
                    val descriptions: Boolean = true
                )
                
                data class JmxProperties(
                    val enabled: Boolean = false,
                    val domain: String = "metrics"
                )
            }
        }
    }
    
    @Bean
    @ConditionalOnProperty(value = ["app.actuator.enabled"], havingValue = "true", matchIfMissing = true)
    fun actuatorCustomizer(): WebEndpointProperties.Exposure {
        val exposure = WebEndpointProperties.Exposure()
        
        when {
            isProduction() -> {
                // Minimal exposure in production
                exposure.include = setOf("health", "info", "metrics", "prometheus")
            }
            isStaging() -> {
                // More endpoints in staging for debugging
                exposure.include = setOf("health", "info", "metrics", "env", "beans", "prometheus")
            }
            else -> {
                // All endpoints in development
                exposure.include = setOf("*")
            }
        }
        
        return exposure
    }
    
    @Bean
    @ConditionalOnProperty(value = ["app.actuator.security.enabled"], havingValue = "true", matchIfMissing = true)
    fun actuatorIpFilter(): FilterRegistrationBean<ActuatorIpFilter> {
        val registration = FilterRegistrationBean<ActuatorIpFilter>()
        registration.filter = ActuatorIpFilter(actuatorProperties.security.allowedIps)
        registration.addUrlPatterns("/actuator/*")
        registration.order = Ordered.HIGHEST_PRECEDENCE
        return registration
    }
    
    private fun isProduction(): Boolean = 
        System.getProperty("spring.profiles.active")?.contains("prod") == true
    
    private fun isStaging(): Boolean = 
        System.getProperty("spring.profiles.active")?.contains("staging") == true
}

// IP filtering for actuator endpoints
class ActuatorIpFilter(
    private val allowedIps: List<String>
) : OncePerRequestFilter() {
    
    private val logger = LoggerFactory.getLogger(ActuatorIpFilter::class.java)
    
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        if (allowedIps.isEmpty()) {
            // If no IPs configured, allow all (useful for development)
            filterChain.doFilter(request, response)
            return
        }
        
        val clientIp = getClientIpAddress(request)
        
        if (isIpAllowed(clientIp)) {
            filterChain.doFilter(request, response)
        } else {
            logger.warn("Access to actuator endpoint denied for IP: {}", clientIp)
            response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access denied")
        }
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
    
    private fun isIpAllowed(ip: String): Boolean {
        return allowedIps.any { allowedIp ->
            when {
                allowedIp == ip -> true
                allowedIp.endsWith("*") -> ip.startsWith(allowedIp.dropLast(1))
                allowedIp.contains("/") -> isIpInCidr(ip, allowedIp)
                else -> false
            }
        }
    }
    
    private fun isIpInCidr(ip: String, cidr: String): Boolean {
        // Simple CIDR implementation - you might want to use a library for production
        try {
            val parts = cidr.split("/")
            val networkIp = parts[0]
            val prefixLength = parts[1].toInt()
            
            // This is a simplified implementation
            // Consider using Apache Commons Net or similar for production
            return ip.startsWith(networkIp.substringBeforeLast("."))
        } catch (e: Exception) {
            return false
        }
    }
}
```

## 11.2 Built-in Endpoints

Actuator provides a rich set of built-in endpoints. Let's explore the most important ones and how to use them effectively.

### Health Endpoint

The health endpoint is crucial for monitoring application health:

```kotlin
// Custom health indicators
@Component
class DatabaseHealthIndicator(
    private val dataSource: DataSource
) : HealthIndicator {
    
    private val logger = LoggerFactory.getLogger(DatabaseHealthIndicator::class.java)
    
    override fun health(): Health {
        return try {
            dataSource.connection.use { connection ->
                val isValid = connection.isValid(5)
                if (isValid) {
                    // Test a simple query to ensure database is responsive
                    connection.prepareStatement("SELECT 1").use { statement ->
                        statement.executeQuery().use { resultSet ->
                            if (resultSet.next()) {
                                Health.up()
                                    .withDetail("database", "PostgreSQL")
                                    .withDetail("connection_pool", getConnectionPoolInfo())
                                    .withDetail("validation_query", "SELECT 1")
                                    .withDetail("response_time", measureResponseTime(connection))
                                    .build()
                            } else {
                                Health.down()
                                    .withDetail("error", "Query returned no results")
                                    .build()
                            }
                        }
                    }
                } else {
                    Health.down()
                        .withDetail("error", "Connection validation failed")
                        .build()
                }
            }
        } catch (ex: SQLException) {
            logger.error("Database health check failed", ex)
            Health.down(ex)
                .withDetail("error", ex.message)
                .withDetail("error_code", ex.errorCode)
                .build()
        } catch (ex: Exception) {
            logger.error("Unexpected error during database health check", ex)
            Health.down(ex).build()
        }
    }
    
    private fun getConnectionPoolInfo(): Map<String, Any> {
        if (dataSource is HikariDataSource) {
            val hikariPool = dataSource.hikariPoolMXBean
            return mapOf(
                "active_connections" to hikariPool.activeConnections,
                "idle_connections" to hikariPool.idleConnections,
                "total_connections" to hikariPool.totalConnections,
                "threads_waiting" to hikariPool.threadsAwaitingConnection
            )
        }
        return mapOf("type" to dataSource.javaClass.simpleName)
    }
    
    private fun measureResponseTime(connection: Connection): String {
        val startTime = System.currentTimeMillis()
        try {
            connection.prepareStatement("SELECT 1").use { it.executeQuery() }
            val responseTime = System.currentTimeMillis() - startTime
            return "${responseTime}ms"
        } catch (ex: Exception) {
            return "N/A"
        }
    }
}

@Component
class RedisHealthIndicator(
    private val redisTemplate: RedisTemplate<String, String>
) : HealthIndicator {
    
    private val logger = LoggerFactory.getLogger(RedisHealthIndicator::class.java)
    
    override fun health(): Health {
        return try {
            val startTime = System.currentTimeMillis()
            val ping = redisTemplate.execute { connection: RedisConnection ->
                connection.ping()
            }
            val responseTime = System.currentTimeMillis() - startTime
            
            if (ping == "PONG") {
                Health.up()
                    .withDetail("redis", "Available")
                    .withDetail("response_time", "${responseTime}ms")
                    .withDetail("connection_info", getRedisConnectionInfo())
                    .build()
            } else {
                Health.down()
                    .withDetail("error", "Unexpected ping response: $ping")
                    .build()
            }
        } catch (ex: Exception) {
            logger.error("Redis health check failed", ex)
            Health.down(ex)
                .withDetail("error", ex.message)
                .build()
        }
    }
    
    private fun getRedisConnectionInfo(): Map<String, Any?> {
        return try {
            redisTemplate.execute { connection: RedisConnection ->
                val info = connection.info()
                mapOf(
                    "version" to info?.getProperty("redis_version"),
                    "mode" to info?.getProperty("redis_mode"),
                    "connected_clients" to info?.getProperty("connected_clients")
                )
            } ?: emptyMap()
        } catch (ex: Exception) {
            emptyMap<String, Any?>()
        }
    }
}

// External service health indicator
@Component
class ExternalApiHealthIndicator(
    private val webClient: WebClient,
    @Value("\${app.external-api.health-check-url}") private val healthCheckUrl: String
) : HealthIndicator {
    
    private val logger = LoggerFactory.getLogger(ExternalApiHealthIndicator::class.java)
    
    override fun health(): Health {
        return try {
            val startTime = System.currentTimeMillis()
            
            val response = webClient
                .get()
                .uri(healthCheckUrl)
                .retrieve()
                .toBodilessEntity()
                .timeout(Duration.ofSeconds(5))
                .block()
            
            val responseTime = System.currentTimeMillis() - startTime
            
            if (response?.statusCode?.is2xxSuccessful == true) {
                Health.up()
                    .withDetail("external_api", "Available")
                    .withDetail("url", healthCheckUrl)
                    .withDetail("response_time", "${responseTime}ms")
                    .withDetail("status_code", response.statusCode.value())
                    .build()
            } else {
                Health.down()
                    .withDetail("external_api", "Unavailable")
                    .withDetail("url", healthCheckUrl)
                    .withDetail("status_code", response?.statusCode?.value())
                    .build()
            }
        } catch (ex: TimeoutException) {
            logger.warn("External API health check timed out")
            Health.down()
                .withDetail("error", "Health check timed out")
                .withDetail("url", healthCheckUrl)
                .build()
        } catch (ex: Exception) {
            logger.error("External API health check failed", ex)
            Health.down(ex)
                .withDetail("url", healthCheckUrl)
                .build()
        }
    }
}

// Application-specific health indicator
@Component
class ApplicationHealthIndicator(
    private val cacheManager: CacheManager,
    private val taskExecutor: TaskExecutor
) : HealthIndicator {
    
    override fun health(): Health {
        val details = mutableMapOf<String, Any>()
        var overallStatus = Status.UP
        
        // Check cache status
        try {
            val cacheNames = cacheManager.cacheNames
            details["cache"] = mapOf(
                "available_caches" to cacheNames.size,
                "cache_names" to cacheNames
            )
        } catch (ex: Exception) {
            details["cache"] = mapOf("error" to ex.message)
            overallStatus = Status.DOWN
        }
        
        // Check thread pool status (if using ThreadPoolTaskExecutor)
        if (taskExecutor is ThreadPoolTaskExecutor) {
            val poolDetails = mapOf(
                "core_pool_size" to taskExecutor.corePoolSize,
                "max_pool_size" to taskExecutor.maxPoolSize,
                "active_count" to taskExecutor.activeCount,
                "pool_size" to taskExecutor.poolSize,
                "queue_size" to taskExecutor.threadPoolExecutor?.queue?.size
            )
            details["thread_pool"] = poolDetails
            
            // Consider unhealthy if queue is too large
            val queueSize = taskExecutor.threadPoolExecutor?.queue?.size ?: 0
            if (queueSize > 100) { // Configurable threshold
                overallStatus = Status.DOWN
                details["thread_pool_warning"] = "Queue size is high: $queueSize"
            }
        }
        
        return Health.Builder(overallStatus)
            .withDetails(details)
            .build()
    }
}
```

### Metrics Endpoint

The metrics endpoint provides detailed application metrics:

```kotlin
@Component
class CustomMetricsConfiguration {
    
    @Bean
    fun commonTags(
        @Value("\${spring.application.name}") applicationName: String,
        @Value("\${management.metrics.tags.environment:unknown}") environment: String
    ): MeterRegistryCustomizer<MeterRegistry> {
        return MeterRegistryCustomizer { registry ->
            registry.config().commonTags(
                "application", applicationName,
                "environment", environment,
                "instance", InetAddress.getLocalHost().hostName
            )
        }
    }
    
    @EventListener
    @Async
    fun handleApplicationEvent(event: ApplicationReadyEvent) {
        // Record application startup time
        Metrics.gauge("application.startup.time", System.currentTimeMillis())
    }
}

// Custom metrics in services
@Service
class UserService(
    private val userRepository: UserRepository,
    private val meterRegistry: MeterRegistry
) {
    
    private val userCreationCounter = Counter.builder("user.creation.total")
        .description("Total number of users created")
        .register(meterRegistry)
    
    private val userCreationTimer = Timer.builder("user.creation.duration")
        .description("Time taken to create a user")
        .register(meterRegistry)
    
    private val activeUsersGauge = Gauge.builder("user.active.count")
        .description("Number of active users")
        .register(meterRegistry) { userRepository.countByActiveTrue() }
    
    fun createUser(request: CreateUserRequest): UserResponse {
        return userCreationTimer.recordCallable {
            try {
                val user = performUserCreation(request)
                userCreationCounter.increment(
                    Tags.of(
                        "status", "success",
                        "method", "api"
                    )
                )
                user
            } catch (ex: Exception) {
                userCreationCounter.increment(
                    Tags.of(
                        "status", "error",
                        "error_type", ex.javaClass.simpleName
                    )
                )
                throw ex
            }
        }!!
    }
    
    private fun performUserCreation(request: CreateUserRequest): UserResponse {
        // Implementation details
        TODO("Implementation")
    }
}

// Method-level metrics with AOP
@Aspect
@Component
class MetricsAspect(private val meterRegistry: MeterRegistry) {
    
    @Around("@annotation(timed)")
    fun timeMethod(joinPoint: ProceedingJoinPoint, timed: Timed): Any? {
        val methodName = joinPoint.signature.name
        val className = joinPoint.signature.declaringType.simpleName
        
        val timer = Timer.builder("method.execution.time")
            .description("Method execution time")
            .tag("class", className)
            .tag("method", methodName)
            .register(meterRegistry)
        
        return timer.recordCallable { joinPoint.proceed() }
    }
    
    @Around("@annotation(counted)")
    fun countMethod(joinPoint: ProceedingJoinPoint, counted: Counted): Any? {
        val methodName = joinPoint.signature.name
        val className = joinPoint.signature.declaringType.simpleName
        
        val counter = Counter.builder("method.invocation.total")
            .description("Method invocation count")
            .tag("class", className)
            .tag("method", methodName)
            .register(meterRegistry)
        
        return try {
            val result = joinPoint.proceed()
            counter.increment(Tags.of("status", "success"))
            result
        } catch (ex: Exception) {
            counter.increment(Tags.of(
                "status", "error",
                "exception", ex.javaClass.simpleName
            ))
            throw ex
        }
    }
}

// Custom annotations for metrics
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Timed(val value: String = "")

@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Counted(val value: String = "")
```

### Info Endpoint

The info endpoint provides application information:

```kotlin
@Component
class CustomInfoContributor : InfoContributor {
    
    override fun contribute(builder: Info.Builder) {
        builder
            .withDetail("application") {
                mapOf(
                    "name" to "Kotlin Spring Boot Application",
                    "description" to "A comprehensive example application",
                    "version" to getApplicationVersion(),
                    "build_time" to getBuildTime(),
                    "commit" to getGitCommitInfo()
                )
            }
            .withDetail("system") {
                mapOf(
                    "java_version" to System.getProperty("java.version"),
                    "java_vendor" to System.getProperty("java.vendor"),
                    "os_name" to System.getProperty("os.name"),
                    "os_version" to System.getProperty("os.version"),
                    "available_processors" to Runtime.getRuntime().availableProcessors(),
                    "max_memory" to "${Runtime.getRuntime().maxMemory() / 1024 / 1024} MB",
                    "total_memory" to "${Runtime.getRuntime().totalMemory() / 1024 / 1024} MB",
                    "free_memory" to "${Runtime.getRuntime().freeMemory() / 1024 / 1024} MB"
                )
            }
            .withDetail("features") {
                mapOf(
                    "database" to "PostgreSQL",
                    "cache" to "Redis",
                    "messaging" to "RabbitMQ",
                    "security" to "JWT + OAuth2",
                    "monitoring" to "Actuator + Prometheus"
                )
            }
    }
    
    private fun getApplicationVersion(): String {
        // Try to read from manifest or properties
        val implementationVersion = javaClass.`package`?.implementationVersion
        return implementationVersion ?: "unknown"
    }
    
    private fun getBuildTime(): String {
        // Try to read from build properties
        return try {
            val properties = Properties()
            properties.load(javaClass.getResourceAsStream("/build-info.properties"))
            properties.getProperty("build.time", "unknown")
        } catch (ex: Exception) {
            "unknown"
        }
    }
    
    private fun getGitCommitInfo(): Map<String, String> {
        return try {
            val properties = Properties()
            properties.load(javaClass.getResourceAsStream("/git.properties"))
            mapOf(
                "commit_id" to properties.getProperty("git.commit.id.abbrev", "unknown"),
                "commit_time" to properties.getProperty("git.commit.time", "unknown"),
                "branch" to properties.getProperty("git.branch", "unknown")
            )
        } catch (ex: Exception) {
            mapOf("status" to "unavailable")
        }
    }
}

// Build info contribution
@Component
@ConditionalOnProperty(value = ["management.info.build.enabled"], havingValue = "true")
class BuildInfoContributor(
    private val buildProperties: BuildProperties? = null
) : InfoContributor {
    
    override fun contribute(builder: Info.Builder) {
        buildProperties?.let { build ->
            builder.withDetail("build") {
                mapOf(
                    "artifact" to build.artifact,
                    "group" to build.group,
                    "name" to build.name,
                    "version" to build.version,
                    "time" to build.time.toString()
                )
            }
        }
    }
}

// Git info contribution
@Component
@ConditionalOnProperty(value = ["management.info.git.enabled"], havingValue = "true")
class GitInfoContributor(
    private val gitProperties: GitProperties? = null
) : InfoContributor {
    
    override fun contribute(builder: Info.Builder) {
        gitProperties?.let { git ->
            builder.withDetail("git") {
                mapOf(
                    "branch" to git.branch,
                    "commit" to mapOf(
                        "id" to git.commitId,
                        "time" to git.commitTime.toString()
                    )
                )
            }
        }
    }
}
```

### Environment Endpoint

The environment endpoint exposes configuration properties:

```kotlin
@Component
class EnvironmentPostProcessor : EnvironmentPostProcessor {
    
    override fun postProcessEnvironment(
        environment: ConfigurableEnvironment,
        application: SpringApplication
    ) {
        // Add custom property sources
        val customProperties = MapPropertySource("custom", mapOf(
            "app.environment.processor.enabled" to true,
            "app.environment.processor.timestamp" to Instant.now().toString()
        ))
        
        environment.propertySources.addLast(customProperties)
    }
}

// Custom environment contributor
@Component
class CustomEnvironmentInfoContributor : InfoContributor {
    
    @Autowired
    private lateinit var environment: Environment
    
    override fun contribute(builder: Info.Builder) {
        val profiles = environment.activeProfiles.takeIf { it.isNotEmpty() } 
            ?: environment.defaultProfiles
        
        builder.withDetail("environment") {
            mapOf(
                "active_profiles" to profiles.toList(),
                "default_profiles" to environment.defaultProfiles.toList(),
                "configuration_classes" to getConfigurationClasses()
            )
        }
    }
    
    private fun getConfigurationClasses(): List<String> {
        // This is a simplified implementation
        // In a real application, you might want to inspect the application context
        return listOf(
            "ActuatorConfiguration",
            "SecurityConfiguration", 
            "DatabaseConfiguration"
        )
    }
}
```

## 11.3 Customizing Actuator

Actuator's real power comes from its extensibility. Let's explore advanced customization options.

### Custom Endpoints

Creating custom endpoints allows you to expose application-specific information:

```kotlin
// Custom endpoint for application statistics
@Component
@Endpoint(id = "application-stats")
class ApplicationStatsEndpoint(
    private val userRepository: UserRepository,
    private val orderRepository: OrderRepository,
    private val cacheManager: CacheManager
) {
    
    @ReadOperation
    fun getApplicationStats(): ApplicationStats {
        return ApplicationStats(
            userStats = getUserStats(),
            orderStats = getOrderStats(),
            cacheStats = getCacheStats(),
            systemStats = getSystemStats()
        )
    }
    
    @ReadOperation
    fun getApplicationStats(@Selector period: String): ApplicationStats {
        val periodDays = when (period.toLowerCase()) {
            "daily" -> 1
            "weekly" -> 7
            "monthly" -> 30
            else -> throw IllegalArgumentException("Invalid period. Use: daily, weekly, monthly")
        }
        
        return getApplicationStatsForPeriod(periodDays)
    }
    
    @WriteOperation
    fun refreshStats(): Map<String, Any> {
        // Trigger stats refresh
        return mapOf(
            "status" to "refreshed",
            "timestamp" to Instant.now()
        )
    }
    
    @DeleteOperation
    fun clearStats(): Map<String, Any> {
        // Clear cached stats
        return mapOf(
            "status" to "cleared",
            "timestamp" to Instant.now()
        )
    }
    
    private fun getUserStats(): UserStats {
        return UserStats(
            totalUsers = userRepository.count(),
            activeUsers = userRepository.countByActiveTrue(),
            newUsersToday = userRepository.countUsersCreatedAfter(LocalDateTime.now().minusDays(1)),
            newUsersThisWeek = userRepository.countUsersCreatedAfter(LocalDateTime.now().minusDays(7))
        )
    }
    
    private fun getOrderStats(): OrderStats {
        return OrderStats(
            totalOrders = orderRepository.count(),
            ordersToday = orderRepository.countOrdersCreatedAfter(LocalDateTime.now().minusDays(1)),
            ordersThisWeek = orderRepository.countOrdersCreatedAfter(LocalDateTime.now().minusDays(7)),
            totalRevenue = orderRepository.getTotalRevenue()
        )
    }
    
    private fun getCacheStats(): CacheStats {
        val cacheStatistics = cacheManager.cacheNames.associate { cacheName ->
            val cache = cacheManager.getCache(cacheName)
            cacheName to if (cache != null) {
                mapOf(
                    "size" to getCacheSize(cache),
                    "hit_ratio" to getCacheHitRatio(cache)
                )
            } else {
                mapOf("status" to "unavailable")
            }
        }
        
        return CacheStats(
            availableCaches = cacheManager.cacheNames.size,
            cacheStatistics = cacheStatistics
        )
    }
    
    private fun getSystemStats(): SystemStats {
        val runtime = Runtime.getRuntime()
        val memoryBean = ManagementFactory.getMemoryMXBean()
        val gcBeans = ManagementFactory.getGarbageCollectorMXBeans()
        
        return SystemStats(
            heapMemoryUsage = memoryBean.heapMemoryUsage.let {
                MemoryUsage(
                    used = it.used,
                    committed = it.committed,
                    max = it.max
                )
            },
            nonHeapMemoryUsage = memoryBean.nonHeapMemoryUsage.let {
                MemoryUsage(
                    used = it.used,
                    committed = it.committed,
                    max = it.max
                )
            },
            garbageCollection = gcBeans.map { gc ->
                GarbageCollectionStats(
                    name = gc.name,
                    collectionCount = gc.collectionCount,
                    collectionTime = gc.collectionTime
                )
            },
            processorCount = runtime.availableProcessors(),
            systemLoadAverage = ManagementFactory.getOperatingSystemMXBean().systemLoadAverage
        )
    }
    
    private fun getApplicationStatsForPeriod(days: Int): ApplicationStats {
        // Implementation for period-specific stats
        val cutoffDate = LocalDateTime.now().minusDays(days.toLong())
        return ApplicationStats(
            userStats = UserStats(
                totalUsers = userRepository.count(),
                activeUsers = userRepository.countByActiveTrue(),
                newUsersToday = userRepository.countUsersCreatedAfter(cutoffDate),
                newUsersThisWeek = userRepository.countUsersCreatedAfter(cutoffDate)
            ),
            orderStats = getOrderStats(),
            cacheStats = getCacheStats(),
            systemStats = getSystemStats()
        )
    }
    
    private fun getCacheSize(cache: Cache): Int {
        // This is cache implementation specific
        return when (cache) {
            is CaffeineCache -> cache.nativeCache.estimatedSize().toInt()
            else -> -1 // Unknown size
        }
    }
    
    private fun getCacheHitRatio(cache: Cache): Double {
        // This is cache implementation specific
        return when (cache) {
            is CaffeineCache -> {
                val stats = cache.nativeCache.stats()
                if (stats.requestCount() > 0) {
                    stats.hitRate()
                } else {
                    0.0
                }
            }
            else -> -1.0 // Unknown hit ratio
        }
    }
}

// Data classes for custom endpoint
data class ApplicationStats(
    val userStats: UserStats,
    val orderStats: OrderStats,
    val cacheStats: CacheStats,
    val systemStats: SystemStats
)

data class UserStats(
    val totalUsers: Long,
    val activeUsers: Long,
    val newUsersToday: Long,
    val newUsersThisWeek: Long
)

data class OrderStats(
    val totalOrders: Long,
    val ordersToday: Long,
    val ordersThisWeek: Long,
    val totalRevenue: BigDecimal
)

data class CacheStats(
    val availableCaches: Int,
    val cacheStatistics: Map<String, Map<String, Any>>
)

data class SystemStats(
    val heapMemoryUsage: MemoryUsage,
    val nonHeapMemoryUsage: MemoryUsage,
    val garbageCollection: List<GarbageCollectionStats>,
    val processorCount: Int,
    val systemLoadAverage: Double
)

data class MemoryUsage(
    val used: Long,
    val committed: Long,
    val max: Long
) {
    val usagePercentage: Double
        get() = if (max > 0) (used.toDouble() / max) * 100 else 0.0
}

data class GarbageCollectionStats(
    val name: String,
    val collectionCount: Long,
    val collectionTime: Long
)
```

### Custom Web Endpoint

For more complex operations that need HTTP method support:

```kotlin
@Component
@WebEndpoint(id = "application-management")
class ApplicationManagementWebEndpoint(
    private val cacheManager: CacheManager,
    private val taskScheduler: TaskScheduler
) {
    
    private val logger = LoggerFactory.getLogger(ApplicationManagementWebEndpoint::class.java)
    
    @ReadOperation
    fun getManagementInfo(): ResponseEntity<ManagementInfo> {
        return ResponseEntity.ok(ManagementInfo(
            cacheInfo = getCacheInfo(),
            scheduledTasks = getScheduledTasksInfo(),
            maintenanceMode = isMaintenanceModeEnabled()
        ))
    }
    
    @WriteOperation
    fun clearCache(@Selector cacheName: String?): ResponseEntity<Map<String, Any>> {
        return try {
            when (cacheName) {
                null -> {
                    // Clear all caches
                    cacheManager.cacheNames.forEach { name ->
                        cacheManager.getCache(name)?.clear()
                    }
                    ResponseEntity.ok(mapOf(
                        "status" to "success",
                        "message" to "All caches cleared",
                        "timestamp" to Instant.now()
                    ))
                }
                else -> {
                    val cache = cacheManager.getCache(cacheName)
                    if (cache != null) {
                        cache.clear()
                        ResponseEntity.ok(mapOf(
                            "status" to "success",
                            "message" to "Cache '$cacheName' cleared",
                            "timestamp" to Instant.now()
                        ))
                    } else {
                        ResponseEntity.badRequest().body(mapOf(
                            "status" to "error",
                            "message" to "Cache '$cacheName' not found"
                        ))
                    }
                }
            }
        } catch (ex: Exception) {
            logger.error("Failed to clear cache", ex)
            ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(mapOf(
                "status" to "error",
                "message" to "Failed to clear cache: ${ex.message}"
            ))
        }
    }
    
    @WriteOperation
    fun enableMaintenanceMode(): ResponseEntity<Map<String, Any>> {
        // Implementation to enable maintenance mode
        System.setProperty("app.maintenance.enabled", "true")
        
        return ResponseEntity.ok(mapOf(
            "status" to "success",
            "message" to "Maintenance mode enabled",
            "timestamp" to Instant.now()
        ))
    }
    
    @DeleteOperation
    fun disableMaintenanceMode(): ResponseEntity<Map<String, Any>> {
        // Implementation to disable maintenance mode
        System.setProperty("app.maintenance.enabled", "false")
        
        return ResponseEntity.ok(mapOf(
            "status" to "success",
            "message" to "Maintenance mode disabled",
            "timestamp" to Instant.now()
        ))
    }
    
    private fun getCacheInfo(): Map<String, Any> {
        return mapOf(
            "available_caches" to cacheManager.cacheNames.toList(),
            "total_caches" to cacheManager.cacheNames.size
        )
    }
    
    private fun getScheduledTasksInfo(): Map<String, Any> {
        // This is a simplified implementation
        // In practice, you'd want to track your scheduled tasks
        return mapOf(
            "scheduler_type" to taskScheduler.javaClass.simpleName,
            "status" to "running"
        )
    }
    
    private fun isMaintenanceModeEnabled(): Boolean {
        return System.getProperty("app.maintenance.enabled", "false").toBoolean()
    }
}

data class ManagementInfo(
    val cacheInfo: Map<String, Any>,
    val scheduledTasks: Map<String, Any>,
    val maintenanceMode: Boolean
)
```

### Custom Health Indicators with Groups

Create health indicator groups for different purposes:

```kotlin
@Configuration
class HealthConfiguration {
    
    @Bean
    fun healthContributorRegistry(): HealthContributorRegistry {
        val registry = DefaultHealthContributorRegistry()
        
        // Group for infrastructure health
        registry.registerContributor("infrastructure", CompositeHealthContributor.fromMap(
            mapOf(
                "database" to DatabaseHealthIndicator(dataSource),
                "redis" to RedisHealthIndicator(redisTemplate),
                "messaging" to RabbitMQHealthIndicator(rabbitTemplate)
            )
        ))
        
        // Group for external services
        registry.registerContributor("external", CompositeHealthContributor.fromMap(
            mapOf(
                "payment-service" to ExternalServiceHealthIndicator("payment"),
                "notification-service" to ExternalServiceHealthIndicator("notification")
            )
        ))
        
        return registry
    }
}

// Configuration for health groups
# application.yml
management:
  endpoint:
    health:
      group:
        liveness:
          include: livenessState,database
          show-details: always
        readiness:
          include: readinessState,redis,external
          show-details: when-authorized
        infrastructure:
          include: database,redis,messaging
          show-details: always
```

### Monitoring and Alerting Integration

Integration with monitoring systems:

```kotlin
@Component
class PrometheusMetricsCustomizer : MeterRegistryCustomizer<PrometheusMeterRegistry> {
    
    override fun customize(registry: PrometheusMeterRegistry) {
        registry.config()
            .meterFilter(MeterFilter.deny { id ->
                // Filter out sensitive metrics
                val name = id.name
                name.contains("password") || 
                name.contains("secret") || 
                name.contains("key")
            })
            .meterFilter(MeterFilter.denyNameStartsWith("jvm.threads.states"))
            .commonTags(
                "service", "kotlin-spring-boot-app",
                "version", getApplicationVersion()
            )
    }
    
    private fun getApplicationVersion(): String {
        return javaClass.`package`?.implementationVersion ?: "unknown"
    }
}

// Custom metrics for business KPIs
@Component
class BusinessMetrics(
    private val meterRegistry: MeterRegistry
) {
    
    private val orderTotalGauge = Gauge.builder("business.order.total")
        .description("Total order value")
        .register(meterRegistry, this) { getTotalOrderValue() }
    
    private val activeUserGauge = Gauge.builder("business.users.active")
        .description("Number of active users")
        .register(meterRegistry, this) { getActiveUserCount() }
    
    private fun getTotalOrderValue(): Double {
        // Implementation to get total order value
        return 0.0 // Placeholder
    }
    
    private fun getActiveUserCount(): Double {
        // Implementation to get active user count
        return 0.0 // Placeholder
    }
}

// Alert configuration (for integration with alerting systems)
@ConfigurationProperties(prefix = "app.monitoring.alerts")
data class AlertingProperties(
    val enabled: Boolean = true,
    val thresholds: AlertThresholds = AlertThresholds()
) {
    data class AlertThresholds(
        val errorRate: Double = 0.05, // 5%
        val responseTime: Duration = Duration.ofSeconds(2),
        val memoryUsage: Double = 0.8, // 80%
        val diskUsage: Double = 0.9 // 90%
    )
}

@Component
@ConditionalOnProperty(value = ["app.monitoring.alerts.enabled"], havingValue = "true")
class AlertingService(
    private val alertingProperties: AlertingProperties,
    private val meterRegistry: MeterRegistry
) {
    
    private val logger = LoggerFactory.getLogger(AlertingService::class.java)
    
    @EventListener
    fun handleMetricEvent(event: MeterRegistryEvent) {
        // Check thresholds and send alerts
        checkErrorRateThreshold()
        checkMemoryUsageThreshold()
    }
    
    private fun checkErrorRateThreshold() {
        val errorRate = getErrorRate()
        if (errorRate > alertingProperties.thresholds.errorRate) {
            sendAlert("High error rate detected: ${errorRate * 100}%")
        }
    }
    
    private fun checkMemoryUsageThreshold() {
        val memoryUsage = getMemoryUsagePercentage()
        if (memoryUsage > alertingProperties.thresholds.memoryUsage) {
            sendAlert("High memory usage detected: ${memoryUsage * 100}%")
        }
    }
    
    private fun getErrorRate(): Double {
        // Calculate error rate from metrics
        return 0.0 // Placeholder
    }
    
    private fun getMemoryUsagePercentage(): Double {
        // Calculate memory usage percentage
        return 0.0 // Placeholder
    }
    
    private fun sendAlert(message: String) {
        logger.warn("ALERT: {}", message)
        // Implementation to send alert (email, Slack, PagerDuty, etc.)
    }
}
```

## 11.4 Summary

Spring Boot Actuator transforms your Kotlin application into a fully observable and manageable system. Throughout this chapter, we've explored how to effectively leverage Actuator's capabilities while maintaining security and performance.

**Dependency and Setup** forms the foundation of effective monitoring. We covered:
- Proper dependency configuration with security considerations
- Production-ready security setup with role-based access control
- IP filtering and SSL configuration for sensitive environments
- Environment-specific endpoint exposure strategies

**Built-in Endpoints** provide comprehensive insights into your application:
- Health endpoints with custom health indicators for databases, external services, and application-specific concerns
- Metrics endpoints with custom business metrics and performance monitoring
- Info endpoints with build information, Git details, and application metadata
- Environment endpoints for configuration inspection and troubleshooting

**Customizing Actuator** enables application-specific monitoring:
- Custom endpoints for domain-specific statistics and management operations
- Health indicator groups for different monitoring scenarios (liveness, readiness, infrastructure)
- Integration with monitoring systems like Prometheus and Grafana
- Custom metrics and alerting based on business KPIs

The key benefits of our comprehensive Actuator implementation include:

1. **Observability**: Complete visibility into application health, performance, and behavior
2. **Operational Excellence**: Tools for managing applications in production environments
3. **Troubleshooting**: Rich diagnostic information for debugging and problem resolution
4. **Monitoring Integration**: Seamless integration with modern monitoring stacks
5. **Security**: Proper access controls and sensitive data protection

**Best practices** we've established:

- Always secure Actuator endpoints in production environments
- Use separate ports for management endpoints when possible
- Implement custom health checks for critical dependencies
- Create business-specific metrics and KPIs
- Filter sensitive information from exposed endpoints
- Integrate with centralized monitoring and alerting systems

**Kotlin-specific advantages** we've leveraged:

- Data classes for clean metric and health check models
- Extension functions for enhanced API usability
- Null safety for robust error handling
- Sealed classes for type-safe status representations

The monitoring and management capabilities provided by Actuator are essential for running Kotlin Spring Boot applications in production. The patterns and configurations demonstrated in this chapter provide a solid foundation for building observable, manageable applications that can be effectively monitored and maintained at scale.

In the next chapter, we'll explore server-to-server communication patterns, including how to monitor and instrument external service calls using the Actuator metrics we've established. The monitoring foundation we've built here will be crucial for tracking the health and performance of distributed system interactions.

The investment in comprehensive Actuator configuration pays dividends in operational excellence, system reliability, and developer productivity. Well-instrumented applications are easier to debug, monitor, and maintain, leading to better user experiences and more robust systems.