# Day 06–07 — Collections: Map & Queue
**Level:** Beginner → Intermediate | **Java Version:** 8+

---

## 🗺️ Map — Key-Value Pairs

A **Map** stores key-value pairs. Each key is unique; values can be duplicated. It's not a `Collection` (separate hierarchy).

```
Map<K, V>
├── HashMap          — unordered, fastest
├── LinkedHashMap    — insertion or access order
├── TreeMap          — sorted by key
├── ConcurrentHashMap — thread-safe
└── Hashtable        — legacy, avoid
```

---

## 🔵 HashMap — Your Default Map

```java
import java.util.*;

// HashMap — O(1) average put/get, no order guarantee
Map<String, Integer> wordCount = new HashMap<>();

// Add / update
wordCount.put("apple",  5);
wordCount.put("banana", 3);
wordCount.put("cherry", 8);
wordCount.put("apple",  7);  // Overwrites previous value!

System.out.println(wordCount);
// {banana=3, cherry=8, apple=7} — ORDER NOT GUARANTEED

// Get value
System.out.println(wordCount.get("apple"));      // 7
System.out.println(wordCount.get("mango"));      // null — key doesn't exist
System.out.println(wordCount.getOrDefault("mango", 0));  // 0 — safe fallback

// Check existence
System.out.println(wordCount.containsKey("banana"));  // true
System.out.println(wordCount.containsValue(8));       // true

// Remove
wordCount.remove("banana");
wordCount.remove("cherry", 99);  // Only removes if value matches — false here, doesn't remove

// Size
System.out.println(wordCount.size());  // 2

// Iterate — THREE ways
// 1. Over entries (most common)
for (Map.Entry<String, Integer> entry : wordCount.entrySet()) {
    System.out.println(entry.getKey() + " → " + entry.getValue());
}

// 2. Over keys only
for (String key : wordCount.keySet()) {
    System.out.println(key + " → " + wordCount.get(key));
}

// 3. Over values only
for (Integer value : wordCount.values()) {
    System.out.println(value);
}

// 4. Java 8+ forEach
wordCount.forEach((key, value) ->
    System.out.println(key + "=" + value));
```

### HashMap — Essential Methods (Java 8+)

```java
Map<String, List<String>> studentCourses = new HashMap<>();

// computeIfAbsent — create value if key doesn't exist
studentCourses.computeIfAbsent("Aman", k -> new ArrayList<>()).add("Java");
studentCourses.computeIfAbsent("Aman", k -> new ArrayList<>()).add("Python");
// If "Aman" already exists, just returns the existing list
System.out.println(studentCourses);  // {Aman=[Java, Python]}

// compute — always compute (can update existing)
Map<String, Integer> scores = new HashMap<>();
scores.put("Player1", 10);
scores.compute("Player1", (k, v) -> v == null ? 1 : v + 5);  // 10 + 5 = 15
scores.compute("Player2", (k, v) -> v == null ? 1 : v + 5);  // null → 1
System.out.println(scores);  // {Player1=15, Player2=1}

// merge — merge new value with existing
Map<String, Integer> inventory = new HashMap<>();
inventory.put("apple", 10);
inventory.merge("apple",  5, Integer::sum);    // 10 + 5 = 15
inventory.merge("banana", 8, Integer::sum);    // null → 8
System.out.println(inventory);  // {apple=15, banana=8}

// putIfAbsent — only puts if key doesn't exist
inventory.putIfAbsent("apple", 999);  // apple already exists — no change
inventory.putIfAbsent("grape", 20);   // grape doesn't exist — adds it
System.out.println(inventory.get("apple")); // 15 — unchanged
```

---

### Word Frequency Counter — Classic HashMap Use Case

```java
public static Map<String, Integer> countWords(String text) {
    Map<String, Integer> frequency = new HashMap<>();

    String[] words = text.toLowerCase().split("\\s+");
    for (String word : words) {
        // Before Java 8:
        // if (frequency.containsKey(word)) frequency.put(word, frequency.get(word) + 1);
        // else frequency.put(word, 1);

        // Java 8+ — cleaner:
        frequency.merge(word, 1, Integer::sum);
    }
    return frequency;
}

Map<String, Integer> freq = countWords("the cat sat on the mat the cat");
System.out.println(freq);
// {on=1, the=3, sat=1, mat=1, cat=2}

// Find top 3 most frequent words
freq.entrySet().stream()
    .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
    .limit(3)
    .forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));
// the: 3
// cat: 2
// on:  1
```

---

## 🔗 LinkedHashMap — Order Matters

```java
// LinkedHashMap — maintains INSERTION ORDER (default) or ACCESS ORDER
Map<String, String> capitals = new LinkedHashMap<>();
capitals.put("India",   "New Delhi");
capitals.put("France",  "Paris");
capitals.put("Germany", "Berlin");
capitals.put("Japan",   "Tokyo");

System.out.println(capitals);
// {India=New Delhi, France=Paris, Germany=Berlin, Japan=Tokyo} — INSERTION ORDER!

// Access-order LinkedHashMap — great for LRU cache!
Map<String, String> lruCache = new LinkedHashMap<>(16, 0.75f, true) {
    // Override removeEldestEntry to auto-evict old entries (LRU cache implementation)
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, String> eldest) {
        return size() > 3;  // Evict when cache exceeds 3 entries
    }
};
lruCache.put("A", "1");
lruCache.put("B", "2");
lruCache.put("C", "3");
lruCache.get("A");         // Access A — moves it to end (most recently used)
lruCache.put("D", "4");   // Cache full — evicts B (least recently used)
System.out.println(lruCache); // {C=3, A=1, D=4} — B evicted
```

---

## 🌳 TreeMap — Sorted Keys

```java
// TreeMap — keys always sorted in natural order (or custom Comparator)
Map<String, Integer> treeMap = new TreeMap<>();
treeMap.put("banana", 3);
treeMap.put("apple",  5);
treeMap.put("cherry", 8);
treeMap.put("date",   2);

System.out.println(treeMap);
// {apple=5, banana=3, cherry=8, date=2} — ALPHABETICAL ORDER!

// Navigation methods (NavigableMap)
NavigableMap<String, Integer> navMap = new TreeMap<>(treeMap);
System.out.println(navMap.firstKey());              // apple
System.out.println(navMap.lastKey());               // date
System.out.println(navMap.headMap("cherry"));       // {apple=5, banana=3} — before "cherry"
System.out.println(navMap.tailMap("cherry"));       // {cherry=8, date=2} — "cherry" and after
System.out.println(navMap.floorKey("blueberry"));   // banana — greatest key ≤ "blueberry"
System.out.println(navMap.ceilingKey("blueberry")); // cherry — smallest key ≥ "blueberry"
```

---

## ⚡ ConcurrentHashMap — Thread-Safe Map

```java
import java.util.concurrent.ConcurrentHashMap;

// ConcurrentHashMap — thread-safe, high-performance (no locking entire map)
// Uses segmented locking (16 segments by default in Java 7, lock-free in Java 8+)
Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();

// All HashMap methods work here
concurrentMap.put("key1", 1);
concurrentMap.computeIfAbsent("counter", k -> 0);
concurrentMap.compute("counter", (k, v) -> v + 1);

// Safe to use from multiple threads simultaneously
// Unlike Collections.synchronizedMap() — which locks the ENTIRE map for each operation
```

---

## 🎯 Map Comparison

| Feature | HashMap | LinkedHashMap | TreeMap | ConcurrentHashMap |
|---------|---------|--------------|---------|-------------------|
| **Order** | None | Insertion/Access | Sorted | None |
| **Null keys** | ✅ 1 null | ✅ 1 null | ❌ No null | ❌ No null |
| **Thread-safe** | ❌ | ❌ | ❌ | ✅ |
| **Performance** | O(1) | O(1) | O(log n) | O(1) |
| **Use when** | Speed | Order matters | Sorted keys | Concurrent access |

---

## 🚦 Queue & Deque — Processing Order

### Queue — FIFO (First In, First Out)

```java
// Queue — elements processed in order they were added
Queue<String> taskQueue = new LinkedList<>();

taskQueue.offer("Task1");   // Add to end — prefer offer() over add()
taskQueue.offer("Task2");   // offer() returns false on failure; add() throws exception
taskQueue.offer("Task3");

System.out.println(taskQueue.peek());   // "Task1" — view front without removing
System.out.println(taskQueue.poll());   // "Task1" — remove and return front
System.out.println(taskQueue.poll());   // "Task2"
System.out.println(taskQueue.size());   // 1

// Process all tasks
while (!taskQueue.isEmpty()) {
    String task = taskQueue.poll();
    System.out.println("Processing: " + task);
}
```

### PriorityQueue — Highest Priority First

```java
// PriorityQueue — smallest element removed first (min-heap by default)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5);
minHeap.offer(1);
minHeap.offer(3);
minHeap.offer(2);

System.out.println(minHeap.peek());   // 1 — smallest
System.out.println(minHeap.poll());   // 1
System.out.println(minHeap.poll());   // 2
System.out.println(minHeap.poll());   // 3

// Max-heap (largest first)
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(5); maxHeap.offer(1); maxHeap.offer(3);
System.out.println(maxHeap.poll());   // 5

// Priority queue with custom priority (Task with priority number)
record Task(String name, int priority) implements Comparable<Task> {
    @Override
    public int compareTo(Task other) {
        return Integer.compare(other.priority, this.priority);  // Higher priority = dequeued first
    }
}

PriorityQueue<Task> taskPQ = new PriorityQueue<>();
taskPQ.offer(new Task("Email",  2));
taskPQ.offer(new Task("Bug Fix", 5));
taskPQ.offer(new Task("Meeting", 1));
taskPQ.offer(new Task("Deploy",  4));

while (!taskPQ.isEmpty()) {
    System.out.println(taskPQ.poll().name());
}
// Bug Fix (5), Deploy (4), Email (2), Meeting (1) — priority order!
```

### ArrayDeque — Double-Ended Queue (Deque)

```java
// ArrayDeque — efficient stack AND queue, faster than LinkedList for both
Deque<String> deque = new ArrayDeque<>();

// Queue operations (FIFO)
deque.offerLast("A");    // Add to end
deque.offerLast("B");
deque.offerFirst("Z");   // Add to front

System.out.println(deque);          // [Z, A, B]
System.out.println(deque.pollFirst()); // Z
System.out.println(deque.pollLast());  // B

// Stack operations (LIFO)
Deque<String> stack = new ArrayDeque<>();
stack.push("First");   // = addFirst()
stack.push("Second");
stack.push("Third");
System.out.println(stack.pop());  // Third — LIFO
System.out.println(stack.pop());  // Second
```

---

## ❓ Interview Questions & Answers

**Q1. How does HashMap work internally?**
> HashMap uses an array of buckets (default 16). When you `put(key, value)`: 1) Compute `key.hashCode()`. 2) Apply bit manipulation to get a bucket index. 3) Store a `Node(key, value)` in that bucket. If two keys hash to the same bucket (collision), they form a linked list in that bucket. In Java 8+, if a bucket's linked list exceeds 8 entries, it converts to a balanced Red-Black tree (O(log n) instead of O(n) worst case). When the map is 75% full (load factor), it resizes to double capacity.

**Q2. What is the load factor in HashMap?**
> Load factor (default 0.75) is the threshold that triggers resizing. When `size / capacity > loadFactor`, HashMap doubles its array size and rehashes all entries. 0.75 balances memory vs. collision rate. A lower load factor (e.g., 0.5) means fewer collisions but more memory usage. A higher load factor (e.g., 0.9) saves memory but increases collisions and degrades to O(n) operations.

**Q3. What is the difference between HashMap and Hashtable?**
> Both are hash tables, but: Hashtable is **synchronized** (thread-safe but slow — locks entire table). HashMap is **not synchronized** (faster, not thread-safe). HashMap allows one null key and null values; Hashtable allows neither. Hashtable is a legacy class — use `ConcurrentHashMap` for thread-safe needs instead of Hashtable.

**Q4. What is the difference between `poll()` and `remove()` in Queue?**
> Both remove and return the head of the queue. The difference is when the queue is EMPTY: `poll()` returns `null` (safe). `remove()` throws `NoSuchElementException` (unsafe). Similarly: `peek()` vs `element()` for viewing the head, and `offer()` vs `add()` for adding.

**Q5. When would you use a PriorityQueue?**
> When elements need to be processed by priority, not arrival order. Examples: task scheduler (high-priority tasks first), Dijkstra's shortest path algorithm, hospital emergency queue (critical patients first), merge K sorted arrays (always extract the smallest element from any list). It's a min-heap by default; use `Comparator.reverseOrder()` for max-heap behaviour.

---

## ✅ Best Practices

1. **Always declare as interface type** — `Map<K,V>` not `HashMap<K,V>`
2. **Use `getOrDefault()`** to avoid null checks on Map lookups
3. **Use `merge()` or `computeIfAbsent()`** for group-by and counting patterns
4. **Use `ConcurrentHashMap`** instead of `Collections.synchronizedMap()` — much faster
5. **Use `LinkedHashMap`** when you need a cache that maintains insertion order
6. **Override `equals()` and `hashCode()`** on objects used as Map keys
7. **Use `ArrayDeque`** instead of `Stack` class (Stack is legacy, synchronised)

---

## 🛠️ Hands-on Practice

1. Write a method `groupByDepartment(List<Employee> employees)` that returns `Map<String, List<Employee>>` using `computeIfAbsent()`.
2. Implement a simple LRU Cache using `LinkedHashMap` with access-order and `removeEldestEntry()`. Test with a capacity of 3.
3. Given a list of strings, count the frequency of each character (across all strings) using a `HashMap`. Sort the result by frequency descending.
4. Write a task scheduler using `PriorityQueue<Task>` where tasks have a priority (1-10, higher = more urgent). Simulate adding 10 tasks and processing them in priority order.
5. Implement a `WordDictionary` using `TreeMap<String, String>` that can find the closest word alphabetically to a given input using `floorKey()` and `ceilingKey()`.
