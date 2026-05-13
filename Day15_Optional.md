# Day 15 — Optional Class
**Level:** Intermediate | **Java Version:** 8+

---

## 🌟 The Problem with `null`

Tony Hoare, the inventor of the null reference, called it his **"Billion-Dollar Mistake."**
In Java, trying to call a method on a `null` reference throws a `NullPointerException` (NPE) — the most common and frustrating error in Java.

To prevent NPEs, code before Java 8 was filled with defensive null-checks:
```java
// Pre-Java 8: Ugly and error-prone null checks
if (user != null) {
    Address address = user.getAddress();
    if (address != null) {
        String city = address.getCity();
        if (city != null) {
            System.out.println(city.toUpperCase());
        }
    }
}
```

---

## 📦 Enter `Optional<T>`

Java 8 introduced `java.util.Optional<T>`. It is a **container object** which may or may not contain a non-null value.

It acts as a *wrapper*. Instead of returning a `String` (which might be null), a method returns `Optional<String>`. This explicitly forces the caller to handle the possibility that the value might be missing.

### 1. Creating Optionals

```java
// 1. Empty Optional
Optional<String> empty = Optional.empty();

// 2. Optional with a guaranteed non-null value (throws NPE if null is passed)
Optional<String> name = Optional.of("Aman");

// 3. Optional that might be null (most common)
String potentialNull = getSomeStringFromDB();
Optional<String> optName = Optional.ofNullable(potentialNull);
// If potentialNull is null, optName is equivalent to Optional.empty()
```

### 2. Extracting Values (The Wrong Way)
Don't do this. It defeats the purpose of Optional.
```java
Optional<String> opt = Optional.of("Aman");
// BAD: get() throws NoSuchElementException if empty. Always requires isPresent() check.
if (opt.isPresent()) {
    System.out.println(opt.get());
}
```

### 3. Extracting Values (The Right Way - Java 8)

```java
Optional<String> opt = Optional.ofNullable(getName());

// 1. orElse (Provides a default value if empty)
String name1 = opt.orElse("Unknown");

// 2. orElseGet (Provides a default value, computed lazily using a Supplier)
// Use this if the default value is expensive to compute
String name2 = opt.orElseGet(() -> fetchDefaultNameFromDB());

// 3. orElseThrow (Throws an exception if empty)
String name3 = opt.orElseThrow(() -> new IllegalArgumentException("Name missing"));

// 4. ifPresent (Execute lambda ONLY if value is present)
opt.ifPresent(n -> System.out.println("Hello, " + n));
```

### 4. Advanced: Java 9+ Additions

```java
// ifPresentOrElse (Java 9): Execute one logic if present, another if empty
opt.ifPresentOrElse(
    n -> System.out.println("Found: " + n),
    () -> System.out.println("Not found")
);

// or (Java 9): Chain optionals
Optional<String> result = opt1.or(() -> opt2).or(() -> opt3);

// stream() (Java 9): Convert Optional to Stream (0 or 1 element)
List<String> names = listOfOptionals.stream()
    .flatMap(Optional::stream) // Automatically strips out empty Optionals!
    .collect(Collectors.toList());
```

---

## 🔄 Transforming with `map` and `flatMap`

Just like Streams, Optionals have `map` and `flatMap` methods to transform the value *only if it is present*.

```java
class User {
    private String name;
    public String getName() { return name; }
}

Optional<User> optUser = database.findUserById(1);

// map: If optUser has a value, extract the name and wrap it in a new Optional<String>.
// If optUser is empty, returns Optional.empty().
Optional<String> optName = optUser.map(User::getName);

// What if getName() itself returned an Optional<String>?
// Then map() would return Optional<Optional<String>>.
// We use flatMap() to flatten it into a single Optional<String>.
Optional<String> flatName = optUser.flatMap(User::getOptionalName);
```

### The Initial Example, Rewritten with Optional
Remember that nested null-check from the beginning? Here it is using Optional:

```java
// Modern Java: Clean, readable, no NPEs!
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .map(String::toUpperCase)
    .orElse("UNKNOWN CITY");
```

---

## ❓ Interview Questions & Answers

**Q1. What is the purpose of the Optional class?**
> Its primary purpose is to provide a clear, type-safe alternative to returning `null` from a method. By returning `Optional<T>`, you explicitly signal to the API user that the value might be absent, forcing them to handle that case and preventing `NullPointerException`.

**Q2. What is the difference between `Optional.of()` and `Optional.ofNullable()`?**
> `Optional.of(value)` expects a non-null value. If you pass `null` to it, it immediately throws a `NullPointerException`. `Optional.ofNullable(value)` accepts both null and non-null values. If you pass `null`, it gracefully returns an `Optional.empty()`.

**Q3. What is the difference between `orElse()` and `orElseGet()`?**
> `orElse(defaultVal)` *always* evaluates `defaultVal`, even if the Optional contains a value (the evaluated default is just discarded). `orElseGet(Supplier)` evaluates the supplier *only* if the Optional is empty (lazy evaluation). Use `orElseGet` if generating the default value requires a database call or complex computation.

**Q4. Should you use Optional as a field in a class or as a method parameter?**
> **No.** Brian Goetz (Java Language Architect) explicitly stated Optional was designed ONLY as a method *return type*.
> - **Fields:** Optional is not `Serializable`, so using it as a class field breaks serialization.
> - **Parameters:** Doing `public void setAge(Optional<Integer> age)` forces callers to wrap values in Optionals, adding boilerplate. Just use standard overloading or allow nulls for parameters.

**Q5. How do you handle primitive Optionals?**
> Like Streams, wrapping primitives in `Optional<Integer>` causes boxing overhead. Java provides `OptionalInt`, `OptionalLong`, and `OptionalDouble` specifically for primitives. They work similarly but avoid boxing/unboxing.

---

## ✅ Best Practices

1. **Only use Optional as a return type.** Do not use it as class fields, method parameters, or collection elements (e.g., `List<Optional<String>>` is a code smell).
2. **Never call `get()` without `isPresent()`.** Actually, try to avoid `get()` and `isPresent()` entirely. Use `orElse`, `map`, or `ifPresent` instead.
3. **Never return `null` from a method that claims to return an `Optional`.** If you have nothing to return, return `Optional.empty()`.
4. **Use `Optional.ofNullable()` when interfacing with legacy APIs** that might return null.

---

## 🛠️ Hands-on Practice

1. Write a method `Optional<Integer> parseInteger(String str)` that uses a try-catch block. If `Integer.parseInt(str)` succeeds, return `Optional.of(val)`. If it throws `NumberFormatException`, return `Optional.empty()`.
2. Given an `Optional<String> username`, write a single line of code that prints the username in uppercase if present, or prints "GUEST" if not.
3. Write a chain of `map()` calls that takes an `Optional<String>` representing a number, converts it to an Integer (assume valid), adds 10, and collects it into a standard `Integer` variable, defaulting to 0 if the original Optional was empty.
4. Refactor a legacy method that returns `null` when a user is not found in the DB to return an `Optional<User>`. Update the calling code to use `orElseThrow` with a custom `UserNotFoundException`.
