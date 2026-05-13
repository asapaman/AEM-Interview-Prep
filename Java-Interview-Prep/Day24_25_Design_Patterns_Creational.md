# Day 24–25 — Design Patterns I (Creational)
**Level:** Intermediate → Advanced | **Java Version:** 8+

---

## 🌟 What are Design Patterns?

Design Patterns are typical, proven solutions to common problems in software design. They are not code you can copy-paste, but rather a blueprint or concept that you can customize to solve a recurring design issue in your code.

**Creational Patterns** deal with object creation mechanisms, trying to create objects in a manner suitable to the situation, rather than just using `new Object()`.

---

## 🏗️ 1. Singleton Pattern

**Goal:** Ensure a class has only ONE instance and provide a global point of access to it.
**Common Uses:** Database connections, Logging services, Configuration managers.

### The Modern, Thread-Safe Approach (Bill Pugh Singleton)
Uses a static inner helper class. It is lazy-loaded and inherently thread-safe without needing `synchronized` blocks.

```java
public class DatabaseConnection {

    // 1. Private constructor prevents instantiation from other classes
    private DatabaseConnection() {
        System.out.println("Initializing DB Connection...");
    }

    // 2. Static inner class - loaded ONLY when getInstance() is called
    private static class ConnectionHolder {
        private static final DatabaseConnection INSTANCE = new DatabaseConnection();
    }

    // 3. Public static method to get the instance
    public static DatabaseConnection getInstance() {
        return ConnectionHolder.INSTANCE;
    }

    public void query(String sql) {
        System.out.println("Executing: " + sql);
    }
}

// Usage
DatabaseConnection db1 = DatabaseConnection.getInstance();
DatabaseConnection db2 = DatabaseConnection.getInstance();
System.out.println(db1 == db2); // true! Both references point to the same object.
```

*Note: Enums are also a fantastic, simple way to create Singletons in Java (`public enum Singleton { INSTANCE; }`) as they handle serialization and reflection attacks natively.*

---

## 🏭 2. Factory Method Pattern

**Goal:** Provide an interface for creating objects in a superclass, but allow subclasses to alter the type of objects that will be created. Let the subclasses decide which class to instantiate.
**Common Uses:** When you have a complex creation process, or when the exact type of object isn't known until runtime.

```java
// 1. Common Interface
public interface Notification {
    void notifyUser();
}

// 2. Concrete Implementations
public class SMSNotification implements Notification {
    @Override public void notifyUser() { System.out.println("Sending SMS..."); }
}

public class EmailNotification implements Notification {
    @Override public void notifyUser() { System.out.println("Sending Email..."); }
}

// 3. The Factory Class
public class NotificationFactory {
    
    // Encapsulates the creation logic
    public Notification createNotification(String channel) {
        if (channel == null || channel.isEmpty()) {
            return null;
        }
        switch (channel.toUpperCase()) {
            case "SMS":
                return new SMSNotification();
            case "EMAIL":
                return new EmailNotification();
            default:
                throw new IllegalArgumentException("Unknown channel " + channel);
        }
    }
}

// Usage - Client code doesn't need to know about SMSNotification or EmailNotification classes directly!
NotificationFactory factory = new NotificationFactory();
Notification notif = factory.createNotification("EMAIL");
notif.notifyUser(); // "Sending Email..."
```

---

## 🔨 3. Builder Pattern

**Goal:** Separate the construction of a complex object from its representation, allowing you to create different representations using the same construction process. Prevents the "Telescoping Constructor" anti-pattern (constructors with 10+ arguments).
**Common Uses:** Creating complex objects like User profiles, HTTP Requests, Immutable configuration objects.

```java
public class User {
    private final String firstName; // Required
    private final String lastName;  // Required
    private final int age;          // Optional
    private final String phone;     // Optional
    private final String address;   // Optional

    // Private constructor takes the Builder
    private User(UserBuilder builder) {
        this.firstName = builder.firstName;
        this.lastName  = builder.lastName;
        this.age       = builder.age;
        this.phone     = builder.phone;
        this.address   = builder.address;
    }

    // Static nested Builder class
    public static class UserBuilder {
        private final String firstName;
        private final String lastName;
        private int age = 0;              // Default
        private String phone = "N/A";     // Default
        private String address = "N/A";   // Default

        // Required parameters in Builder constructor
        public UserBuilder(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }

        // Setter methods return the Builder itself for method chaining
        public UserBuilder age(int age) {
            this.age = age;
            return this;
        }

        public UserBuilder phone(String phone) {
            this.phone = phone;
            return this;
        }

        public UserBuilder address(String address) {
            this.address = address;
            return this;
        }

        // Final build method constructs the actual object
        public User build() {
            return new User(this);
        }
    }

    @Override
    public String toString() {
        return firstName + " " + lastName + " (Age: " + age + ")";
    }
}

// Usage - Fluent, readable, and handles optional parameters perfectly!
User user = new User.UserBuilder("Aman", "Singh")
        .age(28)
        .address("Bangalore")
        // phone is skipped, defaults to "N/A"
        .build();

System.out.println(user);
```
*(Tip: In modern Java projects, the `@Builder` annotation from the Lombok library automatically generates this entire Builder boilerplate for you!)*

---

## ❓ Interview Questions & Answers

**Q1. What is the "Double-Checked Locking" issue with Singleton, and how do we fix it?**
> In older Java versions, a lazy-initialized singleton used a pattern like `if (instance == null) { synchronized(class) { if (instance == null) { instance = new Singleton(); } } }`. However, due to instruction reordering in the JVM, another thread could see a partially constructed object. The fix is to mark the instance variable as `volatile` (Java 5+), which prevents reordering, or use the Bill Pugh static inner class method, which relies on the ClassLoader's inherent thread-safety.

**Q2. Why is the Factory pattern better than just using `new` everywhere?**
> The Factory pattern promotes loose coupling. If you scatter `new EmailNotification()` throughout your codebase, and later need to change how `EmailNotification` is constructed (e.g., it now needs a config object passed to its constructor), you have to change the code in 50 different places. With a Factory, you only change the construction logic in ONE place.

**Q3. What problem does the Builder pattern solve?**
> It solves the "Telescoping Constructor" problem, where a class has multiple constructors with varying numbers of parameters (e.g., `new User(name)`, `new User(name, age)`, `new User(name, age, phone)`). This becomes unreadable and error-prone, especially if there are multiple parameters of the same type (e.g., `new User(name, address, company)` — easy to mix up the order). Builder makes object creation readable, explicit, and handles optional parameters elegantly while keeping the final object immutable.

---

## ✅ Best Practices

1. **Singleton:** Avoid overusing Singletons. They represent global state, which makes unit testing difficult (they can't be easily mocked) and can hide dependencies. Pass dependencies explicitly via constructors where possible.
2. **Factory:** Return interfaces, not concrete classes, from your Factory methods to enforce loose coupling.
3. **Builder:** Use Builder when a class has > 4 parameters, especially if many are optional. It makes the calling code self-documenting.

---

## 🛠️ Hands-on Practice

1. Write a `ConfigurationManager` class using the Bill Pugh Singleton pattern. Add a `Map<String, String>` inside it to store config key-values.
2. Implement an `AnimalFactory`. It takes a string ("DOG", "CAT", "COW") and returns the corresponding object implementing an `Animal` interface.
3. Write a `Pizza` class with a Builder. Required parameters: `size`, `crustType`. Optional parameters: `cheese` (boolean), `pepperoni` (boolean), `mushrooms` (boolean). Create a large thin-crust pizza with cheese and mushrooms.
