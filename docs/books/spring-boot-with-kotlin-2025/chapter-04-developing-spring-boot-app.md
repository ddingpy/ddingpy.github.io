---
layout: default
parent: Spring Boot with Kotlin (2025)
nav_exclude: true
---
# Chapter 04: Developing a Spring Boot Application
- TOC
{:toc}

It's time to write code! In this chapter, we'll create our first Spring Boot application with Kotlin, explore the project structure, understand build configuration, and get our "Hello World" running. By the end, you'll have a solid foundation for building real applications.

## 4.1 Creating a Project

There are two primary ways to create a Spring Boot project: using IntelliJ IDEA's built-in wizard or the official Spring Initializr website. Both approaches generate the same project structure, so choose based on your preference.

### 4.1.1 Creating a Project in IntelliJ IDEA

IntelliJ IDEA provides the most streamlined experience for creating Spring Boot projects with Kotlin.

**Step 1: Start the New Project Wizard**
1. Open IntelliJ IDEA
2. Click "New Project" or File → New → Project
3. Select "Spring Initializr" from the left panel

**Step 2: Configure Project Metadata**
```
Name: my-first-app
Location: ~/projects/my-first-app
Language: Kotlin
Type: Gradle - Kotlin
Group: com.example
Artifact: my-first-app
Package name: com.example.myfirstapp
Project SDK: 21 (or your installed version)
Java: 21
Packaging: Jar
```

**Step 3: Select Spring Boot Version and Dependencies**
```
Spring Boot: 3.2.0 (or latest stable)

Dependencies to add:
Developer Tools:
  ☑ Spring Boot DevTools
  ☑ Lombok (optional for Java interop)

Web:
  ☑ Spring Web
  ☑ Spring Reactive Web (if you want WebFlux)

SQL:
  ☑ Spring Data JPA
  ☑ PostgreSQL Driver
  ☑ H2 Database (for testing)

Ops:
  ☑ Spring Boot Actuator

Click "Create"
```

**What IntelliJ Does Behind the Scenes:**
- Downloads project template from start.spring.io
- Configures Gradle wrapper
- Sets up proper source directories
- Indexes dependencies
- Configures Kotlin compiler
- Sets up run configurations

### 4.1.2 Creating a Project from the Official Spring Site

Sometimes you might want to create a project without an IDE, or share a configuration with team members.

**Step 1: Navigate to Spring Initializr**
Open your browser and go to [https://start.spring.io](https://start.spring.io)

**Step 2: Configure Your Project**

The web interface provides an intuitive form:

```
Project: Gradle - Kotlin
Language: Kotlin
Spring Boot: 3.2.0

Project Metadata:
Group: com.example
Artifact: my-first-app
Name: my-first-app
Description: Demo project for Spring Boot with Kotlin
Package name: com.example.myfirstapp
Packaging: Jar
Java: 21
```

**Step 3: Add Dependencies**

Click "ADD DEPENDENCIES" and search for:
- Spring Web
- Spring Data JPA
- PostgreSQL Driver
- Spring Boot DevTools
- Spring Boot Actuator

**Step 4: Generate and Extract**
1. Click "GENERATE" to download the ZIP file
2. Extract to your projects directory
3. Open in IntelliJ IDEA: File → Open → Select the extracted folder

**Pro Tip: Using the Spring Initializr REST API**

You can also generate projects programmatically:

```bash
# Generate a project using curl
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,postgresql,devtools,actuator \
  -d type=gradle-project \
  -d language=kotlin \
  -d javaVersion=21 \
  -d groupId=com.example \
  -d artifactId=my-first-app \
  -d packageName=com.example.myfirstapp \
  -d bootVersion=3.2.0 \
  -o my-first-app.zip

# Extract
unzip my-first-app.zip -d my-first-app
cd my-first-app
```

### Understanding the Generated Project Structure

Let's explore what Spring Initializr created:

```
my-first-app/
├── .gitignore                 # Git ignore rules
├── build.gradle.kts           # Build configuration (Kotlin DSL)
├── settings.gradle.kts        # Gradle settings
├── gradlew                    # Gradle wrapper script (Unix)
├── gradlew.bat               # Gradle wrapper script (Windows)
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
└── src/
    ├── main/
    │   ├── kotlin/
    │   │   └── com/example/myfirstapp/
    │   │       └── MyFirstAppApplication.kt
    │   └── resources/
    │       ├── application.properties
    │       ├── static/          # Static web resources
    │       └── templates/       # Template files
    └── test/
        ├── kotlin/
        │   └── com/example/myfirstapp/
        │       └── MyFirstAppApplicationTests.kt
        └── resources/
```

**Key Files Explained:**

`MyFirstAppApplication.kt`:
```kotlin
package com.example.myfirstapp

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class MyFirstAppApplication

fun main(args: Array<String>) {
    runApplication<MyFirstAppApplication>(*args)
}
```

The `@SpringBootApplication` annotation is actually three annotations in one:
- `@Configuration`: Marks this as a configuration class
- `@EnableAutoConfiguration`: Enables Spring Boot's auto-configuration
- `@ComponentScan`: Enables component scanning from this package

## 4.2 Exploring build.gradle.kts

The build file is the heart of your project configuration. Let's understand every part of a production-ready `build.gradle.kts`.

### 4.2.1 Build Management Tools

Before diving into Gradle, let's understand why we use build tools:

**What Build Tools Do:**
- Dependency management
- Compilation and packaging
- Running tests
- Code quality checks
- Deployment preparation

**Popular Build Tools for JVM:**
- **Gradle**: Modern, flexible, uses Groovy or Kotlin DSL
- **Maven**: Mature, XML-based, extensive plugin ecosystem
- **Bazel**: Google's build tool, excellent for monorepos
- **SBT**: Scala-focused but works with Kotlin

We use Gradle with Kotlin DSL because:
1. Type-safe build scripts
2. IDE auto-completion
3. Kotlin syntax consistency
4. Better refactoring support

### 4.2.2 Gradle

Gradle is more than a build tool—it's a build automation platform. Let's understand its core concepts:

**Gradle Concepts:**
- **Project**: What you're building (your application)
- **Task**: A unit of work (compile, test, jar)
- **Plugin**: Adds tasks and conventions
- **Dependency**: External libraries your project needs
- **Repository**: Where dependencies are downloaded from

**Gradle Wrapper:**
The wrapper ensures everyone uses the same Gradle version:

```properties
# gradle/wrapper/gradle-wrapper.properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

Always use the wrapper:
```bash
./gradlew build     # Unix/Mac
gradlew.bat build   # Windows
```

### 4.2.3 Managing Dependencies Using Kotlin DSL

Now let's explore a comprehensive `build.gradle.kts`:

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

// Plugin Configuration
plugins {
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"  // Opens classes for Spring
    kotlin("plugin.jpa") version "1.9.21"      // No-arg constructors for JPA
    kotlin("kapt") version "1.9.21"            // Annotation processing
    id("org.jlleitschuh.gradle.ktlint") version "12.0.3"  // Code formatting
}

// Project Metadata
group = "com.example"
version = "0.0.1-SNAPSHOT"
description = "My First Spring Boot App with Kotlin"

// Java Compatibility
java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

// Repository Configuration
repositories {
    mavenCentral()
    maven { url = uri("https://repo.spring.io/milestone") }  // For milestones
    maven { url = uri("https://repo.spring.io/snapshot") }   // For snapshots
}

// Dependency Versions (centralized version management)
extra["kotlinCoroutinesVersion"] = "1.7.3"
extra["mockkVersion"] = "1.13.8"
extra["kotestVersion"] = "5.8.0"

// Dependencies
dependencies {
    // Spring Boot Starters
    implementation("org.springframework.boot:spring-boot-starter-web") {
        exclude(group = "org.springframework.boot", module = "spring-boot-starter-tomcat")
    }
    implementation("org.springframework.boot:spring-boot-starter-undertow")  // Alternative to Tomcat
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-cache")

    // Kotlin Support
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")

    // Kotlin Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:${property("kotlinCoroutinesVersion")}")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor:${property("kotlinCoroutinesVersion")}")

    // Database
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")  // Database migrations

    // Caching
    implementation("com.github.ben-manes.caffeine:caffeine")

    // API Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")

    // Utilities
    implementation("io.github.microutils:kotlin-logging-jvm:3.0.5")
    implementation("org.apache.commons:commons-lang3:3.14.0")

    // Development Only
    developmentOnly("org.springframework.boot:spring-boot-devtools")

    // Annotation Processing
    kapt("org.springframework.boot:spring-boot-configuration-processor")

    // Runtime Only
    runtimeOnly("io.micrometer:micrometer-registry-prometheus")  // Metrics
    runtimeOnly("com.h2database:h2")  // In-memory database for development

    // Test Dependencies
    testImplementation("org.springframework.boot:spring-boot-starter-test") {
        exclude(group = "org.mockito", module = "mockito-core")
    }
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
    testImplementation("org.testcontainers:postgresql:1.19.3")
    testImplementation("org.testcontainers:junit-jupiter:1.19.3")
    testImplementation("io.mockk:mockk:${property("mockkVersion")}")
    testImplementation("com.ninja-squad:springmockk:4.0.2")
    testImplementation("io.kotest:kotest-runner-junit5:${property("kotestVersion")}")
    testImplementation("io.kotest:kotest-assertions-core:${property("kotestVersion")}")
    testImplementation("io.kotest.extensions:kotest-extensions-spring:1.1.3")
}

// Kotlin Compiler Configuration
tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs = listOf(
            "-Xjsr305=strict",           // Strict nullability for Spring
            "-Xemit-jvm-type-annotations", // Better Java interop
            "-Xjvm-default=all"          // Generate default methods in interfaces
        )
        jvmTarget = "21"
        languageVersion = "1.9"
        apiVersion = "1.9"
    }
}

// Test Configuration
tasks.withType<Test> {
    useJUnitPlatform()
    testLogging {
        events("passed", "skipped", "failed")
        exceptionFormat = org.gradle.api.tasks.testing.logging.TestExceptionFormat.FULL
        showStandardStreams = false
    }

    // Memory settings for tests
    maxHeapSize = "1G"
    jvmArgs = listOf("-XX:MaxPermSize=256m")

    // Parallel test execution
    systemProperty("junit.jupiter.execution.parallel.enabled", "true")
    systemProperty("junit.jupiter.execution.parallel.mode.default", "concurrent")
}

// Spring Boot Configuration
springBoot {
    buildInfo {
        properties {
            artifact = "my-first-app"
            version = project.version.toString()
            group = project.group.toString()
            name = "My First Spring Boot App"
            time = null  // Reproducible builds
        }
    }
}

// Jar Configuration
tasks.jar {
    enabled = false  // We only want the executable jar
}

tasks.bootJar {
    enabled = true
    archiveFileName.set("${project.name}.jar")
    launchScript()  // Makes jar executable on Unix-like systems
}

// Ktlint Configuration
ktlint {
    version.set("1.0.1")
    android.set(false)
    outputToConsole.set(true)
    outputColorName.set("RED")
    ignoreFailures.set(false)

    filter {
        exclude("**/generated/**")
        include("**/kotlin/**")
    }
}

// Custom Tasks
tasks.register("printVersion") {
    doLast {
        println("Version: ${project.version}")
    }
}

tasks.register<Copy>("copyDocs") {
    from("src/docs")
    into("build/docs")
}

// Task Dependencies
tasks.build {
    dependsOn(tasks.ktlintCheck)
}

// Gradle Properties Configuration
tasks.withType<JavaCompile> {
    options.encoding = "UTF-8"
}

// Configure ProcessResources to expand properties
tasks.processResources {
    filesMatching("application.yml") {
        expand(project.properties)
    }
}
```

**Understanding Dependency Scopes:**

- **implementation**: Main compile and runtime dependency
- **compileOnly**: Only needed at compile time
- **runtimeOnly**: Only needed at runtime
- **developmentOnly**: Only in development mode
- **testImplementation**: Test compile and runtime
- **kapt**: Kotlin annotation processing

## 4.3 Printing "Hello World"

Now let's write our first Spring Boot endpoint and understand how everything fits together.

### 4.3.1 Writing a Controller

Let's create a simple REST controller:

```kotlin
package com.example.myfirstapp.controller

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api")
class HelloController {

    @GetMapping("/hello")
    fun hello(): String {
        return "Hello, World!"
    }
}
```

But let's make it more interesting and idiomatic:

```kotlin
package com.example.myfirstapp.controller

import mu.KotlinLogging
import org.springframework.web.bind.annotation.*
import java.time.LocalDateTime

private val logger = KotlinLogging.logger {}

@RestController
@RequestMapping("/api")
class HelloController {

    // Simple text response
    @GetMapping("/hello")
    fun hello(): String {
        logger.info { "Hello endpoint called" }
        return "Hello, World from Kotlin and Spring Boot!"
    }

    // JSON response with data class
    @GetMapping("/greeting")
    fun greeting(@RequestParam(defaultValue = "Guest") name: String): GreetingResponse {
        logger.info { "Greeting endpoint called for: $name" }
        return GreetingResponse(
            message = "Hello, $name!",
            timestamp = LocalDateTime.now(),
            kotlin = true
        )
    }

    // Path variable example
    @GetMapping("/hello/{name}")
    fun personalizedHello(@PathVariable name: String): Map<String, String> {
        return mapOf(
            "greeting" to "Hello, $name!",
            "language" to "Kotlin",
            "framework" to "Spring Boot"
        )
    }

    // POST endpoint with request body
    @PostMapping("/message")
    fun receiveMessage(@RequestBody message: MessageRequest): MessageResponse {
        logger.info { "Received message: ${message.content}" }
        return MessageResponse(
            received = true,
            echo = message.content.reversed(),
            length = message.content.length
        )
    }
}

// Data classes for request/response
data class GreetingResponse(
    val message: String,
    val timestamp: LocalDateTime,
    val kotlin: Boolean
)

data class MessageRequest(
    val content: String,
    val sender: String? = "Anonymous"
)

data class MessageResponse(
    val received: Boolean,
    val echo: String,
    val length: Int
)
```

### 4.3.2 Running the Application

There are multiple ways to run your Spring Boot application:

**Method 1: From IntelliJ IDEA**
1. Click the green arrow next to the `main` function
2. Or right-click on `MyFirstAppApplication.kt` → Run

**Method 2: Using Gradle**
```bash
# Using the Gradle wrapper
./gradlew bootRun

# With a specific profile
./gradlew bootRun --args='--spring.profiles.active=dev'

# With debug enabled
./gradlew bootRun --debug-jvm
```

**Method 3: Building and Running the JAR**
```bash
# Build the application
./gradlew build

# Run the JAR
java -jar build/libs/my-first-app.jar

# With specific JVM options
java -Xmx512m -Dspring.profiles.active=prod -jar build/libs/my-first-app.jar
```

**Method 4: Using Spring Boot DevTools**

With DevTools in your dependencies, the application automatically restarts when you make changes:

```kotlin
// application.yml
spring:
  devtools:
    restart:
      enabled: true
      poll-interval: 2s
      quiet-period: 1s
    livereload:
      enabled: true
      port: 35729
```

### 4.3.3 Testing with a Web Browser

Once your application is running, you'll see output like:

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.0)

2024-01-15T10:30:45.123+01:00  INFO 12345 --- [           main] c.e.m.MyFirstAppApplication              : Starting MyFirstAppApplication using Java 21
2024-01-15T10:30:45.125+01:00  INFO 12345 --- [           main] c.e.m.MyFirstAppApplication              : No active profile set, falling back to 1 default profile: "default"
2024-01-15T10:30:46.789+01:00  INFO 12345 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2024-01-15T10:30:46.799+01:00  INFO 12345 --- [           main] c.e.m.MyFirstAppApplication              : Started MyFirstAppApplication in 1.234 seconds (process running for 1.567)
```

Now open your browser and test the endpoints:

- http://localhost:8080/api/hello
- http://localhost:8080/api/greeting?name=Developer
- http://localhost:8080/api/hello/Kotlin

### 4.3.4 Testing with Talend API Tester

For more sophisticated API testing, use a tool like Talend API Tester (Chrome extension) or Postman:

**GET Request:**
```
Method: GET
URL: http://localhost:8080/api/greeting?name=Developer
Headers: Accept: application/json
```

**POST Request:**
```
Method: POST
URL: http://localhost:8080/api/message
Headers:
  Content-Type: application/json
Body:
{
  "content": "Hello from API Tester",
  "sender": "Developer"
}
```

### 4.3.5 Writing Idiomatic Kotlin Controller

Let's refactor our controller to be more idiomatic and production-ready:

```kotlin
package com.example.myfirstapp.controller

import mu.KotlinLogging
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import org.springframework.web.server.ResponseStatusException
import java.time.Instant
import java.util.UUID

private val logger = KotlinLogging.logger {}

@RestController
@RequestMapping("/api/v1")
class GreetingController(
    private val greetingService: GreetingService  // Dependency injection
) {

    // Using ResponseEntity for more control
    @GetMapping("/greetings/{id}")
    fun getGreeting(@PathVariable id: String): ResponseEntity<Greeting> {
        logger.debug { "Fetching greeting with id: $id" }

        return greetingService.findById(id)
            ?.let { ResponseEntity.ok(it) }
            ?: throw ResponseStatusException(
                HttpStatus.NOT_FOUND,
                "Greeting not found with id: $id"
            )
    }

    // Using sealed classes for responses
    @PostMapping("/greetings")
    fun createGreeting(@RequestBody request: CreateGreetingRequest): ResponseEntity<*> {
        logger.info { "Creating new greeting: ${request.message}" }

        return when (val result = greetingService.create(request)) {
            is GreetingResult.Success -> ResponseEntity
                .status(HttpStatus.CREATED)
                .body(result.greeting)

            is GreetingResult.ValidationError -> ResponseEntity
                .badRequest()
                .body(ErrorResponse(result.errors))

            is GreetingResult.Duplicate -> ResponseEntity
                .status(HttpStatus.CONFLICT)
                .body(ErrorResponse(listOf("Greeting already exists")))
        }
    }

    // Pagination support
    @GetMapping("/greetings")
    fun listGreetings(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int,
        @RequestParam(required = false) language: String?
    ): PagedResponse<Greeting> {
        logger.debug { "Listing greetings - page: $page, size: $size, language: $language" }

        val greetings = greetingService.findAll(page, size, language)
        return PagedResponse(
            content = greetings.content,
            page = page,
            size = size,
            totalElements = greetings.totalElements,
            totalPages = greetings.totalPages
        )
    }

    // Async endpoint with coroutines
    @GetMapping("/greetings/random")
    suspend fun getRandomGreeting(): Greeting {
        logger.debug { "Fetching random greeting" }
        return greetingService.getRandomGreeting()
    }

    // File upload example
    @PostMapping("/greetings/import")
    fun importGreetings(
        @RequestParam("file") file: MultipartFile
    ): ImportResponse {
        logger.info { "Importing greetings from file: ${file.originalFilename}" }

        require(!file.isEmpty) { "File must not be empty" }
        require(file.contentType == "text/csv") { "File must be CSV format" }

        val imported = greetingService.importFromCsv(file.inputStream)
        return ImportResponse(
            imported = imported,
            message = "Successfully imported $imported greetings"
        )
    }
}

// Service layer
@Service
class GreetingService(
    private val repository: GreetingRepository
) {
    fun findById(id: String): Greeting? = repository.findById(id).orElse(null)

    fun create(request: CreateGreetingRequest): GreetingResult {
        // Validation
        if (request.message.isBlank()) {
            return GreetingResult.ValidationError(listOf("Message cannot be blank"))
        }

        // Check for duplicates
        if (repository.existsByMessage(request.message)) {
            return GreetingResult.Duplicate
        }

        val greeting = Greeting(
            id = UUID.randomUUID().toString(),
            message = request.message,
            language = request.language ?: "en",
            createdAt = Instant.now()
        )

        return GreetingResult.Success(repository.save(greeting))
    }

    fun findAll(page: Int, size: Int, language: String?): Page<Greeting> {
        val pageable = PageRequest.of(page, size, Sort.by("createdAt").descending())

        return if (language != null) {
            repository.findByLanguage(language, pageable)
        } else {
            repository.findAll(pageable)
        }
    }

    suspend fun getRandomGreeting(): Greeting = withContext(Dispatchers.IO) {
        repository.findRandomGreeting() ?: throw ResponseStatusException(
            HttpStatus.NOT_FOUND,
            "No greetings available"
        )
    }

    fun importFromCsv(inputStream: InputStream): Int {
        // CSV parsing logic here
        return 0
    }
}

// Data classes
data class Greeting(
    val id: String,
    val message: String,
    val language: String,
    val createdAt: Instant
)

data class CreateGreetingRequest(
    val message: String,
    val language: String? = null
)

data class PagedResponse<T>(
    val content: List<T>,
    val page: Int,
    val size: Int,
    val totalElements: Long,
    val totalPages: Int
)

data class ErrorResponse(
    val errors: List<String>,
    val timestamp: Instant = Instant.now()
)

data class ImportResponse(
    val imported: Int,
    val message: String
)

// Sealed class for results
sealed class GreetingResult {
    data class Success(val greeting: Greeting) : GreetingResult()
    data class ValidationError(val errors: List<String>) : GreetingResult()
    object Duplicate : GreetingResult()
}

// Repository interface
@Repository
interface GreetingRepository : JpaRepository<Greeting, String> {
    fun existsByMessage(message: String): Boolean
    fun findByLanguage(language: String, pageable: Pageable): Page<Greeting>

    @Query("SELECT g FROM Greeting g ORDER BY RANDOM() LIMIT 1")
    fun findRandomGreeting(): Greeting?
}
```

**Configuration for Better Development Experience:**

```yaml
# application.yml
spring:
  application:
    name: my-first-app

  jackson:
    property-naming-strategy: SNAKE_CASE
    default-property-inclusion: non_null
    serialization:
      write-dates-as-timestamps: false
      indent-output: true  # Pretty print in development

  web:
    locale: en_US
    resources:
      add-mappings: true
      cache:
        period: 3600

server:
  port: 8080
  error:
    include-message: always
    include-binding-errors: always
    include-stacktrace: on_param
    include-exception: false

  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/xml,text/plain

logging:
  level:
    com.example.myfirstapp: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG
  pattern:
    console: "%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}"

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
      base-path: /actuator
  endpoint:
    health:
      show-details: always
```

## Summary

In this chapter, we've built our first Spring Boot application with Kotlin from the ground up. We've covered:

- **Project Creation**: Using both IntelliJ IDEA and Spring Initializr to scaffold projects with the right dependencies
- **Build Configuration**: Understanding every aspect of `build.gradle.kts` including plugins, dependencies, and custom tasks
- **Hello World and Beyond**: Creating REST controllers from simple to production-ready, with proper structure and patterns
- **Running and Testing**: Multiple ways to run the application and test endpoints
- **Idiomatic Kotlin**: Writing controllers that leverage Kotlin's features like data classes, sealed classes, null safety, and coroutines

You now have a solid foundation for building Spring Boot applications. In the next chapter, we'll dive deeper into building sophisticated REST APIs with various HTTP methods, request/response handling, and API documentation.