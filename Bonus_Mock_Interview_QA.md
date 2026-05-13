# Bonus — Rapid-Fire Mock Interview Q&A
**Level:** All Levels | **Focus:** Core Java, Collections, Concurrency, Spring

---

## ☕ Core Java

**Q1. What is the difference between `==` and `.equals()`?**
> `==` checks for reference equality (do both variables point to the exact same object in memory). `.equals()` checks for logical value equality (do the objects contain the same data), assuming the class has overridden the `equals()` method.

**Q2. Why is String immutable in Java?**
> For security (can't be altered when passed to DB/Network), thread-safety (immutable objects are inherently thread-safe), caching (allows the String Constant Pool to exist), and performance (hashcode can be cached, making it a fast key for HashMaps).

**Q3. What happens if you don't override `hashCode()` when you override `equals()`?**
> It breaks the contract between them. Two objects that are logically equal (according to `equals()`) might return different hash codes. If you use these objects as keys in a `HashMap` or `HashSet`, the collection will not be able to find them later, leading to data loss or duplicates.

**Q4. What is the difference between final, finally, and finalize?**
> **final:** A keyword used to make variables constant, prevent methods from being overridden, and prevent classes from being inherited.
> **finally:** A block used in try-catch to execute crucial cleanup code (like closing resources) regardless of whether an exception occurred.
> **finalize():** A deprecated method in the `Object` class called by the Garbage Collector before destroying an object. Never use it; use `try-with-resources` instead.

**Q5. Can you instantiate an abstract class?**
> No, you cannot use the `new` keyword on an abstract class. You must create a concrete subclass that implements all its abstract methods and instantiate that subclass.

**Q6. What is the difference between an interface and an abstract class?**
> **Abstract class:** Can have state (fields), constructors, and any access modifier. A class can only extend ONE abstract class (single inheritance). Used for strong "IS-A" relationships.
> **Interface:** Pure contract. Only contains constants (public static final) and public methods (abstract, default, static). A class can implement MULTIPLE interfaces. Used for "CAN-DO" capabilities.

**Q7. Does Java support multiple inheritance?**
> Java does not support multiple inheritance of *state* (a class cannot extend multiple classes) to avoid the "Diamond Problem" of ambiguity. However, it does support multiple inheritance of *type* (a class can implement multiple interfaces).

---

## 🗃️ Collections Framework

**Q8. What is the difference between ArrayList and LinkedList?**
> **ArrayList:** Backed by a dynamic array. O(1) for random access (get). O(n) for insertion/deletion in the middle (requires shifting elements). Better cache locality.
> **LinkedList:** Backed by a doubly-linked list. O(n) for random access. O(1) for insertion/deletion at the ends (or at a known node). Usually consumes more memory per element.

**Q9. How does a HashMap handle collisions?**
> When two different keys generate the same hash code (or map to the same bucket index), a collision occurs. HashMap stores them in that same bucket using a Linked List. In Java 8+, if the linked list in a single bucket grows beyond 8 elements, it converts into a balanced Red-Black tree to improve worst-case search time from O(n) to O(log n).

**Q10. Difference between HashSet, LinkedHashSet, and TreeSet?**
> **HashSet:** Unordered, fastest (O(1) average).
> **LinkedHashSet:** Maintains insertion order, slightly slower than HashSet.
> **TreeSet:** Maintains sorted order (natural or custom via Comparator), slower (O(log n)) because it uses a Red-Black tree internally.

**Q11. What is the difference between Comparable and Comparator?**
> **Comparable:** Implemented by the class itself (e.g., `String implements Comparable`). Modifies the class. Defines the *natural* single sorting order using `compareTo()`.
> **Comparator:** An external class/lambda. Does not require modifying the original class. Allows defining *multiple* custom sorting strategies using `compare()`.

---

## 🚦 Multithreading & Concurrency

**Q12. What is the difference between `start()` and `run()` in a Thread?**
> Calling `start()` registers the thread with the OS scheduler and executes `run()` asynchronously in a brand new call stack. Calling `run()` directly just executes the method synchronously on the current thread, defeating the purpose of multithreading.

**Q13. What is a Deadlock and how do you prevent it?**
> A deadlock occurs when Thread A holds Lock 1 waiting for Lock 2, while Thread B holds Lock 2 waiting for Lock 1. Both are blocked forever. Prevention: Always acquire locks in the exact same predefined order across all threads.

**Q14. What does the `volatile` keyword do?**
> It guarantees the *visibility* of a variable's value across threads. It forces threads to read and write the variable directly to main memory rather than caching it in CPU registers. It does *not* guarantee atomicity (e.g., `count++` is still not thread-safe with volatile alone).

**Q15. Callable vs Runnable?**
> **Runnable:** `run()` method returns `void` and cannot throw checked exceptions.
> **Callable:** `call()` method returns a generic value `<T>` and can throw checked exceptions. Used with `ExecutorService.submit()` which returns a `Future`.

**Q16. What is a Thread Pool and why use ExecutorService?**
> Creating native OS threads is expensive. A Thread Pool reuses a fixed number of worker threads to execute tasks from a queue. `ExecutorService` abstracts away the complex thread lifecycle management, improving performance and preventing the app from crashing due to creating too many threads.

---

## 🌊 Java 8 Features

**Q17. What is a Functional Interface?**
> An interface that contains exactly ONE abstract method (though it can have multiple default/static methods). It acts as the target type for Lambda expressions. Examples: `Runnable`, `Predicate`, `Function`, `Consumer`.

**Q18. What is the difference between map() and flatMap() in Streams?**
> `map()` applies a function to each element, returning a stream of values (1-to-1 mapping). `flatMap()` applies a function that returns a *stream* of values for each element, and then "flattens" all those distinct streams into one single unified stream (1-to-many mapping).

**Q19. Why was the Optional class introduced?**
> To provide a clear, type-safe alternative to returning `null`. It forces the API consumer to explicitly handle the case where a value might be missing, thereby preventing `NullPointerException`s.

---

## 🍃 Spring Boot (If Applicable)

**Q20. What is Dependency Injection?**
> A design pattern where an object's dependencies are provided to it (usually via constructor) rather than the object creating them itself (`new`). The Spring IoC Container manages the creation and injection of these Beans, promoting loose coupling and making unit testing easier.

**Q21. How does Spring Boot Auto-Configuration work?**
> Spring Boot scans the classpath (`pom.xml` dependencies). If it finds certain classes (like an H2 database driver), and you haven't manually configured a related Bean (like a DataSource), Spring automatically creates and configures that Bean with sensible defaults.

**Q22. `@Controller` vs `@RestController`?**
> `@Controller` is used for traditional MVC apps; methods return view names (like JSP/HTML pages). `@RestController` combines `@Controller` and `@ResponseBody`; every method automatically serializes its return object into JSON/XML and writes it directly to the HTTP response body (used for REST APIs).

**Q23. ApplicationContext vs BeanFactory?**
> Both are IoC containers. `BeanFactory` is the basic container providing DI and lazy-loading. `ApplicationContext` is an advanced sub-interface of BeanFactory that adds enterprise features: AOP integration, internationalization (i18n), event publishing, and eager-loading of singletons. Always use `ApplicationContext` in modern apps.

---

## 🧠 System Design / Architecture

**Q24. Monolith vs Microservices?**
> **Monolith:** Single deployable unit, shared database. Easy to start, hard to scale specific parts, technology lock-in, a bug crashes the whole app.
> **Microservices:** Small, independent services communicating via network (REST/Events). Independent scaling, tech diversity, fault isolation. But introduces immense complexity in networking, distributed data management, and tracing.

**Q25. What is the SOLID principle? (Briefly name them)**
> **S**ingle Responsibility (One reason to change)
> **O**pen/Closed (Open for extension, closed for modification)
> **L**iskov Substitution (Subclasses must be seamlessly replaceable for parent classes)
> **I**nterface Segregation (Many small specific interfaces over one fat interface)
> **D**ependency Inversion (Depend on abstractions/interfaces, not concrete classes).
