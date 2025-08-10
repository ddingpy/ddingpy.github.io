---
layout: default
parent: Contents
date: 2025-07-18
nav_exclude: true
---

## Debugging Your Kotlin/Spring Boot Application in a Docker Container with IntelliJ IDEA
- TOC
{:toc}

Developing and debugging modern applications often involves containerization with Docker. This guide provides a comprehensive, step-by-step approach to seamlessly debug your Kotlin-based Spring Boot application running inside a Docker container using the powerful debugging tools of IntelliJ IDEA.

### Prerequisites

Before you begin, ensure you have the following installed and configured:

  * **IntelliJ IDEA Ultimate:** The remote debugging features are most robust in the Ultimate edition.
  * **Docker Desktop:** For building and running Docker containers on your local machine.
  * **A Kotlin/Spring Boot Project:** You should have a functional Spring Boot application written in Kotlin.
  * **Basic Docker Knowledge:** Familiarity with `Dockerfile` and `docker-compose.yml` concepts is beneficial.

### Step 1: Configure Your `Dockerfile` for Debugging

To enable debugging, you need to start your Spring Boot application inside the container with the Java Debug Wire Protocol (JDWP) agent enabled. This allows the JVM to listen for a debugger to attach.

Modify your `Dockerfile` to include the necessary JVM arguments. A common and effective method is to use the `JAVA_TOOL_OPTIONS` environment variable.

Here’s an example `Dockerfile`:

```dockerfile
# Use a specific Java version for reproducibility
FROM openjdk:17-jdk-slim

# Set an argument for the JAR file path
ARG JAR_FILE=build/libs/*.jar

# Set the working directory
WORKDIR /app

# Copy the JAR file into the container
COPY ${JAR_FILE} app.jar

# Expose the application port and the debug port
EXPOSE 8080
EXPOSE 5005

# Set the environment variable to enable debugging
ENV JAVA_TOOL_OPTIONS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Explanation of the `JAVA_TOOL_OPTIONS`:**

  * `-agentlib:jdwp`: Loads the JDWP agent.
  * `transport=dt_socket`: Specifies that the debugger will connect via a socket.
  * `server=y`: Tells the JVM to listen for a debugger to attach.
  * `suspend=n`: The JVM will start without waiting for the debugger to attach. If you set this to `y`, the application will not start until you attach the debugger.
  * `address=*:5005`: The JVM will listen for a debugger on port `5005`. The `*` makes it listen on all network interfaces within the container.

### Step 2: Build Your Spring Boot Application

Before building the Docker image, you need to package your Spring Boot application into a JAR file.

Open a terminal in your project's root directory and run the following Gradle command:

```bash
./gradlew bootJar
```

Or, if you are using Maven:

```bash
./mvnw package
```

This will create a JAR file in the `build/libs` (for Gradle) or `target` (for Maven) directory.

### Step 3: Create a `docker-compose.yml` File

Using Docker Compose is a convenient way to manage your container's configuration, including port mappings. Create a `docker-compose.yml` file in the root of your project:

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080" # Map the application port
      - "5005:5005" # Map the debug port
```

This configuration tells Docker to:

  * Build an image from the `Dockerfile` in the current directory.
  * Map port `8080` of the container to port `8080` on your host machine.
  * **Crucially, map port `5005` of the container to port `5005` on your host machine.** This allows IntelliJ's debugger to connect to the JVM inside the container.

### Step 4: Run the Docker Container

Now, start your application in the Docker container using Docker Compose:

```bash
docker-compose up --build
```

The `--build` flag ensures that Docker rebuilds your image with any changes you've made to the `Dockerfile` or your source code. You should see the Spring Boot application starting up in the terminal output.

### Step 5: Configure the Remote Debugger in IntelliJ IDEA

With your application running in a Docker container and the debug port exposed, the final step is to configure IntelliJ IDEA to connect to it.

1.  **Open the "Edit Run/Debug Configurations" Dialog:** In IntelliJ, go to **Run \> Edit Configurations...**.

2.  **Add a New "Remote JVM Debug" Configuration:** Click the **+** button in the top-left corner and select **Remote JVM Debug**.

3.  **Configure the Debugger:**

      * **Name:** Give your configuration a descriptive name, such as "Debug Spring Boot in Docker".
      * **Debugger mode:** Keep the default "Attach to remote JVM".
      * **Host:** Enter `localhost`.
      * **Port:** Enter `5005` (the port you exposed).
      * **Use module classpath:** Select the main module of your project from the dropdown.

    Your configuration should look something like this:

4.  **Apply and Close:** Click **Apply** and then **OK** to save the configuration.

### Step 6: Start Debugging

You are now ready to start debugging\!

1.  **Set Breakpoints:** Open any of your Kotlin source files (e.g., a controller) and set a breakpoint by clicking in the gutter next to a line of code.

2.  **Start the Debugger:** Select your newly created debug configuration from the dropdown in the top-right corner of IntelliJ and click the **Debug** button (the bug icon).

3.  **Trigger the Breakpoint:** Interact with your application to hit the code where you set the breakpoint. For example, if you set a breakpoint in a REST controller's method, send an HTTP request to that endpoint using a tool like `curl`, Postman, or your web browser.

    ```bash
    curl http://localhost:8080/your-endpoint
    ```

4.  **Debug Your Application:** IntelliJ will now suspend the execution at your breakpoint, and you can use the full power of its debugger to inspect variables, step through your code, evaluate expressions, and diagnose any issues within your containerized application.

By following these steps, you can establish a powerful and efficient workflow for developing and debugging your Kotlin/Spring Boot applications directly within Docker containers, ensuring consistency between your development and production environments.