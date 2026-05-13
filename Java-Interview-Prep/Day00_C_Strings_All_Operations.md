# Day 00(C) — Strings: Deep Dive & All Operations
**Level:** Beginner → Intermediate | **Java Version:** 8+

---

## 🌟 1. String Fundamentals

A `String` in Java is an object that represents a sequence of characters.
**Crucial Concept: IMMUTABILITY.** Once a `String` object is created, its data/state cannot be changed. Any operation that modifies a string actually returns a brand **new** `String` object.

### Creation & The String Pool
```java
// 1. String Literal (Uses String Constant Pool in Heap)
// If "Hello" exists in the pool, s1 reuses it. Highly memory efficient.
String s1 = "Hello"; 
String s2 = "Hello"; 
System.out.println(s1 == s2); // true (Same memory reference)

// 2. new Keyword (Creates a new object directly in Heap, bypassing the pool initially)
String s3 = new String("Hello");
System.out.println(s1 == s3); // false (Different memory addresses)

// 3. intern() method (Forces the string into the pool)
String s4 = s3.intern();
System.out.println(s1 == s4); // true
```

---

## 🛠️ 2. Comprehensive String Operations

Here is a categorized list of almost every `String` method you will need for interviews and daily development.

### A. Inspection Methods (Examining the String)
```java
String str = "Java Programming";

int len = str.length();               // 16
char ch = str.charAt(5);              // 'P' (Zero-indexed)

boolean empty = str.isEmpty();        // false (true if length is 0)
boolean blank = str.isBlank();        // false (Java 11+: true if empty OR only whitespaces)

int firstA = str.indexOf('a');        // 1 (Index of first occurrence)
int lastA = str.lastIndexOf('a');     // 10 (Index of last occurrence)
int noMatch = str.indexOf("Python");  // -1 (Returned if substring not found)

boolean hasJava = str.contains("Java"); // true
boolean starts = str.startsWith("Ja");  // true
boolean ends = str.endsWith("ing");     // true
```

### B. Comparison Methods
```java
String s1 = "hello";
String s2 = "HELLO";

// NEVER use == to compare string values!
boolean isEqual = s1.equals(s2);             // false (Case sensitive)
boolean isEqualIg = s1.equalsIgnoreCase(s2); // true

// compareTo returns integer (Dictionary/Lexicographical sorting)
// Returns 0 if equal, negative if s1 < s3, positive if s1 > s3
String s3 = "apple";
int diff = s1.compareTo(s3);                 // Positive number ('h' comes after 'a')
int diffIg = s1.compareToIgnoreCase(s2);     // 0
```

### C. Modification Methods (Returns NEW Strings)
```java
String text = "  Learn Java!  ";

String lower = text.toLowerCase();    // "  learn java!  "
String upper = text.toUpperCase();    // "  LEARN JAVA!  "

String trimmed = text.trim();         // "Learn Java!" (Removes leading/trailing spaces)
String stripped = text.strip();       // "Learn Java!" (Java 11+ version of trim, Unicode aware)

// Replacing
String replaced = text.replace('a', 'o');          // "  Leorn Jovo!  "
String repStr = text.replace("Java", "Python");    // "  Learn Python!  "
String regexRep = text.replaceAll("[aeiou]", "*"); // "  L**rn J*v*!  " (Uses Regex)
String regexFirst = text.replaceFirst("a", "*");   // Replaces only the FIRST 'a'

// Substrings
String sub1 = text.substring(8);      // "ava!  " (From index 8 to end)
String sub2 = text.substring(2, 7);   // "Learn" (From index 2 up to, BUT NOT INCLUDING, 7)
```

### D. Splitting and Joining
```java
String csv = "apple,banana,orange";

// Split string into an Array based on a delimiter (Regex)
String[] fruits = csv.split(","); // ["apple", "banana", "orange"]

// Join an Array or Iterable into a single String (Java 8+)
String joined = String.join(" - ", fruits); // "apple - banana - orange"
```

### E. Conversion Methods
```java
int number = 100;

// Convert primitive/object TO String
String strNum1 = String.valueOf(number); // "100" (Preferred, handles nulls gracefully)
String strNum2 = Integer.toString(number); // "100"
String strNum3 = number + ""; // "100" (Works, but less efficient under the hood)

// Convert String TO primitive
int parsedInt = Integer.parseInt("100");
double parsedDouble = Double.parseDouble("99.99");

// Convert String to Array
char[] charArray = "Hello".toCharArray(); // ['H', 'e', 'l', 'l', 'o']
byte[] byteArray = "Hello".getBytes();    // Array of ASCII/Unicode byte values
```

### F. String Formatting
```java
String name = "Aman";
int age = 28;
double salary = 12345.678;

// %s=String, %d=Integer, %f=Float/Double, %.2f=2 decimal places
String formatted = String.format("Name: %s, Age: %d, Salary: $%.2f", name, age, salary);
System.out.println(formatted); // "Name: Aman, Age: 28, Salary: $12345.68"
```

---

## 🏗️ 3. StringBuilder and StringBuffer

Since `String` is immutable, doing `str += "a"` in a loop creates thousands of temporary garbage objects.
**Solution:** Use `StringBuilder` (or `StringBuffer`) when modifying strings continuously.

| Feature | `String` | `StringBuilder` | `StringBuffer` |
|---------|----------|-----------------|----------------|
| **Mutability** | Immutable | Mutable | Mutable |
| **Thread-Safe** | Yes (Inherently) | **No** (Fastest) | Yes (Synchronized methods, slower) |

### StringBuilder Operations
```java
StringBuilder sb = new StringBuilder("Java");

// 1. Append (Add to end)
sb.append(" Programming"); // "Java Programming"
sb.append(123);            // "Java Programming123"

// 2. Insert (Add at specific index)
sb.insert(4, " is");       // "Java is Programming123"

// 3. Delete (fromIndex, toIndex exclusive)
sb.delete(19, 22);         // "Java is Programming"

// 4. Delete character at index
sb.deleteCharAt(4);        // "Java is Programming" -> "Java s Programming"

// 5. Replace (fromIndex, toIndex, newString)
sb.replace(0, 4, "Python"); // "Python s Programming"

// 6. Reverse (Very common interview requirement!)
sb.reverse();              // "gnimmargorP s nohtyP"

// 7. Convert back to String
String finalResult = sb.toString();
```

---

## ❓ Interview Questions & Answers

**Q1. Why are Strings immutable in Java?**
> 1. **Security:** Strings are used for DB connections, network URLs, and passwords. If mutable, a hacker could change the URL reference after it passed security validation.
> 2. **String Pool:** Immutability allows the JVM to cache string literals. If strings were mutable, changing one would unexpectedly change others pointing to the same literal.
> 3. **Thread Safety:** Immutable objects can be freely shared across multiple threads without synchronization.
> 4. **Caching Hashcode:** Since a string never changes, its hashcode never changes, making it extremely fast and safe as a key in `HashMap`s.

**Q2. What is the difference between `str.length` and `arr.length`?**
> For Arrays, `length` is a `final` variable (property). You access it without parentheses.
> For Strings, `length()` is a method. It calculates and returns the number of characters backed by the internal array.

**Q3. How do you check if a String is a Palindrome?**
> A palindrome reads the same forwards and backwards (e.g., "radar").
> *Quick way:* `return str.equals(new StringBuilder(str).reverse().toString());`
> *Efficient way (Interview preferred):* Use two pointers, one at the start, one at the end, and compare characters moving inward.

**Q4. What is the difference between `replace()` and `replaceAll()`?**
> Both replace all occurrences in a string. However, `replace()` takes literal strings or chars (e.g., `replace("a", "b")`). `replaceAll()` takes a **Regular Expression (Regex)** (e.g., `replaceAll("\\d", "*")` replaces all digits with asterisks).

**Q5. Why should you store passwords in `char[]` instead of `String`?**
> Strings are immutable and are cached in the String Pool. If you store a password in a String, it stays in memory until Garbage Collected (which you can't control). A memory dump could reveal the password in plain text. A `char[]` can be explicitly wiped/overwritten with dummy characters immediately after authentication is complete.

---

## 🛠️ Hands-on Practice

1. Write a program to count the occurrences of a specific character in a String without using loops (Hint: Use `length()` and `replace()`).
2. Write a method to reverse the *words* in a sentence, not the characters. (e.g., "Java is fun" -> "fun is Java"). Use `split(" ")` and `StringBuilder`.
3. Check if two strings are Anagrams of each other (contain exactly the same characters, e.g., "listen" and "silent"). Hint: Convert to `char[]`, use `Arrays.sort()`, and `Arrays.equals()`.
4. Write a program to find the first non-repeated character in a String. (e.g., in "swiss", it should return 'w').
