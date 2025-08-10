---
layout: default
parent: Contents
date: 2025-07-15
nav_exclude: true
---

# Integrating Swagger (OpenAPI) with Spring Boot, Kotlin, and Gradle
- TOC
{:toc}

Here is a step-by-step guide to setting up Swagger (OpenAPI 3) in your Spring Boot project using Kotlin and Gradle.

### Introduction to Swagger and OpenAPI

**Swagger** is a suite of open-source tools for designing, building, documenting, and consuming RESTful web services. The **OpenAPI Specification** (formerly Swagger Specification) is the definition format for these APIs. By integrating Swagger into your Spring Boot application, you can automatically generate interactive API documentation that allows developers and consumers to understand and test your API endpoints directly from their browser.

For modern Spring Boot applications, the `springdoc-openapi` library is the recommended choice. It seamlessly integrates with Spring Boot and automatically generates OpenAPI 3 documentation.

-----

### Step 1: Add the Necessary Dependencies in `build.gradle.kts`

To begin, you need to add the `springdoc-openapi` dependency to your project's `build.gradle.kts` file. This single dependency is sufficient to get Swagger UI up and running.

1.  **Open your `build.gradle.kts` file.**
2.  **Add the following dependency** to the `dependencies` block:

<!-- end list -->

```kotlin
dependencies {
    // ... other dependencies
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.5.0")
    // For WebFlux projects, use this instead:
    // implementation("org.springdoc:springdoc-openapi-starter-webflux-ui:2.5.0")
}
```

**Note on Versions:** The version `2.5.0` is the latest stable version as of this writing. It is compatible with Spring Boot 3.x. You can check for the latest version on the [springdoc-openapi website](https://springdoc.org/) or Maven Central.

After adding the dependency, refresh your Gradle project to download and apply the new library.

-----

### Step 2: Basic Configuration for Swagger (OpenAPI)

While `springdoc-openapi` works out-of-the-box with zero configuration, you can customize the generated documentation to include metadata like the API title, description, version, and more.

Create a new Kotlin file, for example, `OpenApiConfig.kt`, in your project's main source directory (e.g., `src/main/kotlin/com/yourpackage/config`).

Add the following configuration code:

```kotlin
package com.yourpackage.config

import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.info.Info
import io.swagger.v3.oas.models.info.License
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiConfig {

    @Bean
    fun customOpenAPI(): OpenAPI {
        return OpenAPI()
            .info(
                Info()
                    .title("My Awesome API")
                    .version("1.0.0")
                    .description("This is a sample Spring Boot RESTful service using springdoc-openapi and OpenAPI 3.")
                    .termsOfService("http://swagger.io/terms/")
                    .license(License().name("Apache 2.0").url("http://springdoc.org"))
            )
    }
}
```

This `@Configuration` class defines a `@Bean` that creates an `OpenAPI` object. This object is used by `springdoc` to generate the header section of your API documentation.

-----

### Step 3: Annotate Controllers to Enhance Documentation

`springdoc-openapi` will automatically detect your Spring MVC or WebFlux controllers (`@RestController`, `@GetMapping`, etc.) and generate basic documentation. To add more detail, you can use OpenAPI annotations.

Here is an example of a simple `RestController` with OpenAPI annotations:

```kotlin
package com.yourpackage.controller

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.Parameter
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import io.swagger.v3.oas.annotations.responses.ApiResponses
import io.swagger.v3.oas.annotations.tags.Tag
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v1/greetings")
@Tag(name = "Greeting Controller", description = "Endpoints for greetings")
class GreetingController {

    data class Greeting(val message: String)

    @Operation(summary = "Get a greeting by name", description = "Returns a personalized greeting message.")
    @ApiResponses(
        value = [
            ApiResponse(
                responseCode = "200", description = "Successful operation",
                content = [Content(mediaType = "application/json", schema = Schema(implementation = Greeting::class))]
            ),
            ApiResponse(responseCode = "404", description = "Name not found", content = [Content()]),
            ApiResponse(responseCode = "500", description = "Internal server error", content = [Content()])
        ]
    )
    @GetMapping("/{name}")
    fun getGreeting(
        @Parameter(description = "The name to greet.", required = true, example = "World")
        @PathVariable name: String
    ): ResponseEntity<Greeting> {
        if (name.equals("error", ignoreCase = true)) {
            return ResponseEntity.internalServerError().build()
        }
        if (name.isBlank()) {
            return ResponseEntity.notFound().build()
        }
        return ResponseEntity.ok(Greeting("Hello, $name!"))
    }
}
```

#### Key Annotations Explained:

  * `@Tag(name = "...", description = "...")`: Groups related endpoints together in the Swagger UI.
  * `@Operation(summary = "...", description = "...")`: Describes a single endpoint (a controller method).
  * `@Parameter(description = "...", required = ..., example = "...")`: Documents a method parameter (e.g., path variables, request parameters).
  * `@ApiResponses`, `@ApiResponse`: Describe the possible responses from an endpoint, including HTTP status codes, descriptions, and the structure of the response body (`Content` and `Schema`).
  * `@Schema`: Defines the data model for request or response bodies. For Kotlin data classes, `springdoc` is smart enough to infer the schema automatically.

-----

### Step 4: Access and View the Swagger UI

With the dependency added and your application running, you can now access the interactive Swagger UI in your browser.

1.  **Run your Spring Boot application.**

2.  **Open your web browser** and navigate to the following URL:

    [http://localhost:8080/swagger-ui.html](https://www.google.com/search?q=http://localhost:8080/swagger-ui.html)

      * This URL might differ if you have changed the server port or the application's context path. For example, if your server runs on port `8081` and has a context path of `/myapp`, the URL would be `http://localhost:8081/myapp/swagger-ui.html`.

You will see the Swagger UI page, which displays all your documented endpoints. You can expand each endpoint to see its details, including parameters, response models, and an option to "Try it out" and send live requests to your running application.

You can also view the raw OpenAPI 3 specification in JSON format at:

[http://localhost:8080/v3/api-docs](https://www.google.com/search?q=http://localhost:8080/v3/api-docs)

This JSON file is what the Swagger UI uses to render the interactive documentation.

By following these steps, you have successfully integrated and configured Swagger to provide robust and interactive documentation for your Spring Boot, Kotlin, and Gradle-based application.