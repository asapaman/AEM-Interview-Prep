# Day 00(B) — Arrays: Deep Dive & All Operations
**Level:** Beginner → Intermediate | **Java Version:** 8+

---

## 🌟 1. Array Basics

An **Array** in Java is an object that contains a fixed number of values of a single type. The length of an array is established when the array is created. After creation, its length is fixed.

### Declaration and Initialization
```java
// 1. Declaration without initialization
int[] numbers; 
// 2. Initialization with fixed size (elements default to 0, false, or null)
numbers = new int[5]; // [0, 0, 0, 0, 0]

// 3. Declaration and initialization with values (Array Literal)
String[] names = {"Aman", "Priya", "John"};

// 4. Anonymous array (creating an array on the fly to pass to a method)
printArray(new int[]{10, 20, 30});
```

### Accessing and Traversing
Arrays are zero-indexed.
```java
int[] nums = {10, 20, 30, 40};

// Accessing
System.out.println(nums[0]); // 10 (First element)
System.out.println(nums[nums.length - 1]); // 40 (Last element)

// Modifying
nums[1] = 99; // Changes 20 to 99

// Traversing (Standard for loop)
for (int i = 0; i < nums.length; i++) {
    System.out.print(nums[i] + " ");
}

// Traversing (Enhanced for-each loop - Read Only)
for (int num : nums) {
    System.out.print(num + " ");
}
```

---

## 🧰 2. The `java.util.Arrays` Utility Class

This class contains various static methods for manipulating arrays (sorting, searching, filling, etc.). **Memorize these!**

```java
import java.util.Arrays;

public class ArrayUtilsDemo {
    public static void main(String[] args) {
        int[] arr1 = {5, 2, 8, 1, 9};
        
        // 1. Printing (toString) - DO NOT just do System.out.println(arr1)
        System.out.println(Arrays.toString(arr1)); // Prints: [5, 2, 8, 1, 9]
        
        // 2. Sorting
        Arrays.sort(arr1); 
        System.out.println(Arrays.toString(arr1)); // Prints: [1, 2, 5, 8, 9]
        
        // 3. Binary Search (ARRAY MUST BE SORTED FIRST!)
        int index = Arrays.binarySearch(arr1, 5); // Returns 2
        int notFound = Arrays.binarySearch(arr1, 10); // Returns negative number
        
        // 4. Filling an array with a specific value
        int[] zeroes = new int[5];
        Arrays.fill(zeroes, 7); // Makes it [7, 7, 7, 7, 7]
        
        // 5. Copying an array
        // copyOf(originalArray, newLength) - truncates or pads with 0s if length is different
        int[] copy = Arrays.copyOf(arr1, arr1.length); 
        
        // copyOfRange(originalArray, fromIndex, toIndex (exclusive))
        int[] slice = Arrays.copyOfRange(arr1, 1, 4); // [2, 5, 8]
        
        // 6. Comparing Arrays (equals checks values in order)
        int[] arr2 = {1, 2, 5, 8, 9};
        boolean areEqual = Arrays.equals(arr1, arr2); // true
        
        // 7. Converting Array to List (Note: Returns a FIXED-SIZE list)
        String[] colors = {"Red", "Blue", "Green"};
        List<String> colorList = Arrays.asList(colors);
        // colorList.add("Yellow"); // THROWS UnsupportedOperationException!
        
        // 8. Streams (Java 8+)
        int sum = Arrays.stream(arr1).sum();
        int max = Arrays.stream(arr1).max().getAsInt();
    }
}
```

---

## 🔪 3. Custom Array Operations (Common Interview Tasks)

Since arrays are fixed size, operations like insert and delete require creating new arrays or shifting elements.

### A. Inserting an Element at a Specific Index
Because an array cannot grow, we must create a new, larger array.
```java
public static int[] insertElement(int[] original, int element, int index) {
    int[] result = new int[original.length + 1]; // New array, size + 1
    
    for (int i = 0, j = 0; i < result.length; i++) {
        if (i == index) {
            result[i] = element; // Insert new element
        } else {
            result[i] = original[j++]; // Copy existing element
        }
    }
    return result;
}
```

### B. Deleting an Element at a Specific Index
```java
public static int[] deleteElement(int[] original, int index) {
    if (original == null || index < 0 || index >= original.length) {
        return original;
    }
    
    int[] result = new int[original.length - 1]; // New array, size - 1
    
    for (int i = 0, k = 0; i < original.length; i++) {
        if (i == index) continue; // Skip the element to delete
        result[k++] = original[i]; // Copy everything else
    }
    return result;
}
```

### C. Reversing an Array IN-PLACE
A classic interview question. Do not create a new array. Use two pointers.
```java
public static void reverseArray(int[] arr) {
    int left = 0;
    int right = arr.length - 1;
    
    while (left < right) {
        // Swap elements
        int temp = arr[left];
        arr[left] = arr[right];
        arr[right] = temp;
        
        left++;
        right--;
    }
}
```

### D. Finding Min and Max in One Pass
```java
public static void findMinMax(int[] arr) {
    if (arr == null || arr.length == 0) return;
    
    int min = arr[0];
    int max = arr[0];
    
    for (int i = 1; i < arr.length; i++) {
        if (arr[i] > max) max = arr[i];
        if (arr[i] < min) min = arr[i];
    }
    System.out.println("Min: " + min + ", Max: " + max);
}
```

---

## 🎲 4. Multidimensional Arrays

In Java, a 2D array is essentially an "array of arrays".

```java
// Declaration & Initialization
int[][] matrix = {
    {1, 2, 3},       // Row 0
    {4, 5, 6, 7},    // Row 1 (Notice rows can have different lengths! Jagged Arrays)
    {8, 9}           // Row 2
};

// Accessing
System.out.println(matrix[1][2]); // 6 (Row index 1, Column index 2)

// Traversing a 2D Array
for (int row = 0; row < matrix.length; row++) {
    for (int col = 0; col < matrix[row].length; col++) {
        System.out.print(matrix[row][col] + " ");
    }
    System.out.println(); // Newline after each row
}

// Printing a 2D array easily
System.out.println(Arrays.deepToString(matrix)); 
// Prints: [[1, 2, 3], [4, 5, 6, 7], [8, 9]]
```

---

## ❓ Interview Questions & Answers

**Q1. What happens if you try to access an array index that is out of bounds?**
> The JVM throws an `ArrayIndexOutOfBoundsException` at runtime.

**Q2. How do you find the length of an array? Is it a method or a property?**
> It is a `final` property (field) of the array object. You access it using `arr.length` (no parentheses). Note that for Strings, it is a method: `str.length()`.

**Q3. What is the difference between `Arrays.asList()` and `new ArrayList<>(Arrays.asList())`?**
> `Arrays.asList(arr)` returns a fixed-size list backed by the original array. You can replace elements (using `.set()`), but you CANNOT add or remove elements (throws `UnsupportedOperationException`). It acts as a bridge. 
> `new ArrayList<>(Arrays.asList(arr))` creates a brand new, fully modifiable, independent `ArrayList` containing the elements of the array.

**Q4. Can you change the size of an array after it is created?**
> No. Arrays in Java are fixed-size structures. If you need a dynamically resizing data structure, you should use `ArrayList`. To "resize" an array, you must create a completely new array with the new size and use `Arrays.copyOf()` or `System.arraycopy()` to transfer the data.

**Q5. Explain `System.arraycopy()`.**
> It is a native, highly optimized method used to copy data from a source array to a destination array. It is faster than using a `for` loop.
> `System.arraycopy(srcArray, srcPos, destArray, destPos, length);`

---

## ✅ Best Practices

1. **Use `ArrayList` by default.** Unless you are dealing with multi-dimensional matrices, extreme performance/memory constraints, or primitive types (where Boxing overhead matters), `ArrayList` is infinitely easier to work with.
2. **Use `Arrays.toString()` for debugging.** Printing an array directly just prints its memory address hash (e.g., `[I@76ed5528`).
3. **Be careful with Object array copies.** `Arrays.copyOf()` creates a *shallow copy*. The new array holds references to the *same objects* as the old array.

---

## 🛠️ Hands-on Practice

1. Write a method to move all zeros in an integer array to the end while maintaining the relative order of non-zero elements. (e.g., `[0, 1, 0, 3, 12]` -> `[1, 3, 12, 0, 0]`).
2. Write a method that merges two sorted arrays into a single new sorted array in O(n) time.
3. Given an array of integers, find the contiguous subarray (containing at least one number) which has the largest sum and return its sum. (Kadane's Algorithm).
4. Write a program to rotate an array to the right by `k` steps. (e.g., `[1,2,3,4,5,6,7]` rotated by `k=3` becomes `[5,6,7,1,2,3,4]`).
