---
layout: default
parent: Spring Boot with Kotlin (2025)
nav_exclude: true
---
# Chapter 12: Server-to-Server Communication
- TOC
{:toc}

Modern applications rarely operate in isolation. They need to communicate with external APIs, microservices, payment processors, notification services, and countless other systems. In this chapter, we'll explore the various approaches Spring Boot provides for server-to-server communication and how to implement them effectively in Kotlin.

We'll examine three primary approaches: the traditional RestTemplate, the reactive WebClient with Kotlin coroutines support, and the newer RestClient. Each approach has its strengths and appropriate use cases. We'll also cover essential patterns like retry logic, circuit breakers, authentication, and monitoring that are crucial for building resilient distributed systems.

Kotlin brings unique advantages to HTTP client development with its coroutines support, extension functions, and expressive DSL capabilities. We'll leverage these features throughout our implementations to create clean, maintainable, and efficient communication code.

## 12.1 RestTemplate

While RestTemplate is now in maintenance mode, it's still widely used in many applications and remains a solid choice for synchronous HTTP communication. Let's explore how to use it effectively with Kotlin.

### Basic RestTemplate Configuration

```kotlin
@Configuration
class RestTemplateConfiguration {

    @Bean
    @Primary
    fun restTemplate(restTemplateBuilder: RestTemplateBuilder): RestTemplate {
        return restTemplateBuilder
            .setConnectTimeout(Duration.ofSeconds(10))
            .setReadTimeout(Duration.ofSeconds(30))
            .additionalMessageConverters(
                MappingJackson2HttpMessageConverter().apply {
                    objectMapper = jacksonObjectMapper().apply {
                        configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
                        configure(DeserializationFeature.ACCEPT_EMPTY_STRING_AS_NULL_OBJECT, true)
                        registerModule(JavaTimeModule())
                        registerModule(KotlinModule.Builder().build())
                    }
                }
            )
            .additionalInterceptors(
                LoggingInterceptor(),
                RetryInterceptor(),
                AuthenticationInterceptor()
            )
            .errorHandler(CustomErrorHandler())
            .build()
    }

    @Bean
    @Qualifier("externalApiRestTemplate")
    fun externalApiRestTemplate(restTemplateBuilder: RestTemplateBuilder): RestTemplate {
        return restTemplateBuilder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(15))
            .basicAuthentication("api-user", "api-password")
            .additionalInterceptors(ExternalApiInterceptor())
            .build()
    }

    @Bean
    @Qualifier("paymentServiceRestTemplate")
    fun paymentServiceRestTemplate(
        restTemplateBuilder: RestTemplateBuilder,
        @Value("\${app.payment-service.api-key}") apiKey: String
    ): RestTemplate {
        return restTemplateBuilder
            .setConnectTimeout(Duration.ofSeconds(10))
            .setReadTimeout(Duration.ofSeconds(60)) // Payment operations might take longer
            .additionalInterceptors(ApiKeyInterceptor(apiKey))
            .errorHandler(PaymentServiceErrorHandler())
            .build()
    }
}

// Custom interceptors
class LoggingInterceptor : ClientHttpRequestInterceptor {

    private val logger = LoggerFactory.getLogger(LoggingInterceptor::class.java)

    override fun intercept(
        request: HttpRequest,
        body: ByteArray,
        execution: ClientHttpRequestExecution
    ): ClientHttpResponse {

        val startTime = System.currentTimeMillis()

        logger.debug("HTTP {} {}", request.method, request.uri)
        if (body.isNotEmpty() && logger.isTraceEnabled) {
            logger.trace("Request body: {}", String(body, StandardCharsets.UTF_8))
        }

        val response = execution.execute(request, body)
        val duration = System.currentTimeMillis() - startTime

        logger.debug(
            "HTTP {} {} -> {} in {}ms",
            request.method,
            request.uri,
            response.statusCode,
            duration
        )

        return response
    }
}

class RetryInterceptor : ClientHttpRequestInterceptor {

    private val logger = LoggerFactory.getLogger(RetryInterceptor::class.java)
    private val maxRetries = 3
    private val retryDelay = Duration.ofSeconds(1)

    override fun intercept(
        request: HttpRequest,
        body: ByteArray,
        execution: ClientHttpRequestExecution
    ): ClientHttpResponse {

        var lastException: Exception? = null

        repeat(maxRetries) { attempt ->
            try {
                val response = execution.execute(request, body)

                // Retry on server errors (5xx) but not client errors (4xx)
                if (response.statusCode.is5xxServerError && attempt < maxRetries - 1) {
                    logger.warn(
                        "Attempt {} failed with status {}, retrying in {}ms",
                        attempt + 1,
                        response.statusCode,
                        retryDelay.toMillis()
                    )
                    Thread.sleep(retryDelay.toMillis())
                    return@repeat
                }

                return response

            } catch (ex: ResourceAccessException) {
                lastException = ex

                if (attempt < maxRetries - 1) {
                    logger.warn("Attempt {} failed with network error, retrying in {}ms", attempt + 1, retryDelay.toMillis(), ex)
                    Thread.sleep(retryDelay.toMillis())
                } else {
                    logger.error("All {} attempts failed", maxRetries, ex)
                }
            }
        }

        throw lastException ?: RuntimeException("All retry attempts failed")
    }
}

class ApiKeyInterceptor(private val apiKey: String) : ClientHttpRequestInterceptor {

    override fun intercept(
        request: HttpRequest,
        body: ByteArray,
        execution: ClientHttpRequestExecution
    ): ClientHttpResponse {

        request.headers.add("X-API-Key", apiKey)
        return execution.execute(request, body)
    }
}

// Custom error handler
class CustomErrorHandler : ResponseErrorHandler {

    private val logger = LoggerFactory.getLogger(CustomErrorHandler::class.java)

    override fun hasError(response: ClientHttpResponse): Boolean {
        return response.statusCode.isError
    }

    override fun handleError(response: ClientHttpResponse) {
        val statusCode = response.statusCode
        val responseBody = response.body.bufferedReader().use { it.readText() }

        logger.error("HTTP error: {} - {}", statusCode, responseBody)

        when {
            statusCode.is4xxClientError -> {
                when (statusCode) {
                    HttpStatus.NOT_FOUND -> throw ResourceNotFoundException("Resource not found")
                    HttpStatus.UNAUTHORIZED -> throw AuthenticationException("Authentication failed")
                    HttpStatus.FORBIDDEN -> throw AuthorizationException("Access denied")
                    HttpStatus.BAD_REQUEST -> throw BadRequestException("Bad request: $responseBody")
                    else -> throw ClientException("Client error: ${statusCode.value()}")
                }
            }
            statusCode.is5xxServerError -> {
                throw ServerException("Server error: ${statusCode.value()}")
            }
            else -> {
                throw HttpClientException("HTTP error: ${statusCode.value()}")
            }
        }
    }
}
```

### Service Implementation with RestTemplate

```kotlin
@Service
class UserApiService(
    private val restTemplate: RestTemplate,
    private val meterRegistry: MeterRegistry,
    @Value("\${app.external.user-service.base-url}") private val baseUrl: String
) {

    private val logger = LoggerFactory.getLogger(UserApiService::class.java)

    private val userApiTimer = Timer.builder("user.api.request.duration")
        .description("Time taken for user API requests")
        .register(meterRegistry)

    private val userApiCounter = Counter.builder("user.api.request.total")
        .description("Total user API requests")
        .register(meterRegistry)

    /**
     * Get user by ID with comprehensive error handling
     */
    fun getUser(userId: Long): UserDto? {
        return userApiTimer.recordCallable {
            try {
                val url = "$baseUrl/api/users/{userId}"
                val response = restTemplate.getForEntity(url, UserDto::class.java, userId)

                userApiCounter.increment(Tags.of(
                    "operation", "getUser",
                    "status", "success"
                ))

                logger.debug("Successfully retrieved user {}", userId)
                response.body

            } catch (ex: HttpClientErrorException.NotFound) {
                userApiCounter.increment(Tags.of(
                    "operation", "getUser",
                    "status", "not_found"
                ))

                logger.debug("User {} not found", userId)
                null

            } catch (ex: Exception) {
                userApiCounter.increment(Tags.of(
                    "operation", "getUser",
                    "status", "error",
                    "error_type", ex.javaClass.simpleName
                ))

                logger.error("Failed to retrieve user {}", userId, ex)
                throw ExternalServiceException("User service", "getUser", ex.message ?: "Unknown error", ex)
            }
        }
    }

    /**
     * Create a new user
     */
    fun createUser(createUserRequest: CreateUserRequest): UserDto {
        return userApiTimer.recordCallable {
            try {
                val url = "$baseUrl/api/users"
                val response = restTemplate.postForEntity(url, createUserRequest, UserDto::class.java)

                userApiCounter.increment(Tags.of(
                    "operation", "createUser",
                    "status", "success"
                ))

                logger.info("Successfully created user: {}", response.body?.id)
                response.body ?: throw IllegalStateException("Created user response body is null")

            } catch (ex: HttpClientErrorException.BadRequest) {
                userApiCounter.increment(Tags.of(
                    "operation", "createUser",
                    "status", "bad_request"
                ))

                val errorBody = ex.responseBodyAsString
                logger.warn("Bad request when creating user: {}", errorBody)
                throw ValidationException("User creation failed", parseValidationErrors(errorBody))

            } catch (ex: Exception) {
                userApiCounter.increment(Tags.of(
                    "operation", "createUser",
                    "status", "error",
                    "error_type", ex.javaClass.simpleName
                ))

                logger.error("Failed to create user", ex)
                throw ExternalServiceException("User service", "createUser", ex.message ?: "Unknown error", ex)
            }
        }!!
    }

    /**
     * Update an existing user
     */
    fun updateUser(userId: Long, updateUserRequest: UpdateUserRequest): UserDto {
        return userApiTimer.recordCallable {
            try {
                val url = "$baseUrl/api/users/{userId}"
                val requestEntity = HttpEntity(updateUserRequest, createHeaders())

                val response = restTemplate.exchange(
                    url,
                    HttpMethod.PUT,
                    requestEntity,
                    UserDto::class.java,
                    userId
                )

                userApiCounter.increment(Tags.of(
                    "operation", "updateUser",
                    "status", "success"
                ))

                logger.info("Successfully updated user {}", userId)
                response.body ?: throw IllegalStateException("Updated user response body is null")

            } catch (ex: HttpClientErrorException.NotFound) {
                userApiCounter.increment(Tags.of(
                    "operation", "updateUser",
                    "status", "not_found"
                ))

                logger.debug("User {} not found for update", userId)
                throw ResourceNotFoundException("User not found with ID: $userId")

            } catch (ex: Exception) {
                userApiCounter.increment(Tags.of(
                    "operation", "updateUser",
                    "status", "error",
                    "error_type", ex.javaClass.simpleName
                ))

                logger.error("Failed to update user {}", userId, ex)
                throw ExternalServiceException("User service", "updateUser", ex.message ?: "Unknown error", ex)
            }
        }!!
    }

    /**
     * Get users with pagination
     */
    fun getUsers(page: Int = 0, size: Int = 20, sort: String? = null): PagedResponse<UserDto> {
        return userApiTimer.recordCallable {
            try {
                val uriBuilder = UriComponentsBuilder.fromHttpUrl("$baseUrl/api/users")
                    .queryParam("page", page)
                    .queryParam("size", size)

                sort?.let { uriBuilder.queryParam("sort", it) }

                val response = restTemplate.getForEntity(
                    uriBuilder.toUriString(),
                    PagedUserResponse::class.java
                )

                userApiCounter.increment(Tags.of(
                    "operation", "getUsers",
                    "status", "success"
                ))

                logger.debug("Successfully retrieved {} users", response.body?.content?.size ?: 0)

                val pagedResponse = response.body ?: throw IllegalStateException("Paged response body is null")
                PagedResponse(
                    content = pagedResponse.content,
                    totalElements = pagedResponse.totalElements,
                    totalPages = pagedResponse.totalPages,
                    number = pagedResponse.number,
                    size = pagedResponse.size,
                    first = pagedResponse.first,
                    last = pagedResponse.last
                )

            } catch (ex: Exception) {
                userApiCounter.increment(Tags.of(
                    "operation", "getUsers",
                    "status", "error",
                    "error_type", ex.javaClass.simpleName
                ))

                logger.error("Failed to retrieve users", ex)
                throw ExternalServiceException("User service", "getUsers", ex.message ?: "Unknown error", ex)
            }
        }!!
    }

    /**
     * Delete a user
     */
    fun deleteUser(userId: Long): Boolean {
        return userApiTimer.recordCallable {
            try {
                val url = "$baseUrl/api/users/{userId}"
                restTemplate.delete(url, userId)

                userApiCounter.increment(Tags.of(
                    "operation", "deleteUser",
                    "status", "success"
                ))

                logger.info("Successfully deleted user {}", userId)
                true

            } catch (ex: HttpClientErrorException.NotFound) {
                userApiCounter.increment(Tags.of(
                    "operation", "deleteUser",
                    "status", "not_found"
                ))

                logger.debug("User {} not found for deletion", userId)
                false

            } catch (ex: Exception) {
                userApiCounter.increment(Tags.of(
                    "operation", "deleteUser",
                    "status", "error",
                    "error_type", ex.javaClass.simpleName
                ))

                logger.error("Failed to delete user {}", userId, ex)
                throw ExternalServiceException("User service", "deleteUser", ex.message ?: "Unknown error", ex)
            }
        }!!
    }

    private fun createHeaders(): HttpHeaders {
        val headers = HttpHeaders()
        headers.contentType = MediaType.APPLICATION_JSON
        return headers
    }

    private fun parseValidationErrors(errorBody: String): List<ValidationError> {
        return try {
            val objectMapper = jacksonObjectMapper()
            val errorResponse = objectMapper.readValue(errorBody, ErrorResponse::class.java)
            errorResponse.details ?: emptyList()
        } catch (ex: Exception) {
            listOf(ValidationError(
                field = "general",
                message = errorBody,
                code = "EXTERNAL_VALIDATION_ERROR"
            ))
        }
    }
}

// Data classes for API communication
data class UserDto(
    val id: Long,
    val username: String,
    val email: String,
    val firstName: String,
    val lastName: String,
    val active: Boolean,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
)

data class CreateUserRequest(
    val username: String,
    val email: String,
    val firstName: String,
    val lastName: String,
    val password: String
)

data class UpdateUserRequest(
    val username: String? = null,
    val email: String? = null,
    val firstName: String? = null,
    val lastName: String? = null
)

data class PagedUserResponse(
    val content: List<UserDto>,
    val totalElements: Long,
    val totalPages: Int,
    val number: Int,
    val size: Int,
    val first: Boolean,
    val last: Boolean
)

data class PagedResponse<T>(
    val content: List<T>,
    val totalElements: Long,
    val totalPages: Int,
    val number: Int,
    val size: Int,
    val first: Boolean,
    val last: Boolean
)

// Custom exceptions
class ExternalServiceException(
    val service: String,
    val operation: String,
    message: String,
    cause: Throwable? = null
) : RuntimeException("External service '$service' failed during '$operation': $message", cause)

class ResourceNotFoundException(message: String) : RuntimeException(message)
class AuthenticationException(message: String) : RuntimeException(message)
class AuthorizationException(message: String) : RuntimeException(message)
class BadRequestException(message: String) : RuntimeException(message)
class ClientException(message: String) : RuntimeException(message)
class ServerException(message: String) : RuntimeException(message)
class HttpClientException(message: String) : RuntimeException(message)
```

### Advanced RestTemplate Patterns

```kotlin
// Generic API client with RestTemplate
abstract class BaseApiClient(
    protected val restTemplate: RestTemplate,
    protected val baseUrl: String,
    protected val meterRegistry: MeterRegistry
) {

    protected val logger = LoggerFactory.getLogger(this::class.java)

    protected inline fun <reified T> get(
        path: String,
        vararg uriVariables: Any,
        parameters: Map<String, Any> = emptyMap()
    ): T? {
        return executeWithMetrics("GET", path) {
            val uriBuilder = UriComponentsBuilder.fromHttpUrl("$baseUrl$path")
            parameters.forEach { (key, value) -> uriBuilder.queryParam(key, value) }

            val response = restTemplate.getForEntity(
                uriBuilder.toUriString(),
                T::class.java,
                *uriVariables
            )
            response.body
        }
    }

    protected inline fun <reified T> post(
        path: String,
        body: Any,
        vararg uriVariables: Any
    ): T? {
        return executeWithMetrics("POST", path) {
            val response = restTemplate.postForEntity(
                "$baseUrl$path",
                body,
                T::class.java,
                *uriVariables
            )
            response.body
        }
    }

    protected inline fun <reified T> put(
        path: String,
        body: Any,
        vararg uriVariables: Any
    ): T? {
        return executeWithMetrics("PUT", path) {
            val requestEntity = HttpEntity(body, createHeaders())
            val response = restTemplate.exchange(
                "$baseUrl$path",
                HttpMethod.PUT,
                requestEntity,
                T::class.java,
                *uriVariables
            )
            response.body
        }
    }

    protected fun delete(path: String, vararg uriVariables: Any): Boolean {
        return executeWithMetrics("DELETE", path) {
            restTemplate.delete("$baseUrl$path", *uriVariables)
            true
        } ?: false
    }

    private fun <T> executeWithMetrics(method: String, path: String, operation: () -> T): T? {
        val timer = Timer.builder("api.client.request.duration")
            .description("API client request duration")
            .tag("method", method)
            .tag("path", path)
            .tag("service", getServiceName())
            .register(meterRegistry)

        val counter = Counter.builder("api.client.request.total")
            .description("Total API client requests")
            .tag("method", method)
            .tag("path", path)
            .tag("service", getServiceName())
            .register(meterRegistry)

        return timer.recordCallable {
            try {
                val result = operation()
                counter.increment(Tags.of("status", "success"))
                result
            } catch (ex: Exception) {
                counter.increment(Tags.of(
                    "status", "error",
                    "error_type", ex.javaClass.simpleName
                ))
                throw ex
            }
        }
    }

    protected abstract fun getServiceName(): String

    protected fun createHeaders(): HttpHeaders {
        val headers = HttpHeaders()
        headers.contentType = MediaType.APPLICATION_JSON
        return headers
    }
}

// Specific implementation
@Service
class PaymentApiClient(
    @Qualifier("paymentServiceRestTemplate") restTemplate: RestTemplate,
    @Value("\${app.payment-service.base-url}") baseUrl: String,
    meterRegistry: MeterRegistry
) : BaseApiClient(restTemplate, baseUrl, meterRegistry) {

    override fun getServiceName(): String = "payment-service"

    fun processPayment(paymentRequest: PaymentRequest): PaymentResponse? {
        return post("/api/payments", paymentRequest)
    }

    fun getPayment(paymentId: String): PaymentResponse? {
        return get("/api/payments/{paymentId}", paymentId)
    }

    fun refundPayment(paymentId: String, refundRequest: RefundRequest): RefundResponse? {
        return post("/api/payments/{paymentId}/refund", refundRequest, paymentId)
    }

    fun getPaymentsByUser(userId: Long, page: Int = 0, size: Int = 20): PagedResponse<PaymentResponse>? {
        return get(
            "/api/payments/user/{userId}",
            userId,
            parameters = mapOf("page" to page, "size" to size)
        )
    }
}

// Payment-related data classes
data class PaymentRequest(
    val userId: Long,
    val amount: BigDecimal,
    val currency: String,
    val paymentMethod: String,
    val description: String? = null
)

data class PaymentResponse(
    val id: String,
    val userId: Long,
    val amount: BigDecimal,
    val currency: String,
    val status: PaymentStatus,
    val paymentMethod: String,
    val description: String?,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
)

data class RefundRequest(
    val amount: BigDecimal,
    val reason: String
)

data class RefundResponse(
    val id: String,
    val paymentId: String,
    val amount: BigDecimal,
    val status: RefundStatus,
    val reason: String,
    val createdAt: LocalDateTime
)

enum class PaymentStatus {
    PENDING, COMPLETED, FAILED, CANCELLED
}

enum class RefundStatus {
    PENDING, COMPLETED, FAILED
}
```

## 12.2 WebClient (with suspend functions)

WebClient is the modern, reactive HTTP client in Spring Boot. When combined with Kotlin coroutines, it provides powerful asynchronous communication capabilities.

### WebClient Configuration

```kotlin
@Configuration
class WebClientConfiguration {

    @Bean
    @Primary
    fun webClient(webClientBuilder: WebClient.Builder): WebClient {
        return webClientBuilder
            .codecs { configurer ->
                configurer.defaultCodecs().maxInMemorySize(1024 * 1024) // 1MB
                configurer.defaultCodecs().jackson2JsonDecoder(
                    Jackson2JsonDecoder(jacksonObjectMapper().apply {
                        configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
                        registerModule(JavaTimeModule())
                        registerModule(KotlinModule.Builder().build())
                    })
                )
            }
            .clientConnector(
                ReactorClientHttpConnector(
                    HttpClient.create()
                        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000)
                        .responseTimeout(Duration.ofSeconds(30))
                        .doOnConnected { connection ->
                            connection
                                .addHandlerLast(ReadTimeoutHandler(30))
                                .addHandlerLast(WriteTimeoutHandler(30))
                        }
                )
            )
            .filter(logRequest())
            .filter(logResponse())
            .filter(retryFilter())
            .build()
    }

    @Bean
    @Qualifier("userServiceWebClient")
    fun userServiceWebClient(
        webClientBuilder: WebClient.Builder,
        @Value("\${app.external.user-service.base-url}") baseUrl: String
    ): WebClient {
        return webClientBuilder
            .baseUrl(baseUrl)
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .filter(authenticationFilter())
            .build()
    }

    @Bean
    @Qualifier("paymentServiceWebClient")
    fun paymentServiceWebClient(
        webClientBuilder: WebClient.Builder,
        @Value("\${app.payment-service.base-url}") baseUrl: String,
        @Value("\${app.payment-service.api-key}") apiKey: String
    ): WebClient {
        return webClientBuilder
            .baseUrl(baseUrl)
            .defaultHeader("X-API-Key", apiKey)
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .clientConnector(
                ReactorClientHttpConnector(
                    HttpClient.create()
                        .responseTimeout(Duration.ofMinutes(2)) // Longer timeout for payment operations
                )
            )
            .build()
    }

    private fun logRequest(): ExchangeFilterFunction {
        return ExchangeFilterFunction.ofRequestProcessor { request ->
            LoggerFactory.getLogger("WebClient.Request")
                .debug("HTTP {} {}", request.method(), request.url())
            Mono.just(request)
        }
    }

    private fun logResponse(): ExchangeFilterFunction {
        return ExchangeFilterFunction.ofResponseProcessor { response ->
            LoggerFactory.getLogger("WebClient.Response")
                .debug("HTTP Response: {}", response.statusCode())
            Mono.just(response)
        }
    }

    private fun retryFilter(): ExchangeFilterFunction {
        return ExchangeFilterFunction.ofRequestProcessor { request ->
            Mono.just(request)
        }.and(ExchangeFilterFunction.ofResponseProcessor { response ->
            if (response.statusCode().is5xxServerError) {
                Mono.error(ServerException("Server error: ${response.statusCode()}"))
            } else {
                Mono.just(response)
            }
        })
    }

    private fun authenticationFilter(): ExchangeFilterFunction {
        return ExchangeFilterFunction.ofRequestProcessor { request ->
            // Add authentication logic here
            // For example, add JWT token to headers
            val modifiedRequest = ClientRequest.from(request)
                .header(HttpHeaders.AUTHORIZATION, "Bearer ${getAuthToken()}")
                .build()
            Mono.just(modifiedRequest)
        }
    }

    private fun getAuthToken(): String {
        // Implementation to get auth token
        return "sample-token"
    }
}
```

### Reactive Service with Coroutines

```kotlin
@Service
class ReactiveUserService(
    @Qualifier("userServiceWebClient") private val webClient: WebClient,
    private val meterRegistry: MeterRegistry
) {

    private val logger = LoggerFactory.getLogger(ReactiveUserService::class.java)

    /**
     * Get user by ID using coroutines
     */
    suspend fun getUser(userId: Long): UserDto? {
        return withContext(Dispatchers.IO) {
            try {
                recordMetric("getUser", "success") {
                    webClient
                        .get()
                        .uri("/api/users/{userId}", userId)
                        .retrieve()
                        .onStatus(HttpStatus::is4xxClientError) { response ->
                            when (response.statusCode()) {
                                HttpStatus.NOT_FOUND -> Mono.empty()
                                else -> response.createException().flatMap { Mono.error(it) }
                            }
                        }
                        .awaitBodyOrNull<UserDto>()
                }
            } catch (ex: Exception) {
                recordMetric("getUser", "error", ex.javaClass.simpleName)
                logger.error("Failed to retrieve user {}", userId, ex)
                throw ExternalServiceException("User service", "getUser", ex.message ?: "Unknown error", ex)
            }
        }
    }

    /**
     * Get multiple users concurrently
     */
    suspend fun getUsers(userIds: List<Long>): List<UserDto> {
        return withContext(Dispatchers.IO) {
            try {
                userIds.map { userId ->
                    async {
                        webClient
                            .get()
                            .uri("/api/users/{userId}", userId)
                            .retrieve()
                            .onStatus(HttpStatus::is4xxClientError) { response ->
                                if (response.statusCode() == HttpStatus.NOT_FOUND) {
                                    Mono.empty()
                                } else {
                                    response.createException().flatMap { Mono.error(it) }
                                }
                            }
                            .awaitBodyOrNull<UserDto>()
                    }
                }.awaitAll().filterNotNull()
            } catch (ex: Exception) {
                logger.error("Failed to retrieve users {}", userIds, ex)
                throw ExternalServiceException("User service", "getUsers", ex.message ?: "Unknown error", ex)
            }
        }
    }

    /**
     * Create user with comprehensive error handling
     */
    suspend fun createUser(createUserRequest: CreateUserRequest): UserDto {
        return withContext(Dispatchers.IO) {
            try {
                recordMetric("createUser", "success") {
                    webClient
                        .post()
                        .uri("/api/users")
                        .bodyValue(createUserRequest)
                        .retrieve()
                        .onStatus(HttpStatus::is4xxClientError) { response ->
                            response.bodyToMono<ErrorResponse>()
                                .flatMap { errorResponse ->
                                    Mono.error(ValidationException(
                                        "User creation failed",
                                        errorResponse.details ?: emptyList()
                                    ))
                                }
                        }
                        .awaitBody<UserDto>()
                }
            } catch (ex: ValidationException) {
                recordMetric("createUser", "validation_error")
                throw ex
            } catch (ex: Exception) {
                recordMetric("createUser", "error", ex.javaClass.simpleName)
                logger.error("Failed to create user", ex)
                throw ExternalServiceException("User service", "createUser", ex.message ?: "Unknown error", ex)
            }
        }
    }

    /**
     * Update user with optimistic updates
     */
    suspend fun updateUser(userId: Long, updateUserRequest: UpdateUserRequest): UserDto {
        return withContext(Dispatchers.IO) {
            try {
                recordMetric("updateUser", "success") {
                    webClient
                        .put()
                        .uri("/api/users/{userId}", userId)
                        .bodyValue(updateUserRequest)
                        .retrieve()
                        .onStatus(HttpStatus::isError) { response ->
                            when (response.statusCode()) {
                                HttpStatus.NOT_FOUND -> Mono.error(ResourceNotFoundException("User not found with ID: $userId"))
                                HttpStatus.BAD_REQUEST -> response.bodyToMono<ErrorResponse>()
                                    .flatMap { errorResponse ->
                                        Mono.error(ValidationException(
                                            "User update failed",
                                            errorResponse.details ?: emptyList()
                                        ))
                                    }
                                else -> response.createException().flatMap { Mono.error(it) }
                            }
                        }
                        .awaitBody<UserDto>()
                }
            } catch (ex: ValidationException) {
                recordMetric("updateUser", "validation_error")
                throw ex
            } catch (ex: ResourceNotFoundException) {
                recordMetric("updateUser", "not_found")
                throw ex
            } catch (ex: Exception) {
                recordMetric("updateUser", "error", ex.javaClass.simpleName)
                logger.error("Failed to update user {}", userId, ex)
                throw ExternalServiceException("User service", "updateUser", ex.message ?: "Unknown error", ex)
            }
        }
    }

    /**
     * Get users with pagination using reactive streams
     */
    suspend fun getUsersPaginated(page: Int = 0, size: Int = 20, sort: String? = null): PagedResponse<UserDto> {
        return withContext(Dispatchers.IO) {
            try {
                recordMetric("getUsersPaginated", "success") {
                    webClient
                        .get()
                        .uri { uriBuilder ->
                            uriBuilder
                                .path("/api/users")
                                .queryParam("page", page)
                                .queryParam("size", size)
                                .apply { sort?.let { queryParam("sort", it) } }
                                .build()
                        }
                        .retrieve()
                        .awaitBody<PagedUserResponse>()
                        .let { response ->
                            PagedResponse(
                                content = response.content,
                                totalElements = response.totalElements,
                                totalPages = response.totalPages,
                                number = response.number,
                                size = response.size,
                                first = response.first,
                                last = response.last
                            )
                        }
                }
            } catch (ex: Exception) {
                recordMetric("getUsersPaginated", "error", ex.javaClass.simpleName)
                logger.error("Failed to retrieve paginated users", ex)
                throw ExternalServiceException("User service", "getUsersPaginated", ex.message ?: "Unknown error", ex)
            }
        }
    }

    /**
     * Delete user with confirmation
     */
    suspend fun deleteUser(userId: Long): Boolean {
        return withContext(Dispatchers.IO) {
            try {
                recordMetric("deleteUser", "success") {
                    webClient
                        .delete()
                        .uri("/api/users/{userId}", userId)
                        .retrieve()
                        .onStatus(HttpStatus::is4xxClientError) { response ->
                            if (response.statusCode() == HttpStatus.NOT_FOUND) {
                                Mono.empty()
                            } else {
                                response.createException().flatMap { Mono.error(it) }
                            }
                        }
                        .toBodilessEntity()
                        .awaitSingleOrNull()
                    true
                }
            } catch (ex: Exception) {
                recordMetric("deleteUser", "error", ex.javaClass.simpleName)
                logger.error("Failed to delete user {}", userId, ex)
                false
            }
        }
    }

    /**
     * Batch operations with flow control
     */
    suspend fun createUsersBatch(createUserRequests: List<CreateUserRequest>): BatchResult<UserDto> {
        return withContext(Dispatchers.IO) {
            val results = mutableListOf<UserDto>()
            val failures = mutableListOf<BatchFailure>()

            // Process in batches to avoid overwhelming the server
            createUserRequests.chunked(10).forEach { batch ->
                val batchResults = batch.mapIndexed { index, request ->
                    async {
                        try {
                            val user = createUser(request)
                            BatchSuccess(user)
                        } catch (ex: Exception) {
                            BatchError(index, ex)
                        }
                    }
                }.awaitAll()

                batchResults.forEach { result ->
                    when (result) {
                        is BatchSuccess -> results.add(result.data)
                        is BatchError -> failures.add(
                            BatchFailure(
                                index = result.index,
                                error = result.exception.message ?: "Unknown error",
                                originalRequest = batch[result.index]
                            )
                        )
                    }
                }

                // Small delay between batches to be nice to the external service
                delay(100)
            }

            BatchResult(
                successful = results,
                failed = failures,
                totalProcessed = createUserRequests.size
            )
        }
    }

    private suspend inline fun <T> recordMetric(
        operation: String,
        status: String,
        errorType: String? = null,
        block: () -> T
    ): T {
        val timer = Timer.builder("user.service.request.duration")
            .description("User service request duration")
            .tag("operation", operation)
            .register(meterRegistry)

        val counter = Counter.builder("user.service.request.total")
            .description("Total user service requests")
            .tag("operation", operation)
            .register(meterRegistry)

        val start = System.currentTimeMillis()
        return try {
            val result = block()
            val duration = System.currentTimeMillis() - start

            timer.record(duration, TimeUnit.MILLISECONDS)
            counter.increment(Tags.of("status", status))

            result
        } catch (ex: Exception) {
            val duration = System.currentTimeMillis() - start
            timer.record(duration, TimeUnit.MILLISECONDS)
            counter.increment(Tags.of(
                "status", status,
                *if (errorType != null) arrayOf("error_type", errorType) else emptyArray()
            ))
            throw ex
        }
    }
}

// Batch processing result classes
sealed class BatchOperationResult
data class BatchSuccess<T>(val data: T) : BatchOperationResult()
data class BatchError(val index: Int, val exception: Exception) : BatchOperationResult()

data class BatchResult<T>(
    val successful: List<T>,
    val failed: List<BatchFailure>,
    val totalProcessed: Int
) {
    val successCount: Int get() = successful.size
    val failureCount: Int get() = failed.size
    val successRate: Double get() = if (totalProcessed > 0) successCount.toDouble() / totalProcessed else 0.0
}

data class BatchFailure(
    val index: Int,
    val error: String,
    val originalRequest: Any
)
```

### Advanced WebClient Patterns with Circuit Breaker

```kotlin
@Component
class CircuitBreakerWebClientService(
    private val webClient: WebClient,
    private val meterRegistry: MeterRegistry
) {

    private val logger = LoggerFactory.getLogger(CircuitBreakerWebClientService::class.java)

    // Circuit breaker configuration
    private val circuitBreaker = CircuitBreaker.ofDefaults("external-api").apply {
        eventPublisher.onStateTransition { event ->
            logger.info("Circuit breaker state transition: {} -> {}", event.stateTransition.fromState, event.stateTransition.toState)
            meterRegistry.counter(
                "circuit.breaker.state.transition",
                "from", event.stateTransition.fromState.name,
                "to", event.stateTransition.toState.name
            ).increment()
        }
    }

    /**
     * Make HTTP request with circuit breaker protection
     */
    suspend fun <T> executeWithCircuitBreaker(
        operation: String,
        fallback: suspend () -> T,
        request: suspend () -> T
    ): T {
        return withContext(Dispatchers.IO) {
            try {
                // Decorate the operation with circuit breaker
                val decoratedSupplier = CircuitBreaker.decorateSupplier(circuitBreaker) {
                    runBlocking { request() }
                }

                decoratedSupplier.get()
            } catch (ex: CallNotPermittedException) {
                logger.warn("Circuit breaker is OPEN, executing fallback for operation: {}", operation)
                recordCircuitBreakerMetric(operation, "circuit_open")
                fallback()
            } catch (ex: Exception) {
                logger.error("Operation {} failed", operation, ex)
                recordCircuitBreakerMetric(operation, "error")
                fallback()
            }
        }
    }

    /**
     * Get user with circuit breaker and fallback
     */
    suspend fun getUserWithFallback(userId: Long): UserDto? {
        return executeWithCircuitBreaker(
            operation = "getUser",
            fallback = {
                logger.info("Using fallback for user {}", userId)
                createFallbackUser(userId)
            }
        ) {
            webClient
                .get()
                .uri("/api/users/{userId}", userId)
                .retrieve()
                .awaitBodyOrNull<UserDto>()
        }
    }

    /**
     * Create user with retry and circuit breaker
     */
    suspend fun createUserWithResilience(createUserRequest: CreateUserRequest): UserDto {
        return executeWithCircuitBreaker(
            operation = "createUser",
            fallback = {
                logger.error("Failed to create user after circuit breaker opened")
                throw ExternalServiceException(
                    "User service",
                    "createUser",
                    "Service temporarily unavailable"
                )
            }
        ) {
            // Retry logic with exponential backoff
            retry(
                times = 3,
                initialDelay = 1000,
                maxDelay = 5000,
                factor = 2.0
            ) {
                webClient
                    .post()
                    .uri("/api/users")
                    .bodyValue(createUserRequest)
                    .retrieve()
                    .awaitBody<UserDto>()
            }
        }
    }

    private fun createFallbackUser(userId: Long): UserDto {
        return UserDto(
            id = userId,
            username = "fallback-user-$userId",
            email = "fallback-$userId@example.com",
            firstName = "Fallback",
            lastName = "User",
            active = false,
            createdAt = LocalDateTime.now(),
            updatedAt = LocalDateTime.now()
        )
    }

    private fun recordCircuitBreakerMetric(operation: String, status: String) {
        meterRegistry.counter(
            "circuit.breaker.operation",
            "operation", operation,
            "status", status
        ).increment()
    }
}

// Retry utility function
suspend fun <T> retry(
    times: Int,
    initialDelay: Long = 100,
    maxDelay: Long = 1000,
    factor: Double = 2.0,
    block: suspend () -> T
): T {
    var currentDelay = initialDelay
    repeat(times - 1) { attempt ->
        try {
            return block()
        } catch (e: Exception) {
            LoggerFactory.getLogger("RetryUtil")
                .warn("Attempt {} failed, retrying in {}ms", attempt + 1, currentDelay, e)
        }
        delay(currentDelay)
        currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelay)
    }
    return block() // Last attempt without catching exception
}
```

## 12.3 RestClient

RestClient is the newest addition to Spring's HTTP client family, providing a modern, synchronous alternative that's simpler than WebClient but more feature-rich than RestTemplate.

### RestClient Configuration and Usage

```kotlin
@Configuration
class RestClientConfiguration {

    @Bean
    @Primary
    fun restClient(restClientBuilder: RestClient.Builder): RestClient {
        return restClientBuilder
            .baseUrl("https://api.example.com")
            .defaultHeaders { headers ->
                headers.contentType = MediaType.APPLICATION_JSON
                headers.accept = listOf(MediaType.APPLICATION_JSON)
            }
            .requestInterceptor { request, body, execution ->
                // Add request logging
                LoggerFactory.getLogger("RestClient.Request")
                    .debug("HTTP {} {}", request.method, request.uri)
                execution.execute(request, body)
            }
            .build()
    }

    @Bean
    @Qualifier("userServiceRestClient")
    fun userServiceRestClient(
        restClientBuilder: RestClient.Builder,
        @Value("\${app.external.user-service.base-url}") baseUrl: String
    ): RestClient {
        return restClientBuilder
            .baseUrl(baseUrl)
            .defaultHeaders { headers ->
                headers.contentType = MediaType.APPLICATION_JSON
                headers.setBearerAuth(getServiceToken())
            }
            .build()
    }

    @Bean
    @Qualifier("paymentServiceRestClient")
    fun paymentServiceRestClient(
        restClientBuilder: RestClient.Builder,
        @Value("\${app.payment-service.base-url}") baseUrl: String,
        @Value("\${app.payment-service.api-key}") apiKey: String
    ): RestClient {
        return restClientBuilder
            .baseUrl(baseUrl)
            .defaultHeaders { headers ->
                headers.add("X-API-Key", apiKey)
                headers.contentType = MediaType.APPLICATION_JSON
            }
            .build()
    }

    private fun getServiceToken(): String {
        // Implementation to get service token
        return "service-token"
    }
}

@Service
class RestClientUserService(
    @Qualifier("userServiceRestClient") private val restClient: RestClient,
    private val meterRegistry: MeterRegistry
) {

    private val logger = LoggerFactory.getLogger(RestClientUserService::class.java)

    /**
     * Get user by ID with modern RestClient approach
     */
    fun getUser(userId: Long): UserDto? {
        return recordMetrics("getUser") {
            try {
                restClient
                    .get()
                    .uri("/api/users/{userId}", userId)
                    .retrieve()
                    .onStatus(HttpStatus::is4xxClientError) { _, response ->
                        when (response.statusCode) {
                            HttpStatus.NOT_FOUND -> {
                                logger.debug("User {} not found", userId)
                                // Return null for not found instead of throwing exception
                            }
                            else -> {
                                val errorBody = response.body(String::class.java)
                                logger.warn("Client error getting user {}: {}", userId, errorBody)
                                throw ClientException("Client error: ${response.statusCode}")
                            }
                        }
                    }
                    .onStatus(HttpStatus::is5xxServerError) { _, response ->
                        val errorBody = response.body(String::class.java)
                        logger.error("Server error getting user {}: {}", userId, errorBody)
                        throw ServerException("Server error: ${response.statusCode}")
                    }
                    .body(UserDto::class.java)
            } catch (ex: ResourceAccessException) {
                logger.error("Network error getting user {}", userId, ex)
                throw ExternalServiceException("User service", "getUser", "Network error", ex)
            }
        }
    }

    /**
     * Create user with comprehensive error handling
     */
    fun createUser(createUserRequest: CreateUserRequest): UserDto {
        return recordMetrics("createUser") {
            try {
                restClient
                    .post()
                    .uri("/api/users")
                    .body(createUserRequest)
                    .retrieve()
                    .onStatus(HttpStatus::is4xxClientError) { _, response ->
                        val errorResponse = response.body(ErrorResponse::class.java)
                        when (response.statusCode) {
                            HttpStatus.BAD_REQUEST -> {
                                throw ValidationException(
                                    "User creation validation failed",
                                    errorResponse?.details ?: emptyList()
                                )
                            }
                            HttpStatus.CONFLICT -> {
                                throw DuplicateResourceException(
                                    "User",
                                    "username/email",
                                    createUserRequest.username
                                )
                            }
                            else -> {
                                throw ClientException("Client error: ${response.statusCode}")
                            }
                        }
                    }
                    .body(UserDto::class.java)
                    ?: throw IllegalStateException("Created user response is null")
            } catch (ex: ResourceAccessException) {
                logger.error("Network error creating user", ex)
                throw ExternalServiceException("User service", "createUser", "Network error", ex)
            }
        }
    }

    /**
     * Update user with partial data
     */
    fun updateUser(userId: Long, updateUserRequest: UpdateUserRequest): UserDto {
        return recordMetrics("updateUser") {
            try {
                restClient
                    .put()
                    .uri("/api/users/{userId}", userId)
                    .body(updateUserRequest)
                    .retrieve()
                    .onStatus(HttpStatus::is4xxClientError) { _, response ->
                        when (response.statusCode) {
                            HttpStatus.NOT_FOUND -> {
                                throw ResourceNotFoundException("User not found with ID: $userId")
                            }
                            HttpStatus.BAD_REQUEST -> {
                                val errorResponse = response.body(ErrorResponse::class.java)
                                throw ValidationException(
                                    "User update validation failed",
                                    errorResponse?.details ?: emptyList()
                                )
                            }
                            else -> {
                                throw ClientException("Client error: ${response.statusCode}")
                            }
                        }
                    }
                    .body(UserDto::class.java)
                    ?: throw IllegalStateException("Updated user response is null")
            } catch (ex: ResourceAccessException) {
                logger.error("Network error updating user {}", userId, ex)
                throw ExternalServiceException("User service", "updateUser", "Network error", ex)
            }
        }
    }

    /**
     * Get users with advanced query parameters
     */
    fun searchUsers(searchCriteria: UserSearchCriteria): PagedResponse<UserDto> {
        return recordMetrics("searchUsers") {
            try {
                restClient
                    .get()
                    .uri { uriBuilder ->
                        val builder = uriBuilder.path("/api/users/search")

                        searchCriteria.username?.let { builder.queryParam("username", it) }
                        searchCriteria.email?.let { builder.queryParam("email", it) }
                        searchCriteria.active?.let { builder.queryParam("active", it) }
                        builder.queryParam("page", searchCriteria.page)
                        builder.queryParam("size", searchCriteria.size)
                        searchCriteria.sort?.let { builder.queryParam("sort", it) }

                        builder.build()
                    }
                    .retrieve()
                    .body(PagedUserResponse::class.java)
                    ?.let { response ->
                        PagedResponse(
                            content = response.content,
                            totalElements = response.totalElements,
                            totalPages = response.totalPages,
                            number = response.number,
                            size = response.size,
                            first = response.first,
                            last = response.last
                        )
                    }
                    ?: throw IllegalStateException("Search users response is null")
            } catch (ex: ResourceAccessException) {
                logger.error("Network error searching users", ex)
                throw ExternalServiceException("User service", "searchUsers", "Network error", ex)
            }
        }
    }

    /**
     * Delete user with confirmation
     */
    fun deleteUser(userId: Long): Boolean {
        return recordMetrics("deleteUser") {
            try {
                restClient
                    .delete()
                    .uri("/api/users/{userId}", userId)
                    .retrieve()
                    .onStatus(HttpStatus::is4xxClientError) { _, response ->
                        when (response.statusCode) {
                            HttpStatus.NOT_FOUND -> {
                                logger.debug("User {} not found for deletion", userId)
                                // Don't throw exception for not found on delete
                            }
                            else -> {
                                throw ClientException("Client error: ${response.statusCode}")
                            }
                        }
                    }
                    .toBodilessEntity()
                true
            } catch (ex: ResourceAccessException) {
                logger.error("Network error deleting user {}", userId, ex)
                false
            }
        }
    }

    /**
     * Batch user creation with RestClient
     */
    fun createUsersBatch(createUserRequests: List<CreateUserRequest>): BatchCreateResponse {
        return recordMetrics("createUsersBatch") {
            try {
                restClient
                    .post()
                    .uri("/api/users/batch")
                    .body(BatchCreateRequest(createUserRequests))
                    .retrieve()
                    .body(BatchCreateResponse::class.java)
                    ?: throw IllegalStateException("Batch create response is null")
            } catch (ex: ResourceAccessException) {
                logger.error("Network error creating users batch", ex)
                throw ExternalServiceException("User service", "createUsersBatch", "Network error", ex)
            }
        }
    }

    /**
     * Stream users using RestClient for large datasets
     */
    fun streamUsers(callback: (UserDto) -> Unit) {
        recordMetrics("streamUsers") {
            try {
                // RestClient doesn't have native streaming support like WebClient
                // But we can implement pagination-based streaming
                var page = 0
                val size = 100
                var hasMore = true

                while (hasMore) {
                    val pagedResponse = restClient
                        .get()
                        .uri("/api/users?page={page}&size={size}", page, size)
                        .retrieve()
                        .body(PagedUserResponse::class.java)
                        ?: break

                    pagedResponse.content.forEach(callback)

                    hasMore = !pagedResponse.last
                    page++
                }
            } catch (ex: ResourceAccessException) {
                logger.error("Network error streaming users", ex)
                throw ExternalServiceException("User service", "streamUsers", "Network error", ex)
            }
        }
    }

    private fun <T> recordMetrics(operation: String, block: () -> T): T {
        val timer = Timer.builder("rest.client.request.duration")
            .description("RestClient request duration")
            .tag("service", "user-service")
            .tag("operation", operation)
            .register(meterRegistry)

        val counter = Counter.builder("rest.client.request.total")
            .description("Total RestClient requests")
            .tag("service", "user-service")
            .tag("operation", operation)
            .register(meterRegistry)

        return timer.recordCallable {
            try {
                val result = block()
                counter.increment(Tags.of("status", "success"))
                result
            } catch (ex: Exception) {
                counter.increment(Tags.of(
                    "status", "error",
                    "error_type", ex.javaClass.simpleName
                ))
                throw ex
            }
        }!!
    }
}

// Search criteria class
data class UserSearchCriteria(
    val username: String? = null,
    val email: String? = null,
    val active: Boolean? = null,
    val page: Int = 0,
    val size: Int = 20,
    val sort: String? = null
)

// Batch operation classes
data class BatchCreateRequest(
    val users: List<CreateUserRequest>
)

data class BatchCreateResponse(
    val successful: List<UserDto>,
    val failed: List<BatchCreateFailure>,
    val totalRequested: Int,
    val successCount: Int,
    val failureCount: Int
)

data class BatchCreateFailure(
    val index: Int,
    val request: CreateUserRequest,
    val error: String
)
```

### Integration Testing for HTTP Clients

```kotlin
// Test configuration for HTTP clients
@TestConfiguration
class TestHttpClientConfiguration {

    @Bean
    @Primary
    fun mockWebServer(): MockWebServer {
        return MockWebServer()
    }

    @Bean
    @Primary
    fun testRestClient(mockWebServer: MockWebServer): RestClient {
        return RestClient.builder()
            .baseUrl(mockWebServer.url("/").toString())
            .build()
    }

    @Bean
    @Primary
    fun testWebClient(mockWebServer: MockWebServer): WebClient {
        return WebClient.builder()
            .baseUrl(mockWebServer.url("/").toString())
            .build()
    }
}

@SpringBootTest
@Import(TestHttpClientConfiguration::class)
class HttpClientIntegrationTest {

    @Autowired
    private lateinit var mockWebServer: MockWebServer

    @Autowired
    private lateinit var restClientUserService: RestClientUserService

    @Autowired
    private lateinit var reactiveUserService: ReactiveUserService

    @AfterEach
    fun cleanup() {
        // Clean up any remaining requests
        try {
            mockWebServer.takeRequest(100, TimeUnit.MILLISECONDS)
        } catch (e: InterruptedException) {
            // Ignore
        }
    }

    @Test
    fun `should get user successfully with RestClient`() {
        // Given
        val userId = 1L
        val expectedUser = UserDto(
            id = userId,
            username = "testuser",
            email = "test@example.com",
            firstName = "Test",
            lastName = "User",
            active = true,
            createdAt = LocalDateTime.now(),
            updatedAt = LocalDateTime.now()
        )

        mockWebServer.enqueue(
            MockResponse()
                .setResponseCode(200)
                .setHeader("Content-Type", "application/json")
                .setBody(jacksonObjectMapper().writeValueAsString(expectedUser))
        )

        // When
        val result = restClientUserService.getUser(userId)

        // Then
        assertThat(result).isNotNull
        assertThat(result!!.id).isEqualTo(userId)
        assertThat(result.username).isEqualTo("testuser")

        val request = mockWebServer.takeRequest()
        assertThat(request.method).isEqualTo("GET")
        assertThat(request.path).isEqualTo("/api/users/1")
    }

    @Test
    fun `should handle user not found gracefully`() {
        // Given
        val userId = 999L

        mockWebServer.enqueue(
            MockResponse()
                .setResponseCode(404)
                .setHeader("Content-Type", "application/json")
                .setBody("""{"error": "User not found"}""")
        )

        // When
        val result = restClientUserService.getUser(userId)

        // Then
        assertThat(result).isNull()

        val request = mockWebServer.takeRequest()
        assertThat(request.method).isEqualTo("GET")
        assertThat(request.path).isEqualTo("/api/users/999")
    }

    @Test
    fun `should create user successfully with reactive client`() = runBlocking {
        // Given
        val createRequest = CreateUserRequest(
            username = "newuser",
            email = "newuser@example.com",
            firstName = "New",
            lastName = "User",
            password = "password123"
        )

        val createdUser = UserDto(
            id = 2L,
            username = createRequest.username,
            email = createRequest.email,
            firstName = createRequest.firstName,
            lastName = createRequest.lastName,
            active = true,
            createdAt = LocalDateTime.now(),
            updatedAt = LocalDateTime.now()
        )

        mockWebServer.enqueue(
            MockResponse()
                .setResponseCode(201)
                .setHeader("Content-Type", "application/json")
                .setBody(jacksonObjectMapper().writeValueAsString(createdUser))
        )

        // When
        val result = reactiveUserService.createUser(createRequest)

        // Then
        assertThat(result).isNotNull
        assertThat(result.id).isEqualTo(2L)
        assertThat(result.username).isEqualTo("newuser")

        val request = mockWebServer.takeRequest()
        assertThat(request.method).isEqualTo("POST")
        assertThat(request.path).isEqualTo("/api/users")

        val requestBody = jacksonObjectMapper().readValue(request.body.readUtf8(), CreateUserRequest::class.java)
        assertThat(requestBody.username).isEqualTo("newuser")
    }

    @Test
    fun `should handle validation errors appropriately`() = runBlocking {
        // Given
        val createRequest = CreateUserRequest(
            username = "",
            email = "invalid-email",
            firstName = "Test",
            lastName = "User",
            password = "123"
        )

        val errorResponse = ErrorResponse(
            message = "Validation failed",
            errorCode = "VALIDATION_ERROR",
            details = listOf(
                ValidationError("username", "Username is required", "NOT_BLANK"),
                ValidationError("email", "Email must be valid", "EMAIL"),
                ValidationError("password", "Password too short", "SIZE")
            ),
            timestamp = Instant.now()
        )

        mockWebServer.enqueue(
            MockResponse()
                .setResponseCode(400)
                .setHeader("Content-Type", "application/json")
                .setBody(jacksonObjectMapper().writeValueAsString(errorResponse))
        )

        // When & Then
        assertThrows<ValidationException> {
            runBlocking { reactiveUserService.createUser(createRequest) }
        }.let { exception ->
            assertThat(exception.errors).hasSize(3)
            assertThat(exception.errors.map { it.field }).containsExactlyInAnyOrder(
                "username", "email", "password"
            )
        }

        val request = mockWebServer.takeRequest()
        assertThat(request.method).isEqualTo("POST")
        assertThat(request.path).isEqualTo("/api/users")
    }
}
```

## 12.4 Summary

Server-to-server communication is a fundamental aspect of modern application architecture. Throughout this chapter, we've explored three different approaches to HTTP client implementation in Kotlin Spring Boot applications, each with distinct advantages and appropriate use cases.

**RestTemplate** remains a solid choice for synchronous communication:
- Simple, blocking model that's easy to understand and debug
- Comprehensive interceptor support for cross-cutting concerns like logging, retry, and authentication
- Extensive customization options through RestTemplateBuilder
- Well-suited for traditional synchronous workflows and legacy integration

**WebClient with Kotlin coroutines** provides powerful reactive capabilities:
- Non-blocking, asynchronous execution that scales efficiently
- Excellent integration with Kotlin coroutines for natural async programming
- Built-in support for reactive streams and backpressure handling
- Perfect for high-throughput scenarios and reactive architectures

**RestClient** offers a modern, simplified approach:
- Clean, fluent API that's easier to use than RestTemplate
- Synchronous model without the complexity of reactive programming
- Better performance and features than RestTemplate
- Ideal for straightforward HTTP client scenarios

**Key patterns and practices** we've established:

1. **Comprehensive Error Handling**: Custom exception hierarchies that map HTTP status codes to meaningful business exceptions
2. **Retry and Circuit Breaker Logic**: Resilience patterns that handle transient failures and prevent cascade failures
3. **Monitoring and Metrics**: Detailed instrumentation using Micrometer for observability
4. **Security Integration**: Proper authentication and authorization handling for service-to-service communication
5. **Testing Strategies**: MockWebServer integration for reliable, fast integration tests

**Kotlin-specific advantages** we've leveraged:

- Coroutines for natural asynchronous programming with WebClient
- Data classes for clean, immutable request/response models
- Extension functions for enhanced API usability
- Null safety for robust error handling
- DSL-like builders for configuration and request construction

**When to choose each approach:**

- **RestTemplate**: Legacy systems, simple synchronous needs, team familiarity
- **WebClient**: High concurrency, reactive systems, modern asynchronous architectures
- **RestClient**: New projects needing synchronous HTTP communication, migration from RestTemplate

The patterns and implementations demonstrated in this chapter provide a solid foundation for building resilient, observable, and maintainable HTTP client integrations. Whether you're calling external APIs, integrating with microservices, or communicating with third-party services, these approaches will serve you well in production environments.

In the next chapter, we'll explore authentication and authorization, building upon the HTTP client patterns established here to implement secure service-to-service communication and user authentication flows. The monitoring and error handling patterns we've developed will be crucial for tracking security-related events and handling authentication failures gracefully.