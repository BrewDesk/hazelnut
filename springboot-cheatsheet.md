# 🌱 Spring Boot + Kotlin Cheat Sheet

> **Hazelnut Docs** | Spring Boot

Everything you need to build production Spring Boot services in Kotlin. All examples use Kotlin idioms — no Java-style boilerplate.

---

## Table of Contents

1. [Project Setup](#project-setup)
2. [Application Entry Point](#application-entry-point)
3. [REST Controllers](#rest-controllers)
4. [Request Handling](#request-handling)
5. [Response Types](#response-types)
6. [Services & Dependency Injection](#services--dependency-injection)
7. [Spring Data JPA (PostgreSQL / MySQL)](#spring-data-jpa-postgresql--mysql)
8. [Redis](#redis)
9. [MongoDB](#mongodb)
10. [Exception Handling](#exception-handling)
11. [Validation](#validation)
12. [Configuration](#configuration)
13. [Security & JWT](#security--jwt)
14. [WebSocket / STOMP](#websocket--stomp)
15. [Testing](#testing)
16. [Coroutines with Spring](#coroutines-with-spring)
17. [Quick Reference](#quick-reference)

---

## Project Setup

### build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.5"
    kotlin("jvm") version "1.9.24"
    kotlin("plugin.spring") version "1.9.24"   // makes classes open for Spring proxies
    kotlin("plugin.jpa") version "1.9.24"       // no-arg constructor for JPA entities
}

group = "com.example"
version = "0.0.1-SNAPSHOT"

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    implementation("org.springframework.boot:spring-boot-starter-cache")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")

    // Coroutines (optional — for reactive / suspend support)
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor")
    implementation("io.projectreactor.kotlin:reactor-kotlin-extensions")

    runtimeOnly("org.postgresql:postgresql")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}

kotlin {
    compilerOptions {
        freeCompilerArgs.addAll("-Xjsr305=strict")  // strict null checks for Spring annotations
    }
}
```

### application.yml

```yaml
spring:
  application:
    name: my-service

  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASS}

  jpa:
    hibernate:
      ddl-auto: validate        # never use 'create' or 'create-drop' in prod
    show-sql: false
    properties:
      hibernate.format_sql: true

  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379
      password: ${REDIS_PASS:}
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 16
          max-idle: 8

    mongodb:
      uri: ${MONGO_URI:mongodb://localhost:27017/mydb}

  cache:
    type: redis
    redis:
      time-to-live: 600000      # 10 minutes in ms (default TTL)

server:
  port: 8080

logging:
  level:
    com.example: DEBUG
```

---

## Application Entry Point

```kotlin
// No boilerplate needed — one annotation, one function
@SpringBootApplication
class MyServiceApplication

fun main(args: Array<String>) {
    runApplication<MyServiceApplication>(*args)
}
```

> **Kotlin difference:** `main` is a top-level function, not a class method. `runApplication` is the Kotlin-friendly alternative to `SpringApplication.run(...)`.

---

## REST Controllers

```kotlin
@RestController                          // @Controller + @ResponseBody
@RequestMapping("/api/v1/users")
class UserController(
    private val userService: UserService  // constructor injection — no @Autowired needed
) {

    @GetMapping
    fun getAll(): List<UserResponse> =
        userService.findAll()

    @GetMapping("/{id}")
    fun getById(@PathVariable id: Long): UserResponse =
        userService.findById(id)

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun create(@Valid @RequestBody request: CreateUserRequest): UserResponse =
        userService.create(request)

    @PutMapping("/{id}")
    fun update(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateUserRequest
    ): UserResponse = userService.update(id, request)

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun delete(@PathVariable id: Long) =
        userService.delete(id)
}
```

---

## Request Handling

### Path Variables & Query Parameters

```kotlin
@GetMapping("/{id}")
fun getUser(@PathVariable id: Long): UserResponse { }

@GetMapping("/{userId}/posts/{postId}")
fun getPost(
    @PathVariable userId: Long,
    @PathVariable postId: Long
): PostResponse { }

// Query parameters: GET /users?page=0&size=20&sort=name
@GetMapping
fun list(
    @RequestParam(defaultValue = "0") page: Int,
    @RequestParam(defaultValue = "20") size: Int,
    @RequestParam(required = false) sort: String?
): Page<UserResponse> { }

// Spring Data Pageable shorthand
@GetMapping
fun list(pageable: Pageable): Page<UserResponse> { }
```

### Request Body

```kotlin
// Data class (preferred for DTOs)
data class CreateUserRequest(
    val name: String,
    val email: String,
    val role: UserRole = UserRole.USER
)

@PostMapping
fun create(@Valid @RequestBody request: CreateUserRequest): UserResponse { }

// Multipart (file upload)
@PostMapping("/avatar", consumes = [MediaType.MULTIPART_FORM_DATA_VALUE])
fun uploadAvatar(
    @PathVariable id: Long,
    @RequestPart file: MultipartFile
): AvatarResponse { }
```

### Request Headers & Cookies

```kotlin
@GetMapping
fun get(
    @RequestHeader("X-Tenant-Id") tenantId: String,
    @RequestHeader(value = "Authorization", required = false) auth: String?
): Response { }

@GetMapping
fun get(@CookieValue("session") sessionId: String): Response { }
```

---

## Response Types

```kotlin
// Return type directly (Spring serializes to JSON automatically)
@GetMapping("/{id}")
fun get(@PathVariable id: Long): UserResponse = userService.findById(id)

// ResponseEntity — full control over status, headers, body
@GetMapping("/{id}")
fun get(@PathVariable id: Long): ResponseEntity<UserResponse> {
    val user = userService.findById(id) ?: return ResponseEntity.notFound().build()
    return ResponseEntity.ok(user)
}

// ResponseEntity with headers
@PostMapping
fun create(@RequestBody request: CreateUserRequest): ResponseEntity<UserResponse> {
    val user = userService.create(request)
    val location = URI.create("/api/v1/users/${user.id}")
    return ResponseEntity.created(location).body(user)
}

// Streaming / Server-Sent Events
@GetMapping("/stream", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
fun stream(): Flux<UserResponse> = userService.streamAll()

// No content
@DeleteMapping("/{id}")
fun delete(@PathVariable id: Long): ResponseEntity<Void> {
    userService.delete(id)
    return ResponseEntity.noContent().build()
}
```

### DTO Pattern

```kotlin
// Keep your domain model separate from API contracts
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
    val createdAt: Instant
) {
    companion object {
        fun from(user: User) = UserResponse(
            id = user.id,
            name = user.name,
            email = user.email,
            createdAt = user.createdAt
        )
    }
}
```

---

## Services & Dependency Injection

```kotlin
// @Service marks this as a Spring-managed bean
@Service
class UserService(
    private val userRepository: UserRepository,
    private val emailService: EmailService
) {
    // @Transactional on the service layer, not the controller
    @Transactional
    fun create(request: CreateUserRequest): UserResponse {
        val user = User(name = request.name, email = request.email)
        val saved = userRepository.save(user)
        emailService.sendWelcome(saved.email)
        return UserResponse.from(saved)
    }

    @Transactional(readOnly = true)
    fun findAll(): List<UserResponse> =
        userRepository.findAll().map(UserResponse::from)

    @Transactional(readOnly = true)
    fun findById(id: Long): UserResponse =
        userRepository.findById(id)
            .map(UserResponse::from)
            .orElseThrow { UserNotFoundException(id) }
}

// Other stereotype annotations
@Component   // generic bean
@Repository  // persistence layer (also translates JPA exceptions)
@Service     // service/business logic layer
@Controller  // MVC controller (returns view names)
@RestController // REST controller (@Controller + @ResponseBody)
```

### Bean Scopes

```kotlin
@Component
@Scope("prototype")   // new instance per injection point (default is "singleton")
class StatefulHelper

// Injecting a prototype into a singleton
@Service
class MyService(
    @Lookup private fun helper(): StatefulHelper  // Spring generates the impl
) { }
```

### Conditional Beans

```kotlin
@Configuration
class AppConfig {

    @Bean
    @ConditionalOnProperty("feature.notifications.enabled", havingValue = "true")
    fun notificationService(): NotificationService = EmailNotificationService()

    @Bean
    @Profile("local")
    fun localDataSeeder(): DataSeeder = LocalDataSeeder()

    @Bean
    @ConditionalOnMissingBean
    fun defaultClock(): Clock = Clock.systemUTC()
}
```

---

## Spring Data JPA (PostgreSQL / MySQL)

### Entity

```kotlin
@Entity
@Table(name = "users")
class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, length = 100)
    var name: String,

    @Column(nullable = false, unique = true)
    var email: String,

    @Enumerated(EnumType.STRING)
    var role: UserRole = UserRole.USER,

    @CreationTimestamp
    val createdAt: Instant = Instant.now(),

    @UpdateTimestamp
    var updatedAt: Instant = Instant.now(),

    // One-to-many
    @OneToMany(mappedBy = "user", cascade = [CascadeType.ALL], orphanRemoval = true)
    val posts: MutableList<Post> = mutableListOf(),

    // Many-to-one
    // (on the Post side): @ManyToOne @JoinColumn(name = "user_id") val user: User
)

enum class UserRole { USER, ADMIN, MODERATOR }
```

### Repository

```kotlin
@Repository
interface UserRepository : JpaRepository<User, Long> {

    // Spring Data derives the query from the method name
    fun findByEmail(email: String): Optional<User>
    fun findByRole(role: UserRole): List<User>
    fun findByNameContainingIgnoreCase(name: String): List<User>
    fun existsByEmail(email: String): Boolean
    fun deleteByEmail(email: String)

    // Custom JPQL query
    @Query("SELECT u FROM User u WHERE u.role = :role AND u.createdAt > :since")
    fun findActiveAdmins(
        @Param("role") role: UserRole,
        @Param("since") since: Instant
    ): List<User>

    // Native SQL
    @Query("SELECT * FROM users WHERE email ILIKE %:term%", nativeQuery = true)
    fun searchByEmail(@Param("term") term: String): List<User>

    // Pagination
    fun findByRole(role: UserRole, pageable: Pageable): Page<User>

    // Projections (fetch only needed fields)
    fun findByEmail(email: String): Optional<UserSummary>
}

// Projection interface
interface UserSummary {
    val id: Long
    val name: String
    val email: String
}
```

### Specifications (dynamic queries)

```kotlin
object UserSpecs {
    fun hasRole(role: UserRole): Specification<User> =
        Specification { root, _, cb -> cb.equal(root.get<UserRole>("role"), role) }

    fun nameContains(term: String): Specification<User> =
        Specification { root, _, cb -> cb.like(cb.lower(root.get("name")), "%${term.lowercase()}%") }
}

// Usage — repository must extend JpaSpecificationExecutor<User>
val spec = UserSpecs.hasRole(UserRole.ADMIN).and(UserSpecs.nameContains("alice"))
userRepository.findAll(spec, PageRequest.of(0, 20))
```

### JSONB Columns (PostgreSQL)

JSONB lets you store structured data in a column without a fixed schema. Useful for per-row config, modifier lists, or sparse fields.

Add the Hypersistence Utils library — it registers Hibernate's `JsonType` for JSONB mapping:

```kotlin
// build.gradle.kts
implementation("io.hypersistence:hypersistence-utils-hibernate-63:3.9.0")
```

```kotlin
import io.hypersistence.utils.hibernate.type.json.JsonType
import org.hibernate.annotations.Type

@Entity
@Table(name = "orders")
class Order(

    // Store a list of modifier objects as JSONB
    @Type(JsonType::class)
    @Column(columnDefinition = "jsonb")
    var modifiers: List<ModifierSnapshot> = emptyList(),

    // Store a freeform map as JSONB (e.g. per-staff preferences)
    @Type(JsonType::class)
    @Column(columnDefinition = "jsonb")
    var settings: Map<String, Any> = emptyMap(),
)

// The nested type must be serializable by Jackson
data class ModifierSnapshot(val name: String, val option: String, val priceDelta: Double = 0.0)
```

```kotlin
// Querying into JSONB with a native query
@Query(
    "SELECT * FROM staff_users WHERE settings->>'theme' = :theme",
    nativeQuery = true
)
fun findByTheme(@Param("theme") theme: String): List<StaffUser>
```

### Database Migrations (Flyway)

Flyway runs versioned SQL scripts automatically on startup — safer than `spring.jpa.hibernate.ddl-auto=update` in production.

```kotlin
// build.gradle.kts
implementation("org.flywaydb:flyway-core")
implementation("org.flywaydb:flyway-database-postgresql")
```

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true   # set true only for the first run on an existing DB
```

Place migration files in `src/main/resources/db/migration/`. Flyway runs them in version order on every startup, skipping scripts it has already applied.

```
db/migration/
├── V1__init_schema.sql       # create all base tables
├── V2__add_tax_rates.sql     # ALTER TABLE or CREATE TABLE
└── V3__add_loyalty_pin.sql   # ADD COLUMN loyalty_pin_hash VARCHAR(255)
```

Naming rule: `V{version}__{description}.sql` — double underscore, no spaces in the description.

```sql
-- V2__add_tax_rates.sql
CREATE TABLE tax_rates (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id  UUID NOT NULL REFERENCES tenants(id),
    name       VARCHAR(100) NOT NULL,
    rate       DECIMAL(5,4) NOT NULL,
    active     BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

> Never edit a migration file after it has been applied — Flyway checksums each file and will refuse to start if a previously-run script has changed. Create a new `V{n+1}` script instead.

---

## Redis

Redis is used for caching, session storage, rate limiting, pub/sub messaging, and any data that benefits from in-memory speed with optional persistence.

### Configuration

```kotlin
@Configuration
@EnableCaching
class RedisConfig {

    @Bean
    fun redisConnectionFactory(
        @Value("\${spring.data.redis.host}") host: String,
        @Value("\${spring.data.redis.port}") port: Int
    ): LettuceConnectionFactory = LettuceConnectionFactory(host, port)

    // Typed template — use this instead of the default StringRedisTemplate
    @Bean
    fun redisTemplate(factory: LettuceConnectionFactory): RedisTemplate<String, Any> {
        val jackson = Jackson2JsonRedisSerializer(Any::class.java)
        return RedisTemplate<String, Any>().apply {
            connectionFactory = factory
            keySerializer = StringRedisSerializer()
            valueSerializer = jackson
            hashKeySerializer = StringRedisSerializer()
            hashValueSerializer = jackson
            afterPropertiesSet()
        }
    }

    // Cache manager — controls TTL per cache name
    @Bean
    fun cacheManager(factory: LettuceConnectionFactory): RedisCacheManager {
        val defaults = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    Jackson2JsonRedisSerializer(Any::class.java)
                )
            )

        val perCache = mapOf(
            "users"    to defaults.entryTtl(Duration.ofMinutes(30)),
            "sessions" to defaults.entryTtl(Duration.ofHours(2))
        )

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaults)
            .withInitialCacheConfigurations(perCache)
            .build()
    }
}
```

### Spring Cache Abstraction (@Cacheable)

The easiest way to use Redis — annotate service methods and Spring handles get/set automatically.

```kotlin
@Service
class UserService(private val userRepository: UserRepository) {

    // Cache result; key defaults to method args. Evict on update/delete.
    @Cacheable(cacheNames = ["users"], key = "#id")
    fun findById(id: Long): UserResponse =
        userRepository.findById(id)
            .map(UserResponse::from)
            .orElseThrow { NotFoundException("User", id) }

    @CachePut(cacheNames = ["users"], key = "#result.id")   // update cache after save
    @Transactional
    fun update(id: Long, request: UpdateUserRequest): UserResponse {
        val user = userRepository.findById(id).orElseThrow { NotFoundException("User", id) }
        user.name = request.name
        return UserResponse.from(userRepository.save(user))
    }

    @CacheEvict(cacheNames = ["users"], key = "#id")
    @Transactional
    fun delete(id: Long) = userRepository.deleteById(id)

    // Evict all entries in a cache (e.g. after a bulk operation)
    @CacheEvict(cacheNames = ["users"], allEntries = true)
    fun invalidateAll() = Unit
}
```

### RedisTemplate (Low-Level Operations)

Use `RedisTemplate` when you need direct control — counters, TTLs, atomic ops, or data structures.

```kotlin
@Service
class CacheService(private val redis: RedisTemplate<String, Any>) {

    private val ops get() = redis.opsForValue()
    private val hashOps get() = redis.opsForHash<String, Any>()
    private val listOps get() = redis.opsForList()
    private val setOps get() = redis.opsForSet()
    private val zSetOps get() = redis.opsForZSet()

    // String / scalar
    fun set(key: String, value: Any, ttl: Duration) =
        ops.set(key, value, ttl)

    fun get(key: String): Any? = ops.get(key)

    fun increment(key: String): Long = ops.increment(key) ?: 0L

    // Atomic set-if-absent (mutex / distributed lock primitive)
    fun setIfAbsent(key: String, value: Any, ttl: Duration): Boolean =
        ops.setIfAbsent(key, value, ttl) ?: false

    fun delete(key: String) = redis.delete(key)

    fun exists(key: String): Boolean = redis.hasKey(key) ?: false

    fun expire(key: String, ttl: Duration) = redis.expire(key, ttl)

    // Hash (like a nested map — good for objects)
    fun hSet(key: String, field: String, value: Any) =
        hashOps.put(key, field, value)

    fun hGet(key: String, field: String): Any? = hashOps.get(key, field)

    fun hGetAll(key: String): Map<String, Any> =
        hashOps.entries(key)

    // List (queue / stack)
    fun lpush(key: String, value: Any) = listOps.leftPush(key, value)
    fun rpop(key: String): Any? = listOps.rightPop(key)
    fun lrange(key: String, start: Long, end: Long): List<Any> =
        listOps.range(key, start, end) ?: emptyList()

    // Sorted set (leaderboards, priority queues)
    fun zadd(key: String, value: Any, score: Double) =
        zSetOps.add(key, value, score)

    fun zrange(key: String, start: Long, end: Long): Set<Any> =
        zSetOps.range(key, start, end) ?: emptySet()

    fun zrank(key: String, value: Any): Long? = zSetOps.rank(key, value)
}
```

### Rate Limiting

```kotlin
@Service
class RateLimiter(private val redis: RedisTemplate<String, Any>) {

    // Sliding window counter — returns true if the request is allowed
    fun allow(identifier: String, limit: Int, window: Duration): Boolean {
        val key = "rate:$identifier"
        val ops = redis.opsForValue()
        val current = ops.increment(key) ?: 1L
        if (current == 1L) redis.expire(key, window)   // set TTL on first request
        return current <= limit
    }
}

// Use it in a filter or controller
@Component
class RateLimitFilter(private val rateLimiter: RateLimiter) : OncePerRequestFilter() {
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        chain: FilterChain
    ) {
        val ip = request.remoteAddr
        if (!rateLimiter.allow(ip, limit = 100, window = Duration.ofMinutes(1))) {
            response.status = HttpStatus.TOO_MANY_REQUESTS.value()
            return
        }
        chain.doFilter(request, response)
    }
}
```

### Pub/Sub Messaging

```kotlin
// Publisher
@Service
class EventPublisher(private val redis: RedisTemplate<String, Any>) {

    fun publish(channel: String, event: Any) =
        redis.convertAndSend(channel, event)
}

// Subscriber
@Component
class UserEventListener : MessageListener {
    override fun onMessage(message: Message, pattern: ByteArray?) {
        val payload = String(message.body)
        println("Received on ${message.channel}: $payload")
    }
}

// Wire it up
@Configuration
class PubSubConfig {

    @Bean
    fun listenerContainer(
        factory: LettuceConnectionFactory,
        listener: UserEventListener
    ): RedisMessageListenerContainer {
        return RedisMessageListenerContainer().apply {
            connectionFactory = factory
            addMessageListener(listener, ChannelTopic("user-events"))
        }
    }
}
```

### Redis as a Spring Session Store

```kotlin
// build.gradle.kts
// implementation("org.springframework.session:spring-session-data-redis")

@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 3600)
class SessionConfig
// That's it — all HttpSession calls now persist to Redis automatically.
```

---

## MongoDB

MongoDB is a good fit when your schema is evolving fast, your data is document-shaped, or you need flexible querying without a rigid relational model.

### Document (Entity)

```kotlin
@Document(collection = "products")
data class Product(
    @Id val id: String? = null,              // MongoDB uses String ObjectId by default
    val name: String,
    val description: String,
    val price: BigDecimal,
    val tags: List<String> = emptyList(),
    val metadata: Map<String, Any> = emptyMap(),

    @Indexed(unique = true)
    val sku: String,

    @Indexed
    val categoryId: String,

    @Field("created_at")
    val createdAt: Instant = Instant.now(),

    val active: Boolean = true
)

// Embedded document (no @Document — it's stored inside the parent)
data class Address(
    val street: String,
    val city: String,
    val country: String,
    val zip: String
)

@Document(collection = "customers")
data class Customer(
    @Id val id: String? = null,
    val name: String,
    val email: String,
    val address: Address,                    // embedded — no join needed
    val orderIds: List<String> = emptyList() // reference by ID (manual join)
)
```

### Repository

```kotlin
@Repository
interface ProductRepository : MongoRepository<Product, String> {

    fun findByCategoryId(categoryId: String): List<Product>
    fun findByActiveTrue(): List<Product>
    fun findByPriceLessThanEqual(maxPrice: BigDecimal): List<Product>
    fun findByTagsContaining(tag: String): List<Product>
    fun findByNameContainingIgnoreCase(name: String, pageable: Pageable): Page<Product>
    fun existsBySku(sku: String): Boolean
    fun deleteByActiveFalse()

    // Custom MongoDB query (MQL)
    @Query("{ 'price': { '\$lte': ?0 }, 'active': true }")
    fun findCheapActive(maxPrice: BigDecimal): List<Product>

    // Projection — return only specific fields
    @Query(value = "{ 'categoryId': ?0 }", fields = "{ 'name': 1, 'price': 1, 'sku': 1 }")
    fun findSummaryByCategory(categoryId: String): List<ProductSummary>
}

data class ProductSummary(val id: String, val name: String, val price: BigDecimal, val sku: String)
```

### MongoTemplate (Advanced Queries)

Use `MongoTemplate` for dynamic queries, aggregations, and bulk operations that the repository DSL can't express.

```kotlin
@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val mongo: MongoTemplate
) {

    // Dynamic query with Criteria
    fun search(name: String?, tag: String?, maxPrice: BigDecimal?, active: Boolean = true): List<Product> {
        val criteria = Criteria.where("active").`is`(active).apply {
            name?.let { and("name").regex(it, "i") }
            tag?.let { and("tags").`in`(it) }
            maxPrice?.let { and("price").lte(it) }
        }
        val query = Query(criteria).with(Sort.by(Sort.Direction.ASC, "name"))
        return mongo.find(query, Product::class.java)
    }

    // Update a single field without loading the full document
    fun updatePrice(id: String, newPrice: BigDecimal) {
        val query = Query(Criteria.where("_id").`is`(id))
        val update = Update().set("price", newPrice).currentDate("updatedAt")
        mongo.updateFirst(query, update, Product::class.java)
    }

    // Upsert
    fun upsertBySkU(product: Product) {
        val query = Query(Criteria.where("sku").`is`(product.sku))
        val update = Update()
            .set("name", product.name)
            .set("price", product.price)
            .set("active", product.active)
            .setOnInsert("createdAt", Instant.now())
        mongo.upsert(query, update, Product::class.java)
    }

    // Bulk write
    fun bulkDeactivate(ids: List<String>) {
        val query = Query(Criteria.where("_id").`in`(ids))
        val update = Update().set("active", false)
        mongo.updateMulti(query, update, Product::class.java)
    }
}
```

### Aggregation Pipeline

```kotlin
@Service
class AnalyticsService(private val mongo: MongoTemplate) {

    data class CategoryStats(val id: String, val count: Int, val avgPrice: Double)

    // Equivalent to: GROUP BY categoryId, count(*), avg(price)
    fun statsByCategory(): List<CategoryStats> {
        val agg = Aggregation.newAggregation(
            Aggregation.match(Criteria.where("active").`is`(true)),
            Aggregation.group("categoryId")
                .count().`as`("count")
                .avg("price").`as`("avgPrice"),
            Aggregation.sort(Sort.Direction.DESC, "count"),
            Aggregation.limit(20)
        )
        return mongo.aggregate(agg, "products", CategoryStats::class.java).mappedResults
    }

    // Lookup (left join from orders → products)
    fun ordersWithProducts(): List<Document> {
        val agg = Aggregation.newAggregation(
            Aggregation.lookup("products", "productId", "_id", "product"),
            Aggregation.unwind("product"),
            Aggregation.project("quantity", "product.name", "product.price")
        )
        return mongo.aggregate(agg, "orders", Document::class.java).mappedResults
    }
}
```

### Indexes

```kotlin
@Document(collection = "products")
@CompoundIndexes(
    CompoundIndex(name = "category_price_idx", def = "{'categoryId': 1, 'price': -1}"),
    CompoundIndex(name = "sku_active_idx", def = "{'sku': 1, 'active': 1}", unique = true)
)
data class Product( /* ... */ )

// Or programmatically (e.g. in a @PostConstruct or migration)
@Component
class MongoIndexInitializer(private val mongo: MongoTemplate) {

    @PostConstruct
    fun ensureIndexes() {
        val ops = mongo.indexOps("products")
        ops.ensureIndex(
            Index().on("tags", Sort.Direction.ASC).named("tags_idx")
        )
        ops.ensureIndex(
            Index().on("createdAt", Sort.Direction.DESC)
                .expire(Duration.ofDays(90))   // TTL index — auto-delete old docs
                .named("ttl_idx")
        )
    }
}
```

### Reactive MongoDB (Coroutines)

```kotlin
// build.gradle.kts — swap out starter:
// implementation("org.springframework.boot:spring-boot-starter-data-mongodb-reactive")

interface ProductRepository : CoroutineCrudRepository<Product, String> {
    fun findByCategoryId(categoryId: String): Flow<Product>
    suspend fun findBySku(sku: String): Product?
}

@Service
class ProductService(private val repo: ProductRepository) {

    suspend fun findBySku(sku: String): Product =
        repo.findBySku(sku) ?: throw NotFoundException("Product", sku)

    fun streamByCategory(categoryId: String): Flow<Product> =
        repo.findByCategoryId(categoryId)
}
```

---

## Exception Handling

### Custom Exceptions

```kotlin
// Hierarchy keeps things clean
sealed class AppException(message: String, cause: Throwable? = null)
    : RuntimeException(message, cause)

class NotFoundException(resource: String, id: Any)
    : AppException("$resource with id $id not found")

class ValidationException(val field: String, message: String)
    : AppException(message)

class ConflictException(message: String) : AppException(message)
```

### Global Handler

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException::class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    fun handleNotFound(ex: NotFoundException): ErrorResponse =
        ErrorResponse(status = 404, message = ex.message ?: "Not found")

    @ExceptionHandler(ValidationException::class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    fun handleValidation(ex: ValidationException): ErrorResponse =
        ErrorResponse(status = 422, message = ex.message ?: "Validation failed", field = ex.field)

    @ExceptionHandler(MethodArgumentNotValidException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleBeanValidation(ex: MethodArgumentNotValidException): ErrorResponse {
        val errors = ex.bindingResult.fieldErrors.map { "${it.field}: ${it.defaultMessage}" }
        return ErrorResponse(status = 400, message = "Validation failed", errors = errors)
    }

    @ExceptionHandler(Exception::class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    fun handleUnexpected(ex: Exception): ErrorResponse {
        log.error("Unhandled exception", ex)
        return ErrorResponse(status = 500, message = "Internal server error")
    }

    private val log = LoggerFactory.getLogger(javaClass)
}

data class ErrorResponse(
    val status: Int,
    val message: String,
    val field: String? = null,
    val errors: List<String> = emptyList(),
    val timestamp: Instant = Instant.now()
)
```

---

## Validation

```kotlin
// Use Jakarta Validation (Bean Validation) annotations on DTOs
data class CreateUserRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(min = 2, max = 100)
    val name: String,

    @field:Email(message = "Must be a valid email address")
    @field:NotBlank
    val email: String,

    @field:Min(0) @field:Max(150)
    val age: Int? = null,

    @field:Pattern(regexp = "^[A-Z]{2,3}$", message = "Invalid country code")
    val country: String? = null
)

// Trigger with @Valid in the controller
@PostMapping
fun create(@Valid @RequestBody request: CreateUserRequest): UserResponse { }

// Custom validator
@Target(AnnotationTarget.FIELD, AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [UniqueEmailValidator::class])
annotation class UniqueEmail(
    val message: String = "Email already in use",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

class UniqueEmailValidator(
    private val userRepository: UserRepository
) : ConstraintValidator<UniqueEmail, String> {
    override fun isValid(value: String?, context: ConstraintValidatorContext) =
        value == null || !userRepository.existsByEmail(value)
}
```

---

## Configuration

### @Value

```kotlin
@Service
class MailService(
    @Value("\${mail.host}") private val host: String,
    @Value("\${mail.port:587}") private val port: Int,      // 587 is default
    @Value("\${mail.enabled:true}") private val enabled: Boolean
) { }
```

### @ConfigurationProperties (preferred for grouped config)

```kotlin
@ConfigurationProperties(prefix = "app")
@Validated
data class AppProperties(
    val name: String,
    val mail: MailProperties,
    val features: FeaturesProperties = FeaturesProperties()
) {
    data class MailProperties(
        @field:NotBlank val host: String,
        val port: Int = 587,
        val username: String = "",
        val password: String = ""
    )

    data class FeaturesProperties(
        val notifications: Boolean = false,
        val analytics: Boolean = false
    )
}

// Enable in your main class or a @Configuration class:
@SpringBootApplication
@EnableConfigurationProperties(AppProperties::class)
class MyServiceApplication
```

```yaml
# application.yml
app:
  name: My Service
  mail:
    host: smtp.example.com
    port: 587
    username: ${MAIL_USER}
    password: ${MAIL_PASS}
  features:
    notifications: true
```

---

## Security & JWT

### SecurityFilterChain

```kotlin
@Configuration
@EnableWebSecurity
class SecurityConfig(private val jwtAuthFilter: JwtAuthFilter) {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.disable() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/api/v1/auth/**").permitAll()
                    .requestMatchers(HttpMethod.GET, "/api/v1/public/**").permitAll()
                    .requestMatchers("/api/v1/admin/**").hasAnyRole("OWNER", "MANAGER")
                    .requestMatchers("/api/v1/staff/**").hasAnyRole("OWNER", "MANAGER", "STAFF")
                    .anyRequest().authenticated()
            }
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter::class.java)
        return http.build()
    }

    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()
}
```

### JWT Filter (OncePerRequestFilter)

The filter runs once per request, extracts the `Authorization` header, validates the token, and sets the `SecurityContext` so Spring Security knows who is making the request.

```kotlin
// build.gradle.kts
implementation("io.jsonwebtoken:jjwt-api:0.12.6")
runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.6")
runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.6")
```

```kotlin
@Service
class JwtService {

    // For RS256: load a private key for signing, public key for verification.
    // For HS256: use a single secret key (simpler, but both sides share the secret).
    private val secretKey: SecretKey = Keys.hmacShaKeyFor(
        System.getenv("JWT_SECRET").toByteArray()
    )

    fun generate(subject: String, claims: Map<String, Any>, expiryHours: Long = 8): String =
        Jwts.builder()
            .subject(subject)
            .claims(claims)
            .issuedAt(Date())
            .expiration(Date(System.currentTimeMillis() + expiryHours * 3_600_000))
            .signWith(secretKey)
            .compact()

    fun parse(token: String): Claims =
        Jwts.parser()
            .verifyWith(secretKey)
            .build()
            .parseSignedClaims(token)
            .payload
}
```

```kotlin
@Component
class JwtAuthFilter(private val jwtService: JwtService) : OncePerRequestFilter() {

    override fun doFilterInternal(
        req: HttpServletRequest,
        res: HttpServletResponse,
        chain: FilterChain
    ) {
        val token = req.getHeader("Authorization")
            ?.takeIf { it.startsWith("Bearer ") }
            ?.removePrefix("Bearer ")

        if (token != null) {
            runCatching { jwtService.parse(token) }.getOrNull()?.let { claims ->
                val role    = claims["role"] as? String ?: "USER"
                val auth    = UsernamePasswordAuthenticationToken(
                    claims, null, listOf(SimpleGrantedAuthority("ROLE_$role"))
                )
                SecurityContextHolder.getContext().authentication = auth
            }
        }

        chain.doFilter(req, res)
    }
}
```

```kotlin
// Accessing claims in a controller or service
@GetMapping("/me")
fun me(authentication: Authentication): Map<String, Any> {
    @Suppress("UNCHECKED_CAST")
    val claims = authentication.principal as Claims
    return mapOf(
        "sub"  to claims.subject,
        "role" to (claims["role"] ?: ""),
    )
}
```

### Method-Level Security

```kotlin
@Service
@PreAuthorize("hasRole('ADMIN')")          // class-level: all methods require ADMIN
class AdminService {

    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    fun managerAction() { }

    @PostAuthorize("returnObject.ownerId == authentication.principal.subject")
    fun getResource(id: Long): Resource { }
}
```

---

## WebSocket / STOMP

Spring's STOMP broker lets clients subscribe to topics and receive push messages — useful for real-time order updates, kitchen displays, or notifications.

```kotlin
// build.gradle.kts
implementation("org.springframework.boot:spring-boot-starter-websocket")
```

### Configuration

```kotlin
@Configuration
@EnableWebSocketMessageBroker
class WebSocketConfig : WebSocketMessageBrokerConfigurer {

    override fun configureMessageBroker(registry: MessageBrokerRegistry) {
        registry.enableSimpleBroker("/topic", "/queue")   // server → client topics
        registry.setApplicationDestinationPrefixes("/app") // client → server messages
    }

    override fun registerStompEndpoints(registry: StompEndpointRegistry) {
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*")
            .withSockJS()   // fallback for browsers that don't support native WebSocket
    }
}
```

### Publishing Messages (Server → Client)

```kotlin
@Service
class OrderPublisher(private val ws: SimpMessagingTemplate) {

    // Broadcast to all subscribers on a topic
    fun broadcastToTenant(tenantId: UUID, payload: Any) {
        ws.convertAndSend("/topic/orders/$tenantId", payload)
    }

    // Send to a specific user (requires Spring Security principal name)
    fun sendToUser(userId: String, payload: Any) {
        ws.convertAndSendToUser(userId, "/queue/notifications", payload)
    }
}
```

### Receiving Messages from Client (Client → Server)

```kotlin
@Controller
class OrderController {

    // Client sends to /app/orders/advance; result is broadcast to the topic
    @MessageMapping("/orders/advance")
    @SendTo("/topic/orders/{tenantId}")
    fun advanceOrder(@DestinationVariable tenantId: UUID, @Payload cmd: AdvanceOrderCmd): OrderEvent {
        return orderService.advance(cmd.orderId, cmd.status)
    }
}
```

### JavaScript Client

```javascript
import { Client } from '@stomp/stompjs'
import SockJS from 'sockjs-client'

const client = new Client({
    webSocketFactory: () => new SockJS('/ws'),
    connectHeaders: { Authorization: `Bearer ${token}` },
    onConnect: () => {
        // Subscribe to a topic — fires on every push from the server
        client.subscribe(`/topic/orders/${tenantId}`, (msg) => {
            const event = JSON.parse(msg.body)
            console.log('Order update:', event)
        })
    }
})
client.activate()
```

---

## Testing

### Unit Test (Service)

```kotlin
@ExtendWith(MockitoExtension::class)
class UserServiceTest {

    @Mock
    private lateinit var userRepository: UserRepository

    @InjectMocks
    private lateinit var userService: UserService

    @Test
    fun `findById returns user when found`() {
        val user = User(id = 1, name = "Alice", email = "alice@example.com")
        whenever(userRepository.findById(1L)).thenReturn(Optional.of(user))

        val result = userService.findById(1L)

        assertThat(result.name).isEqualTo("Alice")
    }

    @Test
    fun `findById throws NotFoundException when missing`() {
        whenever(userRepository.findById(99L)).thenReturn(Optional.empty())

        assertThatThrownBy { userService.findById(99L) }
            .isInstanceOf(NotFoundException::class.java)
    }
}
```

### Integration Test (Controller)

```kotlin
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @MockBean
    private lateinit var userService: UserService

    @Test
    @WithMockUser(roles = ["ADMIN"])
    fun `GET users returns 200`() {
        val users = listOf(UserResponse(id = 1, name = "Alice", email = "alice@example.com"))
        whenever(userService.findAll()).thenReturn(users)

        mockMvc.perform(get("/api/v1/users").accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$[0].name").value("Alice"))
    }

    @Test
    fun `POST user with invalid body returns 400`() {
        val body = """{"name": "", "email": "not-an-email"}"""

        mockMvc.perform(
            post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(body)
        )
            .andExpect(status().isBadRequest)
    }
}
```

### Repository Test (Database)

```kotlin
@DataJpaTest                                         // only loads JPA slice — fast
@AutoConfigureTestDatabase(replace = Replace.NONE)   // use real DB (not H2)
class UserRepositoryTest {

    @Autowired
    private lateinit var userRepository: UserRepository

    @Test
    fun `findByEmail returns user`() {
        userRepository.save(User(name = "Alice", email = "alice@example.com"))

        val found = userRepository.findByEmail("alice@example.com")

        assertThat(found).isPresent
        assertThat(found.get().name).isEqualTo("Alice")
    }
}
```

---

## Coroutines with Spring

Spring Boot 3.x has first-class coroutine support for WebMVC and WebFlux.

### Suspend Controller Functions (WebMVC)

```kotlin
// Spring MVC supports suspend functions out of the box
@RestController
@RequestMapping("/api/v1/users")
class UserController(private val userService: UserService) {

    @GetMapping
    suspend fun getAll(): List<UserResponse> =
        userService.findAll()                   // suspend call — no thread blocking

    @GetMapping("/{id}")
    suspend fun getById(@PathVariable id: Long): UserResponse =
        userService.findById(id)
}

@Service
class UserService(private val userRepository: UserRepository) {

    suspend fun findAll(): List<UserResponse> =
        withContext(Dispatchers.IO) {
            userRepository.findAll().map(UserResponse::from)
        }
}
```

### WebFlux (Reactive)

```kotlin
// Reactive controller — returns Mono/Flux or suspend
@RestController
class ReactiveUserController(private val userService: ReactiveUserService) {

    @GetMapping
    fun getAll(): Flow<UserResponse> =
        userService.streamAll()

    @GetMapping("/{id}")
    suspend fun getById(@PathVariable id: Long): UserResponse =
        userService.findById(id)
}

// Coroutine-friendly reactive repository
interface UserRepository : CoroutineCrudRepository<User, Long> {
    fun findByRole(role: UserRole): Flow<User>
    suspend fun findByEmail(email: String): User?
}
```

---

## Quick Reference

### Annotations

| Annotation | Purpose |
|-----------|---------|
| `@SpringBootApplication` | Enables autoconfiguration, component scan, bean config |
| `@RestController` | REST controller — serializes return values to JSON |
| `@RequestMapping` | Maps HTTP requests to class or method |
| `@GetMapping` / `@PostMapping` etc. | HTTP method-specific shortcuts |
| `@PathVariable` | Extracts `{placeholder}` from URL |
| `@RequestParam` | Extracts `?key=value` from query string |
| `@RequestBody` | Deserializes request body from JSON |
| `@Valid` | Triggers Bean Validation on the annotated argument |
| `@Service` | Business logic layer bean |
| `@Repository` | Persistence layer bean (also translates JPA exceptions) |
| `@Component` | Generic Spring-managed bean |
| `@Configuration` | Declares `@Bean` methods |
| `@Bean` | Declares a Spring bean in a `@Configuration` class |
| `@Autowired` | Dependency injection (not needed on constructor) |
| `@Value` | Injects a property value from config |
| `@ConfigurationProperties` | Binds a config prefix to a class |
| `@Transactional` | Wraps method in a database transaction |
| `@Entity` | Marks class as a JPA entity |
| `@Id` / `@GeneratedValue` | Primary key and auto-generation strategy |
| `@RestControllerAdvice` | Global exception handler for REST controllers |
| `@ExceptionHandler` | Handles a specific exception type |
| `@Profile` | Activates bean only for specified Spring profile |
| `@Scheduled` | Runs method on a cron schedule |
| `@Async` | Runs method asynchronously in a thread pool |
| `@EnableWebSecurity` | Enables Spring Security |
| `@PreAuthorize` | Method-level authorization (SpEL expression) |

### HTTP Status Shortcuts

| Status | Spring |
|--------|--------|
| 200 OK | Default return |
| 201 Created | `@ResponseStatus(HttpStatus.CREATED)` or `ResponseEntity.created(uri)` |
| 204 No Content | `@ResponseStatus(HttpStatus.NO_CONTENT)` |
| 400 Bad Request | Throw `MethodArgumentNotValidException` or return `ResponseEntity.badRequest()` |
| 401 Unauthorized | Spring Security handles automatically |
| 403 Forbidden | Spring Security handles automatically |
| 404 Not Found | `ResponseEntity.notFound().build()` or `@ResponseStatus(HttpStatus.NOT_FOUND)` |
| 409 Conflict | `ResponseEntity.status(HttpStatus.CONFLICT).body(...)` |
| 500 Internal Server Error | Unhandled exception → `@ExceptionHandler` |

### Spring Data Query Methods

| Method name | SQL equivalent |
|-------------|---------------|
| `findById(id)` | `WHERE id = ?` |
| `findByEmail(email)` | `WHERE email = ?` |
| `findByNameAndRole(name, role)` | `WHERE name = ? AND role = ?` |
| `findByNameOrEmail(name, email)` | `WHERE name = ? OR email = ?` |
| `findByAgeGreaterThan(age)` | `WHERE age > ?` |
| `findByAgeBetween(min, max)` | `WHERE age BETWEEN ? AND ?` |
| `findByNameContaining(str)` | `WHERE name LIKE %?%` |
| `findByNameStartingWith(str)` | `WHERE name LIKE ?%` |
| `findByDeletedFalse()` | `WHERE deleted = false` |
| `findByCreatedAtAfter(date)` | `WHERE created_at > ?` |
| `findByRoleIn(roles)` | `WHERE role IN (...)` |
| `findByNameOrderByCreatedAtDesc(name)` | `WHERE name = ? ORDER BY created_at DESC` |
| `countByRole(role)` | `SELECT count(*) WHERE role = ?` |
| `deleteByEmail(email)` | `DELETE WHERE email = ?` |

---

*Hazelnut Docs — Spring Boot*
