# 📦 Java Collections Deep Dive

---

## 0️⃣ Prerequisites

Before diving into Java Collections, you need to understand:

- **Array**: A fixed-size container holding elements of the same type. Once created, size cannot change.
- **Data Structure**: A way of organizing data for efficient access and modification.
- **Time Complexity**: How operation speed scales with data size. O(1) is constant time, O(n) is linear, O(log n) is logarithmic.
- **Generics**: Java's way of creating type-safe containers. `List<String>` only holds Strings.
- **Interface vs Implementation**: Interface defines what operations exist; implementation defines how they work.

If you understand that different data structures have different performance characteristics, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

Imagine you're building an e-commerce system. You need to:

1. Store a list of products (order matters)
2. Quickly check if a user is logged in (by ID)
3. Count how many times each product was viewed
4. Process orders in the order they arrived (FIFO)

With just arrays, you'd face:

```java
// Arrays are painful for dynamic data
String[] products = new String[100];  // Fixed size!
int productCount = 0;

// Adding a product
if (productCount < products.length) {
    products[productCount++] = "New Product";
} else {
    // Need to create bigger array, copy everything... painful!
    String[] newProducts = new String[products.length * 2];
    System.arraycopy(products, 0, newProducts, 0, products.length);
    products = newProducts;
    products[productCount++] = "New Product";
}

// Finding a product - must scan entire array
for (int i = 0; i < productCount; i++) {
    if (products[i].equals("Target Product")) {
        // Found it!
    }
}

// Removing from middle - must shift everything
// This is O(n) and error-prone
```

**Problems**:

1. **Fixed size**: Arrays can't grow dynamically
2. **No built-in operations**: Must implement add, remove, search yourself
3. **Wrong data structure**: Array isn't optimal for all use cases
4. **Error-prone**: Manual index management leads to bugs

### What Systems Looked Like Before Collections

Before Java 2 (1998), Java only had:

- `Vector`: Thread-safe but slow
- `Hashtable`: Thread-safe but slow
- `Stack`: Extends Vector (design mistake)
- Arrays: Fixed size, primitive operations

Every team built their own data structures, leading to:

- Incompatible implementations
- Bugs from reinventing the wheel
- Performance issues from wrong choices

### What Breaks Without Proper Collections

1. **Performance disasters**: O(n) operations where O(1) is possible
2. **Memory leaks**: Improper cleanup of data structures
3. **Thread safety issues**: Concurrent modification exceptions
4. **Code duplication**: Every project reimplements basic structures

### Real Examples of the Problem

**Twitter's Fail Whale (2008-2010)**: Twitter used Ruby's arrays for timelines. As users grew, O(n) operations killed performance. Switching to proper data structures (Redis sorted sets) was part of the solution.

**Amazon's Early Days**: Before proper caching with HashMaps, every product lookup hit the database. Adding in-memory HashMaps dramatically improved performance.

---

## 2️⃣ Intuition and Mental Model

### The Library Analogy

Think of collections like different ways to organize books in a library:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LIBRARY ORGANIZATION ANALOGY                          │
│                                                                          │
│   ArrayList = Numbered Shelves                                          │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┐                                    │
│   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  Books in order, find by position │
│   └───┴───┴───┴───┴───┴───┴───┴───┘  Fast: get book #5                 │
│                                       Slow: find book by title          │
│                                                                          │
│   LinkedList = Chain of Boxes                                           │
│   ┌───┐   ┌───┐   ┌───┐   ┌───┐                                        │
│   │ A │──►│ B │──►│ C │──►│ D │     Each box points to next            │
│   └───┘   └───┘   └───┘   └───┘     Fast: insert in middle             │
│                                      Slow: find book #5                 │
│                                                                          │
│   HashMap = Card Catalog                                                │
│   ┌─────────────┬─────────────────┐                                    │
│   │ "Moby Dick" │ Shelf 3, Slot 7 │  Look up by title instantly        │
│   │ "1984"      │ Shelf 1, Slot 2 │  Fast: find by title               │
│   │ "Dune"      │ Shelf 5, Slot 1 │  No order, just lookup             │
│   └─────────────┴─────────────────┘                                    │
│                                                                          │
│   TreeMap = Alphabetically Sorted Catalog                               │
│   ┌─────────────┬─────────────────┐                                    │
│   │ "1984"      │ Shelf 1, Slot 2 │  Sorted by title                   │
│   │ "Dune"      │ Shelf 5, Slot 1 │  Can find "all books starting D"   │
│   │ "Moby Dick" │ Shelf 3, Slot 7 │  Slightly slower than HashMap      │
│   └─────────────┴─────────────────┘                                    │
│                                                                          │
│   HashSet = "Do We Have This Book?" List                                │
│   ┌─────────────┐                                                       │
│   │ "Moby Dick" │  Just titles, no duplicates                          │
│   │ "1984"      │  Fast: "Do we have Dune?" → Yes/No                   │
│   │ "Dune"      │  No order, no duplicates                             │
│   └─────────────┘                                                       │
│                                                                          │
│   Queue = Line at Checkout                                              │
│   ┌───┐   ┌───┐   ┌───┐   ┌───┐                                        │
│   │ 1 │──►│ 2 │──►│ 3 │──►│ 4 │     First in, first out (FIFO)        │
│   └───┘   └───┘   └───┘   └───┘     Add at back, remove from front     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insight**: Choose the right data structure for your access pattern. Using ArrayList when you need HashMap is like searching every shelf for a book instead of using the card catalog.

---

## 3️⃣ How It Works Internally

### The Java Collections Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COLLECTIONS FRAMEWORK HIERARCHY                       │
│                                                                          │
│                           Iterable<E>                                   │
│                               │                                          │
│                          Collection<E>                                  │
│                    ┌──────────┼──────────┐                              │
│                    │          │          │                              │
│                 List<E>    Set<E>    Queue<E>                          │
│                    │          │          │                              │
│         ┌─────────┴─┐    ┌───┴───┐   ┌──┴───┐                         │
│         │           │    │       │   │      │                          │
│     ArrayList  LinkedList HashSet TreeSet  PriorityQueue              │
│                          │       │                                      │
│                    LinkedHashSet SortedSet                             │
│                                                                          │
│                           Map<K,V>  (separate hierarchy)                │
│                    ┌──────────┼──────────┐                              │
│                    │          │          │                              │
│                HashMap   TreeMap   LinkedHashMap                       │
│                    │                                                    │
│              ConcurrentHashMap                                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Interface vs Implementation

| Interface | Implementations                        | Use When                                |
| --------- | -------------------------------------- | --------------------------------------- |
| `List`    | ArrayList, LinkedList, Vector          | Ordered, allows duplicates              |
| `Set`     | HashSet, TreeSet, LinkedHashSet        | No duplicates                           |
| `Queue`   | LinkedList, PriorityQueue, ArrayDeque  | FIFO or priority processing             |
| `Deque`   | ArrayDeque, LinkedList                 | Double-ended queue                      |
| `Map`     | HashMap, TreeMap, LinkedHashMap        | Key-value pairs                         |

---

## ArrayList Deep Dive

### What It Is

ArrayList is a **resizable array**. Internally, it's just an array that grows automatically when needed.

### Internal Structure

```java
public class ArrayList<E> {
    // The actual array storing elements
    transient Object[] elementData;
    
    // Number of elements (not array length!)
    private int size;
    
    // Default initial capacity
    private static final int DEFAULT_CAPACITY = 10;
}
```

### How It Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ARRAYLIST INTERNAL STRUCTURE                          │
│                                                                          │
│   Initial state (capacity=10, size=0):                                  │
│   elementData: [null][null][null][null][null][null][null][null][null][null]
│   size: 0                                                                │
│                                                                          │
│   After adding "A", "B", "C" (capacity=10, size=3):                     │
│   elementData: ["A"]["B"]["C"][null][null][null][null][null][null][null]│
│   size: 3        ↑    ↑    ↑                                            │
│                  0    1    2                                             │
│                                                                          │
│   After adding 8 more elements (capacity=10, size=11):                  │
│   RESIZE TRIGGERED! New capacity = oldCapacity + (oldCapacity >> 1)     │
│   = 10 + 5 = 15                                                          │
│                                                                          │
│   elementData: ["A"]["B"]["C"]["D"]["E"]["F"]["G"]["H"]["I"]["J"]["K"][null][null][null][null]
│   size: 11                                                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Time Complexity

| Operation          | Time Complexity | Explanation                                     |
| ------------------ | --------------- | ----------------------------------------------- |
| `get(index)`       | O(1)            | Direct array access                             |
| `set(index, elem)` | O(1)            | Direct array access                             |
| `add(elem)`        | O(1) amortized  | Usually O(1), occasionally O(n) when resizing   |
| `add(index, elem)` | O(n)            | Must shift elements right                       |
| `remove(index)`    | O(n)            | Must shift elements left                        |
| `contains(elem)`   | O(n)            | Must scan entire list                           |
| `size()`           | O(1)            | Just return size field                          |

### Code Example with Explanation

```java
import java.util.ArrayList;
import java.util.List;

public class ArrayListDemo {
    
    public static void main(String[] args) {
        // Create with default capacity (10)
        List<String> names = new ArrayList<>();
        
        // Create with specific initial capacity
        // Use this when you know approximate size to avoid resizing
        List<String> products = new ArrayList<>(1000);
        
        // Add elements - O(1) amortized
        names.add("Alice");   // index 0
        names.add("Bob");     // index 1
        names.add("Charlie"); // index 2
        
        // Access by index - O(1)
        String first = names.get(0);  // "Alice"
        
        // Update by index - O(1)
        names.set(1, "Barbara");  // Replace "Bob" with "Barbara"
        
        // Insert at specific position - O(n)
        // All elements from index 1 must shift right
        names.add(1, "Betty");  // ["Alice", "Betty", "Barbara", "Charlie"]
        
        // Remove by index - O(n)
        // All elements after index must shift left
        names.remove(2);  // ["Alice", "Betty", "Charlie"]
        
        // Remove by value - O(n)
        // Must find element first, then shift
        names.remove("Betty");  // ["Alice", "Charlie"]
        
        // Check if contains - O(n)
        boolean hasAlice = names.contains("Alice");  // true
        
        // Find index - O(n)
        int index = names.indexOf("Charlie");  // 1
        
        // Iterate - O(n)
        for (String name : names) {
            System.out.println(name);
        }
        
        // Size - O(1)
        int size = names.size();  // 2
        
        // Clear - O(n) (sets all references to null for GC)
        names.clear();
    }
}
```

### When to Use ArrayList

✅ **Use ArrayList when**:

- You access elements by index frequently
- You mostly add to the end
- You iterate through all elements
- You need random access

❌ **Don't use ArrayList when**:

- You frequently insert/remove from the middle
- You frequently insert/remove from the beginning
- You need thread safety (use `CopyOnWriteArrayList` or synchronize)

---

## LinkedList Deep Dive

### What It Is

LinkedList is a **doubly-linked list**. Each element (node) contains the data plus references to the previous and next nodes.

### Internal Structure

```java
public class LinkedList<E> {
    transient int size = 0;
    transient Node<E> first;  // Head of the list
    transient Node<E> last;   // Tail of the list
    
    private static class Node<E> {
        E item;           // The actual data
        Node<E> next;     // Reference to next node
        Node<E> prev;     // Reference to previous node
        
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
}
```

### How It Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LINKEDLIST INTERNAL STRUCTURE                         │
│                                                                          │
│   After adding "A", "B", "C":                                           │
│                                                                          │
│   first                                                     last        │
│     │                                                         │         │
│     ▼                                                         ▼         │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐              │
│   │ prev: null  │     │ prev: ─────────┐  │ prev: ─────────┐           │
│   │ item: "A"   │◄────│ item: "B"   │  │  │ item: "C"   │  │           │
│   │ next: ──────────► │ next: ──────────►│ next: null  │  │           │
│   └─────────────┘     └─────────────┘     └─────────────┘              │
│                                                                          │
│   Inserting "X" between "A" and "B":                                    │
│   1. Create new node with item="X"                                      │
│   2. Set X.prev = A, X.next = B                                         │
│   3. Set A.next = X, B.prev = X                                         │
│   No shifting required! Just update 4 references.                       │
│                                                                          │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   │ item: "A"   │◄───►│ item: "X"   │◄───►│ item: "B"   │◄───►│ item: "C"   │
│   └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Time Complexity

| Operation          | Time Complexity | Explanation                                     |
| ------------------ | --------------- | ----------------------------------------------- |
| `get(index)`       | O(n)            | Must traverse from head or tail                 |
| `set(index, elem)` | O(n)            | Must traverse to find position                  |
| `add(elem)`        | O(1)            | Add at tail, just update references             |
| `addFirst(elem)`   | O(1)            | Add at head, just update references             |
| `add(index, elem)` | O(n)            | Must traverse to position, then O(1) insert     |
| `remove(index)`    | O(n)            | Must traverse to position, then O(1) remove     |
| `removeFirst()`    | O(1)            | Just update head reference                      |
| `removeLast()`     | O(1)            | Just update tail reference                      |
| `contains(elem)`   | O(n)            | Must traverse entire list                       |

### Code Example

```java
import java.util.LinkedList;
import java.util.Deque;

public class LinkedListDemo {
    
    public static void main(String[] args) {
        // LinkedList implements both List and Deque
        LinkedList<String> list = new LinkedList<>();
        
        // Add at end - O(1)
        list.add("B");
        list.addLast("C");  // Same as add()
        
        // Add at beginning - O(1)
        list.addFirst("A");  // ["A", "B", "C"]
        
        // Get first/last - O(1)
        String first = list.getFirst();  // "A"
        String last = list.getLast();    // "C"
        
        // Get by index - O(n)
        // LinkedList optimizes: if index < size/2, traverse from head
        // Otherwise, traverse from tail
        String middle = list.get(1);  // "B"
        
        // Remove first/last - O(1)
        list.removeFirst();  // ["B", "C"]
        list.removeLast();   // ["B"]
        
        // Use as Queue (FIFO)
        Deque<String> queue = new LinkedList<>();
        queue.offer("First");   // Add to tail
        queue.offer("Second");
        String next = queue.poll();  // Remove from head: "First"
        
        // Use as Stack (LIFO)
        Deque<String> stack = new LinkedList<>();
        stack.push("Bottom");  // Add to head
        stack.push("Top");
        String top = stack.pop();  // Remove from head: "Top"
    }
}
```

### ArrayList vs LinkedList

| Operation                | ArrayList | LinkedList | Winner       |
| ------------------------ | --------- | ---------- | ------------ |
| Get by index             | O(1)      | O(n)       | ArrayList    |
| Add at end               | O(1)*     | O(1)       | Tie          |
| Add at beginning         | O(n)      | O(1)       | LinkedList   |
| Add in middle            | O(n)      | O(n)**     | Depends      |
| Remove from end          | O(1)      | O(1)       | Tie          |
| Remove from beginning    | O(n)      | O(1)       | LinkedList   |
| Memory per element       | ~4 bytes  | ~24 bytes  | ArrayList    |
| Cache locality           | Excellent | Poor       | ArrayList    |

\* Amortized, occasional O(n) for resize
\** O(n) to find position, then O(1) to insert

**Practical advice**: Use ArrayList by default. LinkedList is rarely the right choice in modern Java due to poor cache locality.

---

## HashMap Deep Dive

### What It Is

HashMap is a **hash table** implementation. It stores key-value pairs and provides O(1) average-case lookup, insertion, and deletion.

### Internal Structure (Java 8+)

```java
public class HashMap<K,V> {
    // Array of buckets
    transient Node<K,V>[] table;
    
    // Number of key-value pairs
    transient int size;
    
    // Resize threshold = capacity * loadFactor
    int threshold;
    
    // When to resize (default 0.75)
    final float loadFactor;
    
    // Initial capacity (default 16)
    static final int DEFAULT_INITIAL_CAPACITY = 16;
    
    // When bucket becomes tree (Java 8+)
    static final int TREEIFY_THRESHOLD = 8;
    
    static class Node<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;  // For collision chaining
    }
}
```

### How Hashing Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HASHMAP INTERNAL STRUCTURE                            │
│                                                                          │
│   Step 1: Compute hash code                                             │
│   key.hashCode() → integer (e.g., "Alice".hashCode() = 92750483)       │
│                                                                          │
│   Step 2: Compute bucket index                                          │
│   index = hash & (capacity - 1)  // Faster than modulo                  │
│   92750483 & 15 = 3  // capacity=16, so indices 0-15                    │
│                                                                          │
│   Step 3: Store in bucket                                               │
│                                                                          │
│   table (capacity=16):                                                  │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐   │
│   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │12 │13 │14 │15 │   │
│   └───┴───┴───┴─┬─┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘   │
│                 │                                                        │
│                 ▼                                                        │
│           ┌─────────────┐                                               │
│           │ "Alice": 25 │  ← Entry stored at index 3                   │
│           │ next: null  │                                               │
│           └─────────────┘                                               │
│                                                                          │
│   COLLISION: Another key hashes to same bucket                          │
│   "Bob".hashCode() & 15 = 3  (same bucket!)                            │
│                                                                          │
│   table[3]:                                                             │
│           ┌─────────────┐     ┌─────────────┐                          │
│           │ "Alice": 25 │────►│ "Bob": 30   │  ← Chaining              │
│           │ next: ──────│     │ next: null  │                          │
│           └─────────────┘     └─────────────┘                          │
│                                                                          │
│   Java 8+: If chain length > 8, convert to Red-Black Tree              │
│   This improves worst-case from O(n) to O(log n)                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Load Factor and Resizing

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LOAD FACTOR AND RESIZING                              │
│                                                                          │
│   Load Factor = size / capacity                                         │
│   Default threshold = 0.75                                              │
│                                                                          │
│   Initial: capacity=16, threshold=12 (16 * 0.75)                        │
│                                                                          │
│   When size > threshold:                                                │
│   1. Create new array with 2x capacity (32)                             │
│   2. Rehash ALL entries to new positions                                │
│   3. New threshold = 24 (32 * 0.75)                                     │
│                                                                          │
│   Why 0.75?                                                             │
│   - Too low (0.5): Wastes memory, frequent resizing                    │
│   - Too high (0.9): More collisions, slower lookups                    │
│   - 0.75 is a good balance                                              │
│                                                                          │
│   Capacity always power of 2:                                           │
│   - Enables fast modulo: hash & (capacity-1)                           │
│   - 16, 32, 64, 128, 256...                                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Time Complexity

| Operation           | Average Case | Worst Case | Notes                              |
| ------------------- | ------------ | ---------- | ---------------------------------- |
| `get(key)`          | O(1)         | O(log n)*  | Worst case with tree buckets       |
| `put(key, value)`   | O(1)         | O(log n)*  | May trigger resize O(n)            |
| `remove(key)`       | O(1)         | O(log n)*  | Worst case with tree buckets       |
| `containsKey(key)`  | O(1)         | O(log n)*  | Same as get                        |
| `containsValue(v)`  | O(n)         | O(n)       | Must scan all values               |
| `keySet()`          | O(1)         | O(1)       | Returns view, not copy             |
| `values()`          | O(1)         | O(1)       | Returns view, not copy             |

\* Before Java 8, worst case was O(n) due to linked list buckets

### Code Example

```java
import java.util.HashMap;
import java.util.Map;

public class HashMapDemo {
    
    public static void main(String[] args) {
        // Create with default capacity (16) and load factor (0.75)
        Map<String, Integer> ages = new HashMap<>();
        
        // Create with specific initial capacity
        // Use when you know approximate size
        Map<String, Integer> largeMap = new HashMap<>(10000);
        
        // Put key-value pairs - O(1) average
        ages.put("Alice", 25);
        ages.put("Bob", 30);
        ages.put("Charlie", 35);
        
        // Get value by key - O(1) average
        Integer aliceAge = ages.get("Alice");  // 25
        Integer unknownAge = ages.get("Unknown");  // null
        
        // Get with default value - O(1)
        int age = ages.getOrDefault("Unknown", 0);  // 0
        
        // Check if key exists - O(1)
        boolean hasAlice = ages.containsKey("Alice");  // true
        
        // Check if value exists - O(n)!
        boolean has30 = ages.containsValue(30);  // true
        
        // Update value - O(1)
        ages.put("Alice", 26);  // Overwrites existing
        
        // Put if absent - O(1)
        ages.putIfAbsent("Alice", 99);  // Does nothing, Alice exists
        ages.putIfAbsent("David", 40);  // Adds David
        
        // Compute if absent (useful for caching)
        ages.computeIfAbsent("Eve", key -> {
            // Only called if key doesn't exist
            return key.length() * 10;  // 30
        });
        
        // Remove - O(1)
        ages.remove("Bob");
        ages.remove("Charlie", 35);  // Only removes if value matches
        
        // Iterate over entries
        for (Map.Entry<String, Integer> entry : ages.entrySet()) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }
        
        // Iterate over keys
        for (String name : ages.keySet()) {
            System.out.println(name);
        }
        
        // Iterate over values
        for (Integer a : ages.values()) {
            System.out.println(a);
        }
        
        // Size - O(1)
        int size = ages.size();
    }
}
```

### Important: hashCode() and equals() Contract

```java
// If you use custom objects as keys, you MUST override both!

public class User {
    private String id;
    private String name;
    
    // MUST override equals
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(id, user.id);
    }
    
    // MUST override hashCode
    // Rule: If a.equals(b), then a.hashCode() == b.hashCode()
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

// Why both?
// HashMap uses hashCode() to find the bucket
// Then uses equals() to find the exact key in the bucket
// If only one is overridden, HashMap breaks!
```

---

## TreeMap Deep Dive

### What It Is

TreeMap is a **Red-Black Tree** implementation of Map. It keeps keys sorted and provides O(log n) operations.

### When to Use TreeMap vs HashMap

| Feature                  | HashMap          | TreeMap          |
| ------------------------ | ---------------- | ---------------- |
| Order                    | No order         | Sorted by key    |
| Get/Put/Remove           | O(1) average     | O(log n)         |
| Null keys                | One allowed      | Not allowed      |
| Range queries            | Not supported    | Supported        |
| Memory                   | Less             | More (tree nodes)|
| Key requirement          | hashCode/equals  | Comparable       |

### Code Example

```java
import java.util.TreeMap;
import java.util.NavigableMap;

public class TreeMapDemo {
    
    public static void main(String[] args) {
        // Keys must be Comparable or provide Comparator
        NavigableMap<String, Integer> scores = new TreeMap<>();
        
        scores.put("Charlie", 85);
        scores.put("Alice", 92);
        scores.put("Bob", 78);
        scores.put("David", 88);
        
        // Iteration is in sorted order!
        for (String name : scores.keySet()) {
            System.out.println(name);  // Alice, Bob, Charlie, David
        }
        
        // Navigation methods
        String firstKey = scores.firstKey();  // "Alice"
        String lastKey = scores.lastKey();    // "David"
        
        // Floor/Ceiling (find nearest)
        String floor = scores.floorKey("Cat");    // "Charlie" (≤ "Cat")
        String ceiling = scores.ceilingKey("Cat"); // "David" (≥ "Cat")
        
        // Range views
        NavigableMap<String, Integer> subMap = scores.subMap(
            "Bob", true,    // From "Bob" (inclusive)
            "David", false  // To "David" (exclusive)
        );  // Contains Bob, Charlie
        
        // Head/Tail maps
        NavigableMap<String, Integer> beforeCharlie = scores.headMap("Charlie", false);
        NavigableMap<String, Integer> fromCharlie = scores.tailMap("Charlie", true);
        
        // Descending order
        NavigableMap<String, Integer> descending = scores.descendingMap();
    }
}
```

---

## HashSet Deep Dive

### What It Is

HashSet is a **Set backed by HashMap**. It stores unique elements with O(1) operations.

### Internal Structure

```java
public class HashSet<E> {
    // HashSet is literally a HashMap where values are ignored!
    private transient HashMap<E,Object> map;
    
    // Dummy value to associate with keys
    private static final Object PRESENT = new Object();
    
    public boolean add(E e) {
        return map.put(e, PRESENT) == null;
    }
    
    public boolean contains(Object o) {
        return map.containsKey(o);
    }
}
```

### Code Example

```java
import java.util.HashSet;
import java.util.Set;

public class HashSetDemo {
    
    public static void main(String[] args) {
        Set<String> uniqueNames = new HashSet<>();
        
        // Add elements - O(1)
        uniqueNames.add("Alice");
        uniqueNames.add("Bob");
        uniqueNames.add("Alice");  // Duplicate, not added
        
        System.out.println(uniqueNames.size());  // 2
        
        // Check membership - O(1)
        boolean hasAlice = uniqueNames.contains("Alice");  // true
        
        // Remove - O(1)
        uniqueNames.remove("Bob");
        
        // Set operations
        Set<String> otherSet = Set.of("Alice", "Charlie", "David");
        
        // Union
        Set<String> union = new HashSet<>(uniqueNames);
        union.addAll(otherSet);
        
        // Intersection
        Set<String> intersection = new HashSet<>(uniqueNames);
        intersection.retainAll(otherSet);
        
        // Difference
        Set<String> difference = new HashSet<>(uniqueNames);
        difference.removeAll(otherSet);
    }
}
```

---

## ConcurrentHashMap Deep Dive

### What It Is

ConcurrentHashMap is a **thread-safe HashMap** that allows concurrent reads and writes without locking the entire map.

### How It Achieves Thread Safety (Java 8+)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONCURRENTHASHMAP INTERNALS                           │
│                                                                          │
│   Java 7 and earlier: Segment-based locking                             │
│   ┌─────────┬─────────┬─────────┬─────────┐                            │
│   │Segment 0│Segment 1│Segment 2│Segment 3│  Each segment has its lock │
│   │ [lock]  │ [lock]  │ [lock]  │ [lock]  │  Concurrent access to      │
│   │ bucket  │ bucket  │ bucket  │ bucket  │  different segments        │
│   │ bucket  │ bucket  │ bucket  │ bucket  │                            │
│   └─────────┴─────────┴─────────┴─────────┘                            │
│                                                                          │
│   Java 8+: Node-level locking with CAS                                  │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┐                                    │
│   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  Lock only the bucket being       │
│   └─┬─┴───┴─┬─┴───┴───┴───┴───┴───┘  modified, not entire segment      │
│     │       │                                                           │
│     ▼       ▼                                                           │
│   [Node]  [Node]  ← synchronized on individual nodes                   │
│     │       │                                                           │
│   [Node]  [Node]                                                        │
│                                                                          │
│   Read operations: No locking (volatile reads)                          │
│   Write operations: CAS for empty buckets, synchronized for collisions │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Code Example

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

public class ConcurrentHashMapDemo {
    
    public static void main(String[] args) {
        ConcurrentHashMap<String, Integer> counters = new ConcurrentHashMap<>();
        
        // Thread-safe operations
        counters.put("pageViews", 0);
        
        // Atomic compute - thread-safe increment
        counters.compute("pageViews", (key, value) -> value + 1);
        
        // Atomic merge - thread-safe accumulation
        counters.merge("pageViews", 1, Integer::sum);
        
        // putIfAbsent - atomic check-and-put
        counters.putIfAbsent("uniqueVisitors", 0);
        
        // computeIfAbsent - atomic check-and-compute
        // Great for caching!
        counters.computeIfAbsent("newMetric", key -> expensiveComputation());
        
        // Bulk operations (parallel)
        counters.forEach(2, (key, value) -> {
            System.out.println(key + ": " + value);
        });
        
        // Search in parallel
        String found = counters.search(2, (key, value) -> 
            value > 100 ? key : null
        );
        
        // Reduce in parallel
        int total = counters.reduce(2, 
            (key, value) -> value,  // Transform
            Integer::sum            // Reduce
        );
    }
    
    private static int expensiveComputation() {
        return 42;
    }
}
```

### HashMap vs ConcurrentHashMap

| Feature                | HashMap                | ConcurrentHashMap       |
| ---------------------- | ---------------------- | ----------------------- |
| Thread safety          | Not thread-safe        | Thread-safe             |
| Null keys/values       | Allowed                | NOT allowed             |
| Performance (single)   | Faster                 | Slightly slower         |
| Performance (multi)    | Requires external sync | Excellent               |
| Iterator behavior      | Fail-fast              | Weakly consistent       |
| Atomic operations      | None                   | compute, merge, etc.    |

---

## 4️⃣ Simulation-First Explanation

Let's trace how a HashMap handles a series of operations.

### Scenario: Building a Word Counter

```java
Map<String, Integer> wordCount = new HashMap<>();
String[] words = {"the", "cat", "sat", "on", "the", "mat"};

for (String word : words) {
    wordCount.merge(word, 1, Integer::sum);
}
```

### Step-by-Step Execution

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HASHMAP WORD COUNTER SIMULATION                       │
│                                                                          │
│   Initial state: capacity=16, size=0, threshold=12                      │
│   table: [null][null][null][null]...[null] (16 nulls)                   │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing "the":                                                      │
│   1. hash("the") = 114801                                               │
│   2. index = 114801 & 15 = 1                                            │
│   3. table[1] is null, create new Node("the", 1)                        │
│   4. size = 1                                                            │
│                                                                          │
│   table: [null][the:1][null][null]...[null]                             │
│                  ↑                                                       │
│               index 1                                                    │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing "cat":                                                      │
│   1. hash("cat") = 98262                                                │
│   2. index = 98262 & 15 = 6                                             │
│   3. table[6] is null, create new Node("cat", 1)                        │
│   4. size = 2                                                            │
│                                                                          │
│   table: [null][the:1][null][null][null][null][cat:1][null]...          │
│                  ↑                              ↑                        │
│               index 1                       index 6                      │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing "sat":                                                      │
│   1. hash("sat") = 113652                                               │
│   2. index = 113652 & 15 = 4                                            │
│   3. table[4] is null, create new Node("sat", 1)                        │
│   4. size = 3                                                            │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing "on":                                                       │
│   1. hash("on") = 3551                                                  │
│   2. index = 3551 & 15 = 15                                             │
│   3. table[15] is null, create new Node("on", 1)                        │
│   4. size = 4                                                            │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing "the" (DUPLICATE):                                          │
│   1. hash("the") = 114801                                               │
│   2. index = 114801 & 15 = 1                                            │
│   3. table[1] is NOT null, check if key equals                          │
│   4. "the".equals("the") = true                                         │
│   5. Apply merge function: oldValue(1) + newValue(1) = 2                │
│   6. Update: Node("the", 2)                                             │
│   7. size stays 4 (no new entry)                                        │
│                                                                          │
│   table: [null][the:2][null][null][sat:1][null][cat:1]...[on:1]        │
│                  ↑                                                       │
│              UPDATED!                                                    │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing "mat":                                                      │
│   1. hash("mat") = 107881                                               │
│   2. index = 107881 & 15 = 9                                            │
│   3. table[9] is null, create new Node("mat", 1)                        │
│   4. size = 5                                                            │
│                                                                          │
│   Final state:                                                          │
│   {the=2, cat=1, sat=1, on=1, mat=1}                                    │
│   size=5, capacity=16 (no resize needed, 5 < 12)                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real Systems at Real Companies

**Netflix's EVCache**:

Netflix uses ConcurrentHashMap internally for local caching:

```java
// Simplified version of Netflix's caching pattern
public class LocalCache<K, V> {
    private final ConcurrentHashMap<K, CacheEntry<V>> cache = new ConcurrentHashMap<>();
    private final long ttlMillis;
    
    public V get(K key) {
        CacheEntry<V> entry = cache.get(key);
        if (entry == null || entry.isExpired()) {
            return null;
        }
        return entry.getValue();
    }
    
    public void put(K key, V value) {
        cache.put(key, new CacheEntry<>(value, System.currentTimeMillis() + ttlMillis));
    }
    
    // Atomic compute for cache-aside pattern
    public V getOrCompute(K key, Function<K, V> loader) {
        return cache.compute(key, (k, existing) -> {
            if (existing != null && !existing.isExpired()) {
                return existing;
            }
            V value = loader.apply(k);
            return new CacheEntry<>(value, System.currentTimeMillis() + ttlMillis);
        }).getValue();
    }
}
```

**Amazon's DynamoDB Client**:

Uses TreeMap for sorted key ranges:

```java
// Simplified key range handling
public class KeyRangeRouter {
    // TreeMap for efficient range lookups
    private final TreeMap<String, Partition> partitions = new TreeMap<>();
    
    public Partition getPartition(String key) {
        // Find the partition responsible for this key
        Map.Entry<String, Partition> entry = partitions.floorEntry(key);
        return entry != null ? entry.getValue() : partitions.firstEntry().getValue();
    }
    
    public List<Partition> getPartitionsInRange(String startKey, String endKey) {
        return new ArrayList<>(
            partitions.subMap(startKey, true, endKey, true).values()
        );
    }
}
```

### Real Workflows and Tooling

**Choosing the Right Collection**:

```java
// Decision tree for collection choice

// Need key-value pairs?
//   Yes → Need sorted keys?
//          Yes → TreeMap
//          No  → Need thread safety?
//                 Yes → ConcurrentHashMap
//                 No  → HashMap

// Need unique elements only?
//   Yes → Need sorted?
//          Yes → TreeSet
//          No  → HashSet

// Need ordered elements with duplicates?
//   Yes → Need frequent insert/remove at ends?
//          Yes → ArrayDeque or LinkedList
//          No  → Need random access?
//                 Yes → ArrayList
//                 No  → LinkedList

// Need FIFO processing?
//   Yes → ArrayDeque (faster) or LinkedList

// Need priority processing?
//   Yes → PriorityQueue
```

### Production War Stories

**The ArrayList.remove() Disaster**:

A team had code removing elements while iterating:

```java
// BUG: ConcurrentModificationException
List<Order> orders = getOrders();
for (Order order : orders) {
    if (order.isCancelled()) {
        orders.remove(order);  // BOOM!
    }
}

// FIX 1: Use Iterator
Iterator<Order> it = orders.iterator();
while (it.hasNext()) {
    if (it.next().isCancelled()) {
        it.remove();  // Safe!
    }
}

// FIX 2: Use removeIf (Java 8+)
orders.removeIf(Order::isCancelled);

// FIX 3: Create new list
List<Order> activeOrders = orders.stream()
    .filter(o -> !o.isCancelled())
    .collect(Collectors.toList());
```

**The HashMap Memory Leak**:

A caching system used objects as keys without proper equals/hashCode:

```java
// BUG: Memory leak
Map<User, Session> sessions = new HashMap<>();

// Each request creates new User object
User user = new User(userId);  // New object each time!
sessions.put(user, newSession);

// Old entries never found (different object identity)
// Map grows forever!

// FIX: Override equals and hashCode in User
// OR use userId (String) as key instead
Map<String, Session> sessions = new HashMap<>();
sessions.put(userId, newSession);
```

---

## 6️⃣ How to Implement: Complete Example

Let's build a complete LRU (Least Recently Used) Cache using collections.

```java
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * LRU Cache implementation using LinkedHashMap.
 * 
 * LinkedHashMap maintains insertion order (or access order with special flag).
 * When accessOrder=true, accessing an entry moves it to the end.
 * Override removeEldestEntry to auto-evict when capacity exceeded.
 */
public class LRUCache<K, V> {
    
    private final int capacity;
    private final Map<K, V> cache;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        
        // accessOrder=true: iteration order is access order (LRU)
        // loadFactor=0.75: standard load factor
        // capacity+1: slightly larger to avoid immediate resize
        this.cache = new LinkedHashMap<K, V>(capacity + 1, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                // Remove oldest entry when size exceeds capacity
                return size() > LRUCache.this.capacity;
            }
        };
    }
    
    /**
     * Get value from cache.
     * Accessing moves the entry to "most recently used" position.
     */
    public synchronized V get(K key) {
        return cache.get(key);
    }
    
    /**
     * Put value in cache.
     * If cache is full, least recently used entry is automatically removed.
     */
    public synchronized void put(K key, V value) {
        cache.put(key, value);
    }
    
    /**
     * Check if key exists without affecting LRU order.
     */
    public synchronized boolean containsKey(K key) {
        // Note: containsKey doesn't update access order in LinkedHashMap
        return cache.containsKey(key);
    }
    
    /**
     * Remove entry from cache.
     */
    public synchronized V remove(K key) {
        return cache.remove(key);
    }
    
    /**
     * Get current cache size.
     */
    public synchronized int size() {
        return cache.size();
    }
    
    /**
     * Clear all entries.
     */
    public synchronized void clear() {
        cache.clear();
    }
    
    @Override
    public synchronized String toString() {
        return cache.toString();
    }
}
```

### Usage Example

```java
public class LRUCacheDemo {
    
    public static void main(String[] args) {
        LRUCache<String, String> cache = new LRUCache<>(3);
        
        // Add entries
        cache.put("A", "Value A");
        cache.put("B", "Value B");
        cache.put("C", "Value C");
        System.out.println(cache);  // {A=Value A, B=Value B, C=Value C}
        
        // Access A (moves to end)
        cache.get("A");
        System.out.println(cache);  // {B=Value B, C=Value C, A=Value A}
        
        // Add D (evicts B, the least recently used)
        cache.put("D", "Value D");
        System.out.println(cache);  // {C=Value C, A=Value A, D=Value D}
        
        // B was evicted
        System.out.println(cache.get("B"));  // null
    }
}
```

### Thread-Safe Version with ConcurrentHashMap

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedDeque;

/**
 * Thread-safe LRU Cache using ConcurrentHashMap.
 * 
 * More complex but better for high-concurrency scenarios.
 */
public class ConcurrentLRUCache<K, V> {
    
    private final int capacity;
    private final ConcurrentHashMap<K, V> cache;
    private final ConcurrentLinkedDeque<K> accessOrder;
    
    public ConcurrentLRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new ConcurrentHashMap<>(capacity);
        this.accessOrder = new ConcurrentLinkedDeque<>();
    }
    
    public V get(K key) {
        V value = cache.get(key);
        if (value != null) {
            // Update access order
            accessOrder.remove(key);
            accessOrder.addLast(key);
        }
        return value;
    }
    
    public void put(K key, V value) {
        if (cache.containsKey(key)) {
            // Update existing
            cache.put(key, value);
            accessOrder.remove(key);
            accessOrder.addLast(key);
        } else {
            // Add new
            while (cache.size() >= capacity) {
                K oldest = accessOrder.pollFirst();
                if (oldest != null) {
                    cache.remove(oldest);
                }
            }
            cache.put(key, value);
            accessOrder.addLast(key);
        }
    }
    
    public int size() {
        return cache.size();
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

**1. Using Wrong Collection Type**

```java
// WRONG: Using ArrayList for frequent lookups
List<User> users = new ArrayList<>();
// O(n) lookup!
boolean exists = users.contains(new User("123"));

// RIGHT: Use HashSet for membership checks
Set<User> users = new HashSet<>();
// O(1) lookup!
boolean exists = users.contains(new User("123"));
```

**2. Not Specifying Initial Capacity**

```java
// WRONG: Default capacity, many resizes
Map<String, Integer> map = new HashMap<>();
for (int i = 0; i < 100000; i++) {
    map.put("key" + i, i);  // Multiple resizes!
}

// RIGHT: Specify initial capacity
Map<String, Integer> map = new HashMap<>(150000);  // capacity > expected size / 0.75
for (int i = 0; i < 100000; i++) {
    map.put("key" + i, i);  // No resizes!
}
```

**3. Modifying Collection While Iterating**

```java
// WRONG: ConcurrentModificationException
for (String item : list) {
    if (shouldRemove(item)) {
        list.remove(item);  // BOOM!
    }
}

// RIGHT: Use Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (shouldRemove(it.next())) {
        it.remove();  // Safe!
    }
}

// RIGHT: Use removeIf (Java 8+)
list.removeIf(this::shouldRemove);
```

**4. Forgetting hashCode/equals**

```java
// WRONG: Custom class as key without hashCode/equals
class Product {
    String id;
    String name;
    // No hashCode/equals!
}

Map<Product, Integer> stock = new HashMap<>();
stock.put(new Product("123", "Widget"), 100);
stock.get(new Product("123", "Widget"));  // Returns null! Different object

// RIGHT: Override both hashCode and equals
class Product {
    String id;
    String name;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Product)) return false;
        Product product = (Product) o;
        return Objects.equals(id, product.id);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

**5. Using HashMap in Multi-threaded Code**

```java
// WRONG: Race condition
Map<String, Integer> counters = new HashMap<>();

// Multiple threads doing this:
Integer count = counters.get(key);
counters.put(key, count + 1);  // Race condition!

// RIGHT: Use ConcurrentHashMap with atomic operations
ConcurrentHashMap<String, Integer> counters = new ConcurrentHashMap<>();
counters.merge(key, 1, Integer::sum);  // Atomic!
```

### Performance Pitfalls

| Pitfall                              | Impact              | Fix                                       |
| ------------------------------------ | ------------------- | ----------------------------------------- |
| ArrayList.add(0, elem)               | O(n) every time     | Use LinkedList or ArrayDeque              |
| HashMap without initial capacity     | Multiple resizes    | Specify capacity                          |
| LinkedList.get(index)                | O(n) every time     | Use ArrayList                             |
| TreeMap when order not needed        | O(log n) vs O(1)    | Use HashMap                               |
| Creating new HashSet for contains()  | O(n) construction   | Keep Set as field                         |

---

## 8️⃣ When NOT to Use Standard Collections

### Situations Requiring Specialized Collections

**1. Very Large Collections (Millions+)**

```java
// Standard collections may not be optimal for huge data

// Consider:
// - Trove (primitive collections, less memory)
// - Eclipse Collections (optimized implementations)
// - Chronicle Map (off-heap, persistent)
```

**2. Primitive Types**

```java
// WRONG: Boxing overhead
Map<Integer, Integer> map = new HashMap<>();  // Boxing every int!

// RIGHT: Use primitive collections
// Trove: TIntIntHashMap
// Eclipse Collections: IntIntHashMap
// Fastutil: Int2IntOpenHashMap
```

**3. Specialized Access Patterns**

```java
// Circular buffer: ArrayDeque
// Priority queue: PriorityQueue
// Bidirectional map: Guava's BiMap
// Multimap: Guava's Multimap
// Range map: Guava's RangeMap
```

---

## 9️⃣ Comparison: Choosing the Right Collection

### Quick Reference Chart

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COLLECTION SELECTION GUIDE                            │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    Need Key-Value Pairs?                         │   │
│   │                           │                                      │   │
│   │              ┌────────────┴────────────┐                        │   │
│   │              │                         │                        │   │
│   │             YES                        NO                       │   │
│   │              │                         │                        │   │
│   │    ┌─────────┴─────────┐      ┌───────┴───────┐                │   │
│   │    │ Need sorted keys? │      │ Allow dupes?  │                │   │
│   │    │                   │      │               │                │   │
│   │   YES                 NO     YES             NO                 │   │
│   │    │                   │      │               │                 │   │
│   │ TreeMap          ┌────┴────┐ │          ┌────┴────┐            │   │
│   │                  │ Thread  │ │          │ Sorted? │            │   │
│   │                  │ safe?   │ │          │         │            │   │
│   │                  │         │ │         YES       NO            │   │
│   │                 YES       NO │          │         │            │   │
│   │                  │         │ │       TreeSet  HashSet          │   │
│   │           Concurrent   HashMap                                  │   │
│   │           HashMap       │                                       │   │
│   │                    ┌────┴────┐                                  │   │
│   │                    │ Insert  │                                  │   │
│   │                    │ order?  │                                  │   │
│   │                   YES       NO                                  │   │
│   │                    │         │                                  │   │
│   │              LinkedHashMap  HashMap                             │   │
│   │                                                                 │   │
│   │         ┌─────────┴─────────┐                                  │   │
│   │         │  (For Lists)      │                                  │   │
│   │         │  Random access?   │                                  │   │
│   │         │                   │                                  │   │
│   │        YES                 NO                                  │   │
│   │         │                   │                                  │   │
│   │     ArrayList        ┌─────┴─────┐                             │   │
│   │                      │ Queue/    │                             │   │
│   │                      │ Stack?    │                             │   │
│   │                     YES         NO                             │   │
│   │                      │           │                             │   │
│   │                  ArrayDeque  LinkedList                        │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Time Complexity Summary

| Collection       | Add      | Remove   | Get/Contains | Notes                    |
| ---------------- | -------- | -------- | ------------ | ------------------------ |
| ArrayList        | O(1)*    | O(n)     | O(1)/O(n)    | *Amortized               |
| LinkedList       | O(1)     | O(1)**   | O(n)         | **If have reference      |
| HashSet          | O(1)     | O(1)     | O(1)         | Average case             |
| TreeSet          | O(log n) | O(log n) | O(log n)     | Sorted                   |
| HashMap          | O(1)     | O(1)     | O(1)         | Average case             |
| TreeMap          | O(log n) | O(log n) | O(log n)     | Sorted keys              |
| LinkedHashMap    | O(1)     | O(1)     | O(1)         | Insertion order          |
| PriorityQueue    | O(log n) | O(log n) | O(1) peek    | Heap-based               |
| ArrayDeque       | O(1)     | O(1)     | O(n)         | Double-ended             |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What's the difference between ArrayList and LinkedList?**

A: ArrayList uses a dynamic array internally. It's great for random access (O(1) to get element by index) and adding at the end. But inserting or removing from the middle is O(n) because elements must shift.

LinkedList uses a doubly-linked list. Each element points to the next and previous. Adding/removing at the ends is O(1), but getting element by index is O(n) because you must traverse from the head.

In practice, ArrayList is almost always better due to CPU cache locality. LinkedList's nodes are scattered in memory, causing cache misses. Use ArrayList by default.

**Q: How does HashMap work internally?**

A: HashMap uses an array of buckets. When you put a key-value pair:
1. Compute the key's hashCode()
2. Use the hash to find the bucket index: `hash & (capacity - 1)`
3. If the bucket is empty, store the entry there
4. If there's a collision (same bucket), chain entries in a linked list
5. In Java 8+, if the chain gets too long (>8), convert to a red-black tree

For get(), it's the reverse: compute hash, find bucket, search the chain/tree for matching key using equals().

The load factor (default 0.75) determines when to resize. When size exceeds capacity * loadFactor, the array doubles and all entries are rehashed.

### L5 (Mid-Level) Questions

**Q: Why must you override both hashCode() and equals() when using custom objects as HashMap keys?**

A: HashMap uses hashCode() to find the bucket and equals() to find the exact key within the bucket.

If you only override equals(): Two equal objects might have different hashCodes, landing in different buckets. You'll never find the key.

If you only override hashCode(): Two objects in the same bucket won't be recognized as equal, so you'll get duplicate entries.

The contract is: if `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must be true. The reverse isn't required (different objects can have the same hashCode).

**Q: When would you use TreeMap instead of HashMap?**

A: Use TreeMap when you need:
1. **Sorted keys**: TreeMap keeps keys in natural order (or custom Comparator order)
2. **Range queries**: `subMap(from, to)`, `headMap(to)`, `tailMap(from)`
3. **Navigation**: `firstKey()`, `lastKey()`, `floorKey()`, `ceilingKey()`

The tradeoff is performance: TreeMap is O(log n) for all operations vs HashMap's O(1) average. If you don't need sorting or range queries, HashMap is faster.

Example: Storing time-series data where you need to query "all entries between timestamp A and B" → TreeMap. Storing user sessions by ID → HashMap.

### L6 (Senior) Questions

**Q: How would you implement a thread-safe cache with expiration?**

A: I'd use ConcurrentHashMap with a wrapper that tracks expiration:

```java
public class ExpiringCache<K, V> {
    private final ConcurrentHashMap<K, Entry<V>> cache = new ConcurrentHashMap<>();
    private final long ttlMillis;
    
    record Entry<V>(V value, long expiresAt) {
        boolean isExpired() {
            return System.currentTimeMillis() > expiresAt;
        }
    }
    
    public V get(K key) {
        Entry<V> entry = cache.get(key);
        if (entry == null || entry.isExpired()) {
            cache.remove(key);  // Lazy cleanup
            return null;
        }
        return entry.value();
    }
    
    public void put(K key, V value) {
        cache.put(key, new Entry<>(value, System.currentTimeMillis() + ttlMillis));
    }
}
```

For production, I'd add:
1. Background cleanup thread or scheduled task
2. Size limits with LRU eviction
3. Metrics (hit rate, size, evictions)
4. Consider Caffeine library which handles all this

**Q: Explain the performance characteristics of ConcurrentHashMap vs Collections.synchronizedMap().**

A: `Collections.synchronizedMap()` wraps a HashMap with synchronized methods. Every operation locks the entire map. Only one thread can read or write at a time. Simple but terrible for high concurrency.

`ConcurrentHashMap` (Java 8+) uses fine-grained locking:
- Reads are lock-free using volatile reads
- Writes lock only the affected bucket, not the entire map
- Multiple threads can write to different buckets simultaneously
- Uses CAS (Compare-And-Swap) for empty buckets

In benchmarks, ConcurrentHashMap can be 10-100x faster than synchronizedMap under high concurrency. The only downside is no null keys/values allowed.

For read-heavy workloads, ConcurrentHashMap is nearly as fast as unsynchronized HashMap. For write-heavy workloads, it's still much faster than full synchronization.

---

## 1️⃣1️⃣ One Clean Mental Summary

Java Collections are like different ways to organize a library. ArrayList is numbered shelves (fast random access), LinkedList is a chain of boxes (fast insert/remove at ends), HashMap is a card catalog (instant lookup by key), and TreeMap is an alphabetically sorted catalog (ordered iteration, range queries). Choose based on your access pattern: random access → ArrayList, unique elements → HashSet, key-value with fast lookup → HashMap, sorted keys → TreeMap, thread-safe → ConcurrentHashMap. Always override hashCode() and equals() for custom keys, specify initial capacity for large collections, and never modify a collection while iterating over it. When in doubt, start with ArrayList and HashMap. They're right 90% of the time.

