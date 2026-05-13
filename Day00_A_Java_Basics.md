# Day 00(A) — Java Basics & Fundamentals
**Level:** Beginner | **Java Version:** 8+

---

## 🌟 1. Java Program Structure

Every Java application begins with a class name, and that class must match the filename. The entry point of a Java application is the `public static void main(String[] args)` method.

```java
// File: HelloWorld.java
public class HelloWorld {
    
    // The main method is where execution begins
    public static void main(String[] args) {
        System.out.println("Hello, World!"); // Prints to the console
    }
}
```

---

## 📦 2. Variables and Data Types

Java is **statically typed**, meaning all variables must first be declared before they can be used.

### Primitive Data Types (Stored in Stack Memory)
There are 8 primitive data types in Java.

| Type | Size | Description / Range |
|------|------|---------------------|
| `byte` | 1 byte | -128 to 127 |
| `short` | 2 bytes | -32,768 to 32,767 |
| `int` | 4 bytes | Standard integer (approx. ±2 billion) |
| `long` | 8 bytes | Large integer. **Must end with 'L'** (e.g., `100L`) |
| `float` | 4 bytes | Decimal number. **Must end with 'f'** (e.g., `3.14f`) |
| `double`| 8 bytes | Standard precise decimal (e.g., `3.14`) |
| `boolean`| 1 bit | `true` or `false` |
| `char` | 2 bytes | Single 16-bit Unicode character (e.g., `'A'`) |

```java
int age = 25;
double price = 19.99;
boolean isOnline = true;
char grade = 'A';
```

### Reference Data Types (Stored in Heap Memory)
Any type that is not a primitive is a reference type. This includes `String`, Arrays, and all Objects. Reference variables store the *memory address* of the object, not the object itself.

```java
String name = "Aman"; // String is a Class, not a primitive!
```

---

## 🔁 3. Control Flow Statements

### If-Else Statements
```java
int score = 85;

if (score >= 90) {
    System.out.println("Grade: A");
} else if (score >= 80) {
    System.out.println("Grade: B");
} else {
    System.out.println("Grade: C");
}
```

### Switch Statement (Enhanced in Java 14+)
```java
String day = "MONDAY";

// Traditional Switch
switch(day) {
    case "MONDAY":
    case "TUESDAY":
        System.out.println("Weekday");
        break; // Crucial! Without break, it falls through to the next case
    case "SATURDAY":
        System.out.println("Weekend");
        break;
    default:
        System.out.println("Invalid day");
}

// Modern Switch Expression (Java 14+) - No breaks needed!
String type = switch(day) {
    case "MONDAY", "TUESDAY" -> "Weekday";
    case "SATURDAY", "SUNDAY" -> "Weekend";
    default -> "Invalid day";
};
```

---

## 🔄 4. Loops

### For Loop
Best when you know exactly how many times you want to loop.
```java
for (int i = 0; i < 5; i++) {
    System.out.println("Count: " + i); // Prints 0, 1, 2, 3, 4
}
```

### Enhanced For Loop (For-Each)
Best for iterating through arrays or Collections.
```java
int[] numbers = {10, 20, 30};
for (int num : numbers) {
    System.out.println(num);
}
```

### While Loop
Best when you don't know how many iterations are needed beforehand.
```java
int count = 0;
while (count < 3) {
    System.out.println("While count: " + count);
    count++;
}
```

### Do-While Loop
Guaranteed to execute **at least once**, even if the condition is false initially.
```java
int x = 10;
do {
    System.out.println("This runs once even though x > 5");
} while (x < 5);
```

---

## 🛠️ 5. Methods (Functions)

Methods break code into reusable blocks. 

```java
public class MathUtils {
    
    // 1. Access Modifier (public)
    // 2. Static (belongs to class, no object needed)
    // 3. Return Type (int)
    // 4. Method Name (addNumbers)
    // 5. Parameters (int a, int b)
    public static int addNumbers(int a, int b) {
        return a + b;
    }

    public static void main(String[] args) {
        int result = addNumbers(5, 10); // Calling the method
        System.out.println("Result: " + result);
    }
}
```

### Pass-by-Value vs Pass-by-Reference
Java is **STRICTLY PASS-BY-VALUE**.
- **Primitives:** A copy of the actual value is passed. Modifying it inside the method does not affect the original variable.
- **Objects:** A copy of the *reference* (memory address) is passed. If you modify the object's properties inside the method, the original object changes (because both references point to the same object in the heap). However, if you reassign the reference itself to a `new` object inside the method, the original reference outside the method is completely unaffected.

---

## 🧱 6. Operators

- **Arithmetic:** `+`, `-`, `*`, `/`, `%` (modulo - returns remainder)
- **Increment/Decrement:** `++`, `--` 
  - `x++` (Post-increment: uses current value, then increments)
  - `++x` (Pre-increment: increments first, then uses new value)
- **Relational:** `==`, `!=`, `>`, `<`, `>=`, `<=`
- **Logical:** `&&` (AND), `||` (OR), `!` (NOT)
  - Java uses *Short-Circuit Evaluation*: In `A && B`, if `A` is false, `B` is never evaluated because the whole expression is already guaranteed to be false.
- **Ternary Operator:** `condition ? valueIfTrue : valueIfFalse`
  - `String status = (age >= 18) ? "Adult" : "Minor";`

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between JDK, JRE, and JVM?**
> **JVM (Java Virtual Machine):** Executes the bytecode. It provides the runtime environment.
> **JRE (Java Runtime Environment):** Contains the JVM + core class libraries needed to *run* Java programs.
> **JDK (Java Development Kit):** Contains the JRE + development tools (like the `javac` compiler, debuggers, JavaDoc) needed to *write* and compile Java programs.

**Q2. Is Java 100% Object-Oriented?**
> No. Java uses primitive data types (`int`, `char`, `boolean`, etc.) which are not objects. This was a design choice made for performance reasons. However, Java provides Wrapper classes (`Integer`, `Character`) to treat them as objects when necessary (like when using Collections).

**Q3. Explain `public static void main(String[] args)`.**
> `public`: Accessible by the JVM from anywhere.
> `static`: Can be called without creating an instance (object) of the class (since at startup, no objects exist yet).
> `void`: Returns nothing to the operating system.
> `main`: The configured entry point name the JVM looks for.
> `String[] args`: Command-line arguments passed to the program at startup.

**Q4. What is Type Casting and what are the two types?**
> Type casting is converting a value from one data type to another.
> **Implicit Casting (Widening):** Automatic conversion of a smaller type to a larger type (e.g., `int` to `double`). Safe, no data loss.
> **Explicit Casting (Narrowing):** Manual conversion of a larger type to a smaller type (e.g., `double` to `int`). Requires parentheses `(int)`. Unsafe, can cause data loss (e.g., losing decimal precision).

---

## 🛠️ Hands-on Practice

1. Write a program that calculates the factorial of a number using a `for` loop.
2. Write a method `boolean isPrime(int n)` that returns true if a number is prime.
3. Write a program demonstrating pass-by-value using a primitive `int`, and pass-by-value-of-reference using an array (change the 0th element inside the method and observe the original array).
4. Use a switch statement to take an integer (1-7) and print the corresponding day of the week.
