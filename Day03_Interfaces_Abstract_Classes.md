# Day 03 — Interfaces & Abstract Classes
**Level:** Beginner → Intermediate | **Java Version:** 8+

---

## 🌟 The Problem They Solve

Both `interface` and `abstract class` define a **contract** — what a class must do, without necessarily saying how. They enable abstraction: "I know this object can `makeSound()`, I don't care if it's a Dog or Cat."

---

## 📋 Abstract Classes

An **abstract class** is a class that:
- Cannot be instantiated directly (you can't do `new Animal()`)
- Can have both abstract methods (no body) AND concrete methods (with body)
- Can have fields (state), constructors, and static methods

```java
// Abstract class — partially implemented
public abstract class Animal {
    // Fields (state) — allowed in abstract class
    private String name;
    private int age;

    // Constructor — abstract classes CAN have constructors
    public Animal(String name, int age) {
        this.name = name;
        this.age  = age;
    }

    // ABSTRACT method — subclass MUST implement this
    public abstract String makeSound();

    // ABSTRACT method — subclass MUST implement this
    public abstract String getAnimalType();

    // CONCRETE method — shared by all animals (no override needed)
    public void eat() {
        System.out.println(name + " is eating.");
    }

    // CONCRETE method using abstract method — template method pattern
    public void introduce() {
        System.out.printf("Hi! I'm %s, a %s, and I say: %s%n",
            name, getAnimalType(), makeSound());
    }

    // Getters
    public String getName() { return name; }
    public int getAge()     { return age;  }
}

// Concrete subclass — MUST implement all abstract methods
public class Dog extends Animal {
    private String breed;

    public Dog(String name, int age, String breed) {
        super(name, age);
        this.breed = breed;
    }

    @Override
    public String makeSound()     { return "Woof!"; }

    @Override
    public String getAnimalType() { return "Dog (" + breed + ")"; }
}

public class Cat extends Animal {
    public Cat(String name, int age) {
        super(name, age);
    }

    @Override
    public String makeSound()     { return "Meow!"; }

    @Override
    public String getAnimalType() { return "Cat"; }
}

// Usage
// Animal a = new Animal("X", 1);  // COMPILE ERROR — cannot instantiate abstract class
Dog dog = new Dog("Buddy", 3, "Labrador");
Cat cat = new Cat("Whiskers", 5);

dog.introduce();  // Hi! I'm Buddy, a Dog (Labrador), and I say: Woof!
cat.introduce();  // Hi! I'm Whiskers, a Cat, and I say: Meow!
```

---

## 📐 Interfaces

An **interface** defines a pure contract — only what a class must do, not how. It's a 100% abstract type (before Java 8). Since Java 8, interfaces can have `default` and `static` methods with implementations.

```java
// Interface — defines a capability/contract
public interface Printable {
    // All methods are public abstract by default (before Java 8)
    void print();
    String getContent();

    // Default method (Java 8+) — optional to override
    default void printWithBorder() {
        System.out.println("=".repeat(40));
        print();
        System.out.println("=".repeat(40));
    }

    // Static method (Java 8+) — belongs to the interface, not instances
    static Printable of(String content) {
        return new SimplePrintable(content);
    }

    // Private method (Java 9+) — helper for default methods
    private void log(String message) {
        System.out.println("[LOG] " + message);
    }
}

// A class can implement MULTIPLE interfaces
public interface Saveable {
    void save(String filename);
    boolean isModified();
}

public interface Shareable {
    void share(String recipient);
}

// One class, multiple interfaces
public class Document implements Printable, Saveable, Shareable {
    private String title;
    private String content;
    private boolean modified;

    public Document(String title, String content) {
        this.title   = title;
        this.content = content;
        this.modified = true;
    }

    @Override
    public void print() {
        System.out.println("Document: " + title);
        System.out.println(content);
    }

    @Override
    public String getContent() { return content; }

    @Override
    public void save(String filename) {
        System.out.println("Saving '" + title + "' to " + filename);
        this.modified = false;
    }

    @Override
    public boolean isModified() { return modified; }

    @Override
    public void share(String recipient) {
        System.out.println("Sharing '" + title + "' with " + recipient);
    }
}

// Usage
Document doc = new Document("Report", "Q1 results are excellent.");
doc.print();           // Uses own print()
doc.printWithBorder(); // Uses default method from Printable
doc.save("report.txt");
doc.share("boss@company.com");
```

---

## 🔑 Interface Constants & Functional Interfaces

```java
// Interfaces can have constants (public static final — implicit)
public interface MathConstants {
    double PI = 3.14159265;          // implicitly public static final
    double E  = 2.71828182;
    int MAX_PRECISION = 10;
}

// Functional Interface — exactly ONE abstract method
// Can be used as a lambda expression (Java 8+)
@FunctionalInterface
public interface Calculator {
    int calculate(int a, int b);  // The one abstract method

    // Can still have default methods
    default void printResult(int a, int b) {
        System.out.println("Result: " + calculate(a, b));
    }
}

// Using as lambda
Calculator add      = (a, b) -> a + b;
Calculator subtract = (a, b) -> a - b;
Calculator multiply = (a, b) -> a * b;

add.printResult(5, 3);       // Result: 8
subtract.printResult(10, 4); // Result: 6
multiply.printResult(3, 7);  // Result: 21
```

---

## ⚖️ Interface vs Abstract Class — The Decision Matrix

| Feature | Interface | Abstract Class |
|---------|-----------|---------------|
| **Multiple inheritance** | ✅ A class can implement many | ❌ A class extends only ONE |
| **Fields / State** | ❌ Only constants (`final static`) | ✅ Any fields |
| **Constructor** | ❌ Cannot have | ✅ Can have |
| **Method types** | `default`, `static`, `private` (Java 9) | abstract + concrete |
| **Access modifiers** | All methods are `public` by default | Any access modifier |
| **"is-a" relationship** | No (capability/role) | Yes (type hierarchy) |
| **When to use** | Define a capability/role | Share common state + behaviour |

```
Decision guide:
┌─────────────────────────────────────────────────────────────┐
│ Do you need to share STATE (fields)?                        │
│   YES → Abstract class                                      │
│   NO  ↓                                                     │
│ Do you need multiple inheritance?                           │
│   YES → Interface                                           │
│   NO  ↓                                                     │
│ Is this a clear IS-A relationship?                          │
│   YES → Abstract class  (Dog IS-A Animal)                   │
│   NO  → Interface  (Dog CAN Swim, CAN Fetch)               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🎯 Interface Segregation — Keep Them Small

```java
// ❌ FAT INTERFACE — forces implementors to implement methods they don't need
public interface Worker {
    void work();
    void eat();         // Robots don't eat!
    void sleep();       // Robots don't sleep!
}

// ✅ SEGREGATED INTERFACES — each interface represents one responsibility
public interface Workable { void work(); }
public interface Eatable  { void eat();  }
public interface Sleepable { void sleep(); }

public class Human implements Workable, Eatable, Sleepable {
    @Override public void work()  { System.out.println("Human working"); }
    @Override public void eat()   { System.out.println("Human eating");  }
    @Override public void sleep() { System.out.println("Human sleeping");}
}

public class Robot implements Workable {
    @Override public void work() { System.out.println("Robot working 24/7"); }
    // No eat() or sleep() needed!
}
```

---

## 🔄 Default Method Conflict Resolution

When a class implements two interfaces with the same `default` method:

```java
public interface A {
    default String greet() { return "Hello from A"; }
}

public interface B {
    default String greet() { return "Hello from B"; }
}

// Class must override to resolve the conflict
public class C implements A, B {
    @Override
    public String greet() {
        // Choose which one to use, or provide completely new implementation
        return A.super.greet() + " and " + B.super.greet();
        // "Hello from A and Hello from B"
    }
}
```

---

## 🏗️ Real-World Design Example

```java
// Payment processing system — using interfaces for flexibility

public interface PaymentProcessor {
    boolean processPayment(double amount, String currency);
    String getProcessorName();

    default String formatAmount(double amount, String currency) {
        return String.format("%s %.2f", currency, amount);
    }
}

public interface RefundProcessor {
    boolean processRefund(String transactionId, double amount);
}

public interface FraudDetector {
    boolean isSuspicious(double amount, String userId);
}

// Stripe: can process payments and detect fraud
public class StripeProcessor implements PaymentProcessor, FraudDetector {
    private final String apiKey;

    public StripeProcessor(String apiKey) {
        this.apiKey = apiKey;
    }

    @Override
    public boolean processPayment(double amount, String currency) {
        System.out.println("Stripe processing: " + formatAmount(amount, currency));
        // Call Stripe API...
        return true;
    }

    @Override
    public String getProcessorName() { return "Stripe"; }

    @Override
    public boolean isSuspicious(double amount, String userId) {
        return amount > 10000;  // Flag transactions over ₹10,000
    }
}

// PayPal: can process payments AND refunds
public class PayPalProcessor implements PaymentProcessor, RefundProcessor {
    @Override
    public boolean processPayment(double amount, String currency) {
        System.out.println("PayPal processing: " + formatAmount(amount, currency));
        return true;
    }

    @Override
    public String getProcessorName() { return "PayPal"; }

    @Override
    public boolean processRefund(String transactionId, double amount) {
        System.out.println("PayPal refunding " + amount + " for tx: " + transactionId);
        return true;
    }
}

// Usage — program to the interface, not the implementation
public class CheckoutService {
    private final PaymentProcessor processor;

    public CheckoutService(PaymentProcessor processor) {
        this.processor = processor;  // Can be Stripe, PayPal, or anything
    }

    public void checkout(double amount) {
        if (processor instanceof FraudDetector detector) {
            if (detector.isSuspicious(amount, "user123")) {
                System.out.println("Suspicious transaction flagged!");
                return;
            }
        }
        processor.processPayment(amount, "INR");
    }
}

CheckoutService stripeCheckout = new CheckoutService(new StripeProcessor("sk_test_xxx"));
stripeCheckout.checkout(500.0);    // Processes fine
stripeCheckout.checkout(15000.0);  // Flagged as suspicious
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between an interface and an abstract class?**
> Interface: pure contract, no state (only constants), multiple implementation, all methods public. Abstract class: partial implementation, can have fields/constructors/any access modifier, only single inheritance. Use interface when defining a role/capability (Printable, Runnable). Use abstract class when sharing common state + behavior in a type hierarchy.

**Q2. Can an interface have a constructor?**
> No. Interfaces cannot have constructors because they cannot be instantiated directly. You instantiate a class that implements the interface.

**Q3. What are `default` methods in interfaces (Java 8+) and why were they added?**
> Default methods are interface methods with a body (`default void method() {}`). They were added to allow adding new methods to existing interfaces WITHOUT breaking all implementing classes. Before Java 8, adding a method to an interface would break every class implementing it. Default methods provide backward compatibility.

**Q4. A class implements two interfaces with the same default method. What happens?**
> Compile error — "inherits unrelated defaults from..." The class MUST override the conflicting method and explicitly choose which implementation to use: `InterfaceA.super.method()` or `InterfaceB.super.method()`, or provide a completely new implementation.

**Q5. What is a marker interface?**
> An interface with NO methods — used purely to "mark" a class for some purpose. Examples: `Serializable`, `Cloneable`, `RandomAccess`. The JVM or framework checks `instanceof MarkerInterface` to apply special behaviour. In modern Java, annotations are often preferred over marker interfaces.

**Q6. When should you use an abstract class over an interface?**
> Use abstract class when: 1) You need to share common fields/state (interfaces can't have instance fields). 2) You need constructors to initialize state. 3) Some methods have a default implementation that most subclasses will use (vs. interface default methods which are more about backward compatibility). 4) The relationship is a strong IS-A hierarchy (not just capability). Example: `abstract class HttpServlet` — servlets share common lifecycle state and behaviour.

---

## ✅ Best Practices

1. **Program to interfaces, not implementations** — `List<String> list = new ArrayList<>()` not `ArrayList<String> list = new ArrayList<>()`
2. **Keep interfaces small and focused** — one responsibility per interface (Interface Segregation Principle)
3. **Use `@FunctionalInterface`** for single-method interfaces — enables lambda usage
4. **Prefer interfaces over abstract classes** for defining contracts — allows multiple inheritance
5. **Don't put state in interfaces** — constants only; state belongs in abstract classes
6. **Name interfaces as adjectives** (Printable, Runnable, Closeable) OR nouns for roles (Repository, Service)

---

## 🛠️ Hands-on Practice

1. Create an interface `Shape` with `area()` and `perimeter()`. Implement `Circle`, `Rectangle`, `Triangle`. Process a `List<Shape>` to find the shape with the largest area.
2. Create a `Flyable` interface with `fly()`. Create `Bird`, `Airplane`, `Superhero` all implementing `Flyable`. Demonstrate multiple dispatch.
3. Create an abstract class `DatabaseConnection` with abstract `connect()`, `disconnect()`, and concrete `executeQuery(String sql)`. Implement `MySQLConnection` and `PostgreSQLConnection`.
4. Create a `@FunctionalInterface` named `Transformer<T>` with `T transform(T input)`. Use it as a lambda to: convert strings to uppercase, double integers, reverse a string.
5. Simulate the default method conflict: two interfaces with same `default void log()`. Implement a class that resolves the conflict by calling both using `Interface.super.log()`.
