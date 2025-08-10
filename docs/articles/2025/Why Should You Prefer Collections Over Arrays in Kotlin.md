---
layout: default
parent: Contents
date: 2025-07-14
nav_exclude: true
---
# Why Should You Prefer Collections Over Arrays in Kotlin?
- TOC
{:toc}

### The Core Difference: Arrays vs. Collections

At a high level, both arrays and collections are used to hold multiple items. However, they have fundamental differences in their structure, capabilities, and how you interact with them.

#### Arrays (`Array<T>`)

  * **Fixed Size:** This is the most critical characteristic of an array. Once you create an array of a certain size, you cannot change that size. If you need to add more elements than its capacity, you have to create a new, larger array and copy the elements over.
  * **Mutable Elements:** The elements within an array are mutable. You can change the value at any index (e.g., `myArray[2] = "new value"`).
  * **Performance:** Arrays can have a slight performance advantage in some specific, low-level scenarios because they map directly to a contiguous block of memory. This can lead to faster access, especially with primitive types (`IntArray`, `CharArray`, etc.), as it avoids the "boxing" of primitives into objects.
  * **Limited API:** The built-in functions available for an array are relatively basic. You have `size`, `get()`, and `set()`, but not the rich set of manipulation functions that collections offer.

**Example of an Array:**

```kotlin
val numbers = arrayOf(1, 2, 3) // Creates an array of size 3
numbers[0] = 10 // This is fine, you are changing an element
// numbers.add(4) // This will not compile. You cannot change the size.
```

#### Collections (`List`, `Set`, `Map`)

  * **Flexible Size (for mutable collections):** This is the primary reason for their preference. Collections come in two main flavors:
      * **Read-only:** `List`, `Set`, `Map`. Their size and elements are fixed after creation.
      * **Mutable:** `MutableList`, `MutableSet`, `MutableMap`. You can easily add, remove, or reorder elements. This dynamic nature is what you usually need in application development.
  * **Rich API:** The Kotlin standard library provides a vast and powerful set of extension functions for all collections. These functions allow you to perform complex operations in a concise and readable way, such as `map`, `filter`, `forEach`, `groupBy`, `sorted`, and many more. This encourages a more functional and declarative style of programming.
  * **Abstraction:** Collections are interfaces (`List`, `Set`, `Map`). This means you code against the interface, and you can easily swap out the underlying implementation (e.g., from an `ArrayList` to a `LinkedList`) without changing your code, depending on your specific performance needs.

**Example of a Collection (a Mutable List):**

```kotlin
val names = mutableListOf("Alice", "Bob") // Creates a mutable list
names[0] = "Alicia" // This is fine
names.add("Charlie") // This is also fine! The list now has 3 elements.

// You can use the rich API:
val longNames = names.filter { it.length > 4 }
println(longNames) // Output: [Alicia, Charlie]
```

### Why Prefer Collections by Default?

Here is a summary of the reasons behind the recommendation:

| Feature | Arrays (`Array<T>`) | Collections (`List<T>`, `Set<T>`, etc.) | Why Collections are Preferred |
| :--- | :--- | :--- | :--- |
| **Size** | **Fixed**. Cannot be changed after creation. | **Dynamic** (for mutable collections). Can grow and shrink as needed. | Most application logic requires adding or removing items, which is cumbersome with arrays. |
| **API** | **Limited**. Basic get/set/size operations. | **Rich & Powerful**. Huge number of extension functions (`filter`, `map`, `sorted`, etc.). | The rich API makes code more expressive, concise, and less error-prone. You write what you want to achieve, not how to do it. |
| **Immutability** | Elements are always mutable. The array itself is a mutable container. | **Clear separation**. `List` is read-only. `MutableList` is mutable. | This design makes your code safer. You can expose a read-only `List` to prevent unintended modifications. |
| **Abstraction** | Concrete implementation. | **Interface-based**. You program against `List`, not `ArrayList`. | This provides flexibility to change the underlying data structure without affecting the rest of the code. |

### When Should You Use Arrays?

Despite the general recommendation, there are specific situations where arrays are the right choice:

1.  **Performance-Critical, Low-Level Code:** When you are working with a large number of primitive types (like `Int`, `Double`, `Boolean`) and you need to squeeze out every last bit of performance, the specialized primitive arrays (`IntArray`, `DoubleArray`, etc.) are ideal. They avoid the overhead of boxing primitives into wrapper objects (`Integer`, `Double`).

2.  **Interoperability with Java:** When you are calling a Java library or framework that requires an array as a parameter, you will need to provide one.

3.  **Varargs:** When you define a function that accepts a variable number of arguments, Kotlin uses an array under the hood for the `vararg` parameter.

### Summary

Think of it this way:

  * **Start with `List` or `MutableList` by default.** For 95% of your programming tasks, a collection will be more convenient, readable, and flexible.
  * **Reach for an `Array` only when you have a specific, justifiable reason to do so,** such as a clear performance bottleneck with primitives or a requirement from a Java API.

The advice from "Kotlin in Action" encourages you to use the more modern, powerful, and safer tool (collections) for your general-purpose work, while still acknowledging that the older, more primitive tool (arrays) has its specific, necessary uses.