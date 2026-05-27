# 📘 → 🎯 TypeScript to Kotlin Guide

> **Hazelnut Docs** | TypeScript → Kotlin

For frontend/fullstack engineers moving to backend Kotlin. This guide maps TypeScript concepts to their Kotlin equivalents and highlights where the two diverge.

---

## Table of Contents

1. [Variables & Types](#variables--types)
2. [Type System Comparison](#type-system-comparison)
3. [Functions](#functions)
4. [Interfaces & Type Aliases](#interfaces--type-aliases)
5. [Classes](#classes)
6. [Null & Undefined vs Null Safety](#null--undefined-vs-null-safety)
7. [Generics](#generics)
8. [Union Types → Sealed Classes](#union-types--sealed-classes)
9. [Collections & Array Operations](#collections--array-operations)
10. [Async: Promises vs Coroutines](#async-promises-vs-coroutines)
11. [Destructuring](#destructuring)
12. [Modules & Imports](#modules--imports)
13. [Enums](#enums)
14. [Error Handling](#error-handling)
15. [Type Guards vs Smart Casts](#type-guards-vs-smart-casts)
16. [Utility Types Mapping](#utility-types-mapping)

---

## Variables & Types

### TypeScript
```typescript
let name: string = "Alice";
const age: number = 30;
let flag: boolean = true;
let anything: any = "hello";
let unknown: unknown = 42;

// Type inference
const greeting = "Hello";  // inferred as string
```

### Kotlin
```kotlin
var name: String = "Alice"    // var = mutable
val age: Int = 30             // val = immutable (like const)
val flag: Boolean = true
val anything: Any = "hello"   // Any ≈ object (but NOT nullable by default)

// Type inference
val greeting = "Hello"        // inferred as String

// Kotlin primitive equivalents
// TypeScript number  → Kotlin Int, Long, Double, Float
// TypeScript string  → Kotlin String
// TypeScript boolean → Kotlin Boolean
// TypeScript any     → Kotlin Any
// TypeScript unknown → Kotlin Any?  (nullable Any)
// TypeScript void    → Kotlin Unit
// TypeScript never   → Kotlin Nothing
```

---

## Type System Comparison

| TypeScript | Kotlin | Notes |
|------------|--------|-------|
| `string` | `String` | Kotlin's is always an object |
| `number` | `Int`, `Long`, `Double`, `Float` | Kotlin has distinct numeric types |
| `boolean` | `Boolean` | Same concept |
| `any` | `Any` | TS's any disables checking; Kotlin's Any is the root type |
| `unknown` | `Any?` | Must check type before use in both |
| `void` | `Unit` | Function with no meaningful return |
| `never` | `Nothing` | Function that never returns normally |
| `null` / `undefined` | `null` | Kotlin has only null (no undefined) |
| `T \| null` | `T?` | Nullable types |
| `T[]` / `Array<T>` | `List<T>` / `Array<T>` | Prefer List in Kotlin |
| `Record<K, V>` | `Map<K, V>` | |
| `[T, U]` (tuple) | `Pair<T, U>`, `Triple<T,U,V>` | Or data classes |
| `as` (type assertion) | `as` (cast) | Kotlin's throws if wrong; `as?` for safe cast |

---

## Functions

### TypeScript
```typescript
// Named function
function add(a: number, b: number): number {
    return a + b;
}

// Arrow function
const add = (a: number, b: number): number => a + b;

// Optional parameters
function greet(name: string, greeting?: string): string {
    return `${greeting ?? "Hello"}, ${name}!`;
}

// Default parameters
function greet(name: string, greeting: string = "Hello"): string {
    return `${greeting}, ${name}!`;
}

// Rest parameters
function sum(...numbers: number[]): number {
    return numbers.reduce((a, b) => a + b, 0);
}

// Overloads
function process(input: string): string;
function process(input: number): number;
function process(input: any): any {
    return input;
}
```

### Kotlin
```kotlin
// Named function
fun add(a: Int, b: Int): Int = a + b

// Lambda (assigned to variable)
val add: (Int, Int) -> Int = { a, b -> a + b }
val addInferred = { a: Int, b: Int -> a + b }

// Optional → use default values
fun greet(name: String, greeting: String = "Hello") = "$greeting, $name!"

// Named arguments (call any param by name)
greet("Alice")                      // "Hello, Alice!"
greet("Alice", greeting = "Hi")     // "Hi, Alice!"
greet(greeting = "Hey", name = "Bob")

// Varargs (rest params)
fun sum(vararg numbers: Int) = numbers.sum()
sum(1, 2, 3, 4)
val nums = intArrayOf(1, 2, 3)
sum(*nums)   // spread operator

// Kotlin doesn't have overloads like TS — use default params or sealed types
fun process(input: String) = input.uppercase()
fun process(input: Int) = input * 2
// Or with generic + reified:
inline fun <reified T> process(input: T): T = input

// Single expression functions
fun square(n: Int) = n * n
fun isPositive(n: Int) = n > 0
```

---

## Interfaces & Type Aliases

### TypeScript
```typescript
// Interface
interface User {
    readonly id: number;
    name: string;
    email?: string;   // optional
    greet(): string;
}

// Type alias
type ID = string | number;
type Callback = (error: Error | null, result?: string) => void;

// Intersection type
type Admin = User & { role: string };

// Implementing interface
class UserImpl implements User {
    constructor(
        public readonly id: number,
        public name: string,
        public email?: string
    ) {}
    greet() { return `Hi, I'm ${this.name}`; }
}
```

### Kotlin
```kotlin
// Interface (can have default implementations)
interface User {
    val id: Int          // abstract property
    var name: String
    val email: String?   // nullable = optional
    fun greet(): String
    fun describe() = "User $name"   // default implementation
}

// Type alias
typealias ID = String
typealias Callback = (Error?, String?) -> Unit
typealias UserMap = Map<Int, User>

// No intersection types — use interface composition
interface Timestamped {
    val createdAt: Long
}
interface Identifiable {
    val id: Int
}
// Implement both
class Entity(override val id: Int, override val createdAt: Long)
    : Identifiable, Timestamped

// Data class for simple value objects (replaces TS interfaces often)
data class UserData(
    val id: Int,
    val name: String,
    val email: String? = null   // optional with default
)

// Implementing interface
class UserImpl(
    override val id: Int,
    override var name: String,
    override val email: String? = null
) : User {
    override fun greet() = "Hi, I'm $name"
}
```

---

## Classes

### TypeScript
```typescript
class Animal {
    protected name: string;
    #privateField: string;     // ES private field

    constructor(name: string) {
        this.name = name;
        this.#privateField = "secret";
    }

    speak(): string {
        return `${this.name} makes a sound`;
    }

    static create(name: string): Animal {
        return new Animal(name);
    }

    get displayName(): string {
        return this.name.toUpperCase();
    }

    set displayName(value: string) {
        this.name = value.toLowerCase();
    }
}

class Dog extends Animal {
    constructor(name: string, public breed: string) {
        super(name);
    }

    override speak(): string {
        return `${this.name} barks`;
    }
}
```

### Kotlin
```kotlin
// Kotlin classes are final by default; 'open' allows inheritance
open class Animal(protected val name: String) {
    private val privateField = "secret"

    open fun speak(): String = "$name makes a sound"

    // Computed property (like TS getter)
    val displayName: String
        get() = name.uppercase()

    // Property with custom setter
    var nickname: String = name
        set(value) {
            field = value.lowercase()
        }

    companion object {
        fun create(name: String) = Animal(name)
    }
}

class Dog(name: String, val breed: String) : Animal(name) {
    override fun speak() = "$name barks"
}

// Primary constructor shorthand (no body needed for simple classes)
class Point(val x: Double, val y: Double) {
    fun distanceTo(other: Point) =
        Math.sqrt((x - other.x).pow(2) + (y - other.y).pow(2))
}

// Abstract class
abstract class Shape {
    abstract fun area(): Double
    fun describe() = "Shape with area ${area()}"
}
```

---

## Null & Undefined vs Null Safety

This is where Kotlin fundamentally differs from TypeScript.

### TypeScript
```typescript
// TypeScript has both null and undefined
let a: string | null = null;
let b: string | undefined = undefined;
let c: string | null | undefined;

// Optional chaining
const len = user?.name?.length;

// Nullish coalescing
const name = user?.name ?? "Anonymous";

// Non-null assertion
const len2 = user!.name.length;

// Type narrowing
if (user !== null && user !== undefined) {
    console.log(user.name);  // narrowed to non-null
}
```

### Kotlin
```kotlin
// Kotlin has only null (no undefined)
var a: String? = null   // nullable
var b: String = "hi"    // non-nullable — CANNOT be null

// Safe call (like optional chaining)
val len = user?.name?.length     // Int? (nullable Int)

// Elvis operator (like nullish coalescing)
val name = user?.name ?: "Anonymous"

// Non-null assertion (throws KotlinNullPointerException)
val len2 = user!!.name.length

// Smart cast after null check
if (user != null) {
    println(user.name)   // user is smart-cast to non-nullable here
}

// Safe cast
val str = anyValue as? String   // null if not a String

// let for null-safe block
user?.let { u ->
    println("User: ${u.name}")
    sendEmail(u.email)
}

// requireNotNull / checkNotNull
val safeUser = requireNotNull(user) { "User must not be null" }

// Nullable collections
val items: List<String>? = null
val size = items?.size ?: 0
val first = items?.firstOrNull()
```

---

## Generics

### TypeScript
```typescript
// Generic function
function identity<T>(value: T): T { return value; }

// Generic interface
interface Container<T> {
    value: T;
    transform<U>(fn: (v: T) => U): Container<U>;
}

// Constraints
function getLength<T extends { length: number }>(value: T): number {
    return value.length;
}

// Conditional types
type IsString<T> = T extends string ? true : false;

// Mapped types
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Partial<T> = { [K in keyof T]?: T[K] };
```

### Kotlin
```kotlin
// Generic function
fun <T> identity(value: T): T = value

// Generic class
class Container<T>(val value: T) {
    fun <U> transform(fn: (T) -> U): Container<U> = Container(fn(value))
}

// Constraints (upper bounds)
fun <T : Comparable<T>> max(a: T, b: T) = if (a > b) a else b

// Multiple constraints
fun <T> clone(value: T): T where T : Cloneable, T : Serializable {
    return value.clone() as T
}

// Variance
// 'out' = covariant (like TS 'extends' in position) — producer
class ReadOnlyBox<out T>(val value: T)

// 'in' = contravariant (like TS 'super' in position) — consumer
class WriteOnlyBox<in T> {
    fun store(value: T) { /* ... */ }
}

// Reified — access type at runtime (TS has no equivalent)
inline fun <reified T> Any.castOrNull(): T? = this as? T
inline fun <reified T> parseJson(json: String): T =
    objectMapper.readValue(json, T::class.java)

// Star projection (like TS unknown generic)
fun printList(list: List<*>) {
    list.forEach { println(it) }
}
```

---

## Union Types → Sealed Classes

TypeScript's union types are a structural concept; Kotlin uses sealed classes/interfaces for discriminated unions.

### TypeScript
```typescript
// Union type
type Result<T> = { kind: "success"; data: T } | { kind: "error"; message: string };

// Discriminated union
type Shape =
    | { kind: "circle"; radius: number }
    | { kind: "rect"; width: number; height: number };

function area(shape: Shape): number {
    switch (shape.kind) {
        case "circle": return Math.PI * shape.radius ** 2;
        case "rect":   return shape.width * shape.height;
    }
}

// Simple union
type StringOrNumber = string | number;
```

### Kotlin
```kotlin
// Sealed class (the idiomatic Kotlin discriminated union)
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

sealed class Shape {
    data class Circle(val radius: Double) : Shape()
    data class Rect(val width: Double, val height: Double) : Shape()
}

fun area(shape: Shape): Double = when (shape) {
    is Shape.Circle -> Math.PI * shape.radius * shape.radius
    is Shape.Rect   -> shape.width * shape.height
    // No 'else' needed — compiler enforces exhaustiveness
}

// For "string | number" style — use a sealed interface
sealed interface StringOrNumber {
    @JvmInline value class Str(val value: String) : StringOrNumber
    @JvmInline value class Num(val value: Int) : StringOrNumber
}

// Or just use Any with type checks (less safe)
fun process(input: Any) = when (input) {
    is String -> "String: $input"
    is Int    -> "Int: $input"
    else      -> "Unknown"
}
```

---

## Collections & Array Operations

### TypeScript
```typescript
const nums = [1, 2, 3, 4, 5];

// Array methods
nums.filter(n => n % 2 === 0);          // [2, 4]
nums.map(n => n * 2);                   // [2, 4, 6, 8, 10]
nums.reduce((acc, n) => acc + n, 0);    // 15
nums.find(n => n > 3);                  // 4
nums.findIndex(n => n > 3);             // 3
nums.some(n => n > 4);                  // true
nums.every(n => n > 0);                 // true
nums.includes(3);                       // true
nums.slice(1, 3);                       // [2, 3]
nums.flat();
nums.flatMap(n => [n, n * 2]);

// Spread
const combined = [...nums, 6, 7];

// Object spread
const user = { name: "Alice", age: 30 };
const updated = { ...user, age: 31 };

// Map & Set
const map = new Map<string, number>();
map.set("a", 1);
map.get("a");

const set = new Set([1, 2, 3]);
```

### Kotlin
```kotlin
val nums = listOf(1, 2, 3, 4, 5)   // immutable

// Collection operations (no .stream() needed)
nums.filter { it % 2 == 0 }         // [2, 4]
nums.map { it * 2 }                  // [2, 4, 6, 8, 10]
nums.reduce { acc, n -> acc + n }    // 15
nums.fold(0) { acc, n -> acc + n }   // 15 (with initial value)
nums.find { it > 3 }                 // 4
nums.indexOfFirst { it > 3 }         // 3
nums.any { it > 4 }                  // true
nums.all { it > 0 }                  // true
nums.contains(3)                     // true — or: 3 in nums
nums.subList(1, 3)                   // [2, 3]
nums.flatten()                       // (on List<List<T>>)
nums.flatMap { listOf(it, it * 2) }  // [1,2, 2,4, 3,6...]

// Combining collections
val combined = nums + listOf(6, 7)
val combined2 = buildList { addAll(nums); add(6); add(7) }

// Data class "spread" (copy with changes)
data class User(val name: String, val age: Int)
val user = User("Alice", 30)
val updated = user.copy(age = 31)

// Mutable collections
val mutable = mutableListOf(1, 2, 3)
mutable.add(4)
mutable.removeAt(0)

// Map
val map = mutableMapOf("a" to 1, "b" to 2)
map["c"] = 3
map.getOrDefault("z", 0)
map.getOrPut("d") { 4 }      // insert if absent

// Set
val set = setOf(1, 2, 3)
val mutableSet = mutableSetOf(1, 2, 3)

// Useful extras
nums.sumOf { it * 2 }
nums.maxOrNull()
nums.minOrNull()
nums.sorted()
nums.sortedBy { it }
nums.groupBy { it % 2 == 0 }  // Map<Boolean, List<Int>>
nums.chunked(2)               // [[1,2], [3,4], [5]]
nums.windowed(3)              // [[1,2,3], [2,3,4], [3,4,5]]
nums.zip(listOf("a","b","c")) // [(1,a), (2,b), (3,c)]
nums.take(3)                  // [1, 2, 3]
nums.drop(3)                  // [4, 5]
nums.distinct()               // removes duplicates
```

---

## Async: Promises vs Coroutines

### TypeScript
```typescript
// Promise
async function fetchUser(id: number): Promise<User> {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) throw new Error("Not found");
    return response.json();
}

// Promise.all
const [user, posts] = await Promise.all([
    fetchUser(1),
    fetchPosts(1)
]);

// Error handling
try {
    const user = await fetchUser(1);
} catch (error) {
    console.error(error);
}

// Promise chaining
fetchUser(1)
    .then(user => fetchPosts(user.id))
    .then(posts => console.log(posts))
    .catch(err => console.error(err));
```

### Kotlin
```kotlin
// Suspend function (like async function)
suspend fun fetchUser(id: Int): User {
    val response = httpClient.get("/api/users/$id")
    if (!response.ok) throw Exception("Not found")
    return response.body<User>()
}

// Parallel (like Promise.all)
val (user, posts) = coroutineScope {
    val u = async { fetchUser(1) }
    val p = async { fetchPosts(1) }
    u.await() to p.await()
}

// Error handling
try {
    val user = fetchUser(1)
} catch (e: Exception) {
    println("Error: ${e.message}")
}

// Result type (cleaner error handling — no try/catch)
val result = runCatching { fetchUser(1) }
result.onSuccess { user -> println(user) }
result.onFailure { error -> println(error) }
val user = result.getOrElse { User.empty() }

// Flow (like async generator / RxJS Observable)
fun userStream(): Flow<User> = flow {
    while (true) {
        emit(fetchLatestUser())
        delay(5000)
    }
}

scope.launch {
    userStream()
        .filter { it.isActive }
        .collect { user -> updateUI(user) }
}

// Timeout
withTimeout(5000) {    // throws TimeoutCancellationException
    fetchUser(1)
}
val result = withTimeoutOrNull(5000) { fetchUser(1) }  // null on timeout
```

---

## Destructuring

### TypeScript
```typescript
// Array destructuring
const [first, second, ...rest] = [1, 2, 3, 4];

// Object destructuring
const { name, age, address: { city } } = user;

// Renaming
const { name: userName } = user;

// Default values
const { name = "Anonymous" } = user;

// Function parameter destructuring
function greet({ name, age }: { name: string; age: number }) {
    return `${name} is ${age}`;
}
```

### Kotlin
```kotlin
// Destructuring declarations (works with data classes, Pair, Triple, Map entries)
val (first, second) = Pair(1, 2)
val (name, age) = Pair("Alice", 30)

data class User(val name: String, val age: Int, val city: String)
val user = User("Alice", 30, "NY")
val (name, age, city) = user

// Skip with underscore
val (_, age) = user

// In loops
for ((key, value) in map) { }
for ((index, item) in list.withIndex()) { }

// Lambda destructuring
listOf(Pair("Alice", 30)).forEach { (name, age) ->
    println("$name is $age")
}

// No rest destructuring (no ...rest equivalent)
// No nested destructuring directly — access via property

// No renaming in destructuring — use a new val
val userName = user.name
```

---

## Modules & Imports

### TypeScript
```typescript
// export
export const MAX = 100;
export function helper() {}
export default class MyClass {}

// import
import MyClass from "./MyClass";
import { helper, MAX } from "./utils";
import * as utils from "./utils";
```

### Kotlin
```kotlin
// In Kotlin, files are in packages — no default export concept
// Everything in a package is accessible within the same package

package com.hazelnut.utils

const val MAX = 100
fun helper() {}
class MyClass

// Importing
import com.hazelnut.utils.MAX
import com.hazelnut.utils.helper
import com.hazelnut.utils.*          // import all

// Aliased import
import com.hazelnut.utils.MyClass as UtilsClass

// Top-level functions and properties are first-class (no wrapping class needed)
// com/hazelnut/StringUtils.kt
package com.hazelnut

fun String.isPalindrome() = this == reversed()
val EMPTY_STRING = ""
```

---

## Enums

### TypeScript
```typescript
// Numeric enum
enum Direction { Up, Down, Left, Right }
Direction.Up   // 0

// String enum
enum Status { Active = "ACTIVE", Inactive = "INACTIVE" }

// Const enum
const enum Size { Small = 1, Medium = 2, Large = 3 }

// Accessing enum names
Object.keys(Direction)
```

### Kotlin
```kotlin
// Basic enum
enum class Direction { UP, DOWN, LEFT, RIGHT }
Direction.UP

// Enum with properties
enum class Status(val displayName: String) {
    ACTIVE("Active"),
    INACTIVE("Inactive"),
    PENDING("Pending");

    fun isActive() = this == ACTIVE
}

Status.ACTIVE.displayName    // "Active"
Status.ACTIVE.isActive()     // true
Status.ACTIVE.name           // "ACTIVE" (built-in)
Status.ACTIVE.ordinal        // 0 (built-in)

// Get enum from string
val status = Status.valueOf("ACTIVE")
val statusOrNull = runCatching { Status.valueOf("UNKNOWN") }.getOrNull()

// Iterate
Status.values().forEach { println(it.displayName) }
Status.entries.forEach { println(it) }  // Kotlin 1.9+

// When with enum (exhaustive — no else needed)
when (direction) {
    Direction.UP    -> moveUp()
    Direction.DOWN  -> moveDown()
    Direction.LEFT  -> moveLeft()
    Direction.RIGHT -> moveRight()
}
```

---

## Error Handling

### TypeScript
```typescript
// try/catch
try {
    const data = JSON.parse(input);
} catch (error) {
    if (error instanceof SyntaxError) {
        console.error("Invalid JSON:", error.message);
    }
    throw error;
}

// Custom error
class ValidationError extends Error {
    constructor(message: string, public field: string) {
        super(message);
        this.name = "ValidationError";
    }
}

// Result pattern (fp-ts / custom)
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
```

### Kotlin
```kotlin
// try/catch (no checked exceptions in Kotlin)
try {
    val data = json.parse(input)
} catch (e: JsonSyntaxException) {
    println("Invalid JSON: ${e.message}")
    throw e
} finally {
    cleanup()
}

// try as expression
val parsed = try {
    json.parse(input)
} catch (e: Exception) {
    null
}

// Custom exception
class ValidationException(
    message: String,
    val field: String
) : RuntimeException(message)

// Result type (built-in)
val result: Result<User> = runCatching { fetchUser(1) }

result.isSuccess
result.isFailure
result.getOrNull()
result.exceptionOrNull()
result.getOrDefault(User.empty())
result.getOrElse { exception -> User.empty() }
result.getOrThrow()
result.map { user -> user.name }
result.recover { exception -> User.empty() }

// Custom sealed Result (for typed errors)
sealed class AppError {
    object NetworkError : AppError()
    data class ValidationError(val field: String) : AppError()
    data class NotFound(val id: Int) : AppError()
}

sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Failure(val error: AppError) : ApiResult<Nothing>()
}
```

---

## Type Guards vs Smart Casts

### TypeScript
```typescript
// typeof guard
function process(input: string | number) {
    if (typeof input === "string") {
        return input.toUpperCase();  // narrowed to string
    }
    return input * 2;  // narrowed to number
}

// instanceof guard
if (error instanceof ValidationError) {
    console.log(error.field);  // narrowed to ValidationError
}

// User-defined type guard
function isUser(obj: any): obj is User {
    return typeof obj.name === "string";
}
```

### Kotlin
```kotlin
// Smart cast with 'is' (automatic — no extra assertion needed)
fun process(input: Any) {
    if (input is String) {
        println(input.uppercase())   // smart cast to String
        println(input.length)        // compiler knows it's String
    }
    if (input is Int) {
        println(input * 2)           // smart cast to Int
    }
}

// when + smart cast
fun describe(obj: Any) = when (obj) {
    is String -> "String: ${obj.uppercase()}"  // smart cast inside branch
    is Int    -> "Int: ${obj * 2}"
    is List<*> -> "List of ${obj.size} items"
    else      -> "Unknown: $obj"
}

// Contract (user-defined smart cast — advanced)
fun requireString(value: Any): String {
    require(value is String) { "Expected String, got ${value::class.simpleName}" }
    return value  // smart cast
}
```

---

## Utility Types Mapping

| TypeScript | Kotlin Equivalent |
|------------|-------------------|
| `Partial<T>` | Data class with all nullable fields or use a builder |
| `Required<T>` | All fields non-nullable (default) |
| `Readonly<T>` | `val` properties / immutable `List` |
| `Record<K, V>` | `Map<K, V>` |
| `Pick<T, K>` | New data class with subset of fields |
| `Omit<T, K>` | New data class without the field |
| `Exclude<T, U>` | Sealed class / filtering |
| `Extract<T, U>` | Sealed class / filtering |
| `ReturnType<F>` | Not directly; use `: TypeName` annotation |
| `Parameters<F>` | Reflection or explicit types |
| `Awaited<T>` | The type returned by `suspend fun` |
| `NonNullable<T>` | `T` (just don't use `?`) |

---

## Quick Reference

| TypeScript | Kotlin |
|------------|--------|
| `let x = 5` | `var x = 5` |
| `const x = 5` | `val x = 5` |
| `undefined` / `null` | `null` only |
| `T \| null` | `T?` |
| `?.` (optional chain) | `?.` (safe call) |
| `??` (nullish coalescing) | `?:` (Elvis) |
| `!` (non-null assert) | `!!` |
| `as Type` (assertion) | `as Type` / `as? Type` |
| `instanceof` | `is` |
| `interface` | `interface` / `data class` |
| `type Alias = ...` | `typealias Alias = ...` |
| `enum` | `enum class` |
| `async/await` | `suspend` + coroutines |
| `Promise<T>` | `Deferred<T>` / `suspend fun` |
| `Array<T>.map()` | `List<T>.map { }` |
| `...spread` | `+` operator / `addAll()` |
| `{...obj, key: val}` | `obj.copy(key = val)` |
| Union type `A \| B` | `sealed class` |
| Template literal `` `Hi ${name}` `` | `"Hi $name"` |

---

*Hazelnut Docs — TypeScript to Kotlin*