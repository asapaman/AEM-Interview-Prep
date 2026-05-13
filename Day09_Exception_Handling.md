# Day 09 — Exception Handling
**Level:** Beginner → Intermediate | **Java Version:** 8+

---

## 🌟 The Exception Hierarchy

An **Exception** is an event that disrupts the normal flow of the program. Java uses an object-oriented approach to handle errors.

```
Object
  └── Throwable
        ├── Error (Out of our control — e.g., OutOfMemoryError, StackOverflowError)
        └── Exception (Things we can catch and handle)
              ├── IOException, SQLException (Checked Exceptions)
              └── RuntimeException (Unchecked Exceptions)
                    ├── NullPointerException
                    ├── IllegalArgumentException
                    └── ArithmeticException
```

### Checked vs Unchecked Exceptions

| Feature | Checked Exception | Unchecked (Runtime) Exception |
|---------|-------------------|-------------------------------|
| **Base Class** | Extends `Exception` (but not `RuntimeException`) | Extends `RuntimeException` |
| **Compiler Check** | **YES.** Compiler FORCES you to handle it (`try/catch` or `throws`) | **NO.** Compiler doesn't force handling |
| **Meaning** | Recoverable conditions outside your control (e.g., file missing, DB down) | Programming errors (e.g., bug, bad logic, null access) |
| **Examples** | `IOException`, `FileNotFoundException` | `NullPointerException`, `IndexOutOfBoundsException` |

---

## 🛠️ try-catch-finally Blocks

```java
import java.io.*;

public class ExceptionDemo {
    public static void main(String[] args) {
        // 1. The try block contains code that MIGHT throw an exception
        try {
            int result = 10 / 0; // Throws ArithmeticException
            System.out.println("Result: " + result); // Skipped
        }
        // 2. The catch block handles the exception
        catch (ArithmeticException e) {
            System.err.println("Cannot divide by zero: " + e.getMessage());
            // e.printStackTrace(); // Useful for debugging
        }
        // 3. The finally block ALWAYS executes (whether exception occurred or not)
        // Usually used for cleanup (closing connections, files)
        finally {
            System.out.println("Execution completed.");
        }
    }
}
```

### Multi-catch (Java 7+)

You can catch multiple related exceptions in a single block using the pipe `|` operator. The exceptions must not have a parent-child relationship.

```java
try {
    // Code that might throw IOException or SQLException
} catch (IOException | SQLException e) {
    // Handle both types here
    System.err.println("Database or File error: " + e.getMessage());
} catch (Exception e) {
    // Catch-all (must be the LAST block, as it's the parent of all exceptions)
    System.err.println("Unknown error");
}
```

---

## 📦 try-with-resources (Java 7+)

Before Java 7, you had to manually close resources (files, DB connections) in the `finally` block, which was verbose and prone to errors. `try-with-resources` automatically closes any resource that implements `AutoCloseable`.

```java
// ✅ Modern approach
public void readFile() {
    // The resource is declared inside the parentheses
    // It will be AUTOMATICALLY closed at the end of the block
    try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
        String line = reader.readLine();
        System.out.println(line);
    } catch (IOException e) {
        System.err.println("Error reading file: " + e.getMessage());
    }
    // No finally block needed just to call reader.close()!
}

// You can declare multiple resources separated by semicolons
try (
    FileReader fr = new FileReader("in.txt");
    BufferedReader br = new BufferedReader(fr);
    FileWriter fw = new FileWriter("out.txt")
) {
    // Read from br, write to fw
}
```

---

## 🚀 Throwing Exceptions

### The `throw` Keyword
Used to explicitly throw an exception from within a method.

```java
public void setAge(int age) {
    if (age < 0 || age > 150) {
        // Explicitly throw an unchecked exception
        throw new IllegalArgumentException("Invalid age: " + age);
    }
    this.age = age;
}
```

### The `throws` Keyword
Used in the method signature to declare that a method *might* throw a checked exception. It passes the responsibility of handling the exception to the caller.

```java
// Method declares it might throw a checked exception
public void loadData(String filename) throws FileNotFoundException {
    FileReader file = new FileReader(filename); // Can throw FileNotFoundException
}

// The caller MUST handle it
public void process() {
    try {
        loadData("config.xml");
    } catch (FileNotFoundException e) {
        System.err.println("Config file not found.");
    }
}
```

---

## ⚙️ Custom Exceptions

You can create your own exception classes by extending `Exception` (checked) or `RuntimeException` (unchecked). Modern Java leans heavily towards **unchecked** custom exceptions.

```java
// Custom Unchecked Exception
public class UserNotFoundException extends RuntimeException {

    // Default constructor
    public UserNotFoundException() {
        super();
    }

    // Message constructor
    public UserNotFoundException(String message) {
        super(message);
    }

    // Message + cause (for exception wrapping)
    public UserNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Usage
public User getUser(String id) {
    User user = database.findById(id);
    if (user == null) {
        throw new UserNotFoundException("User not found with id: " + id);
    }
    return user;
}
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between `Error` and `Exception`?**
> Both extend `Throwable`. `Error` represents severe problems that a reasonable application should not try to catch (e.g., `OutOfMemoryError`, `StackOverflowError` caused by infinite recursion). `Exception` represents conditions that a reasonable application might want to catch and recover from (e.g., `IOException`, `NullPointerException`).

**Q2. What is the difference between Checked and Unchecked Exceptions?**
> **Checked Exceptions** (extend `Exception`) must be explicitly handled via `try-catch` or declared using `throws` in the method signature. They represent external, recoverable failures (e.g., file I/O). **Unchecked Exceptions** (extend `RuntimeException`) do not require explicit handling. They represent programming errors (e.g., null access, invalid arguments). Modern frameworks (like Spring) prefer unchecked exceptions.

**Q3. Does the `finally` block always execute?**
> Yes, the `finally` block executes almost always: whether an exception occurs, is caught, or isn't caught, or even if there is a `return` statement in the `try` or `catch` block. The only times it does NOT execute are: 1) If `System.exit(0)` is called. 2) If the JVM crashes or the machine loses power. 3) If the thread executing the try block dies or is interrupted before reaching finally.

**Q4. What is Exception Wrapping (or Exception Translation)?**
> Exception wrapping is catching a lower-level exception (like `SQLException`) and re-throwing a higher-level custom exception (like `DatabaseConnectionException`), passing the original exception as the "cause". This encapsulates implementation details. Example: `throw new CustomException("DB failed", originalSqlException);`.

**Q5. Can a subclass overriding a method throw a broader exception than the parent method?**
> No. An overriding method cannot throw *new* or *broader* **checked** exceptions than the overridden method. However, it can throw *fewer* checked exceptions, or *any* **unchecked** exceptions. This maintains the Liskov Substitution Principle (the caller of the parent method shouldn't be surprised by new checked exceptions from a subclass).

---

## ✅ Best Practices

1. **Catch the most specific exception first.** (e.g., catch `FileNotFoundException` before `IOException`).
2. **Never catch `Throwable` or `Error`.** You can't recover from them.
3. **Never leave a catch block empty.** Empty catch blocks swallow errors and make debugging impossible. At minimum, log the error.
4. **Prefer unchecked exceptions** for custom exceptions in modern Java applications (cleaner code, less boilerplate `throws` declarations).
5. **Use `try-with-resources`** for any object implementing `AutoCloseable` to prevent resource leaks.
6. **Include context in exception messages.** Don't just throw `new Exception("Failed")`. Throw `new UserNotFoundException("Failed to load user ID: " + userId)`.
7. **Throw early, catch late.** Validate arguments at the top of a method and throw immediately. Catch exceptions high up in the application architecture (e.g., at the controller/API layer) to return a unified error response.

---

## 🛠️ Hands-on Practice

1. Write a program that divides two numbers provided by the user. Handle `ArithmeticException` (division by zero) and `InputMismatchException` (user enters letters instead of numbers).
2. Create a custom checked exception `InsufficientFundsException`. Write a `BankAccount.withdraw(amount)` method that throws it if balance < amount. Handle it in a `main` method.
3. Refactor code that reads a file using `finally` to close the reader into code that uses `try-with-resources`.
4. Demonstrate exception wrapping: Create a method that throws `IOException`. Catch it in another method and throw a `RuntimeException` with the `IOException` as the cause. Print the stack trace to see the "Caused by" section.
5. Create an array of size 5. Try to access the 10th element in a try block. Add a `finally` block that prints "Cleanup done". Observe the execution flow.
