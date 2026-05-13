# Day 02 — Inheritance & Polymorphism
**Level:** Beginner → Intermediate | **Java Version:** 8+

---

## 🌟 What is Inheritance?

**Inheritance** lets a child class reuse fields and methods from a parent class, extending or specialising its behaviour. Use the `extends` keyword.

```
       Animal (parent/superclass)
       /         \
     Dog          Cat
   (child)      (child)
   /    \
Labrador Poodle
(grandchild)
```

### When to Use Inheritance
- When there is a clear **"is-a"** relationship: `Dog IS-A Animal`, `Labrador IS-A Dog`
- When the child class is a **specialised version** of the parent
- Avoid it for **"has-a"** relationships (use composition instead)

---

## 💻 Inheritance — Complete Example

```java
// Parent class (superclass)
public class Animal {
    private String name;
    private int age;

    public Animal(String name, int age) {
        this.name = name;
        this.age  = age;
    }

    // Method available to ALL animals
    public void eat() {
        System.out.println(name + " is eating.");
    }

    public void sleep() {
        System.out.println(name + " is sleeping.");
    }

    // Can be overridden by subclasses
    public String makeSound() {
        return "...";  // Generic sound
    }

    public String getName() { return name; }
    public int getAge()     { return age;  }

    @Override
    public String toString() {
        return getClass().getSimpleName() + "{name='" + name + "', age=" + age + "}";
    }
}

// Child class — inherits from Animal
public class Dog extends Animal {
    private String breed;

    public Dog(String name, int age, String breed) {
        super(name, age);   // MUST call parent constructor first using super()
        this.breed = breed;
    }

    // Override parent method — provide Dog-specific implementation
    @Override
    public String makeSound() {
        return "Woof!";
    }

    // Add Dog-specific behaviour
    public void fetch() {
        System.out.println(getName() + " fetches the ball!");
    }

    public String getBreed() { return breed; }
}

// Another child class
public class Cat extends Animal {
    private boolean isIndoor;

    public Cat(String name, int age, boolean isIndoor) {
        super(name, age);
        this.isIndoor = isIndoor;
    }

    @Override
    public String makeSound() {
        return "Meow!";
    }

    public boolean isIndoor() { return isIndoor; }
}

// Usage
Dog dog = new Dog("Buddy", 3, "Labrador");
Cat cat = new Cat("Whiskers", 5, true);

dog.eat();                       // Inherited from Animal
dog.fetch();                     // Dog-specific
System.out.println(dog.makeSound());  // "Woof!" — overridden

cat.eat();                       // Inherited from Animal
System.out.println(cat.makeSound()); // "Meow!" — overridden

System.out.println(dog);  // Dog{name='Buddy', age=3}
```

---

## 🔗 `super` Keyword

```java
public class Vehicle {
    private String brand;
    private int year;

    public Vehicle(String brand, int year) {
        this.brand = brand;
        this.year  = year;
    }

    public String describe() {
        return brand + " (" + year + ")";
    }
}

public class ElectricCar extends Vehicle {
    private int batteryRange;  // km

    public ElectricCar(String brand, int year, int batteryRange) {
        super(brand, year);           // 1. Call parent constructor — MUST be first line
        this.batteryRange = batteryRange;
    }

    @Override
    public String describe() {
        String parentDesc = super.describe();  // 2. Call parent method
        return parentDesc + " | Electric | Range: " + batteryRange + "km";
    }
}

ElectricCar tesla = new ElectricCar("Tesla", 2024, 500);
System.out.println(tesla.describe());
// Output: Tesla (2024) | Electric | Range: 500km
```

---

## 🎭 Polymorphism — One Interface, Many Forms

**Polymorphism** means the same method call behaves differently depending on the actual object type at runtime.

### Runtime Polymorphism (Method Overriding)

```java
public class Shape {
    public double area() {
        return 0.0;  // Default — subclasses override this
    }
    public String describe() {
        return getClass().getSimpleName() + " with area " + area();
    }
}

public class Circle extends Shape {
    private double radius;
    public Circle(double radius) { this.radius = radius; }

    @Override
    public double area() {
        return Math.PI * radius * radius;  // πr²
    }
}

public class Rectangle extends Shape {
    private double width, height;
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public double area() {
        return width * height;  // w × h
    }
}

public class Triangle extends Shape {
    private double base, height;
    public Triangle(double base, double height) {
        this.base = base;
        this.height = height;
    }

    @Override
    public double area() {
        return 0.5 * base * height;  // ½bh
    }
}

// Polymorphism in action — same variable type, different behaviour
Shape[] shapes = {
    new Circle(5),
    new Rectangle(4, 6),
    new Triangle(3, 8)
};

for (Shape shape : shapes) {
    // At runtime, JVM calls the correct area() based on actual object type
    System.out.println(shape.describe());
}
// Circle with area 78.53981633974483
// Rectangle with area 24.0
// Triangle with area 12.0

// Calculate total area of all shapes
double totalArea = 0;
for (Shape shape : shapes) {
    totalArea += shape.area();  // Each call dispatched to correct subclass
}
```

### Compile-Time Polymorphism (Method Overloading)

```java
public class MathHelper {
    // Same method name, different parameter signatures

    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }

    public String add(String a, String b) {
        return a + b;  // String concatenation
    }
}

MathHelper math = new MathHelper();
System.out.println(math.add(2, 3));            // 5
System.out.println(math.add(2.5, 3.5));        // 6.0
System.out.println(math.add(1, 2, 3));         // 6
System.out.println(math.add("Hello", " World")); // Hello World
// Method resolution happens at COMPILE TIME based on argument types
```

---

## 🔍 `instanceof` and Type Casting

```java
Animal animal = new Dog("Rex", 2, "German Shepherd");

// instanceof check before casting
if (animal instanceof Dog) {
    Dog dog = (Dog) animal;  // Downcast — safe because we checked
    dog.fetch();
    System.out.println("Breed: " + dog.getBreed());
}

// Modern Java (16+) — pattern matching instanceof
if (animal instanceof Dog dog) {        // Declare and cast in one line
    dog.fetch();                        // 'dog' is already Dog type
    System.out.println(dog.getBreed()); // No explicit cast needed
}

// Check inheritance hierarchy
System.out.println(animal instanceof Animal); // true
System.out.println(animal instanceof Dog);    // true
System.out.println(animal instanceof Cat);    // false
```

---

## ⬆️ Upcasting vs Downcasting

```java
Dog dog = new Dog("Rex", 2, "Husky");

// UPCASTING — always safe (implicit)
Animal animal = dog;         // Dog IS-A Animal — OK
animal.eat();                // Works — eat() is in Animal
// animal.fetch();           // COMPILE ERROR — Animal doesn't have fetch()

// DOWNCASTING — risky (needs explicit cast)
Dog sameDog = (Dog) animal;  // OK — actual object IS a Dog
sameDog.fetch();             // Works!

Animal cat = new Cat("Tom", 1, true);
// Dog wrongCast = (Dog) cat;  // COMPILE OK but RUNTIME ClassCastException!
// ALWAYS use instanceof before downcasting
```

---

## 🚫 Overriding Rules

```java
public class Parent {
    public void    method1() {}   // Can be overridden
    protected void method2() {}   // Can be overridden (can widen to public)
    void           method3() {}   // Package-private, can be overridden in same package
    private void   method4() {}   // CANNOT be overridden (not visible to subclass)
    public static  void method5() {} // CANNOT be overridden — belongs to class, not object
    public final   void method6() {} // CANNOT be overridden — final!
}

public class Child extends Parent {
    @Override  // @Override annotation — compiler checks if you're actually overriding
    public void method1() {
        // ✅ Overriding — same or wider access (public → public OK)
    }

    @Override
    public void method2() {
        // ✅ Widening from protected → public is allowed
    }

    // Overriding rules:
    // 1. Same method name and parameter types
    // 2. Return type must be same OR a subtype (covariant return)
    // 3. Access modifier can be same or WIDER (never narrower)
    // 4. Cannot throw NEW or BROADER checked exceptions
}
```

---

## 🔗 Inheritance vs Composition

```java
// ❌ Inheritance for "has-a" — WRONG approach
public class Car extends Engine {  // A Car IS-NOT an Engine!
    // ...
}

// ✅ Composition for "has-a" — CORRECT
public class Engine {
    private int horsepower;
    private String fuelType;

    public Engine(int horsepower, String fuelType) {
        this.horsepower = horsepower;
        this.fuelType   = fuelType;
    }

    public void start() { System.out.println("Engine started (" + fuelType + ")"); }
    public int getHorsepower() { return horsepower; }
}

public class Car {
    private String brand;
    private Engine engine;   // Car HAS-AN Engine (composition)
    private String color;

    public Car(String brand, Engine engine, String color) {
        this.brand  = brand;
        this.engine = engine;
        this.color  = color;
    }

    public void start() {
        engine.start();  // Delegate to Engine
        System.out.println(brand + " is ready to drive!");
    }

    public int getPower() { return engine.getHorsepower(); }
}

// Usage
Engine v8 = new Engine(450, "Petrol");
Car sportsCar = new Car("Mustang", v8, "Red");
sportsCar.start();
// Engine started (Petrol)
// Mustang is ready to drive!
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between method overriding and method overloading?**
> **Overriding** (runtime polymorphism): A subclass provides its own implementation of a method defined in the parent class. Same name, same parameters, different class. Resolved at runtime. Requires `@Override`. **Overloading** (compile-time polymorphism): Same class has multiple methods with the same name but different parameter lists (different number, types, or order of parameters). Resolved at compile time.

**Q2. Can a constructor be inherited?**
> No. Constructors are NOT inherited. However, a child class's constructor MUST call a parent constructor (explicitly with `super(...)` or implicitly — Java calls `super()` for the no-arg parent constructor). If the parent has no no-arg constructor, the child MUST explicitly call `super(args)`.

**Q3. What is the difference between `super` and `this`?**
> `this` refers to the current object instance — used to access current class's members or call another constructor in the same class. `super` refers to the parent class — used to call parent's constructor (`super(args)`) or parent's overridden method (`super.methodName()`).

**Q4. When should you use inheritance vs composition?**
> Use inheritance for **"is-a"** relationships (Dog IS-A Animal). Use composition for **"has-a"** relationships (Car HAS-AN Engine). Composition is generally preferred because: it's more flexible (you can change the composed object at runtime), avoids fragile base class problems, and doesn't break encapsulation. "Favour composition over inheritance" is a core design principle.

**Q5. What is a covariant return type?**
> A method in a subclass can override a parent method and return a MORE specific (subtype) return type. Example: if parent's `create()` returns `Animal`, the child's override can return `Dog` (which is a subtype of `Animal`). This is allowed because the child's return type IS-A parent's return type.

**Q6. What happens if you don't use `@Override`?**
> The code still compiles and works IF you actually override correctly. But `@Override` gives you compile-time safety — if you misspell the method name or wrong parameter types, the compiler throws an error ("method does not override or implement a method from a supertype"). Without `@Override`, you'd silently create a NEW method instead of overriding.

---

## ✅ Best Practices

1. **Always use `@Override`** when overriding methods — compile-time safety
2. **Call `super()` appropriately** — parent needs to initialize its own state
3. **Prefer composition over inheritance** for "has-a" relationships
4. **Don't override `static` or `private` methods** — they're not polymorphic
5. **Keep inheritance hierarchies shallow** — deep inheritance (>2–3 levels) becomes hard to understand
6. **Use `instanceof` before downcasting** — prevents `ClassCastException`
7. **Override `toString()`** in each class — helps with debugging

---

## 🛠️ Hands-on Practice

1. Create a class hierarchy: `Vehicle → Car → ElectricCar` and `Vehicle → Motorcycle`. Each has `fuelType()` overridden to return different strings. Create an array of `Vehicle` and call `fuelType()` on each — observe polymorphism.
2. Implement method overloading in a `Calculator` class: `calculate(int a, int b)`, `calculate(double a, double b)`, `calculate(int a, int b, int c)` — all named `calculate`.
3. Write a `Animal[] animals = {new Dog(...), new Cat(...), new Bird(...)}` loop that prints each animal's sound without if/else — purely via polymorphism.
4. Refactor a class that extends `ArrayList` to instead USE `ArrayList` (composition). Verify the behaviour is the same but the design is now flexible.
5. Use `instanceof` with pattern matching (Java 16+): write a method `describe(Object obj)` that checks if it's a `String`, `Integer`, or `List` and prints accordingly without casting.
