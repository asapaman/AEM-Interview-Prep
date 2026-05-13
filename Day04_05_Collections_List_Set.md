# Day 04–05 — Collections: List & Set
**Level:** Beginner → Intermediate | **Java Version:** 8+

---

## 🌟 What is the Java Collections Framework?

The **Collections Framework** is a unified architecture for storing and manipulating groups of objects. Instead of writing your own linked list or hash table from scratch, you use battle-tested, optimised implementations.

```
Collection (interface)
├── List     — ordered, allows duplicates (ArrayList, LinkedList, Vector, Stack)
├── Set      — no duplicates (HashSet, LinkedHashSet, TreeSet)
└── Queue    — ordered for processing (PriorityQueue, ArrayDeque)

Map (separate hierarchy — key-value pairs)
└── HashMap, LinkedHashMap, TreeMap, ConcurrentHashMap
```

---

## 📋 List — Ordered, Allows Duplicates

### ArrayList — Your Default Choice

```java
import java.util.*;

// ArrayList — dynamic array, O(1) random access, O(n) insert/delete in middle
List<String> fruits = new ArrayList<>();

// Add elements
fruits.add("Apple");
fruits.add("Banana");
fruits.add("Cherry");
fruits.add("Apple");   // Duplicates ALLOWED

System.out.println(fruits);       // [Apple, Banana, Cherry, Apple]
System.out.println(fruits.size()); // 4

// Access by index — O(1)
System.out.println(fruits.get(0)); // Apple
System.out.println(fruits.get(2)); // Cherry

// Modify
fruits.set(1, "Blueberry");
System.out.println(fruits); // [Apple, Blueberry, Cherry, Apple]

// Remove
fruits.remove("Apple");        // Removes FIRST occurrence
fruits.remove(Integer.valueOf(0));  // ⚠️ Use Integer.valueOf() to remove by value not index
fruits.remove(0);              // Removes by INDEX
System.out.println(fruits);    // [Blueberry, Cherry]

// Check existence
System.out.println(fruits.contains("Cherry")); // true
System.out.println(fruits.indexOf("Cherry"));  // 1

// Iterate
for (String fruit : fruits) {
    System.out.println(fruit);
}

// Java 8+ forEach
fruits.forEach(f -> System.out.println(f.toUpperCase()));

// Sort
List<Integer> numbers = new ArrayList<>(Arrays.asList(5, 2, 8, 1, 9, 3));
Collections.sort(numbers);                      // Natural order: [1, 2, 3, 5, 8, 9]
Collections.sort(numbers, Comparator.reverseOrder()); // Reverse: [9, 8, 5, 3, 2, 1]
numbers.sort(Comparator.naturalOrder());        // Same as Collections.sort

// Create with initial values (Java 9+)
List<String> immutableList = List.of("A", "B", "C");  // Immutable!
List<String> mutableCopy   = new ArrayList<>(List.of("A", "B", "C")); // Mutable copy
```

### ArrayList Performance

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `get(index)` | O(1) | Direct array access |
| `add(element)` (end) | O(1) amortised | Occasionally O(n) when resizing |
| `add(index, element)` | O(n) | Shifts elements right |
| `remove(index)` | O(n) | Shifts elements left |
| `contains(element)` | O(n) | Linear search |
| `size()` | O(1) | Stored as field |

---

### LinkedList — When You Need Frequent Insert/Delete

```java
// LinkedList — doubly-linked list, O(1) add/remove at head/tail, O(n) access by index
LinkedList<String> queue = new LinkedList<>();

// Works as both List AND Deque (double-ended queue)
queue.addFirst("First");
queue.addLast("Last");
queue.add("Middle");

System.out.println(queue);          // [First, Middle, Last]
System.out.println(queue.getFirst()); // First
System.out.println(queue.getLast());  // Last

queue.removeFirst();  // Remove from front
queue.removeLast();   // Remove from end

// Use as a Stack (LIFO)
LinkedList<String> stack = new LinkedList<>();
stack.push("A");     // Add to front
stack.push("B");
stack.push("C");
System.out.println(stack.pop());   // C (LIFO)
System.out.println(stack.pop());   // B

// Use as a Queue (FIFO)
Queue<String> fifoQueue = new LinkedList<>();
fifoQueue.offer("Task1");   // Add to end
fifoQueue.offer("Task2");
fifoQueue.offer("Task3");
System.out.println(fifoQueue.poll());   // Task1 (FIFO — remove from front)
System.out.println(fifoQueue.peek());   // Task2 (view without removing)
```

### ArrayList vs LinkedList — Choose Wisely

| Criteria | ArrayList | LinkedList |
|---------|-----------|-----------|
| **Access by index** | ✅ O(1) fast | ❌ O(n) slow |
| **Insert at beginning/middle** | ❌ O(n) slow | ✅ O(1) fast |
| **Memory** | ✅ Less (contiguous) | ❌ More (node + pointers) |
| **Iteration** | ✅ Cache-friendly | ❌ Cache-unfriendly |
| **Default choice** | ✅ Yes | ❌ Only for frequent insert/delete |

---

## 📏 Sorting: Comparable vs Comparator

### Comparable — Natural Ordering (built into the class)

```java
public class Student implements Comparable<Student> {
    private String name;
    private int grade;
    private double gpa;

    public Student(String name, int grade, double gpa) {
        this.name  = name;
        this.grade = grade;
        this.gpa   = gpa;
    }

    // Define natural ordering — sorted by name alphabetically
    @Override
    public int compareTo(Student other) {
        return this.name.compareTo(other.name);  // Alphabetical by name
        // Return: negative if this < other, 0 if equal, positive if this > other
    }

    @Override
    public String toString() {
        return name + " (Grade:" + grade + ", GPA:" + gpa + ")";
    }

    public String getName() { return name; }
    public int getGrade()   { return grade; }
    public double getGpa()  { return gpa; }
}

List<Student> students = new ArrayList<>(Arrays.asList(
    new Student("Priya", 12, 3.8),
    new Student("Aman", 11, 3.5),
    new Student("Zara", 10, 4.0)
));

Collections.sort(students);  // Uses compareTo — alphabetical by name
System.out.println(students);
// [Aman (Grade:11, GPA:3.5), Priya (Grade:12, GPA:3.8), Zara (Grade:10, GPA:4.0)]
```

### Comparator — Custom Ordering (external)

```java
// Multiple sorting strategies without modifying the Student class

// Sort by GPA descending
Comparator<Student> byGpaDesc = Comparator.comparingDouble(Student::getGpa).reversed();

// Sort by grade ascending, then by name alphabetically (multi-key sort)
Comparator<Student> byGradeThenName = Comparator
    .comparingInt(Student::getGrade)
    .thenComparing(Student::getName);

// Sort by name length
Comparator<Student> byNameLength = Comparator.comparingInt(s -> s.getName().length());

students.sort(byGpaDesc);
System.out.println(students);
// [Zara (Grade:10, GPA:4.0), Priya (Grade:12, GPA:3.8), Aman (Grade:11, GPA:3.5)]

students.sort(byGradeThenName);
System.out.println(students);
// [Zara (Grade:10, GPA:4.0), Aman (Grade:11, GPA:3.5), Priya (Grade:12, GPA:3.8)]
```

---

## 🔷 Set — No Duplicates

### HashSet — Fastest, No Order Guarantee

```java
Set<String> hashSet = new HashSet<>();
hashSet.add("Banana");
hashSet.add("Apple");
hashSet.add("Cherry");
hashSet.add("Apple");   // Duplicate — silently ignored

System.out.println(hashSet);       // [Banana, Cherry, Apple] — ORDER NOT GUARANTEED
System.out.println(hashSet.size()); // 3 — Apple wasn't added twice

// Common operations
System.out.println(hashSet.contains("Apple")); // true — O(1) average
hashSet.remove("Banana");

// Set operations
Set<Integer> a = new HashSet<>(Arrays.asList(1, 2, 3, 4, 5));
Set<Integer> b = new HashSet<>(Arrays.asList(3, 4, 5, 6, 7));

// Union (all elements from both)
Set<Integer> union = new HashSet<>(a);
union.addAll(b);
System.out.println(union); // [1, 2, 3, 4, 5, 6, 7]

// Intersection (only common elements)
Set<Integer> intersection = new HashSet<>(a);
intersection.retainAll(b);
System.out.println(intersection); // [3, 4, 5]

// Difference (in a but not in b)
Set<Integer> difference = new HashSet<>(a);
difference.removeAll(b);
System.out.println(difference); // [1, 2]
```

### LinkedHashSet — Insertion Order Preserved

```java
// LinkedHashSet — HashSet + maintains insertion order
Set<String> linkedSet = new LinkedHashSet<>();
linkedSet.add("Banana");
linkedSet.add("Apple");
linkedSet.add("Cherry");
linkedSet.add("Apple");  // Duplicate ignored

System.out.println(linkedSet);
// [Banana, Apple, Cherry] — INSERTION ORDER PRESERVED!
```

### TreeSet — Sorted Order

```java
// TreeSet — elements sorted in natural order (or custom Comparator)
Set<Integer> treeSet = new TreeSet<>();
treeSet.add(5);
treeSet.add(1);
treeSet.add(3);
treeSet.add(2);

System.out.println(treeSet);      // [1, 2, 3, 5] — always sorted!

// TreeSet has navigation methods
NavigableSet<Integer> navSet = new TreeSet<>(Arrays.asList(10, 20, 30, 40, 50));
System.out.println(navSet.first());         // 10 — smallest
System.out.println(navSet.last());          // 50 — largest
System.out.println(navSet.headSet(30));     // [10, 20] — less than 30
System.out.println(navSet.tailSet(30));     // [30, 40, 50] — 30 and greater
System.out.println(navSet.floor(25));       // 20 — greatest element ≤ 25
System.out.println(navSet.ceiling(25));     // 30 — smallest element ≥ 25

// TreeSet with custom Comparator (by string length)
Set<String> byLength = new TreeSet<>(Comparator.comparingInt(String::length));
byLength.add("Banana");
byLength.add("Fig");
byLength.add("Apple");
byLength.add("Kiwi");
System.out.println(byLength); // [Fig, Kiwi, Apple, Banana] — sorted by length!
```

### Set Comparison

| Feature | HashSet | LinkedHashSet | TreeSet |
|---------|---------|--------------|---------|
| **Order** | No order | Insertion order | Sorted order |
| **Null elements** | ✅ 1 null | ✅ 1 null | ❌ No null |
| **Performance** | O(1) avg | O(1) avg | O(log n) |
| **Implements** | Set | Set | SortedSet + NavigableSet |
| **Use when** | Speed matters | Need order of add | Need sorted elements |

---

## 🛑 Common Pitfalls

```java
// ❌ PITFALL 1: Modifying list while iterating (ConcurrentModificationException)
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C", "D"));
// for (String s : list) {
//     if (s.equals("B")) list.remove(s);  // ❌ ConcurrentModificationException!
// }

// ✅ FIX: Use Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("B")) it.remove();  // Safe!
}

// ✅ OR use removeIf() (Java 8+)
list.removeIf(s -> s.equals("C"));

// ❌ PITFALL 2: HashSet with mutable objects — hashCode must not change after add
Set<List<Integer>> set = new HashSet<>();
List<Integer> innerList = new ArrayList<>(Arrays.asList(1, 2, 3));
set.add(innerList);
System.out.println(set.contains(innerList)); // true

innerList.add(4);  // ⚠️ List's hashCode changes!
System.out.println(set.contains(innerList)); // false — can't find it now!
// Use immutable objects in Sets where possible

// ❌ PITFALL 3: Using == instead of equals() for object comparison
String a = "hello";
String b = new String("hello");
System.out.println(a == b);       // false (different reference)
System.out.println(a.equals(b));  // true (same content)
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between ArrayList and LinkedList?**
> ArrayList uses a dynamic array internally — O(1) random access but O(n) for insert/delete in the middle. LinkedList uses a doubly-linked list — O(1) insert/delete at front/back but O(n) random access. ArrayList is almost always preferred because real-world operations are dominated by reads, and ArrayList has better cache locality. Use LinkedList only when you primarily add/remove from the ends.

**Q2. What is the difference between HashSet, LinkedHashSet, and TreeSet?**
> All three don't allow duplicates. HashSet: unordered, fastest O(1). LinkedHashSet: maintains insertion order, slightly slower. TreeSet: always sorted (natural or custom order), O(log n) — uses a Red-Black tree internally. Use HashSet for speed, LinkedHashSet to preserve insertion order, TreeSet when you need sorted traversal.

**Q3. How does HashSet prevent duplicate elements?**
> HashSet uses `hashCode()` and `equals()`. When you add an element: 1) Compute `hashCode()` to find the bucket. 2) In that bucket, check `equals()` against all existing elements. If `equals()` returns true for any existing element, the new element is rejected (duplicate). This is why if you don't override `hashCode()` and `equals()`, two "logically equal" objects can both end up in the Set.

**Q4. What is the difference between Comparable and Comparator?**
> `Comparable` is implemented by the class itself — defines its NATURAL ordering via `compareTo()`. One class = one natural order. `Comparator` is an external class — defines a CUSTOM ordering. You can have multiple Comparators for the same class (by name, by age, by salary). Comparator is more flexible and doesn't require modifying the original class.

**Q5. Why can't you add null to a TreeSet?**
> TreeSet uses `compareTo()` (or a Comparator) to position elements. When you add null, Java tries to call `null.compareTo(existingElement)` — which throws a NullPointerException. HashSet and LinkedHashSet allow one null because they use `hashCode()` (with a null check before calling it).

---

## ✅ Best Practices

1. **Use `ArrayList` by default** — it outperforms `LinkedList` in most real scenarios
2. **Declare as interface type** — `List<String>` not `ArrayList<String>` — easier to swap implementation
3. **Use `List.of()` for immutable lists** (Java 9+) — safe to pass around
4. **Override `equals()` and `hashCode()` on objects stored in Sets/as Map keys**
5. **Use `removeIf()` instead of removing during iteration** — cleaner and safer
6. **Use `TreeSet` only when you need sorted traversal** — has O(log n) overhead
7. **For thread-safe List, use `CopyOnWriteArrayList`** from `java.util.concurrent`

---

## 🛠️ Hands-on Practice

1. Create a `List<Integer>` of 10 random numbers. Sort it ascending, then descending. Find the max, min, and average using Collections and streams.
2. Remove all even numbers from an `ArrayList` using `removeIf()` and separately using `Iterator.remove()`.
3. Given two lists of student names, find: the union (all students), the intersection (students in both lists), and the difference (students only in first list) using Set operations.
4. Create a `TreeSet<Employee>` sorted by salary (descending), then by name (ascending). Add 10 employees and print them.
5. Create a custom `Comparator<String>` that sorts strings by length first, then alphabetically for same-length strings. Apply it to sort a list of city names.
