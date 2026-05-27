# ☕ → 🎯 Java to Kotlin Migration Guide

> **Hazelnut Docs** | Java → Kotlin

This guide covers everything your team needs to migrate from Java to Kotlin. Kotlin is 100% interoperable with Java, so you can migrate incrementally — file by file, class by class.

---

## Table of Contents

1. [Variable Declarations](#variable-declarations)
2. [Null Safety](#null-safety)
3. [Functions](#functions)
4. [Classes](#classes)
5. [Data Classes](#data-classes)
6. [Inheritance & Interfaces](#inheritance--interfaces)
7. [Control Flow](#control-flow)
8. [Collections](#collections)
9. [Lambdas & Functional Programming](#lambdas--functional-programming)
10. [Extension Functions](#extension-functions)
11. [Object & Companion Object](#object--companion-object)
12. [Coroutines (Async/Await)](#coroutines-asyncawait)
13. [Sealed Classes](#sealed-classes)
14. [Generics](#generics)
15. [Destructuring](#destructuring)
16. [Scope Functions](#scope-functions)
17. [Java Interop Tips](#java-interop-tips)

---

## Variable Declarations

### Java
```java
// Mutable
String name = "Alice";
int age = 30;

// Final (immutable)
final String NAME = "Alice";
final int AGE = 30;
```

### Kotlin
```kotlin
// var = mutable, val = immutable (prefer val by default)
var name: String = "Alice"
var age: Int = 30

val NAME = "Alice"   // type inferred
val AGE = 30         // type inferred

// Explicit types
val greeting: String = "Hello"
```

> **Rule of thumb:** Always prefer `val`. Only use `var` when mutation is genuinely needed.

---

## Null Safety

This is one of the biggest quality-of-life improvements in Kotlin.

### Java
```java
String name = null;

// Must manually check
if (name != null) {
    System.out.println(name.length());
}

// Optional (Java 8+)
Optional<String> maybeName = Optional.ofNullable(name);
maybeName.ifPresent(n -> System.out.println(n.length()));
```

### Kotlin
```kotlin
// Non-nullable (cannot be null — compiler enforced)
var name: String = "Alice"
// name = null  ← Compile error!

// Nullable type uses '?'
var name: String? = null

// Safe call operator ?.
println(name?.length)           // prints null if name is null

// Elvis operator ?: (default value)
val len = name?.length ?: 0     // 0 if name is null

// Non-null assertion !! (use sparingly — throws NPE if null)
val len2 = name!!.length

// Smart cast after null check
if (name != null) {
    println(name.length)        // compiler knows name is non-null here
}

// let block for null-safe operations
name?.let { n ->
    println("Name is $n, length is ${n.length}")
}

// Safe cast
val obj: Any = "hello"
val str: String? = obj as? String   // null if cast fails
```

---

## Functions

### Java
```java
public int add(int a, int b) {
    return a + b;
}

public String greet(String name) {
    return "Hello, " + name + "!";
}

// Void method
public void printMessage(String msg) {
    System.out.println(msg);
}
```

### Kotlin
```kotlin
// Standard function
fun add(a: Int, b: Int): Int {
    return a + b
}

// Single-expression function
fun add(a: Int, b: Int) = a + b

// Unit return type (equivalent to void)
fun printMessage(msg: String) {
    println(msg)
}

// Default parameter values
fun greet(name: String = "World"): String {
    return "Hello, $name!"
}

// Named arguments
greet(name = "Alice")
greet()  // uses default → "Hello, World!"

// Varargs
fun sum(vararg numbers: Int): Int = numbers.sum()
sum(1, 2, 3, 4)

// String templates
fun greetFull(first: String, last: String) = "Hello, $first $last! Length: ${first.length}"
```

---

## Classes

### Java
```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }

    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + "}";
    }
}
```

### Kotlin
```kotlin
// Properties are auto-generated; getters/setters are implicit
class Person(var name: String, var age: Int) {

    // Custom getter
    val isAdult: Boolean
        get() = age >= 18

    // Custom setter with validation
    var email: String = ""
        set(value) {
            require(value.contains("@")) { "Invalid email" }
            field = value   // 'field' refers to the backing field
        }

    // Secondary constructor
    constructor(name: String) : this(name, 0)

    // init block (runs after primary constructor)
    init {
        require(age >= 0) { "Age cannot be negative" }
    }

    override fun toString() = "Person(name=$name, age=$age)"
}

// Usage
val person = Person("Alice", 30)
println(person.name)     // no .getName() needed
person.age = 31          // no .setAge() needed
println(person.isAdult)  // true
```

---

## Data Classes

Replaces Java's boilerplate POJOs / records — `equals()`, `hashCode()`, `toString()`, `copy()` are auto-generated.

### Java
```java
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) { this.x = x; this.y = y; }

    public int getX() { return x; }
    public int getY() { return y; }

    @Override
    public boolean equals(Object o) { /* boilerplate */ }

    @Override
    public int hashCode() { /* boilerplate */ }

    @Override
    public String toString() { return "Point{x=" + x + ", y=" + y + "}"; }
}
```

### Kotlin
```kotlin
data class Point(val x: Int, val y: Int)

// That's it. You get for free:
// - equals() / hashCode()
// - toString() → "Point(x=1, y=2)"
// - copy()
// - componentN() for destructuring

val p1 = Point(1, 2)
val p2 = p1.copy(y = 10)        // Point(x=1, y=10)
val (x, y) = p1                  // destructuring

// Nested data class
data class Address(val street: String, val city: String)
data class User(val name: String, val address: Address)

val user = User("Alice", Address("123 Main St", "Springfield"))
val movedUser = user.copy(address = user.address.copy(city = "Shelbyville"))
```

---

## Inheritance & Interfaces

### Java
```java
// Abstract class
public abstract class Shape {
    protected String color;

    public Shape(String color) { this.color = color; }

    public abstract double area();

    public String describe() {
        return color + " shape with area " + area();
    }
}

public class Circle extends Shape {
    private double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double area() { return Math.PI * radius * radius; }
}

// Interface
public interface Drawable {
    void draw();
    default String getDescription() { return "A drawable"; }
}
```

### Kotlin
```kotlin
// Classes are 'final' by default; use 'open' to allow inheritance
open class Shape(protected val color: String) {
    open fun area(): Double = 0.0

    fun describe() = "$color shape with area ${area()}"
}

class Circle(color: String, private val radius: Double) : Shape(color) {
    override fun area() = Math.PI * radius * radius
}

// Interface (can have default implementations)
interface Drawable {
    fun draw()
    fun getDescription(): String = "A drawable"   // default implementation
    val name: String                               // abstract property
}

// Implementing multiple interfaces
class MyShape(color: String) : Shape(color), Drawable {
    override val name = "MyShape"
    override fun draw() { println("Drawing $color shape") }
}

// Abstract class
abstract class Animal(val name: String) {
    abstract fun speak(): String
    fun introduce() = "I am $name and I say ${speak()}"
}

class Dog(name: String) : Animal(name) {
    override fun speak() = "Woof!"
}
```

---

## Control Flow

### Java
```java
// if-else
int max;
if (a > b) {
    max = a;
} else {
    max = b;
}

// switch
String day = "MONDAY";
switch (day) {
    case "MONDAY":
    case "TUESDAY":
        System.out.println("Weekday");
        break;
    default:
        System.out.println("Other");
}

// for loop
for (int i = 0; i < 10; i++) { }
for (String s : list) { }
```

### Kotlin
```kotlin
// if is an expression (returns a value)
val max = if (a > b) a else b

// when (replaces switch — much more powerful)
val day = "MONDAY"
when (day) {
    "MONDAY", "TUESDAY" -> println("Weekday")
    "SATURDAY", "SUNDAY" -> println("Weekend")
    else -> println("Other")
}

// when as an expression
val type = when {
    age < 13  -> "child"
    age < 18  -> "teen"
    age < 65  -> "adult"
    else      -> "senior"
}

// when with type checking
fun describe(obj: Any) = when (obj) {
    is String -> "String of length ${obj.length}"
    is Int    -> "Int: $obj"
    is List<*> -> "List with ${obj.size} elements"
    else      -> "Unknown"
}

// Ranges & for loops
for (i in 1..10) { }           // 1 to 10 inclusive
for (i in 1 until 10) { }      // 1 to 9
for (i in 10 downTo 1) { }     // 10, 9, 8...
for (i in 1..10 step 2) { }    // 1, 3, 5, 7, 9
for (item in list) { }
for ((index, item) in list.withIndex()) { }

// while
while (condition) { }
do { } while (condition)

// Ranges in conditions
if (age in 18..65) { println("Working age") }
if (name in listOf("Alice", "Bob")) { println("Known user") }
```

---

## Collections

### Java
```java
// Immutable list
List<String> names = List.of("Alice", "Bob", "Charlie");

// Mutable list
List<String> mutableNames = new ArrayList<>(Arrays.asList("Alice", "Bob"));
mutableNames.add("Dave");

// Map
Map<String, Integer> scores = new HashMap<>();
scores.put("Alice", 95);

// Stream operations
List<String> filtered = names.stream()
    .filter(n -> n.startsWith("A"))
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

### Kotlin
```kotlin
// Immutable collections (default)
val names = listOf("Alice", "Bob", "Charlie")
val scores = mapOf("Alice" to 95, "Bob" to 87)
val set = setOf(1, 2, 3)

// Mutable collections
val mutableNames = mutableListOf("Alice", "Bob")
mutableNames.add("Dave")
mutableNames.remove("Bob")

val mutableMap = mutableMapOf("Alice" to 95)
mutableMap["Bob"] = 87
mutableMap.getOrDefault("Charlie", 0)

// Collection operations (no .stream() needed!)
val filtered = names.filter { it.startsWith("A") }
val upper = names.map { it.uppercase() }
val lengths = names.map { it.length }
val total = lengths.sum()
val any = names.any { it.length > 4 }
val all = names.all { it.isNotEmpty() }
val first = names.first { it.startsWith("B") }
val firstOrNull = names.firstOrNull { it.startsWith("Z") }

// Grouping
val byLength = names.groupBy { it.length }
// → {5=[Alice, "Charlie" wait...}, etc.

// FlatMap
val lists = listOf(listOf(1,2), listOf(3,4))
val flat = lists.flatten()          // [1, 2, 3, 4]
val flatMapped = lists.flatMap { it.map { n -> n * 2 } }

// Zip
val zipped = listOf(1, 2, 3).zip(listOf("a", "b", "c"))
// → [(1, a), (2, b), (3, c)]

// Sorting
val sorted = names.sorted()
val sortedByLength = names.sortedBy { it.length }
val sortedDesc = names.sortedByDescending { it.length }

// Associate (to map)
val nameMap = names.associateBy { it.first() }
// → {A=Alice, B=Bob, C=Charlie}
```

---

## Lambdas & Functional Programming

### Java
```java
// Lambda
Runnable r = () -> System.out.println("Hello");
Function<String, Integer> len = s -> s.length();
BiFunction<Int, Int, Int> add = (a, b) -> a + b;

// Method reference
List<String> names = List.of("Alice", "Bob");
names.forEach(System.out::println);
```

### Kotlin
```kotlin
// Lambda (type inferred)
val r: () -> Unit = { println("Hello") }
val len: (String) -> Int = { s -> s.length }
val add: (Int, Int) -> Int = { a, b -> a + b }

// 'it' implicit for single parameter
val double: (Int) -> Int = { it * 2 }

// Higher-order functions
fun applyTwice(n: Int, f: (Int) -> Int) = f(f(n))
applyTwice(2, { it * 3 })   // → 18
applyTwice(2) { it * 3 }    // trailing lambda syntax

// Function references
fun square(n: Int) = n * n
val ref = ::square
listOf(1, 2, 3).map(::square)

// Member references
listOf("Alice", "Bob").map(String::length)
listOf("Alice", "Bob").forEach(::println)

// Inline lambdas for control flow
val result = run {
    val x = computeSomething()
    val y = computeOther()
    x + y   // last expression is the value
}
```

---

## Extension Functions

One of Kotlin's most powerful features — add methods to existing classes without inheritance.

### Java
```java
// Must use utility class
public class StringUtils {
    public static boolean isPalindrome(String s) {
        return s.equals(new StringBuilder(s).reverse().toString());
    }
}
StringUtils.isPalindrome("racecar");
```

### Kotlin
```kotlin
// Extend any class
fun String.isPalindrome() = this == this.reversed()
"racecar".isPalindrome()   // true

fun String.truncate(maxLen: Int) =
    if (length <= maxLen) this else take(maxLen) + "..."

fun Int.isEven() = this % 2 == 0
fun List<Int>.second() = this[1]

// Extension on nullable type
fun String?.orEmpty() = this ?: ""

// Extending third-party / standard library classes
fun <T> List<T>.second(): T = this[1]
fun <T> MutableList<T>.swap(i: Int, j: Int) {
    val temp = this[i]; this[i] = this[j]; this[j] = temp
}

// Extension properties
val String.wordCount: Int get() = split(" ").size
```

---

## Object & Companion Object

### Java
```java
// Singleton
public class Config {
    private static Config INSTANCE;
    private Config() {}
    public static Config getInstance() {
        if (INSTANCE == null) INSTANCE = new Config();
        return INSTANCE;
    }
}

// Static methods/fields
public class MathUtils {
    public static final double PI = 3.14159;
    public static int square(int n) { return n * n; }
}
```

### Kotlin
```kotlin
// Singleton — use 'object'
object Config {
    val dbUrl = "jdbc:postgresql://localhost/mydb"
    fun load() { /* ... */ }
}
Config.load()
Config.dbUrl

// Companion object (static-like members inside a class)
class MathUtils {
    companion object {
        const val PI = 3.14159
        fun square(n: Int) = n * n

        // Factory pattern
        fun create(): MathUtils = MathUtils()
    }
}
MathUtils.square(5)
MathUtils.PI
MathUtils.create()

// Named companion object
class MyClass {
    companion object Factory {
        fun create() = MyClass()
    }
}

// Anonymous object (ad-hoc implementations)
val comparator = object : Comparator<String> {
    override fun compare(a: String, b: String) = a.length - b.length
}
```

---

## Coroutines (Async/Await)

Replaces Java's `CompletableFuture`, threads, and callback hell.

### Java
```java
// CompletableFuture
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchData())
    .thenApply(data -> process(data));

// Thread
new Thread(() -> {
    String result = fetchData();
    // update UI... but you're on a background thread
}).start();
```

### Kotlin
```kotlin
// Add to build.gradle.kts:
// implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")

import kotlinx.coroutines.*

// Basic coroutine
fun main() = runBlocking {
    val result = async { fetchData() }
    println(result.await())
}

// Suspend function (can be paused without blocking a thread)
suspend fun fetchUser(id: Int): User {
    delay(100) // non-blocking delay
    return User(id, "Alice")
}

// Launch (fire and forget)
val scope = CoroutineScope(Dispatchers.IO)
scope.launch {
    val user = fetchUser(1)
    println(user)
}

// Async (returns Deferred<T> — like a future)
val deferred = scope.async { fetchUser(1) }
val user = deferred.await()

// Parallel execution
val (user, posts) = coroutineScope {
    val u = async { fetchUser(1) }
    val p = async { fetchPosts(1) }
    u.await() to p.await()
}

// Dispatchers
Dispatchers.Main    // Android/UI thread
Dispatchers.IO      // I/O operations (network, disk)
Dispatchers.Default // CPU-intensive work

// withContext — switch dispatcher mid-coroutine
suspend fun loadAndProcess(): String = withContext(Dispatchers.IO) {
    val raw = fetchFromNetwork()
    withContext(Dispatchers.Default) {
        process(raw)
    }
}

// Flow (reactive streams)
import kotlinx.coroutines.flow.*

fun numberFlow(): Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)
        emit(i)
    }
}

scope.launch {
    numberFlow()
        .filter { it % 2 == 0 }
        .map { it * 10 }
        .collect { println(it) }   // 20, 40
}
```

---

## Sealed Classes

Represent restricted class hierarchies — great for modeling states and results.

### Java
```java
// Java 17 sealed classes (limited)
public sealed interface Result<T> permits Success, Failure {}
public record Success<T>(T value) implements Result<T> {}
public record Failure<T>(String error) implements Result<T> {}
```

### Kotlin
```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val cause: Throwable? = null) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// Exhaustive when — compiler ensures all cases are covered
fun handleResult(result: Result<User>) = when (result) {
    is Result.Success -> println("Got user: ${result.data}")
    is Result.Error   -> println("Error: ${result.message}")
    Result.Loading    -> println("Loading...")
    // No 'else' needed — sealed class is exhaustive
}

// Real-world usage with network calls
suspend fun getUser(id: Int): Result<User> {
    return try {
        val user = api.fetchUser(id)
        Result.Success(user)
    } catch (e: Exception) {
        Result.Error("Failed to fetch user", e)
    }
}

// Sealed interface (Kotlin 1.5+)
sealed interface UiState {
    object Idle : UiState
    object Loading : UiState
    data class Success(val items: List<Item>) : UiState
    data class Error(val msg: String) : UiState
}
```

---

## Generics

### Java
```java
public <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) > 0 ? a : b;
}

// Wildcards
public void printList(List<? extends Number> list) { }
```

### Kotlin
```kotlin
// Basic generic function
fun <T : Comparable<T>> max(a: T, b: T) = if (a > b) a else b

// Multiple constraints
fun <T> T.toJson(): String where T : Serializable, T : Any {
    return /* serialize */ ""
}

// Covariance (out = producer, like Java ? extends)
class Box<out T>(val value: T)     // can only READ T
val intBox: Box<Int> = Box(42)
val numBox: Box<Number> = intBox   // allowed because of 'out'

// Contravariance (in = consumer, like Java ? super)
class Comparator<in T> {           // can only CONSUME T
    fun compare(a: T, b: T): Int = 0
}

// Reified type parameters (access type at runtime — impossible in Java)
inline fun <reified T> isType(value: Any) = value is T

isType<String>("hello")  // true
isType<Int>("hello")     // false

// Useful for JSON deserialization
inline fun <reified T> fromJson(json: String): T {
    return objectMapper.readValue(json, T::class.java)
}
val user = fromJson<User>("""{"name":"Alice"}""")
```

---

## Destructuring

### Kotlin
```kotlin
// Data class destructuring
data class Point(val x: Int, val y: Int)
val (x, y) = Point(10, 20)

// Pair and Triple
val (first, second) = Pair("Alice", 30)
val (a, b, c) = Triple(1, 2, 3)

// Map entries
val map = mapOf("Alice" to 95, "Bob" to 87)
for ((name, score) in map) {
    println("$name: $score")
}

// List
val list = listOf(1, 2, 3)
val (head, second, third) = list

// Skip components with _
val (_, lastName) = Pair("John", "Doe")

// Lambda parameters
val pairs = listOf(Pair("Alice", 30), Pair("Bob", 25))
pairs.forEach { (name, age) -> println("$name is $age") }
```

---

## Scope Functions

Kotlin's `let`, `run`, `with`, `apply`, `also` — for cleaner object configuration and chaining.

```kotlin
// let — transform object, null safety, result is lambda result
val result = user?.let {
    println("Processing ${it.name}")
    it.name.uppercase()
}

// apply — configure object, returns the object itself
val button = Button().apply {
    text = "Click Me"
    isEnabled = true
    setOnClickListener { doSomething() }
}

// also — side effects, returns the object itself
val list = mutableListOf(1, 2, 3)
    .also { println("Created list: $it") }

// run — like let but uses 'this', returns lambda result
val length = "Hello".run {
    println("String: $this")
    length   // returns this
}

// with — call multiple methods on same object, returns lambda result
val html = with(StringBuilder()) {
    append("<html>")
    append("<body>Hello</body>")
    append("</html>")
    toString()
}

// When to use which:
// apply / also  → configuring/building objects
// let / run     → transforming values, null checks
// with          → multiple operations on same object
```

---

## Java Interop Tips

```kotlin
// Calling Java from Kotlin — mostly seamless
val list = ArrayList<String>()    // Java class works fine
list.add("hello")

// Java getters/setters become Kotlin properties
val file = java.io.File("path.txt")
val name = file.name              // calls getName()
val length = file.length()        // method call (no backing property)

// @JvmStatic — expose companion functions as Java static
class Utils {
    companion object {
        @JvmStatic
        fun helper() = "help"
    }
}
// Now callable from Java as: Utils.helper()

// @JvmOverloads — generate Java overloads for default params
class Greeting @JvmOverloads constructor(
    val name: String = "World",
    val greeting: String = "Hello"
)

// @JvmField — expose property as Java field (no getter/setter)
class Config {
    @JvmField val MAX_SIZE = 100
}

// Kotlin nulls and Java
// Java methods return platform types (String!) — assume nullable
val javaString: String? = javaClass.returnsString()

// @Throws — declare exceptions for Java callers
@Throws(IOException::class)
fun readFile(path: String): String { /* ... */ }
```

---

## Quick Reference Cheat Sheet

| Java | Kotlin |
|------|--------|
| `String name = "x"` | `val name = "x"` |
| `final String NAME` | `val NAME` |
| `void method()` | `fun method(): Unit` |
| `class Foo extends Bar` | `class Foo : Bar()` |
| `instanceof` | `is` |
| `(String) obj` | `obj as String` |
| `x == null ? y : x` | `x ?: y` |
| `x != null ? x.y : null` | `x?.y` |
| `System.out.println()` | `println()` |
| `new Foo()` | `Foo()` |
| `static` members | `companion object` |
| Singleton | `object` |
| `switch` | `when` |
| `@Override` | `override` |
| Checked exceptions | No checked exceptions |
| `List<? extends T>` | `List<out T>` |
| `List<? super T>` | `List<in T>` |

---

*Hazelnut Docs — Java to Kotlin*