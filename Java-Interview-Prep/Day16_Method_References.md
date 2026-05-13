# Day 16 — Method References
**Level:** Intermediate | **Java Version:** 8+

---

## 🌟 What are Method References?

A **Method Reference** is a shorthand syntax for a Lambda Expression that executes *just ONE method*. It makes code more readable and concise by referring to an existing method by name instead of writing a full lambda block.

They are created using the double-colon `::` operator.

**The Golden Rule:** You can only replace a lambda with a method reference if the lambda's parameters match the referenced method's arguments perfectly.

---

## 🛠️ The 4 Types of Method References

### 1. Static Method Reference
`ClassName::staticMethodName`

```java
// Using Lambda
Function<String, Integer> parseIntLambda = str -> Integer.parseInt(str);

// Using Method Reference
Function<String, Integer> parseIntRef = Integer::parseInt;

System.out.println(parseIntRef.apply("123")); // 123
```
*Why it works:* `Integer.parseInt(String s)` takes a `String` and returns an `Integer`. The `Function<String, Integer>` interface (`apply` method) also takes a `String` and returns an `Integer`. It's a perfect match.

### 2. Instance Method Reference of a Particular Object
`instanceRef::instanceMethodName`

```java
String prefix = "Hello, ";

// Using Lambda
Function<String, String> greetLambda = name -> prefix.concat(name);

// Using Method Reference (using the existing 'prefix' object)
Function<String, String> greetRef = prefix::concat;

System.out.println(greetRef.apply("Aman")); // "Hello, Aman"
```
*Why it works:* `concat(String str)` is called *on* the `prefix` object, taking the lambda argument as its argument.

### 3. Instance Method Reference of an Arbitrary Object of a Particular Type
`ClassName::instanceMethodName`

This is the trickiest one. The first parameter of the lambda becomes the *target object* upon which the method is invoked. Any subsequent parameters become the arguments to the method.

```java
// Using Lambda
Function<String, String> upperLambda = str -> str.toUpperCase();

// Using Method Reference
Function<String, String> upperRef = String::toUpperCase;

System.out.println(upperRef.apply("java")); // "JAVA"

// Let's look at a BiFunction example for clarity
// Lambda: (str1, str2) -> str1.compareTo(str2)
BiFunction<String, String, Integer> compareLambda = (s1, s2) -> s1.compareTo(s2);

// Method Reference
BiFunction<String, String, Integer> compareRef = String::compareTo;
```
*Why it works:* Even though `toUpperCase()` takes no arguments, the `Function` provides one argument (`str`). Because `toUpperCase()` is an instance method, Java knows to call `str.toUpperCase()`.

### 4. Constructor Reference
`ClassName::new`

```java
class User {
    private String name;
    public User(String name) { this.name = name; }
    // ... getters/setters
}

// Using Lambda
Function<String, User> factoryLambda = name -> new User(name);

// Using Constructor Reference
Function<String, User> factoryRef = User::new;

User u = factoryRef.apply("Aman");
```
*Why it works:* Java looks at the functional interface (a `Function` taking a `String`). It then looks for a constructor in the `User` class that takes a single `String` argument and uses it.

---

## 🌊 Usage in Streams

Method references shine brightest when used within Stream pipelines, reducing visual clutter significantly.

```java
List<String> names = Arrays.asList("Aman", "Priya", "John", "Alice");

// With Lambdas
names.stream()
     .filter(name -> name.startsWith("A"))
     .map(name -> name.toUpperCase())
     .forEach(name -> System.out.println(name));

// With Method References
names.stream()
     .filter(name -> name.startsWith("A"))
     .map(String::toUpperCase)          // Arbitrary object instance method
     .forEach(System.out::println);     // Particular object instance method (System.out)
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between a Lambda Expression and a Method Reference?**
> A Method Reference is essentially a simplified, shorthand notation for a Lambda Expression that only calls an existing method. While a lambda allows you to write custom inline logic, a method reference just points to existing code. They compile down to the same functional interfaces.

**Q2. Can I use a Method Reference if I need to pass additional parameters that aren't provided by the functional interface?**
> No. Method references require a strict mapping between the functional interface's parameters and the referenced method's arguments. If you need to hardcode a parameter or manipulate the input before passing it to the method, you *must* use a full lambda expression.
> *Example:* `x -> myMethod(x, true)` cannot be written as a method reference.

**Q3. How does Java know which constructor to call when using `ClassName::new`?**
> Java uses Type Inference based on the Functional Interface it is being assigned to. If it's assigned to a `Supplier<User>`, it looks for a no-arg constructor. If it's assigned to a `Function<String, User>`, it looks for a constructor that takes a `String`. If no matching constructor exists, a compile error occurs.

**Q4. Explain `System.out::println` in terms of Method Reference types.**
> It is an "Instance Method Reference of a Particular Object". `System.out` is a static field that holds an instance of `PrintStream`. We are referring to the `println` instance method of that specific `PrintStream` object.

---

## ✅ Best Practices

1. **Prefer Method References over Lambdas** when the lambda does nothing but pass its arguments to an existing method. It improves readability.
2. **Stick to Lambdas** if using a method reference makes the code less readable or if you need to perform additional logic (like null checks or calculations) before calling the method.
3. **Beware of Overloading.** If a class has multiple methods with the same name (overloading), method references can sometimes confuse the compiler if the functional interface target isn't perfectly clear. You may have to revert to a lambda.

---

## 🛠️ Hands-on Practice

1. Convert this lambda to a method reference: `Function<Double, Double> sqrt = x -> Math.sqrt(x);`
2. Create a `List<Integer>` of numbers. Use `Collections.sort()` passing a method reference to sort it. (Hint: look at the `Integer` class methods).
3. Given a class `Employee` with a no-arg constructor, create a `Supplier<Employee>` using a constructor reference.
4. Convert this stream pipeline to use method references entirely:
   ```java
   List<String> words = Arrays.asList("apple", "banana", "cherry");
   words.stream()
        .map(w -> w.length())
        .forEach(len -> System.out.println(len));
   ```
