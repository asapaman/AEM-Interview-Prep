# Bonus — Spring Boot Basics
**Level:** Intermediate → Advanced | **Java Version:** 8+ / 17+ (Spring Boot 3)

---

## 🌟 What is Spring Boot?

**Spring Framework** is an enormous dependency injection and enterprise application framework. It historically required heavy XML configuration.
**Spring Boot** sits on top of Spring. It provides:
1. **Auto-Configuration:** Automatically configures your application based on the `.jar` dependencies on your classpath. (e.g., if it sees `H2` in your `pom.xml`, it auto-configures an in-memory database).
2. **Embedded Servers:** Packages a Tomcat/Jetty server *inside* your application. You just run `java -jar app.jar` instead of deploying a `.war` file to an external server.
3. **Opinionated Defaults:** Provides sensible defaults for everything, so you don't have to manually configure boilerplate.

---

## 🧱 The Core Concepts

### 1. Inversion of Control (IoC) & Dependency Injection (DI)
Instead of classes instantiating their own dependencies (`new Database()`), the **Spring IoC Container** creates the objects (called **Beans**) and "injects" them where needed.

```java
// Spring creates this Bean and manages its lifecycle
@Service 
public class EmailService {
    public void sendEmail(String msg) { ... }
}

@RestController
public class UserController {
    
    private final EmailService emailService;

    // Spring INJECTS the EmailService Bean into this constructor automatically
    // (In modern Spring, @Autowired is optional if there's only one constructor)
    public UserController(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

### 2. Core Annotations

| Annotation | Purpose |
|------------|---------|
| `@SpringBootApplication` | Placed on the main class. It's a combination of `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`. |
| `@Component` | Generic stereotype for any Spring-managed component (Bean). |
| `@Service` | Specialized `@Component` for Business Logic classes. |
| `@Repository` | Specialized `@Component` for Data Access (Database) classes. Translates DB exceptions into Spring exceptions. |
| `@Controller` / `@RestController` | Specialized `@Component` for handling HTTP requests. `@RestController` automatically adds `@ResponseBody` to serialize returns into JSON. |
| `@Configuration` | Marks a class that defines manual Bean configurations using `@Bean` methods. |

---

## 🌐 Building a REST API in 5 Minutes

Spring Boot makes creating REST APIs incredibly simple.

### 1. The Controller
```java
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController // Tells Spring this class handles HTTP requests and returns JSON
@RequestMapping("/api/users") // Base URL
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    // GET /api/users
    @GetMapping
    public List<User> getAllUsers() {
        return userService.getAll(); // Automatically serialized to JSON array
    }

    // GET /api/users/123
    @GetMapping("/{id}")
    public User getUserById(@PathVariable Long id) {
        return userService.findById(id);
    }

    // POST /api/users
    @PostMapping
    public User createUser(@RequestBody User newUser) {
        // @RequestBody deserializes incoming JSON into a Java Object
        return userService.save(newUser);
    }
}
```

### 2. Application Properties
Configuration is externalized in `application.properties` (or `application.yml`).
```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret
```

---

## 💾 Spring Data JPA

Spring Data drastically reduces boilerplate code for database access. You just define an interface, and Spring generates the implementation at runtime!

```java
import org.springframework.data.jpa.repository.JpaRepository;

// Entity Class mapping to a DB table
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String email;
    private String name;
    // getters/setters...
}

// The Repository - That's it! No SQL needed for basic operations.
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Custom query method! Spring parses the method name and writes the SQL!
    // "SELECT * FROM users WHERE email = ?"
    Optional<User> findByEmail(String email);
    
    // "SELECT * FROM users WHERE name LIKE ? ORDER BY name ASC"
    List<User> findByNameContainingOrderByNameAsc(String nameFragment);
}
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between `@Controller` and `@RestController`?**
> Both handle web requests. `@Controller` is typically used for traditional web applications where methods return a String representing a View (like a JSP or Thymeleaf template) to be rendered. `@RestController` is a convenience annotation that combines `@Controller` and `@ResponseBody`. Every method in a `@RestController` automatically serializes its return value directly into the HTTP response body (usually as JSON or XML), making it ideal for REST APIs.

**Q2. What are the different types of Dependency Injection in Spring, and which is best?**
> 1. **Constructor Injection:** (Recommended) Dependencies are passed via the constructor. Ensures the bean cannot be created without its dependencies (fail-fast), allows fields to be `final` (immutable), and makes unit testing easy without Spring.
> 2. **Setter Injection:** Dependencies are passed via setter methods. Used for optional dependencies.
> 3. **Field Injection:** (`@Autowired` directly on the field). Heavily discouraged! It makes unit testing difficult, hides dependencies, and prevents immutability.

**Q3. How does Spring Boot Auto-Configuration work?**
> The `@EnableAutoConfiguration` annotation (part of `@SpringBootApplication`) triggers it. Spring looks at the `META-INF/spring.factories` file in its libraries. It then uses `@Conditional` annotations to evaluate your classpath. For example, `@ConditionalOnClass(DataSource.class)` checks if a database driver is present. If it is, and you haven't defined your own DataSource bean, Spring automatically creates and configures one for you.

**Q4. What is the Spring Bean Lifecycle?**
> 1. Instantiation (Constructor called).
> 2. Populate Properties (Dependency Injection occurs).
> 3. `setBeanName`, `setBeanFactory`, etc. (Aware interfaces).
> 4. `postProcessBeforeInitialization` (BeanPostProcessors).
> 5. Custom `init()` method (or `@PostConstruct` annotated method).
> 6. `postProcessAfterInitialization`.
> 7. Bean is ready for use.
> 8. Container shuts down -> Custom `destroy()` method (or `@PreDestroy` annotated method).

**Q5. What is the difference between `@Component`, `@Service`, and `@Repository`?**
> Technically, `@Service` and `@Repository` are just meta-annotations on top of `@Component`. They all register the class as a Spring Bean. However, they clarify the architecture (semantics). Furthermore, `@Repository` has special behavior: it enables automatic translation of vendor-specific database exceptions (like `SQLException`) into Spring's unified `DataAccessException` hierarchy.

---

## ✅ Best Practices

1. **Use Constructor Injection.** Avoid `@Autowired` on fields.
2. **Keep Controllers thin.** Controllers should only handle HTTP routing, validation, and JSON serialization. Put all business logic in `@Service` classes.
3. **Use DTOs (Data Transfer Objects).** Do not return `@Entity` classes directly from your REST controllers. Entities represent your DB schema and might expose sensitive data (passwords) or cause infinite recursion loops in JSON serialization (due to bidirectional relationships). Map Entities to DTOs before returning.
4. **Use `@ControllerAdvice` for global exception handling.** Instead of `try/catch` in every controller method, create a single class to catch specific exceptions globally and return standardized error JSON responses.

---

## 🛠️ Hands-on Practice

1. Setup a Spring Boot project using [start.spring.io](https://start.spring.io/) with "Spring Web" dependency.
2. Create a simple `HelloWorldController` with a `@GetMapping("/hello")` that returns "Hello Spring Boot!". Run the app and visit `localhost:8080/hello`.
3. Create a `Book` class (id, title, author). Create a `@RestController` that maintains a `List<Book>` in memory. Implement POST to add a book, and GET to retrieve all books.
4. Research and implement `@ControllerAdvice` to catch an `IllegalArgumentException` thrown by your controller and return a custom JSON error response with a 400 Bad Request status.
