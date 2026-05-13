# Day 08 — Generics
**Level:** Intermediate | **Java Version:** 5+

---

## 🌟 What are Generics?

Generics allow you to write classes, interfaces, and methods that operate on types specified by the caller. They provide **compile-time type safety** and eliminate the need for explicit casting.

Before Generics (Java 1.4):
```java
List list = new ArrayList();
list.add("Hello");
list.add(123); // Allowed, but dangerous!

String s = (String) list.get(0); // Explicit cast required
String s2 = (String) list.get(1); // RUNTIME ERROR: ClassCastException (Integer to String)
```

With Generics (Java 5+):
```java
List<String> list = new ArrayList<>(); // Type inferred by diamond operator <>
list.add("Hello");
// list.add(123); // COMPILE-TIME ERROR: Type safety!

String s = list.get(0); // No cast needed
```

---

## 📦 Generic Classes and Interfaces

You can define your own classes that take type parameters. Common naming conventions:
- `E` - Element (used extensively by the Java Collections Framework)
- `K` - Key (used in Map)
- `N` - Number
- `T` - Type
- `V` - Value (used in Map)
- `S`, `U`, `V` etc. - 2nd, 3rd, 4th types

```java
// A generic box that can hold any type 'T'
public class Box<T> {
    private T item;

    public void set(T item) {
        this.item = item;
    }

    public T get() {
        return item;
    }
}

// Usage
Box<String> stringBox = new Box<>();
stringBox.set("Secret Message");
String msg = stringBox.get(); // Returns String

Box<Integer> intBox = new Box<>();
intBox.set(42);
Integer num = intBox.get(); // Returns Integer

// Multiple Type Parameters
public interface Pair<K, V> {
    K getKey();
    V getValue();
}

public class OrderedPair<K, V> implements Pair<K, V> {
    private K key;
    private V value;

    public OrderedPair(K key, V value) {
        this.key = key;
        this.value = value;
    }

    @Override
    public K getKey() { return key; }

    @Override
    public V getValue() { return value; }
}

Pair<String, Integer> p1 = new OrderedPair<>("Age", 30);
```

---

## 🛠️ Generic Methods

You can define generic methods inside both generic and non-generic classes. The type parameter is declared *before* the return type.

```java
public class Util {
    // Generic method: Takes an array of type T and an element of type T
    public static <T> int countOccurrences(T[] array, T target) {
        int count = 0;
        for (T element : array) {
            if (element.equals(target)) {
                count++;
            }
        }
        return count;
    }
}

// Usage
String[] words = {"apple", "banana", "apple"};
int appleCount = Util.<String>countOccurrences(words, "apple"); // Explicit type witness
int count = Util.countOccurrences(words, "apple"); // Type inference (common)
```

---

## 🚧 Bounded Type Parameters

Sometimes you want to restrict the types that can be used as type arguments. Use `extends` to specify an upper bound.

```java
// T must be a subclass of Number (Integer, Double, etc.)
public class NumberBox<T extends Number> {
    private T number;

    public NumberBox(T number) {
        this.number = number;
    }

    public double getDoubleValue() {
        // We know 'number' has doubleValue() because it extends Number
        return number.doubleValue();
    }
}

NumberBox<Integer> intBox = new NumberBox<>(10);
NumberBox<Double> doubleBox = new NumberBox<>(3.14);
// NumberBox<String> strBox = new NumberBox<>("10"); // COMPILE ERROR: String is not a Number

// Multiple bounds (Class first, then Interfaces)
public class ComparableBox<T extends Number & Comparable<T>> {
    // T must extend Number AND implement Comparable<T>
}
```

---

## 🃏 Wildcards (`?`)

Wildcards represent an unknown type. They are crucial for writing flexible methods that operate on collections.

### 1. Unbounded Wildcard (`?`)
Matches any type. Use when you only need methods from `Object` (like `toString()`).

```java
public static void printList(List<?> list) {
    for (Object elem : list) {
        System.out.println(elem);
    }
    // list.add("Hello"); // COMPILE ERROR! You can't add to a List<?> (except null)
}

List<Integer> ints = Arrays.asList(1, 2, 3);
List<String> strings = Arrays.asList("A", "B");
printList(ints);   // OK
printList(strings); // OK
```

### 2. Upper Bounded Wildcard (`? extends Type`)
Matches `Type` or any subclass of `Type`. Use for **reading** data (Producer).

```java
// List can be List<Number>, List<Integer>, List<Double>
public static double sumOfList(List<? extends Number> list) {
    double sum = 0.0;
    for (Number n : list) {
        sum += n.doubleValue(); // Safe to read as Number
    }
    // list.add(10); // COMPILE ERROR! We don't know if it's a List<Double> or List<Integer>
    return sum;
}

List<Integer> integers = Arrays.asList(1, 2, 3);
System.out.println(sumOfList(integers)); // OK
```

### 3. Lower Bounded Wildcard (`? super Type`)
Matches `Type` or any superclass of `Type`. Use for **writing** data (Consumer).

```java
// List can be List<Integer>, List<Number>, List<Object>
public static void addNumbers(List<? super Integer> list) {
    list.add(1); // Safe to add Integer
    list.add(2); // Safe to add Integer
    // Number n = list.get(0); // COMPILE ERROR! Could be a List<Object>, get() returns Object
}

List<Number> numList = new ArrayList<>();
addNumbers(numList); // OK, Integer is a subclass of Number
```

### The PECS Rule (Producer Extends, Consumer Super)
- Use `? extends T` when you only **get** values out of a structure (Producer).
- Use `? super T` when you only **put** values into a structure (Consumer).
- Use `T` (no wildcard) when you need to do both.

---

## 🧹 Type Erasure

Java implements Generics using **Type Erasure**. The compiler enforces type safety, but then *removes* (erases) all generic type information during compilation to maintain backward compatibility with pre-Java 5 code.

```java
// What you write:
List<String> list = new ArrayList<>();
list.add("Hello");
String s = list.get(0);

// What the compiler translates it to (roughly):
List list = new ArrayList();
list.add("Hello");
String s = (String) list.get(0); // Cast inserted by compiler
```

**Consequences of Type Erasure:**
1. Cannot instantiate generic types: `T obj = new T(); // ERROR`
2. Cannot create arrays of generic types: `T[] array = new T[10]; // ERROR`
3. Cannot use primitive types as generic arguments: `List<int> // ERROR` (must use `List<Integer>`)
4. Cannot use `instanceof` with parameterized types: `if (obj instanceof List<String>) // ERROR`
5. Overloading conflicts after erasure:
   ```java
   // COMPILE ERROR: Both erase to print(List)
   public void print(List<String> list) {}
   public void print(List<Integer> list) {}
   ```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between `List<Object>` and `List<?>`?**
> `List<Object>` can only accept exactly a `List<Object>`. You cannot pass a `List<String>` to a method expecting `List<Object>` because `List<String>` is NOT a subclass of `List<Object>` (this is called *invariance*). `List<?>` is a wildcard that accepts a List of ANY type (`List<String>`, `List<Integer>`, etc.). However, you cannot add elements (other than null) to a `List<?>`.

**Q2. Explain the PECS rule with an example.**
> PECS stands for "Producer Extends, Consumer Super". It guides wildcard usage. If a method reads from a collection (the collection is a producer), use `? extends T`. For example, `void copyAll(List<? extends T> src, List<? super T> dest)`: `src` produces elements, so it's `? extends T`. `dest` consumes elements, so it's `? super T`.

**Q3. What is Type Erasure and why does Java use it?**
> Type Erasure is the process where the Java compiler removes generic type parameters and replaces them with their bounds (or `Object` if unbounded) and inserts necessary casts. Java uses it to ensure backward compatibility with older versions of Java (pre-Java 5) that did not support generics, allowing generic code to interoperate with legacy code.

**Q4. Can you use primitive types as type arguments in Generics? Why?**
> No, you cannot use primitive types (like `int`, `double`). You must use their wrapper classes (`Integer`, `Double`). This is due to Type Erasure. The erased type defaults to `Object`, and primitive types do not inherit from `Object`. Autoboxing/unboxing usually handles the conversion automatically, but it incurs a slight performance penalty.

**Q5. Why can't you create an array of a generic type `T[] array = new T[10]`?**
> Arrays in Java are *covariant* and reified (they know their element type at runtime). Generics are *invariant* and erased (type info is lost at runtime). If `new T[10]` were allowed, it would erase to `new Object[10]`. At runtime, the array wouldn't know what type it's supposed to hold, potentially leading to `ArrayStoreException` if wrong types were added later. You usually work around this using an `Object[]` and casting, or using `ArrayList<T>`.

---

## ✅ Best Practices

1. **Always use parameterized types** (e.g., `List<String>`), avoid raw types (e.g., `List`) to catch errors at compile time.
2. **Use the diamond operator `<>`** to reduce verbosity when the type can be inferred.
3. **Follow the PECS rule** for flexible API design when accepting collections as arguments.
4. **Use bounded wildcards** to increase API flexibility (e.g., accepting `Collection<? extends Number>` instead of just `Collection<Number>`).
5. **Prefer Lists over Arrays** when working with generics to avoid array-generic friction.

---

## 🛠️ Hands-on Practice

1. Create a generic class `Storage<T>` with `store(T item)` and `retrieve()` methods.
2. Write a generic method `swap(T[] array, int i, int j)` that swaps two elements in an array. Test it with `Integer[]` and `String[]`.
3. Create a method `merge(List<? extends Number> list1, List<? extends Number> list2)` that returns a new `List<Double>` containing the sum of corresponding elements (handle different sizes).
4. Implement a `Predicate<T>` interface with a `boolean test(T t)` method. Write a generic `filter(List<T> list, Predicate<T> p)` method that returns a new list of items that pass the test.
5. Create a class hierarchy: `Animal -> Dog -> Poodle`. Write a method `void addDog(List<? super Dog> list)` and test calling it with `List<Animal>`, `List<Dog>`, and `List<Poodle>`. Observe which ones compile.
