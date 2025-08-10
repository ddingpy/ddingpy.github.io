---
layout: default
parent: Contents
date: 2025-07-16
nav_exclude: true
---

## RestTemplateBuilder: A Modern Approach to HTTP Clients in Spring Boot
- TOC
{:toc}

In the landscape of Java's Spring Boot framework, making HTTP requests to external services is a cornerstone of building interconnected microservices and applications. While the classic `RestTemplate` has long been the go-to tool for this task, Spring Boot offers a more powerful and elegant solution: the `RestTemplateBuilder`. This builder utility simplifies the configuration and creation of `RestTemplate` instances, promoting cleaner code and centralized management of HTTP client settings.

### What is RestTemplateBuilder?

At its core, `RestTemplateBuilder` is a factory for creating and customizing `RestTemplate` objects. In a Spring Boot application, a `RestTemplateBuilder` bean is automatically configured and made available for injection, carrying with it sensible defaults and the ability to be further tailored to specific needs. This eliminates the need for manual and often verbose setup of `RestTemplate`, allowing developers to focus on the business logic of their applications.

### The Advantages of Using RestTemplateBuilder

Using `RestTemplateBuilder` offers several distinct advantages over creating `RestTemplate` instances directly:

  * **Simplified Configuration:** The builder provides a fluent API for setting various properties of the `RestTemplate`, such as connection and read timeouts, message converters for handling different data formats (like JSON and XML), and interceptors for modifying requests and responses. This approach is more readable and less error-prone than manual configuration.
  * **Centralized Customization:** `RestTemplateBuilder` encourages the centralization of `RestTemplate` configurations. By defining a `RestTemplate` bean using the builder in a configuration class, you can ensure that all parts of your application use a consistently configured HTTP client.
  * **Automatic Defaults:** Spring Boot's auto-configured `RestTemplateBuilder` comes with a pre-configured set of `HttpMessageConverter` instances, making it ready to handle common data types out of the box.
  * **Seamless Integration with Testing:** Spring Boot provides excellent testing support for code that uses `RestTemplateBuilder`, allowing for the easy mocking of external services in your tests.

### Practical Usage and Common Configurations

Getting started with `RestTemplateBuilder` is straightforward. You can inject it directly into your services or create a `RestTemplate` bean in a configuration class.

#### Basic Instantiation

Here's a simple example of how to use `RestTemplateBuilder` to create a `RestTemplate` instance within a service:

```java
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class MyService {

    private final RestTemplate restTemplate;

    public MyService(RestTemplateBuilder restTemplateBuilder) {
        this.restTemplate = restTemplateBuilder.build();
    }

    // ... methods that use restTemplate
}
```

For application-wide use, it's best practice to define a `RestTemplate` bean in a configuration class:

```java
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.build();
    }
}
```

#### Common Configuration Examples

The true power of `RestTemplateBuilder` lies in its ability to easily configure various aspects of the `RestTemplate`.

**Setting Timeouts:**

To prevent your application from being blocked by unresponsive services, it's crucial to set connection and read timeouts.

```java
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;
import java.time.Duration;

@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
                .setConnectTimeout(Duration.ofSeconds(5))
                .setReadTimeout(Duration.ofSeconds(5))
                .build();
    }
}
```

**Adding a Root URI:**

If you frequently make requests to the same base URL, you can set a root URI to simplify your request calls.

```java
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
                .rootUri("https://api.example.com")
                .build();
    }
}
```

Now, when you use this `RestTemplate`, you only need to provide the path relative to the root URI: `restTemplate.getForObject("/users/{id}", User.class, 1L);`

**Configuring Basic Authentication:**

For services that require basic authentication, you can easily configure the credentials.

```java
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
                .basicAuthentication("username", "password")
                .build();
    }
}
```

### Advanced Customization with `RestTemplateCustomizer`

For more advanced or application-wide customizations, Spring Boot offers the `RestTemplateCustomizer` interface. You can create beans of this type to apply customizations to the auto-configured `RestTemplateBuilder`. This is particularly useful for adding interceptors or message converters that should be applied globally.

For example, to add a custom logging interceptor to all `RestTemplate` instances created by the builder:

```java
import org.springframework.boot.web.client.RestTemplateCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.ClientHttpRequestInterceptor;

@Configuration
public class RestTemplateCustomizationConfig {

    @Bean
    public RestTemplateCustomizer restTemplateCustomizer() {
        return restTemplate -> {
            restTemplate.getInterceptors().add(loggingInterceptor());
        };
    }

    @Bean
    public ClientHttpRequestInterceptor loggingInterceptor() {
        return (request, body, execution) -> {
            // Log request details
            return execution.execute(request, body);
        };
    }
}
```

### Testing with `RestTemplateBuilder`

Spring Boot's testing framework seamlessly integrates with `RestTemplateBuilder`. The `@RestClientTest` annotation is a powerful tool for testing code that makes HTTP calls. It auto-configures a `MockRestServiceServer` which allows you to define mock responses for specific requests.

Here's an example of how to test a service that uses a `RestTemplate` created by a `RestTemplateBuilder`:

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.client.RestClientTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.client.MockRestServiceServer;

import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;

@RestClientTest(MyService.class)
public class MyServiceTest {

    @Autowired
    private MyService myService;

    @Autowired
    private MockRestServiceServer server;

    @Test
    void testGetUser() {
        String userJson = "{\"id\": 1, \"name\": \"John Doe\"}";

        this.server.expect(requestTo("/users/1"))
                .andRespond(withSuccess(userJson, MediaType.APPLICATION_JSON));

        // Call the service method that uses RestTemplate
        // and assert the result
    }
}
```

In this test, `@RestClientTest` focuses on the `MyService` and provides a `MockRestServiceServer`. We then set an expectation on the server that a request to `/users/1` will be made and should be responded to with a success status and a predefined JSON body. This allows for isolated and reliable testing of your service's logic without making actual network calls.

In conclusion, `RestTemplateBuilder` is an indispensable tool in the modern Spring Boot developer's arsenal. It promotes best practices for creating and managing `RestTemplate` instances, leading to more robust, maintainable, and easily testable applications. By leveraging the power of the builder and its associated testing utilities, you can confidently integrate your Spring Boot applications with the vast world of external RESTful services.