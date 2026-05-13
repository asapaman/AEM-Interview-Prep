# Day 13–14 — Streams API
**Level:** Intermediate | **Java Version:** 8+

---

## 🌊 What is a Stream?

Introduced in Java 8, a **Stream** is a sequence of elements supporting sequential and parallel aggregate operations. It is NOT a data structure; it doesn't store data. It pulls data from a source (like a Collection), processes it through a pipeline, and produces a result.

**Key Concepts:**
1. **Source:** Where the data comes from (List, Set, Array, etc.).
2. **Intermediate Operations:** Transform the stream into another stream. They are *lazy* (they don't execute until a terminal operation is called).
3. **Terminal Operation:** Produces a result or side-effect. Once a terminal operation executes, the stream is **consumed** and cannot be used again.

---

## 🛠️ The Stream Pipeline

```java
List<String> names = Arrays.asList("Aman", "Priya", "John", "Alice", "Bob", "Anna");

// Let's find all names starting with 'A', convert to uppercase, and sort them
List<String> aNames = names.stream()               // 1. Source
    .filter(name -> name.startsWith("A"))          // 2. Intermediate (Predicate)
    .map(name -> name.toUpperCase())               // 3. Intermediate (Function)
    .sorted()                                      // 4. Intermediate
    .collect(Collectors.toList());                 // 5. Terminal

System.out.println(aNames); // [ALICE, AMAN, ANNA]
```

---

## 🔄 Intermediate Operations (Lazy)

These return a new `Stream` and are not executed until the terminal operation is invoked.

### 1. `filter(Predicate<T>)`
Keeps elements that match the condition.
```java
Stream.of(1, 2, 3, 4, 5).filter(n -> n % 2 == 0); // yields 2, 4
```

### 2. `map(Function<T, R>)`
Transforms each element into something else.
```java
Stream.of("a", "b", "c").map(s -> s.toUpperCase()); // yields "A", "B", "C"
```

### 3. `flatMap(Function<T, Stream<R>>)`
Flattens nested streams into a single stream. Crucial when your `map` operation returns a Stream or Collection for each element.
```java
List<List<Integer>> listOfLists = Arrays.asList(
    Arrays.asList(1, 2),
    Arrays.asList(3, 4)
);

List<Integer> flatList = listOfLists.stream()
    .flatMap(list -> list.stream()) // Flattens Stream<List<Integer>> to Stream<Integer>
    .collect(Collectors.toList());
// flatList is [1, 2, 3, 4]
```

### 4. `distinct()`, `limit(n)`, `skip(n)`, `sorted()`
```java
Stream.of(1, 1, 2, 3, 3)
    .distinct()          // 1, 2, 3
    .skip(1)             // 2, 3
    .limit(1)            // 2
    .forEach(System.out::println);
```

---

## 🛑 Terminal Operations (Eager)

These trigger the execution of the pipeline.

### 1. `collect()`
Gathers the results into a Collection or Map.
```java
List<String> list = stream.collect(Collectors.toList());
Set<String> set = stream.collect(Collectors.toSet());
String joined = stream.collect(Collectors.joining(", ")); // "A, B, C"
```

### 2. `forEach(Consumer<T>)`
Iterates over elements (usually for side-effects like printing).
```java
stream.forEach(System.out::println);
```

### 3. `reduce(BinaryOperator<T>)`
Combines all elements into a single result (e.g., sum, max).
```java
int sum = Stream.of(1, 2, 3, 4, 5)
    .reduce(0, (a, b) -> a + b); // Returns 15
```

### 4. Matching & Finding
```java
boolean allEven = stream.allMatch(n -> n % 2 == 0);
boolean anyEven = stream.anyMatch(n -> n % 2 == 0);
boolean noneEven = stream.noneMatch(n -> n % 2 == 0);

Optional<String> first = stream.findFirst();
Optional<String> any = stream.findAny(); // Better for parallel streams
```

---

## 🗃️ Advanced Grouping (`Collectors.groupingBy`)

SQL-like `GROUP BY` functionality in Java.

```java
record Employee(String name, String department, double salary) {}

List<Employee> employees = Arrays.asList(
    new Employee("Aman", "IT", 5000),
    new Employee("Priya", "HR", 4000),
    new Employee("John", "IT", 6000)
);

// 1. Group by Department
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department));
// {IT=[Aman, John], HR=[Priya]}

// 2. Group by Department, Count Employees
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department, Collectors.counting()));
// {IT=2, HR=1}

// 3. Group by Department, Sum of Salaries
Map<String, Double> sumSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department, Collectors.summingDouble(Employee::salary)));
// {IT=11000.0, HR=4000.0}
```

---

## ⚡ Parallel Streams

Streams can easily be executed in parallel utilizing multiple CPU cores. Just call `.parallelStream()` instead of `.stream()`.

```java
long count = list.parallelStream()
    .filter(x -> isPrime(x)) // Heavy computation
    .count();
```

**⚠️ Warning:** Do NOT use parallel streams blindly.
- Only beneficial for LARGE datasets (>10,000 items) with CPU-intensive operations.
- The overhead of splitting the data and managing threads can make parallel streams *slower* than sequential streams for small lists or quick operations.
- Ensure your lambda functions are **stateless and thread-safe**.

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between `map()` and `flatMap()`?**
> `map()` applies a function to each element and returns a stream of the results (One-to-One). `flatMap()` applies a function that returns a *stream* for each element, and then *flattens* those multiple streams into a single, unified stream (One-to-Many). Use `flatMap` when dealing with nested collections like `List<List<T>>`.

**Q2. Are Streams lazy? What does that mean?**
> Yes, intermediate operations (like `filter`, `map`) are lazy. They don't process data when they are called. Instead, they build a pipeline of operations. Data only begins to flow through the pipeline when a **terminal operation** (like `collect`, `forEach`) is invoked. This enables optimizations (like loop fusion and short-circuiting).

**Q3. Can you reuse a Stream?**
> No. Once a terminal operation is called on a stream, it is considered "consumed" or "closed". Attempting to perform another operation on it will throw an `IllegalStateException`. You must create a new stream from the source to process it again.

**Q4. What is short-circuiting in Streams?**
> Short-circuiting operations do not need to process the entire stream to produce a result. Examples include `limit(n)`, `findFirst()`, `anyMatch()`. If you have an infinite stream, a short-circuiting operation is necessary to terminate it.

**Q5. When should you use Parallel Streams?**
> Use parallel streams only when: 1) You have a large dataset. 2) The operations are CPU-intensive. 3) The elements are independent and order doesn't matter. 4) The source collection is easy to split (like ArrayList, not LinkedList). Always benchmark before and after applying `parallelStream()` to verify it actually improved performance.

---

## ✅ Best Practices

1. **Avoid side-effects in lambdas.** Lambdas used in streams should not modify variables outside their scope. Use `collect()` to gather results instead of modifying an external list inside `forEach()`.
2. **Format stream pipelines vertically.** Place each intermediate/terminal operation on a new line for readability.
3. **Prefer specialized streams for primitives.** Use `IntStream`, `DoubleStream`, `LongStream` instead of `Stream<Integer>` to avoid boxing/unboxing overhead.
4. **Don't overuse streams.** Simple `for` loops are sometimes faster and easier to read, especially for simple iterations over arrays.

---

## 🛠️ Hands-on Practice

1. Given a `List<String> words`, find the 3 longest words that start with the letter 'c', convert them to uppercase, and join them with a comma.
2. Given a `List<Integer>`, use `reduce()` to find the maximum element.
3. Use `IntStream.rangeClosed(1, 100)` to generate numbers 1 to 100. Filter for prime numbers and collect them into a List.
4. Define a `Person` object. Create a list of persons. Use `Collectors.groupingBy` and `Collectors.mapping` to create a `Map<String, List<String>>` that maps a City to a list of names of people living in that city.
5. Create an infinite stream using `Stream.iterate(0, n -> n + 2)`. Use `limit(10)` and `forEach` to print the first 10 even numbers.
