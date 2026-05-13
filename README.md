# ☕ Java Interview Preparation Guide
### Complete 30-Day Roadmap — Beginner to 4 YOE

> **Who is this for?** Complete beginners who want to start Java development AND developers with 2–4 years of experience preparing for technical interviews.
>
> **What does each file contain?** Concept explanation → Key theory → Complete code examples → Interview Q&A → Best practices → Hands-on exercises.

---

## 📚 Full Index

### 🟢 Core Java (Days 1–10)

| Day | File | Topics |
|-----|------|--------|
| 01 | [OOP Fundamentals](./Day01_OOP_Fundamentals.md) | Classes, Objects, Encapsulation, Constructors, Access Modifiers |
| 02 | [Inheritance & Polymorphism](./Day02_Inheritance_Polymorphism.md) | extends, super, Method Overriding vs Overloading, instanceof |
| 03 | [Interfaces & Abstract Classes](./Day03_Interfaces_Abstract_Classes.md) | interface, abstract class, default methods, when to use which |
| 04–05 | [Collections — List & Set](./Day04_05_Collections_List_Set.md) | ArrayList, LinkedList, HashSet, TreeSet, Comparable, Comparator |
| 06–07 | [Collections — Map & Queue](./Day06_07_Collections_Map_Queue.md) | HashMap, LinkedHashMap, TreeMap, ConcurrentHashMap, PriorityQueue |
| 08 | [Generics](./Day08_Generics.md) | Generic classes, wildcards, bounded types, type erasure |
| 09 | [Exception Handling](./Day09_Exception_Handling.md) | Checked vs Unchecked, try-with-resources, custom exceptions |
| 10 | [String Handling](./Day10_String_Handling.md) | String pool, StringBuilder, StringBuffer, regex, formatting |

### 🔵 Java 8–17+ Features (Days 11–16)

| Day | File | Topics |
|-----|------|--------|
| 11–12 | [Lambdas & Functional Interfaces](./Day11_12_Lambdas_Functional_Interfaces.md) | Lambda syntax, Function, Predicate, Consumer, Supplier |
| 13–14 | [Stream API](./Day13_14_Stream_API.md) | map, filter, flatMap, collect, reduce, parallel streams |
| 15 | [Optional & Method References](./Day15_Optional_Method_References.md) | Optional chaining, method references (::) |
| 16 | [Java 9–17 Features](./Day16_Java9_17_Features.md) | var, records, sealed classes, switch expressions, text blocks |

### 🔴 Advanced Topics (Days 17–22)

| Day | File | Topics |
|-----|------|--------|
| 17–18 | [Multithreading Basics](./Day17_18_Multithreading_Basics.md) | Thread lifecycle, synchronized, volatile, wait/notify |
| 19–20 | [Concurrency Advanced](./Day19_20_Concurrency_Advanced.md) | ExecutorService, Future, Callable, Locks, atomic classes |
| 21–22 | [JVM & Memory & GC](./Day21_22_JVM_Memory_GC.md) | Stack vs Heap, GC algorithms, memory leaks, OOM diagnosis |

### 🟣 Patterns & Tools (Days 23–30)

| Day | File | Topics |
|-----|------|--------|
| 23 | [SOLID Principles](./Day23_SOLID_Principles.md) | SRP, OCP, LSP, ISP, DIP with Java examples |
| 24 | [Design Patterns — Creational](./Day24_Design_Patterns_Creational.md) | Singleton, Factory, Builder, Prototype |
| 25 | [Design Patterns — Structural](./Day25_Design_Patterns_Structural.md) | Adapter, Decorator, Facade, Proxy, Composite |
| 26 | [Design Patterns — Behavioral](./Day26_Design_Patterns_Behavioral.md) | Observer, Strategy, Command, Template Method |
| 27 | [Annotations & Reflection](./Day27_Annotations_Reflection.md) | Custom annotations, retention, reflection API |
| 28 | [Java I/O & NIO](./Day28_Java_IO_NIO.md) | File I/O, Streams, Path, Files API, serialization |
| 29 | [JDBC & Databases](./Day29_JDBC_Database.md) | Connection, PreparedStatement, transactions, connection pools |
| 30 | [REST APIs & HTTP](./Day30_REST_APIs_HTTP.md) | HTTP methods, Jackson JSON, Java 11 HttpClient |

### 🏆 Bonus Modules

| File | Topics |
|------|--------|
| [Collections Internals](./Bonus_Collections_Internals.md) | HashMap internals, ConcurrentHashMap segments, TreeMap red-black tree |
| [JVM Memory Deep Dive](./Bonus_JVM_Memory_Deep_Dive.md) | GC tuning flags, metaspace, memory profiling |
| [Coding Patterns](./Bonus_Coding_Patterns.md) | Sliding window, two pointers, BFS/DFS, DP patterns |
| [Serialization](./Bonus_Serialization.md) | Serializable, Externalizable, transient, SerialVersionUID |
| [Records & Sealed Classes](./Bonus_Records_Sealed_Classes.md) | Java 16–17 records, sealed classes, pattern matching |
| [CompletableFuture](./Bonus_CompletableFuture.md) | Async chains, thenApply, thenCompose, allOf, exception handling |
| [Spring DI & IoC](./Bonus_Spring_DI_IoC.md) | @Component, @Bean, @Autowired, @Qualifier, ApplicationContext |
| [Spring Boot REST](./Bonus_Spring_Boot_REST.md) | @RestController, @RequestMapping, ResponseEntity, validation |
| [Testing with JUnit 5](./Bonus_Testing_JUnit5.md) | @Test, @BeforeEach, Mockito, ArgumentCaptor, test structure |
| [Maven & Gradle](./Bonus_Maven_Gradle.md) | Build lifecycle, dependency scopes, plugins, profiles |
| [Java Security](./Bonus_Java_Security.md) | OWASP top 10, SQL injection, input validation, secrets management |
| [Mock Interview Q&A](./Bonus_Mock_Interview_QA.md) | 75 rapid-fire Java Q&As + power phrases + scoring guide |

---

## 📅 Recommended Study Schedule

### Week 1 — Core Foundations
```
Day 1–2:  OOP (Day01 + Day02)
Day 3–4:  Interfaces + Collections List/Set (Day03 + Day04-05)
Day 5–6:  Collections Map + Generics (Day06-07 + Day08)
Day 7:    Exceptions + Strings (Day09 + Day10)
```

### Week 2 — Java 8+ & Modern Java
```
Day 8–9:  Lambdas + Streams (Day11-12 + Day13-14)
Day 10:   Optional + Java 9-17 features (Day15 + Day16)
Day 11–12: Multithreading Basics + Advanced (Day17-18 + Day19-20)
Day 13:   JVM + GC (Day21-22)
```

### Week 3 — Patterns & Architecture
```
Day 14:   SOLID Principles (Day23)
Day 15–16: Design Patterns (Day24 + Day25 + Day26)
Day 17:   Annotations + Reflection (Day27)
Day 18–19: I/O + JDBC + REST (Day28 + Day29 + Day30)
Day 20:   Bonus: CompletableFuture + Collections Internals
```

### Week 4 — Interview Sprint
```
Day 21–22: Spring Boot REST + Testing (Bonus)
Day 23–24: Coding Patterns (Bonus_Coding_Patterns.md)
Day 25:   Mock Interview Q&A — all 75 questions
Day 26–28: Weak areas revisit
Day 29–30: Practice coding problems out loud
```

---

## 🎯 How to Use This Guide

### For Beginners
1. Read each file top to bottom
2. Type out every code example (don't copy-paste — muscle memory matters)
3. Complete all hands-on tasks before moving to the next file
4. Answer interview questions out loud (say them, don't just read)

### For 2–4 YOE Developers
1. Skim to the interview Q&A section first — if you know all answers, skip the file
2. Focus on files where you're uncertain: Collections Internals, Concurrency, JVM
3. Use `Bonus_Mock_Interview_QA.md` as a daily 15-minute rapid-fire drill
4. Practice coding problems in `Bonus_Coding_Patterns.md`

---

## ⚙️ Environment Setup

```bash
# Verify Java version
java -version   # Should be Java 11+ (Java 17 recommended)

# Compile and run
javac Main.java
java Main

# With Maven
mvn archetype:generate -DgroupId=com.learn -DartifactId=java-prep \
    -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
cd java-prep
mvn compile
mvn exec:java -Dexec.mainClass="com.learn.App"

# Recommended IDE: IntelliJ IDEA Community Edition (free)
# Alternative: VS Code + Extension Pack for Java
```

---

## 📊 Java Version Quick Reference

| Version | LTS? | Key Features |
|---------|------|-------------|
| Java 8 | ✅ | Lambdas, Streams, Optional, default methods |
| Java 11 | ✅ | var, HTTP Client, String methods (strip, isBlank) |
| Java 17 | ✅ | Records, Sealed classes, Pattern matching, Text blocks |
| Java 21 | ✅ | Virtual threads, Sequenced collections, Pattern matching switch |

> This guide targets Java 11 (baseline) with Java 17 features covered in Day16 and Bonus modules.
