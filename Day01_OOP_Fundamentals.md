# Day 01 — OOP Fundamentals
**Level:** Beginner → Intermediate | **Java Version:** 8+

---

## 🌟 What is Object-Oriented Programming?

**OOP** is a programming paradigm that organises code around **objects** (data + behaviour) rather than functions and logic. Java is a fully object-oriented language — everything lives inside a class.

### The 4 Pillars of OOP

| Pillar | In Simple Words | Java Mechanism |
|--------|----------------|----------------|
| **Encapsulation** | Hide internal data, expose only what's needed | `private` fields + public getters/setters |
| **Inheritance** | Child class reuses parent class code | `extends` keyword |
| **Polymorphism** | One interface, many implementations | Method overriding / overloading |
| **Abstraction** | Hide complexity, show only essentials | `abstract` classes, `interface` |

---

## 📦 Classes & Objects

A **class** is a blueprint. An **object** is an instance of that blueprint.

```java
// Blueprint
public class Car {
    // Fields (state / attributes)
    private String brand;
    private String color;
    private int speed;

    // Constructor — called when creating an object
    public Car(String brand, String color) {
        this.brand = brand;   // 'this' refers to current object
        this.color = color;
        this.speed = 0;       // Default value
    }

    // Methods (behaviour)
    public void accelerate(int amount) {
        this.speed += amount;
        System.out.println(brand + " is now going at " + speed + " km/h");
    }

    public void brake() {
        this.speed = 0;
        System.out.println(brand + " stopped.");
    }

    // Getter
    public String getBrand() { return brand; }
    public int getSpeed()    { return speed; }

    // toString — always override for readable output
    @Override
    public String toString() {
        return "Car{brand='" + brand + "', color='" + color + "', speed=" + speed + "}";
    }
}

// Using the class
public class Main {
    public static void main(String[] args) {
        Car myCar = new Car("Toyota", "Red");  // Create object
        myCar.accelerate(60);                  // Call method
        myCar.accelerate(20);
        System.out.println(myCar.getSpeed());  // 80
        System.out.println(myCar);             // Car{brand='Toyota', color='Red', speed=80}
    }
}
```

---

## 🔐 Encapsulation — Hiding the "How"

```java
public class BankAccount {
    private String owner;
    private double balance;    // PRIVATE — cannot be accessed directly from outside

    public BankAccount(String owner, double initialBalance) {
        this.owner = owner;
        // Validate in constructor
        if (initialBalance < 0) {
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        this.balance = initialBalance;
    }

    // Controlled access via methods — NEVER expose setBalance() directly
    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Deposit must be positive");
        balance += amount;
        System.out.printf("Deposited %.2f. New balance: %.2f%n", amount, balance);
    }

    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Withdrawal must be positive");
        if (amount > balance) throw new IllegalStateException("Insufficient funds");
        balance -= amount;
        System.out.printf("Withdrawn %.2f. Remaining: %.2f%n", amount, balance);
    }

    // Read-only access
    public double getBalance() { return balance; }
    public String getOwner()   { return owner; }

    // No setBalance() — balance can only change via deposit() or withdraw()
}

// Usage
BankAccount account = new BankAccount("Aman", 1000.0);
account.deposit(500.0);
account.withdraw(200.0);
System.out.println(account.getBalance());  // 1300.0
// account.balance = 999999;  ← COMPILE ERROR — private field
```

---

## 🏗️ Constructors — Deep Dive

```java
public class Person {
    private String name;
    private int age;
    private String email;

    // 1. No-arg constructor
    public Person() {
        this("Unknown", 0);  // Delegates to another constructor using this()
    }

    // 2. Partial constructor
    public Person(String name, int age) {
        this(name, age, "not-provided@email.com");
    }

    // 3. Full constructor — primary constructor
    public Person(String name, int age, String email) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name cannot be blank");
        }
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Invalid age: " + age);
        }
        this.name  = name;
        this.age   = age;
        this.email = email;
    }

    // Copy constructor — creates a new Person with same data
    public Person(Person other) {
        this(other.name, other.age, other.email);
    }

    @Override
    public String toString() {
        return String.format("Person{name='%s', age=%d, email='%s'}", name, age, email);
    }

    // Getters
    public String getName()  { return name; }
    public int getAge()      { return age; }
    public String getEmail() { return email; }
}
```

---

## 🔑 Access Modifiers

| Modifier | Same Class | Same Package | Subclass | Everywhere |
|----------|-----------|-------------|----------|------------|
| `private` | ✅ | ❌ | ❌ | ❌ |
| (default/package) | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

```java
public class AccessDemo {
    private   int privateField   = 1;  // Only inside this class
    int       packageField       = 2;  // Only in same package
    protected int protectedField = 3;  // Subclasses + same package
    public    int publicField    = 4;  // Everywhere
}
```

---

## ⚡ `static` vs Instance

```java
public class Counter {
    // STATIC — shared by ALL instances (class-level)
    private static int totalCount = 0;

    // INSTANCE — each object has its own copy
    private int id;
    private String name;

    public Counter(String name) {
        totalCount++;          // Shared counter increments
        this.id   = totalCount;
        this.name = name;
    }

    // Static method — no 'this', no instance fields
    public static int getTotalCount() {
        return totalCount;
    }

    // Instance method — has access to 'this'
    public String getInfo() {
        return "Counter #" + id + ": " + name;
    }

    // Static block — runs ONCE when class is loaded
    static {
        System.out.println("Counter class loaded!");
    }
}

// Usage
Counter c1 = new Counter("Alpha");   // totalCount = 1
Counter c2 = new Counter("Beta");    // totalCount = 2
Counter c3 = new Counter("Gamma");   // totalCount = 3

System.out.println(Counter.getTotalCount());  // 3 — call on class, not object
System.out.println(c1.getInfo());  // Counter #1: Alpha
System.out.println(c2.getInfo());  // Counter #2: Beta
```

---

## 🏛️ `final` Keyword

```java
public class ImmutablePoint {
    // final field — must be set in constructor, never changed
    private final double x;
    private final double y;

    public ImmutablePoint(double x, double y) {
        this.x = x;
        this.y = y;
    }

    // final method — cannot be overridden by subclasses
    public final double distanceTo(ImmutablePoint other) {
        return Math.sqrt(Math.pow(this.x - other.x, 2) + Math.pow(this.y - other.y, 2));
    }

    public double getX() { return x; }
    public double getY() { return y; }
}

// final class — cannot be extended (e.g., String, Integer)
public final class Constant {
    public static final double PI = 3.14159265;
    public static final int MAX_RETRIES = 3;
}
```

---

## 🔄 `equals()` and `hashCode()` — The Contract

> **Rule:** If two objects are equal (`equals()` returns true), they MUST have the same `hashCode()`. Always override both together.

```java
public class Employee {
    private String employeeId;
    private String name;
    private String department;

    public Employee(String employeeId, String name, String department) {
        this.employeeId = employeeId;
        this.name       = name;
        this.department = department;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;          // Same reference
        if (obj == null) return false;          // Null check
        if (!(obj instanceof Employee)) return false;  // Type check

        Employee other = (Employee) obj;
        return Objects.equals(this.employeeId, other.employeeId);
        // Equality based on employeeId only (business key)
    }

    @Override
    public int hashCode() {
        return Objects.hash(employeeId);  // Same fields as equals()
    }

    @Override
    public String toString() {
        return "Employee{id='" + employeeId + "', name='" + name + "'}";
    }
}

// Why this matters:
Employee e1 = new Employee("E001", "Aman",  "Engineering");
Employee e2 = new Employee("E001", "Aman S", "Engineering");  // Different name

System.out.println(e1.equals(e2));    // true  — same employeeId
System.out.println(e1 == e2);        // false — different references

Set<Employee> set = new HashSet<>();
set.add(e1);
set.add(e2);
System.out.println(set.size());      // 1 — HashSet uses equals+hashCode to detect duplicates
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between a class and an object?**
> A **class** is a template/blueprint that defines properties and behaviours. An **object** is a specific instance of that class with actual values. Example: `Car` is a class; `new Car("Toyota", "Red")` is an object. Multiple objects can be created from one class.

**Q2. What is encapsulation and why is it important?**
> Encapsulation means hiding internal state (fields) and exposing only controlled operations (methods). It's important because: 1) It protects data integrity (e.g., `BankAccount.balance` can only change via `deposit()`/`withdraw()`). 2) It allows internal implementation to change without affecting callers. 3) It reduces coupling between classes.

**Q3. What is the `this` keyword?**
> `this` refers to the current object instance. Used to: 1) Distinguish between field names and parameter names when they're the same (`this.name = name`). 2) Call another constructor from a constructor (`this(name, 0)`). 3) Pass the current object as an argument to another method.

**Q4. Why must `equals()` and `hashCode()` be overridden together?**
> The Java contract states: objects that are equal must have the same hash code. HashMap and HashSet use `hashCode()` first to find the bucket, then `equals()` to confirm identity. If you override `equals()` but not `hashCode()`, two "equal" objects may end up in different buckets — making HashSet contain "duplicates" and HashMap lookups return null.

**Q5. What is the difference between `static` and instance members?**
> **Static members** belong to the class itself — shared across all objects. Accessed via `ClassName.method()`. Initialized once when the class loads. **Instance members** belong to each individual object — each object has its own copy. Accessed via `objectReference.method()`. Initialized when the object is created with `new`.

**Q6. What does `final` mean on a field, method, and class?**
> **`final` field:** Value set once in constructor — never changed (immutable). **`final` method:** Cannot be overridden by any subclass. **`final` class:** Cannot be subclassed at all (e.g., `String`, `Integer` are `final`).

---

## ✅ Best Practices

1. **Always declare fields `private`** — use getters/setters for controlled access
2. **Override `toString()`** on every entity class — invaluable for debugging
3. **Override `equals()` and `hashCode()` together** — always, never just one
4. **Validate in constructors** — ensure objects are always in a valid state
5. **Use `this()` for constructor chaining** — avoid duplicating validation logic
6. **Prefer `final` fields for immutable state** — clearly signals intent, thread-safe
7. **Don't expose mutable internal state** — return defensive copies if needed

---

## 🛠️ Hands-on Practice

1. Create a `Student` class with `name`, `studentId`, `grade` (A/B/C/D/F). Add validation in the constructor. Override `toString()`, `equals()` (by studentId), `hashCode()`.
2. Create a `Circle` class with `final double radius`. Add methods `area()` and `perimeter()`. Make the class immutable.
3. Create a `Counter` class with a `static` field `totalCreated` and verify it increments every time you create a new instance.
4. Create a `BankAccount` class with `deposit()`, `withdraw()`, and `transfer(BankAccount target, double amount)` methods. Ensure balance can never go negative.
5. Create a `Person` class with all 4 constructors (no-arg, name-only, name+age, full). Use constructor chaining with `this()`.
