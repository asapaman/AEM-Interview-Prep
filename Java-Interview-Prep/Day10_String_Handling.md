# Day 10 — String Handling
**Level:** Beginner → Intermediate | **Java Version:** 8+ (with 11+ features)

---

## 🌟 The `String` Class

In Java, `String` is an **object**, not a primitive. It is backed by an array of characters (or bytes in Java 9+).

**Key Characteristic:** Strings in Java are **IMMUTABLE**. Once created, a `String` object cannot be changed. Any operation that seems to modify a string actually creates a *new* string object.

```java
String s1 = "Hello";
s1.concat(" World"); // Creates "Hello World", but s1 still points to "Hello"
s1 = s1.concat(" World"); // Now s1 points to the new "Hello World" object
```

### Why Immutable?
1. **Security:** Strings are used for passwords, network URLs, and DB connections. Immutability ensures they can't be altered after validation.
2. **Thread Safety:** Immutable objects are inherently thread-safe. Multiple threads can read the same string without synchronization.
3. **Caching (String Pool):** Because they don't change, identical string literals can share the same memory location.
4. **HashMap Keys:** Immutability guarantees the `hashCode()` won't change, making `String` an ideal key for Maps.

---

## 🏊 The String Constant Pool

To save memory, Java maintains a special area in the Heap called the **String Constant Pool**.

```java
// 1. Literal creation (Uses the String Pool)
String a = "Java";
String b = "Java";
System.out.println(a == b); // TRUE: Both point to the exact same object in the pool

// 2. new keyword creation (Bypasses the pool, creates a new object on the heap)
String c = new String("Java");
String d = new String("Java");
System.out.println(a == c); // FALSE: Different memory addresses
System.out.println(c == d); // FALSE: Different memory addresses

// 3. intern() method
// Forces the string into the pool (or returns the reference if already there)
String e = c.intern();
System.out.println(a == e); // TRUE: e now points to the pool instance
```

> **Golden Rule:** ALWAYS use `.equals()` to compare string contents, NEVER use `==`. `==` compares memory addresses, not the actual text.

---

## 🛠️ Essential String Methods

```java
String text = "  Java Programming  ";

// Length and basics
int len = text.length();            // 20
boolean empty = text.isEmpty();     // false
boolean blank = text.isBlank();     // false (Java 11+) - true if empty or only whitespace

// Trimming
String trimmed = text.trim();       // "Java Programming"
String stripped = text.strip();     // "Java Programming" (Java 11+ - better Unicode support)

// Changing Case
String lower = text.toLowerCase();  // "  java programming  "
String upper = text.toUpperCase();  // "  JAVA PROGRAMMING  "

// Inspection
boolean starts = text.startsWith("  Ja"); // true
boolean ends = text.endsWith("ng  ");     // true
boolean contains = text.contains("Pro");  // true
int index = text.indexOf("a");            // 3 (first occurrence)
int lastIndex = text.lastIndexOf("a");    // 12 (last occurrence)

// Extraction
String sub1 = text.substring(2);       // "Java Programming  " (from index to end)
String sub2 = text.substring(2, 6);    // "Java" (from index to index-1)

// Modification (creates NEW strings)
String replaced = text.replace("Java", "Python"); // "  Python Programming  "
String noSpaces = text.replace(" ", "");          // "JavaProgramming"

// Splitting
String csv = "apple,banana,orange";
String[] fruits = csv.split(","); // ["apple", "banana", "orange"]

// Joining (Java 8+)
String joined = String.join(" - ", fruits); // "apple - banana - orange"
```

---

## 🏗️ `StringBuilder` and `StringBuffer`

Since `String` is immutable, doing many string concatenations in a loop creates many temporary objects, wasting memory and CPU (Garbage Collection overhead).

Instead, use **mutable** alternatives when building strings iteratively.

### `StringBuilder` (Fast, Not Thread-Safe)
Use this 99% of the time when you need mutable strings.

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 5; i++) {
    // Modifies the SAME object internally. No new objects created.
    sb.append("Item").append(i).append(" ");
}
String result = sb.toString(); // Convert back to String when done
System.out.println(result); // "Item0 Item1 Item2 Item3 Item4 "
```

### `StringBuffer` (Slower, Thread-Safe)
Almost identical to `StringBuilder`, but all its methods are `synchronized`. Only use if multiple threads are appending to the *same* buffer concurrently (rare).

```java
StringBuffer buffer = new StringBuffer();
buffer.append("Safe ").append("for ").append("threads");
```

---

## 🎨 String Formatting

Use `String.format()` to create formatted strings (similar to `printf` in C).

```java
String name = "Aman";
int age = 28;
double salary = 85000.50;

// %s = String, %d = Integer, %f = Float/Double, %n = newline
String info = String.format("Name: %s, Age: %d, Salary: $%.2f", name, age, salary);
System.out.println(info); // "Name: Aman, Age: 28, Salary: $85000.50"
```

---

## 📝 Text Blocks (Java 15+)

Multi-line strings are now much easier. No more `\n` and `+` concatenation.

```java
// Traditional
String jsonOld = "{\n" +
                 "  \"name\": \"Aman\",\n" +
                 "  \"age\": 28\n" +
                 "}";

// Java 15+ Text Blocks (use """ )
String jsonNew = """
                 {
                   "name": "Aman",
                   "age": 28
                 }
                 """;
```

---

## ❓ Interview Questions & Answers

**Q1. Why are Strings immutable in Java?**
> Security (parameters passed to network connections/databases can't be maliciously altered), Thread Safety (immutable objects are inherently thread-safe), Caching (String pool is only possible if strings don't change), and Performance (hashCode can be cached at creation, making String ideal for Map keys).

**Q2. What is the String Constant Pool?**
> A special storage area in the Java heap. When a string is created using a literal (e.g., `String s = "Hello"`), Java checks the pool. If "Hello" exists, it returns a reference to it. If not, it creates a new string in the pool. This saves memory by reusing identical strings.

**Q3. What is the difference between `==` and `.equals()` for Strings?**
> `==` checks for reference equality (do they point to the exact same memory location?). `.equals()` checks for value equality (do they contain the exact same characters in the same order?). Always use `.equals()` for strings.

**Q4. What is the difference between `String`, `StringBuilder`, and `StringBuffer`?**
> `String` is immutable. Every modification creates a new object. `StringBuilder` is mutable and fast, but not thread-safe. Used for string concatenation in loops. `StringBuffer` is mutable and thread-safe (synchronized methods), but slower due to locking overhead.

**Q5. How does the `intern()` method work?**
> When `s.intern()` is invoked, if the string pool already contains a string equal to `s`, the reference from the pool is returned. Otherwise, this `String` object is added to the pool and a reference to it is returned. It forces string deduplication.

---

## ✅ Best Practices

1. **Never use `==` to compare Strings.** Use `s1.equals(s2)` or `s1.equalsIgnoreCase(s2)`.
2. **Use `StringBuilder` for concatenation in loops.** Doing `str += "x"` in a loop creates `n` new objects.
3. **Use literal notation `String s = "text"`** instead of `new String("text")` to utilize the string pool.
4. **Use `String.valueOf(object)` instead of `object.toString()`** if the object might be null, to avoid `NullPointerException` (returns "null" instead).
5. **Use Java 11's `isBlank()` instead of `isEmpty()`** when validating user input (it checks for whitespaces too).

---

## 🛠️ Hands-on Practice

1. Write a method `boolean isPalindrome(String s)` that ignores case and spaces. (e.g., "A man a plan a canal Panama" -> true). Use `StringBuilder.reverse()`.
2. Write a program that counts the number of vowels and consonants in a given string.
3. Demonstrate the difference between `+` concatenation in a loop (10,000 iterations) vs `StringBuilder`. Measure the time taken using `System.currentTimeMillis()`.
4. Use `String.split()` to parse a CSV line: `"101,Aman,Engineering,85000"`. Extract and print the department.
5. Create a multi-line JSON string using Java Text Blocks (or standard concatenation if on older Java) and use `String.format()` to inject dynamic values into it.
