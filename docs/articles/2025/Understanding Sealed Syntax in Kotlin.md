---
layout: default
parent: Contents
date: 2025-07-20
nav_exclude: true
---

# Understanding Sealed Syntax in Kotlin
- TOC
{:toc}

In the world of Kotlin, the `sealed` keyword provides a powerful mechanism for creating restricted class hierarchies. This allows for more controlled and predictable code, especially when dealing with a fixed set of possible types. This concept applies to both classes and interfaces, offering a significant advantage in terms of type safety and expressiveness, particularly when used with `when` expressions.

### Understanding Sealed Syntax

At its core, the `sealed` modifier restricts the inheritance of a class or interface. When you declare a class or interface as `sealed`, you are essentially telling the compiler that all its direct subclasses or implementations are known at compile time and are located within the same module. This "closed" or "sealed" nature of the hierarchy allows the compiler to perform exhaustive checks, leading to more robust and maintainable code.

### Sealed Classes

A sealed class is an abstract class that cannot be instantiated directly. Its constructors are `protected` by default. The key characteristic of a sealed class is that all its direct subclasses must be declared in the same file.

**Key Characteristics:**

  * **Restricted Hierarchy:** Subclasses can only be defined within the same file as the sealed class itself.
  * **Abstract by Nature:** A sealed class is implicitly abstract and cannot be instantiated.
  * **Stateful Subclasses:** Unlike enums, which typically represent a set of constants, the subclasses of a sealed class can have their own state and behavior. Each subclass can be a `data class`, a regular `class`, an `object`, or even another `sealed class`.

**Example:**

A common use case for sealed classes is to represent the different states of a network request or a user interface.

```kotlin
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val exception: Exception) : Result()
    object Loading : Result()
}

fun handleResult(result: Result) {
    when (result) {
        is Result.Success -> println("Success: ${result.data}")
        is Result.Error -> println("Error: ${result.exception.message}")
        Result.Loading -> println("Loading...")
    }
}
```

In this example, `Result` can only be one of the three defined types: `Success`, `Error`, or `Loading`.

### Sealed Interfaces

Introduced in Kotlin 1.5, sealed interfaces extend the concept of restricted hierarchies to interfaces. Similar to sealed classes, all direct implementations of a sealed interface must be in the same package and module.

**Key Characteristics:**

  * **Flexible Inheritance:** A class can implement multiple sealed interfaces, which is not possible with sealed classes due to single inheritance.
  * **Known Implementations:** The compiler is aware of all the classes that implement the sealed interface at compile time.

**Example:**

Imagine you have different types of user actions that can be logged. A sealed interface can be a great way to model this.

```kotlin
sealed interface UserAction {
    fun log()
}

data class LoginAction(val username: String) : UserAction {
    override fun log() {
        println("User logged in: $username")
    }
}

data class LogoutAction(val username: String) : UserAction {
    override fun log() {
        println("User logged out: $username")
    }
}

object AppOpenedAction : UserAction {
    override fun log() {
        println("Application was opened.")
    }
}
```

### The Power of `when` Expressions

The primary benefit of using sealed syntax becomes evident when combined with `when` expressions. Because the compiler knows all the possible subtypes of a sealed class or interface, it can perform an exhaustive check. This means that you are forced to handle all possible cases in your `when` block. If you forget a case, the code will not compile, preventing potential runtime errors.

Consider the `handleResult` function from the sealed class example. If a new subclass were added to the `Result` sealed class, the `when` expression in `handleResult` would produce a compile-time error, reminding the developer to handle the new case. This eliminates the need for an `else` branch in the `when` expression, making the code cleaner and more type-safe.

### Sealed Classes vs. Enums

While sealed classes might seem similar to enums, they offer more flexibility. Here's a quick comparison:

| Feature | Enum | Sealed Class |
| :--- | :--- | :--- |
| **Instances** | Single instance per constant. | Multiple instances of subclasses can exist. |
| **State** | Constants have a fixed state. | Subclasses can hold their own state. |
| **Hierarchy** | A flat list of constants. | Can have a deeper, more complex inheritance hierarchy. |

In essence, you can think of a sealed class as an extension of an enum. If you just need a set of named constants, an enum is sufficient. However, if you need to associate data with each type or have more complex behavior, a sealed class is the better choice.

### Conclusion

Kotlin's sealed syntax provides a robust way to model restricted hierarchies, enhancing type safety and code clarity. By ensuring that all possible subtypes are known at compile time, sealed classes and interfaces enable the compiler to perform exhaustive checks, especially within `when` expressions. This leads to more predictable and maintainable code, making it a valuable tool in any Kotlin developer's arsenal for representing well-defined sets of states, events, or results.