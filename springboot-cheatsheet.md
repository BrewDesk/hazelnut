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
7. [Spring Data JPA](#spring-data-jpa)
8. [Exception Handling](#exception-handling)
9. [Validation](#validation)
10. [Configuration](#configuration)
11. [Security Basics](#security-basics)
12. [Testing](#testing)
13. [Coroutines with Spring](#coroutines-with-spring)
14. [Quick Reference](#quick-reference)

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

## Spring Data JPA

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

## Security Basics

```kotlin
@Configuration
@EnableWebSecurity
class SecurityConfig {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.disable() }                              // disable for REST APIs
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/api/v1/auth/**").permitAll()
                    .requestMatchers(HttpMethod.GET, "/api/v1/public/**").permitAll()
                    .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
            }
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter::class.java)
        return http.build()
    }

    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()

    @Bean
    fun authenticationManager(config: AuthenticationConfiguration): AuthenticationManager =
        config.authenticationManager
}

// Method-level security
@Service
@PreAuthorize("hasRole('ADMIN')")          // class-level: all methods require ADMIN
class AdminService {

    @PreAuthorize("hasAnyRole('ADMIN', 'MODERATOR')")
    fun moderatorAction() { }

    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    fun getResource(id: Long): Resource { }
}

// Get the current user
@GetMapping("/me")
fun me(authentication: Authentication): UserResponse {
    val principal = authentication.principal as UserDetails
    return userService.findByEmail(principal.username)
}
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
