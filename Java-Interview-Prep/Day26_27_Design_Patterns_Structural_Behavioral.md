# Day 26–27 — Design Patterns II (Structural & Behavioral)
**Level:** Advanced | **Java Version:** 8+

---

## 🌟 Structural Patterns
These patterns deal with object composition or the way classes and objects are structured to form larger, more complex structures.

### 🔌 1. Adapter Pattern

**Goal:** Allow incompatible interfaces to work together. It acts as a bridge between two objects.
**Common Uses:** Integrating third-party libraries, modernizing legacy code interfaces.

```java
// 1. The interface our application expects
public interface MediaPlayer {
    void play(String audioType, String fileName);
}

// 2. An incompatible 3rd party class we want to use
public class VlcPlayer {
    public void playVlc(String fileName) {
        System.out.println("Playing vlc file: " + fileName);
    }
}

// 3. The Adapter - Implements our interface, but delegates to the 3rd party class
public class MediaAdapter implements MediaPlayer {
    private VlcPlayer advancedMusicPlayer;

    public MediaAdapter() {
        this.advancedMusicPlayer = new VlcPlayer();
    }

    @Override
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("vlc")) {
            advancedMusicPlayer.playVlc(fileName); // Delegate!
        } else {
            System.out.println("Invalid media type.");
        }
    }
}

// Usage
MediaPlayer player = new MediaAdapter();
player.play("vlc", "movie.vlc"); // "Playing vlc file: movie.vlc"
```

### ☕ 2. Decorator Pattern

**Goal:** Attach new behaviors to an object dynamically at runtime without subclassing (which applies changes statically to an entire class).
**Common Uses:** `java.io` package (`BufferedReader(new FileReader())`), adding features to UI components.

```java
// 1. Base Interface
public interface Coffee {
    String getDescription();
    double getCost();
}

// 2. Concrete Component
public class SimpleCoffee implements Coffee {
    @Override public String getDescription() { return "Simple Coffee"; }
    @Override public double getCost() { return 2.0; }
}

// 3. Base Decorator (implements the interface AND holds a reference to it)
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee decoratedCoffee;

    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }
    
    public String getDescription() { return decoratedCoffee.getDescription(); }
    public double getCost() { return decoratedCoffee.getCost(); }
}

// 4. Concrete Decorators
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }

    @Override
    public String getDescription() { return super.getDescription() + ", Milk"; }

    @Override
    public double getCost() { return super.getCost() + 0.5; }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) { super(coffee); }
    @Override public String getDescription() { return super.getDescription() + ", Sugar"; }
    @Override public double getCost() { return super.getCost() + 0.2; }
}

// Usage - dynamically wrap the object!
Coffee myCoffee = new SimpleCoffee();
System.out.println(myCoffee.getDescription() + " $" + myCoffee.getCost()); // Simple Coffee $2.0

// Add milk and sugar dynamically
myCoffee = new MilkDecorator(myCoffee);
myCoffee = new SugarDecorator(myCoffee);

System.out.println(myCoffee.getDescription() + " $" + myCoffee.getCost()); 
// Simple Coffee, Milk, Sugar $2.7
```

---

## 🎭 Behavioral Patterns
These patterns deal with algorithms and the assignment of responsibilities between objects.

### 👀 3. Observer Pattern

**Goal:** Define a one-to-many dependency so that when one object changes state, all its dependents are notified and updated automatically (Pub-Sub pattern).
**Common Uses:** UI event listeners (e.g., button clicks), State management (Redux/Vuex style), Message queues.

```java
import java.util.ArrayList;
import java.util.List;

// 1. Observer Interface
public interface Observer {
    void update(float temperature);
}

// 2. Subject (Publisher)
public class WeatherStation {
    private List<Observer> observers = new ArrayList<>();
    private float temperature;

    public void addObserver(Observer o) { observers.add(o); }
    public void removeObserver(Observer o) { observers.add(o); }

    public void setTemperature(float temp) {
        System.out.println("\nWeatherStation: new temperature is " + temp);
        this.temperature = temp;
        notifyObservers(); // Notify everyone!
    }

    private void notifyObservers() {
        for (Observer o : observers) {
            o.update(temperature);
        }
    }
}

// 3. Concrete Observers (Subscribers)
public class PhoneDisplay implements Observer {
    @Override
    public void update(float temp) {
        System.out.println("Phone Display: Temperature updated to " + temp);
    }
}

public class WindowDisplay implements Observer {
    @Override
    public void update(float temp) {
        System.out.println("Window Display: Temperature is now " + temp);
    }
}

// Usage
WeatherStation station = new WeatherStation();
Observer phone = new PhoneDisplay();
Observer window = new WindowDisplay();

station.addObserver(phone);
station.addObserver(window);

station.setTemperature(25.5f); // Both displays print
station.setTemperature(26.0f); // Both displays print
```

### ♟️ 4. Strategy Pattern

**Goal:** Define a family of algorithms, encapsulate each one, and make them interchangeable at runtime.
**Common Uses:** Payment processing (Credit Card vs PayPal), Sorting algorithms.

```java
// 1. Strategy Interface
public interface PaymentStrategy {
    void pay(int amount);
}

// 2. Concrete Strategies
public class CreditCardPayment implements PaymentStrategy {
    @Override public void pay(int amount) { System.out.println("Paid " + amount + " using Credit Card."); }
}

public class PaypalPayment implements PaymentStrategy {
    @Override public void pay(int amount) { System.out.println("Paid " + amount + " using PayPal."); }
}

// 3. Context (The class that USES the strategy)
public class ShoppingCart {
    private PaymentStrategy paymentStrategy; // Inject strategy here

    // Strategy can be swapped at runtime!
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }

    public void checkout(int amount) {
        if (paymentStrategy == null) {
            System.out.println("Please select a payment method.");
            return;
        }
        paymentStrategy.pay(amount); // Delegate to strategy
    }
}

// Usage
ShoppingCart cart = new ShoppingCart();
cart.setPaymentStrategy(new PaypalPayment());
cart.checkout(100); // Paid 100 using PayPal

cart.setPaymentStrategy(new CreditCardPayment()); // Switched strategy dynamically!
cart.checkout(200); // Paid 200 using Credit Card
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between Decorator and Inheritance?**
> Inheritance adds behavior *statically* to a class at compile time. It applies to all instances of the class and can lead to class explosion (e.g., `MilkCoffee`, `SugarCoffee`, `MilkAndSugarCoffee`). Decorator adds behavior *dynamically* to an individual object at runtime using composition. You can mix and match decorators flexibly.

**Q2. Give a real-world example of the Adapter pattern in Java.**
> The `Arrays.asList()` method acts as an adapter, taking an array and returning a `List` interface wrapper over it. Another classic example is `java.io.InputStreamReader`, which acts as an adapter bridging the byte-based `InputStream` to the character-based `Reader`.

**Q3. How does the Strategy pattern adhere to the Open/Closed Principle?**
> The Open/Closed Principle states classes should be open for extension but closed for modification. In the Strategy pattern, if you want to add a new payment method (e.g., `CryptoPayment`), you simply create a new class implementing the `PaymentStrategy` interface. You do *not* have to modify the `ShoppingCart` class or any existing strategies.

---

## ✅ Best Practices

1. **Favor Composition over Inheritance.** The Decorator and Strategy patterns both rely heavily on composition to provide immense flexibility at runtime, avoiding brittle inheritance hierarchies.
2. **Use Lambdas for simple Strategies.** In Java 8+, if a Strategy interface has only one method (a Functional Interface), you can pass lambda expressions directly instead of creating concrete class files! (e.g., `cart.setPaymentStrategy(amount -> System.out.println("Paid " + amount));`)
3. **Keep Observers fast.** In the Observer pattern, `notifyObservers()` typically runs synchronously. If one observer performs a slow database write, it blocks the publisher and all other observers. Consider asynchronous notification if observers perform heavy I/O.

---

## 🛠️ Hands-on Practice

1. Implement the Strategy pattern for a `SortingContext` that can sort an array using either `BubbleSortStrategy` or `QuickSortStrategy`.
2. Use Java 8 Lambdas to pass the sorting strategies directly to the context without creating concrete strategy classes.
3. Implement the Observer pattern for a `YouTubeChannel` (Publisher) and `Subscriber` (Observer). When the channel uploads a video, all subscribers should receive a notification string.
4. Implement the Adapter pattern to connect a legacy `EuropeanPlug` (2 round pins) into a modern `AmericanSocket` (2 flat pins) interface.
