# Day 11–12 — Lambdas & Functional Interfaces
**Level:** Intermediate | **Java Version:** 8+

---

## 🌟 What are Lambdas?

Introduced in Java 8, Lambda Expressions enable **Functional Programming** in Java. They allow you to treat functionality (code) as a method argument, or treat code as data.

Before Lambdas, we used verbose **Anonymous Inner Classes**.

```java
// Before Java 8: Anonymous Inner Class
Runnable taskOld = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running old way!");
    }
};

// With Java 8: Lambda Expression
Runnable taskNew = () -> System.out.println("Running new way!");
```

### Lambda Syntax

`(parameters) -> { body }`

1. **No parameters:** `() -> System.out.println("Hello")`
2. **One parameter:** `x -> x * 2` (parentheses optional for single param)
3. **Multiple parameters:** `(a, b) -> a + b`
4. **Multiple lines:**
   ```java
   (a, b) -> {
       int sum = a + b;
       return sum; // Must use 'return' if using {}
   }
   ```

---

## 🎯 Functional Interfaces

A Lambda expression must be assigned to a **Functional Interface**.
A Functional Interface is an interface that has **exactly ONE abstract method**. (It can have any number of `default` or `static` methods).

```java
@FunctionalInterface // Optional, but recommended for compiler check
public interface MathOperation {
    int operate(int a, int b); // Only ONE abstract method
}

// Usage
MathOperation addition = (a, b) -> a + b;
MathOperation multiplication = (a, b) -> a * b;

System.out.println(addition.operate(5, 3));       // 8
System.out.println(multiplication.operate(5, 3)); // 15
```

---

## 📦 The `java.util.function` Package

Java 8 introduced standard functional interfaces so you don't have to write your own `MathOperation` or `Transformer` interfaces every time.

There are 4 main categories you MUST know:

### 1. Predicate<T>
Takes an input, returns a `boolean`. Used for **filtering**.

```java
// interface Predicate<T> { boolean test(T t); }

Predicate<Integer> isEven = num -> num % 2 == 0;
Predicate<String> isLong = str -> str.length() > 5;

System.out.println(isEven.test(4)); // true

// You can chain predicates!
Predicate<Integer> isEvenAndPositive = isEven.and(num -> num > 0);
System.out.println(isEvenAndPositive.test(-4)); // false
```

### 2. Function<T, R>
Takes an input of type `T`, returns an output of type `R`. Used for **transformation/mapping**.

```java
// interface Function<T, R> { R apply(T t); }

Function<String, Integer> getLength = str -> str.length();
System.out.println(getLength.apply("Hello")); // 5

Function<Integer, Integer> multiplyBy10 = x -> x * 10;
// Chain functions
Function<String, Integer> lengthTimes10 = getLength.andThen(multiplyBy10);
System.out.println(lengthTimes10.apply("Java")); // 40
```

### 3. Consumer<T>
Takes an input, returns `void` (nothing). Used for **consuming/printing** data.

```java
// interface Consumer<T> { void accept(T t); }

Consumer<String> printUpper = str -> System.out.println(str.toUpperCase());
printUpper.accept("hello"); // HELLO

// Used in Iterable.forEach
List<String> names = Arrays.asList("Aman", "John");
names.forEach(printUpper);
```

### 4. Supplier<T>
Takes NO input, returns a value of type `T`. Used for **generating/providing** data.

```java
// interface Supplier<T> { T get(); }

Supplier<Double> randomValue = () -> Math.random();
System.out.println(randomValue.get()); // e.g., 0.723...

Supplier<String> defaultName = () -> "Unknown User";
```

### Bi-Interfaces
Variants that take TWO arguments:
- `BiPredicate<T, U>`: `(a, b) -> boolean`
- `BiFunction<T, U, R>`: `(a, b) -> R`
- `BiConsumer<T, U>`: `(a, b) -> void`

---

## 🔁 Variables Scope & Effectively Final

Lambdas can access variables defined outside their scope, but those variables must be **final or effectively final** (meaning their value is never changed after initialization).

```java
public void testScope() {
    int multiplier = 2; // Not marked final, but it is "effectively final"

    Function<Integer, Integer> math = x -> x * multiplier; // OK

    // multiplier = 3; // UNCOMMENTING THIS CAUSES COMPILE ERROR IN LAMBDA ABOVE
}
```
*Why?* Lambdas might be executed in a different thread later. Local variables live on the stack and are destroyed when the method ends. Java copies the value of the local variable into the lambda. If the value were allowed to change, it would lead to synchronization nightmares.

---

## ❓ Interview Questions & Answers

**Q1. What is a Functional Interface?**
> An interface with exactly one abstract method. It may contain multiple default and static methods. The `@FunctionalInterface` annotation is used to ensure the interface meets this criteria at compile time.

**Q2. What is the difference between a Predicate and a Function?**
> A `Predicate<T>` takes an argument and returns a `boolean`. It's primarily used for filtering or condition checking. A `Function<T, R>` takes an argument of type `T` and transforms it, returning a result of type `R`.

**Q3. Can you modify a local variable from within a lambda expression?**
> No. Local variables accessed inside a lambda must be `final` or "effectively final" (never modified after initialization). If you need to modify state from a lambda, you must use an instance variable, a static variable, or a mutable wrapper object (like `AtomicInteger` or an array of size 1).

**Q4. Why were Default Methods introduced in Java 8 interfaces?**
> To support backward compatibility. When introducing Streams in Java 8, the designers needed to add methods like `.stream()` to the `Collection` interface. Without default methods, doing this would have broken every custom implementation of `Collection` in existence, as they wouldn't have the new method implemented.

**Q5. Explain the concept of "Effectively Final".**
> A variable is effectively final if it is not modified after it is first initialized. Java 8 relaxed the rule that required the explicit `final` keyword for variables accessed in anonymous classes/lambdas; the compiler simply checks if the variable is ever reassigned.

---

## ✅ Best Practices

1. **Keep Lambdas short.** If a lambda is more than 3-4 lines, extract it into a separate named method and use a Method Reference (covered later).
2. **Use standard functional interfaces.** Don't create custom interfaces like `Transformer` or `Action` when `Function` or `Consumer` already exist in `java.util.function`.
3. **Use `@FunctionalInterface`.** Always annotate your custom functional interfaces to prevent someone from accidentally adding a second abstract method later.
4. **Avoid complex logic in lambdas.** Lambdas should express *what* to do, not complex *how* mechanics.

---

## 🛠️ Hands-on Practice

1. Write a `Predicate<String>` that checks if a string starts with "A" and has a length > 3. Test it.
2. Write a `Function<String, Integer>` that takes a date string "YYYY-MM-DD" and returns the year as an Integer.
3. Write a `Supplier<String>` that generates a random 6-character alphanumeric password.
4. Use a `BiConsumer<String, Integer>` to print out a person's name and age in the format: "Name: [name], Age: [age]".
5. Create a `List<Integer>`, use `.removeIf()` passing a lambda (Predicate) to remove all odd numbers from the list.
