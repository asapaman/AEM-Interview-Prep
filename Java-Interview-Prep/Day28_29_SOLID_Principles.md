# Day 28–29 — SOLID Principles
**Level:** Advanced | **Java Version:** 8+

---

## 🌟 What is SOLID?

SOLID is an acronym for five design principles intended to make software designs more understandable, flexible, and maintainable. Coined by Robert C. Martin (Uncle Bob), they form the core philosophy of modern Object-Oriented Design.

---

## 1️⃣ Single Responsibility Principle (SRP)
> **"A class should have one, and only one, reason to change."**

Every class, module, or method should have responsibility over a single part of the functionality.

**❌ Bad Example:**
```java
public class Employee {
    public void calculatePay() { /* business logic */ }
    public void saveToDatabase() { /* database logic */ }
    public void printReport() { /* UI/formatting logic */ }
}
```
*Why is this bad?* If the database schema changes, you modify `Employee`. If the UI report format changes, you modify `Employee`. It has 3 reasons to change.

**✅ Good Example:**
```java
public class Employee { /* Only holds state/data */ }
public class PayrollCalculator { public void calculatePay(Employee e) { ... } }
public class EmployeeRepository { public void save(Employee e) { ... } }
public class EmployeeReportFormatter { public void print(Employee e) { ... } }
```

---

## 2️⃣ Open/Closed Principle (OCP)
> **"Software entities should be open for extension, but closed for modification."**

You should be able to add new functionality without changing existing, tested code. (This is exactly what the Strategy pattern achieves!).

**❌ Bad Example:**
```java
public class DiscountCalculator {
    public double calculate(String customerType, double amount) {
        if (customerType.equals("REGULAR")) {
            return amount * 0.1;
        } else if (customerType.equals("VIP")) {
            return amount * 0.2;
        }
        return 0;
    }
}
```
*Why is this bad?* If you add a "SUPER_VIP" tier, you have to open this class and modify the `if/else` block, risking breaking existing logic.

**✅ Good Example:**
```java
public interface DiscountStrategy {
    double applyDiscount(double amount);
}

public class RegularDiscount implements DiscountStrategy {
    public double applyDiscount(double amount) { return amount * 0.1; }
}

public class VipDiscount implements DiscountStrategy {
    public double applyDiscount(double amount) { return amount * 0.2; }
}

public class DiscountCalculator {
    public double calculate(DiscountStrategy strategy, double amount) {
        return strategy.applyDiscount(amount); // Never needs modification!
    }
}
```

---

## 3️⃣ Liskov Substitution Principle (LSP)
> **"Objects of a superclass should be replaceable with objects of its subclasses without breaking the application."**

If Class B extends Class A, you should be able to pass B to any method expecting A, and it should behave correctly. A subclass should **override** methods, but not **break** the parent's contract.

**❌ Bad Example (The Classic Rectangle/Square Problem):**
```java
public class Rectangle {
    protected int width, height;
    public void setWidth(int w) { width = w; }
    public void setHeight(int h) { height = h; }
    public int getArea() { return width * height; }
}

public class Square extends Rectangle {
    // A square must have equal sides, so we override to force this.
    @Override public void setWidth(int w) { width = w; height = w; }
    @Override public void setHeight(int h) { width = h; height = h; }
}

// Client Code
public void resize(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    // If 'r' is a Rectangle, area = 50.
    // If 'r' is a Square, area = 100! The behavior unexpectedly changed!
    assert r.getArea() == 50; 
}
```
*Why is this bad?* A Square IS-A Rectangle in geometry, but in OOP behavior, setting width and height independently breaks the Square. They shouldn't use inheritance here.

---

## 4️⃣ Interface Segregation Principle (ISP)
> **"Clients should not be forced to depend upon interfaces that they do not use."**

Keep interfaces small and focused. Better to have many small, specific interfaces than one large, "fat" interface.

**❌ Bad Example:**
```java
public interface MultiFunctionPrinter {
    void print();
    void scan();
    void fax();
}

public class BasicPrinter implements MultiFunctionPrinter {
    @Override public void print() { /* prints */ }
    @Override public void scan() { throw new UnsupportedOperationException(); } // Forced to implement!
    @Override public void fax()  { throw new UnsupportedOperationException(); } // Forced to implement!
}
```

**✅ Good Example:**
```java
public interface Printer { void print(); }
public interface Scanner { void scan(); }
public interface Fax { void fax(); }

// Now BasicPrinter only implements what it needs
public class BasicPrinter implements Printer {
    @Override public void print() { /* prints */ }
}

public class AdvancedPrinter implements Printer, Scanner, Fax {
    // Implements all three seamlessly
}
```

---

## 5️⃣ Dependency Inversion Principle (DIP)
> **"High-level modules should not depend on low-level modules. Both should depend on abstractions (interfaces)."**
> **"Abstractions should not depend on details. Details should depend on abstractions."**

This is the core concept behind Dependency Injection (e.g., Spring Framework).

**❌ Bad Example:**
```java
public class LightBulb {
    public void turnOn() { }
    public void turnOff() { }
}

public class Switch {
    private LightBulb bulb; // HIGH-LEVEL module tightly coupled to a LOW-LEVEL concrete class!

    public Switch() {
        this.bulb = new LightBulb();
    }
    public void operate() { bulb.turnOn(); }
}
```
*Why is this bad?* What if we want the switch to turn on a Fan instead? We have to rewrite the Switch class.

**✅ Good Example:**
```java
// The Abstraction
public interface Switchable {
    void turnOn();
    void turnOff();
}

// Low-level modules depend on the abstraction
public class LightBulb implements Switchable { ... }
public class Fan implements Switchable { ... }

// High-level module depends on the abstraction
public class Switch {
    private Switchable device; // Decoupled!

    // Dependency Injection via constructor
    public Switch(Switchable device) {
        this.device = device;
    }
    public void operate() { device.turnOn(); }
}
```

---

## ❓ Interview Questions & Answers

**Q1. How do SOLID principles relate to Design Patterns?**
> SOLID principles are the foundational *rules* or guidelines for writing good object-oriented code. Design Patterns are the *tools* or specific templates used to implement those rules. For example, the Strategy Pattern is a direct implementation of the Open/Closed Principle. Dependency Injection directly implements the Dependency Inversion Principle.

**Q2. Explain Dependency Injection vs Dependency Inversion.**
> **Dependency Inversion** is the *principle* (High-level modules shouldn't depend on low-level modules, both should depend on abstractions). **Dependency Injection (DI)** is a *technique* used to achieve Dependency Inversion, usually by passing interfaces into a class's constructor rather than the class instantiating dependencies itself using `new`.

**Q3. How can you identify a violation of the Single Responsibility Principle?**
> Look for classes with names containing "And", "Manager", or "Processor" (e.g., `UserDataAndEmailManager`). Look for classes that import packages from entirely different layers (e.g., a class importing `java.sql.*` and `javax.swing.*`). Look for classes with thousands of lines of code or methods that do more than what their name implies.

---

## ✅ Best Practices

1. **Don't force SOLID prematurely.** Applying all 5 principles to a simple 10-line script is over-engineering. Apply them when the codebase begins to grow and change becomes painful.
2. **Favor Composition over Inheritance.** Inheritance tightly couples classes and often leads to violations of the Liskov Substitution Principle (like the Rectangle/Square problem). Composition allows more flexibility.
3. **Use interfaces for boundaries.** The boundaries between different layers of your application (Controller -> Service -> Repository) should always be defined by interfaces to adhere to DIP.

---

## 🛠️ Hands-on Practice

1. Refactor a "God Object" class (e.g., `OrderManager` that validates inventory, calculates tax, charges a credit card, saves to DB, and sends an email) into 5 separate classes following SRP.
2. Review your last project. Find an `if/else if` or `switch` statement that checks a type to determine behavior. Refactor it using an Interface and Polymorphism to follow the Open/Closed Principle.
3. Create an interface `Bird` with `fly()` and `walk()`. Create an `Ostrich` class that implements `Bird`. Identify why this violates LSP and ISP, and refactor the interfaces to fix it.
