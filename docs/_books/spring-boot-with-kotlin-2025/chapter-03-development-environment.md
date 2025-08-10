---
layout: default
---
# Chapter 03: Setting Up the Development Environment

Getting your development environment right from the start will save you countless hours of frustration. In this chapter, we'll walk through setting up a professional Kotlin and Spring Boot development environment that will serve you well throughout your journey.

## 3.1 Installing Java JDK

Spring Boot 3.x requires Java 17 or later. While you could use any JDK distribution, we recommend either Amazon Corretto or Eclipse Temurin for their long-term support and excellent performance.

### Choosing the Right JDK Version

For Spring Boot development with Kotlin, we recommend:
- **Minimum**: Java 17 (LTS)
- **Recommended**: Java 21 (Latest LTS)
- **Experimental**: Java 22+ (For trying new features)

### Installation Methods

**macOS (using Homebrew):**
```bash
# Install Homebrew if you haven't already
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Java 21 (Temurin)
brew install --cask temurin@21

# Or install Corretto
brew install --cask corretto@21

# Verify installation
java -version
```

**Windows (using Scoop or Chocolatey):**
```powershell
# Using Scoop
scoop bucket add java
scoop install temurin21-jdk

# Using Chocolatey
choco install temurin21

# Verify installation
java -version
```

**Linux (Ubuntu/Debian):**
```bash
# Add Adoptium repository
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo apt-key add -
echo "deb https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list

# Update and install
sudo apt update
sudo apt install temurin-21-jdk

# Verify installation
java -version
```

### Managing Multiple JDK Versions

If you work on multiple projects requiring different Java versions, consider using a JDK version manager:

**Using SDKMAN! (Recommended for Unix-like systems):**
```bash
# Install SDKMAN!
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# List available Java versions
sdk list java

# Install specific versions
sdk install java 21.0.1-tem  # Temurin 21
sdk install java 17.0.9-tem  # Temurin 17

# Switch between versions
sdk use java 21.0.1-tem
sdk default java 21.0.1-tem  # Set as default

# Verify current version
java -version
```

**Using jEnv (Alternative for macOS/Linux):**
```bash
# Install jEnv
brew install jenv  # macOS
git clone https://github.com/jenv/jenv.git ~/.jenv  # Linux

# Add to shell profile
echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(jenv init -)"' >> ~/.zshrc

# Add JDK installations
jenv add /Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home
jenv add /Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home

# Set global version
jenv global 21.0

# Set project-specific version
cd my-project
jenv local 17.0
```

### Setting JAVA_HOME

Ensure JAVA_HOME is properly set in your environment:

**macOS/Linux (.zshrc or .bashrc):**
```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 21)  # macOS
export JAVA_HOME=/usr/lib/jvm/temurin-21-jdk-amd64  # Linux
export PATH=$JAVA_HOME/bin:$PATH
```

**Windows (System Environment Variables):**
1. Open System Properties → Advanced → Environment Variables
2. Add new System Variable:
   - Variable name: `JAVA_HOME`
   - Variable value: `C:\Program Files\Eclipse Adoptium\jdk-21.0.1.12-hotspot`
3. Add `%JAVA_HOME%\bin` to PATH

## 3.2 Installing IntelliJ IDEA

IntelliJ IDEA is the gold standard for Kotlin and Spring Boot development. The integration is seamless, and the productivity features are unmatched.

### Choosing the Right Edition

- **Community Edition**: Free, sufficient for basic Spring Boot development
- **Ultimate Edition**: Paid, includes Spring-specific features, database tools, and advanced frameworks support

For professional Spring Boot development, we recommend Ultimate Edition for features like:
- Spring Boot run configurations
- Application properties assistance
- Database tools
- HTTP client
- Advanced debugging tools

### Installation Steps

**Download and Install:**
1. Visit [https://www.jetbrains.com/idea/download/](https://www.jetbrains.com/idea/download/)
2. Download the appropriate version for your OS
3. Install following the platform-specific instructions

**Using JetBrains Toolbox (Recommended):**
```bash
# The Toolbox App manages all JetBrains IDEs
# Download from: https://www.jetbrains.com/toolbox-app/

# Benefits:
# - Easy updates
# - Multiple versions
# - Project management
# - Settings sync
```

### Initial Configuration

When you first launch IntelliJ IDEA:

1. **Choose UI Theme**: Darcula (dark) or Light
2. **Configure Keymap**: Keep IntelliJ IDEA Classic or choose from Eclipse/VS Code
3. **Install Essential Plugins** (we'll cover this in detail)
4. **Configure JDK**: Point to your installed JDK

### Essential IntelliJ Settings for Spring Boot

**Optimize imports:**
```
Settings → Editor → General → Auto Import
☑ Add unambiguous imports on the fly
☑ Optimize imports on the fly
```

**Code style for Kotlin:**
```
Settings → Editor → Code Style → Kotlin
- Set from... → Kotlin style guide
- Hard wrap at: 120 columns
- Use tab character: No (use 4 spaces)
```

**Enable annotation processing:**
```
Settings → Build, Execution, Deployment → Compiler → Annotation Processors
☑ Enable annotation processing
```

**Increase memory for better performance:**
```
Help → Edit Custom VM Options
-Xms2048m
-Xmx4096m
-XX:ReservedCodeCacheSize=512m
```

## 3.3 Setting Up Kotlin in IntelliJ IDEA

While IntelliJ IDEA comes with excellent Kotlin support out of the box, let's ensure everything is properly configured.

### Kotlin Plugin Configuration

The Kotlin plugin is bundled with IntelliJ IDEA, but let's verify it's updated:

1. Go to `Settings → Plugins → Installed`
2. Search for "Kotlin"
3. Ensure it's enabled and updated to the latest version
4. Restart IDE if you updated the plugin

### Kotlin Compiler Settings

Configure the Kotlin compiler for optimal Spring Boot development:

```
Settings → Build, Execution, Deployment → Compiler → Kotlin Compiler

Target JVM version: 21 (or your Java version)
Language version: 1.9 (or latest stable)
API version: 1.9 (or latest stable)

Additional command line parameters:
-Xjsr305=strict  # Strict nullability for Spring annotations
-java-parameters  # Preserve parameter names for Spring
```

### Kotlin Code Style

Configure Kotlin code style to match common conventions:

```
Settings → Editor → Code Style → Kotlin

Tabs and Indents:
- Use tab character: No
- Tab size: 4
- Indent: 4
- Continuation indent: 4

Blank Lines:
- Keep maximum blank lines: 1
- Minimum blank lines after package: 1

Wrapping and Braces:
- Class annotations: Wrap always
- Method annotations: Wrap always
- Field annotations: Do not wrap
```

### Project Structure for Kotlin

Set up the standard project structure:

```
my-spring-boot-app/
├── src/
│   ├── main/
│   │   ├── kotlin/
│   │   │   └── com/example/app/
│   │   │       ├── Application.kt
│   │   │       ├── config/
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       └── model/
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       └── static/
│   └── test/
│       ├── kotlin/
│       │   └── com/example/app/
│       └── resources/
├── build.gradle.kts
└── settings.gradle.kts
```

## 3.4 Enabling Kotlin Plugin

Let's ensure all necessary Kotlin-related plugins are installed and configured for optimal Spring Boot development.

### Essential Kotlin Plugins

**1. Kotlin Plugin (Built-in)**
Already included, provides core Kotlin support.

**2. Spring Boot Plugin**
For IntelliJ Ultimate:
```
Settings → Plugins → Marketplace
Search: "Spring Boot"
Install: Spring Boot (by JetBrains)
```

**3. Additional Helpful Plugins:**

```
Recommended plugins to install:

1. "Kotlin Fill Class" - Auto-generate data class constructors
2. "JSON to Kotlin Class" - Convert JSON to Kotlin data classes
3. ".env files support" - Environment variable management
4. "Rainbow Brackets" - Colored bracket matching
5. "Key Promoter X" - Learn keyboard shortcuts
6. "String Manipulation" - Text transformation utilities
7. "GitToolBox" - Enhanced Git integration
8. "Kotest" - If using Kotest for testing
```

### Gradle Configuration for Kotlin

Configure your `build.gradle.kts` for optimal Kotlin and Spring Boot integration:

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"
    kotlin("plugin.jpa") version "1.9.21"  // If using JPA
    kotlin("kapt") version "1.9.21"  // For annotation processing
}

group = "com.example"
version = "0.0.1-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot starters
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    
    // Kotlin specific
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    
    // Kotlin coroutines for async
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor")
    
    // Development tools
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    
    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
    testImplementation("io.kotest.extensions:kotest-extensions-spring:1.1.3")
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs = listOf(
            "-Xjsr305=strict",  // Strict null-safety for Java interop
            "-Xemit-jvm-type-annotations"  // Emit JVM type annotations
        )
        jvmTarget = "21"
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}

// Configure Spring Boot plugin
springBoot {
    buildInfo()  // Generate build info for Actuator
}

// Kotlin-specific configurations
kotlin {
    jvmToolchain(21)  // Ensure Kotlin uses Java 21
}

// All-open plugin configuration for Spring
allOpen {
    annotation("org.springframework.stereotype.Component")
    annotation("org.springframework.stereotype.Service")
    annotation("org.springframework.stereotype.Repository")
    annotation("org.springframework.stereotype.Controller")
    annotation("org.springframework.stereotype.RestController")
    annotation("org.springframework.boot.context.properties.ConfigurationProperties")
}

// No-arg plugin for JPA entities
noArg {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}
```

### IntelliJ Run Configurations

Set up run configurations for different scenarios:

**1. Development Configuration:**
```
Run → Edit Configurations → + → Spring Boot

Name: Application (Dev)
Main class: com.example.app.ApplicationKt
Use classpath of module: app.main
Environment variables: SPRING_PROFILES_ACTIVE=dev
Working directory: $MODULE_WORKING_DIR$
```

**2. Debug Configuration with DevTools:**
```
Run → Edit Configurations → + → Spring Boot

Name: Application (Debug)
Main class: com.example.app.ApplicationKt
VM options: -Dspring.devtools.restart.enabled=true
           -Dspring.devtools.livereload.enabled=true
Program arguments: --debug
```

**3. Test Configuration:**
```
Run → Edit Configurations → + → JUnit

Name: All Tests
Test kind: All in package
Package: com.example.app
Use classpath of module: app.test
```

### Productivity Tips and Shortcuts

**Essential IntelliJ Shortcuts for Spring Boot Development:**

```
Navigation:
Ctrl+N (Cmd+O)         - Go to class
Ctrl+Shift+N           - Go to file
Ctrl+Alt+Shift+N       - Go to symbol
Ctrl+B (Cmd+B)         - Go to declaration
Ctrl+Alt+B             - Go to implementation
Alt+F7                 - Find usages

Code Generation:
Alt+Insert             - Generate (constructor, getters, etc.)
Ctrl+O                 - Override methods
Ctrl+I                 - Implement methods
Ctrl+Alt+V             - Extract variable
Ctrl+Alt+M             - Extract method

Spring-Specific:
Ctrl+Shift+F12         - Maximize editor
Double Shift           - Search everywhere
Ctrl+Shift+T           - Navigate to test
Ctrl+Alt+T             - Surround with (try-catch, if, etc.)

Refactoring:
Shift+F6               - Rename
Ctrl+F6                - Change signature
F6                     - Move
Ctrl+Alt+N             - Inline
```

### Setting Up Terminal and Version Control

**Configure Terminal:**
```
Settings → Tools → Terminal

Shell path:
- Windows: C:\Program Files\Git\bin\bash.exe (Git Bash)
- macOS: /bin/zsh
- Linux: /bin/bash

Start directory: $PROJECT_DIR$
```

**Git Configuration:**
```
Settings → Version Control → Git

Path to Git executable: (auto-detect or specify)
☑ Use credential helper
☑ Warn if CRLF line separators are about to be committed
```

### Database Tools (Ultimate Edition)

Configure database connections for development:

```
View → Tool Windows → Database

+ → Data Source → PostgreSQL

Host: localhost
Port: 5432
Database: myapp_dev
User: myapp
Password: ****

Test Connection → OK → Apply
```

### HTTP Client Setup

IntelliJ's HTTP Client is excellent for testing APIs:

Create `http-requests/api-tests.http`:
```http
### Get all users
GET {{host}}/api/users
Accept: application/json
Authorization: Bearer {{auth-token}}

### Create new user
POST {{host}}/api/users
Content-Type: application/json

{
  "username": "john_doe",
  "email": "john@example.com",
  "role": "USER"
}

### Update user
PUT {{host}}/api/users/1
Content-Type: application/json
Authorization: Bearer {{auth-token}}

{
  "email": "newemail@example.com"
}
```

Create `http-client.env.json`:
```json
{
  "dev": {
    "host": "http://localhost:8080",
    "auth-token": "dev-token-here"
  },
  "prod": {
    "host": "https://api.example.com",
    "auth-token": "prod-token-here"
  }
}
```

### Performance Optimization

**IDE Performance Settings:**
```
Help → Edit Custom VM Options

-Xms2g
-Xmx4g
-XX:ReservedCodeCacheSize=512m
-XX:+UseG1GC
-XX:+UseStringDeduplication
-XX:+ParallelRefProcEnabled
-Dsun.io.useCanonCaches=false
-Djava.net.preferIPv4Stack=true
```

**Exclude Folders from Indexing:**
```
Right-click on folder → Mark Directory as → Excluded

Common exclusions:
- build/
- .gradle/
- out/
- target/
- node_modules/ (if you have frontend code)
```

### Troubleshooting Common Issues

**Issue: Kotlin version conflicts**
```kotlin
// In build.gradle.kts, ensure all Kotlin plugins use the same version
plugins {
    val kotlinVersion = "1.9.21"
    kotlin("jvm") version kotlinVersion
    kotlin("plugin.spring") version kotlinVersion
    kotlin("plugin.jpa") version kotlinVersion
}
```

**Issue: IntelliJ not recognizing Spring annotations**
```
File → Invalidate Caches and Restart
Settings → Build, Execution, Deployment → Build Tools → Gradle
Use Gradle from: 'wrapper' task in Gradle build script
Run tests using: IntelliJ IDEA
```

**Issue: Slow IDE performance**
```
1. Increase memory allocation
2. Disable unnecessary plugins
3. Exclude build directories from indexing
4. Use Power Save Mode for battery preservation
5. File → Invalidate Caches and Restart
```

## Summary

You now have a professional development environment configured for Spring Boot and Kotlin development. We've covered:

- **JDK Installation**: Setting up Java 21 with proper environment variables and version management
- **IntelliJ IDEA Configuration**: Installing and optimizing the IDE for maximum productivity
- **Kotlin Setup**: Configuring the Kotlin plugin, compiler settings, and code style for Spring Boot development
- **Essential Plugins**: Installing and configuring plugins that enhance your development experience
- **Build Configuration**: Setting up Gradle with Kotlin DSL for optimal Spring Boot integration
- **Productivity Tools**: Configuring shortcuts, database tools, HTTP client, and performance optimizations

With this solid foundation, you're ready to start building Spring Boot applications. In the next chapter, we'll create our first Spring Boot application and explore the project structure in detail.