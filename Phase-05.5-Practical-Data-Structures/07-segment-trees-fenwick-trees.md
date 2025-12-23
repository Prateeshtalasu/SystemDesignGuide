# 🎯 Segment Trees & Fenwick Trees

## 0️⃣ Prerequisites

Before diving into Segment Trees and Fenwick Trees, you need to understand:

### Arrays and Indexing
An **array** is a contiguous block of memory storing elements accessible by index.

```java
int[] arr = {1, 3, 5, 7, 9, 11};
// Index:    0  1  2  3  4   5
arr[2];  // Returns 5
```

### Range Queries
A **range query** asks for aggregate information about a contiguous portion of an array.

```java
// Array: [1, 3, 5, 7, 9, 11]
// Range sum query: sum of elements from index 1 to 4
// Answer: 3 + 5 + 7 + 9 = 24

// Range min query: minimum element from index 2 to 5
// Answer: min(5, 7, 9, 11) = 5
```

### Binary Trees
A **binary tree** is a tree where each node has at most two children (left and right).

```
        1
       / \
      2   3
     / \
    4   5
```

### Binary Representation
Numbers can be represented in binary (base 2). Fenwick Trees rely on binary properties.

```
Decimal 12 = Binary 1100
Decimal 10 = Binary 1010
Decimal 7  = Binary 0111
```

### Time Complexity
- **O(1)**: Constant time
- **O(log n)**: Logarithmic time
- **O(n)**: Linear time

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Core Problem: Efficient Range Queries with Updates

Imagine you're building a stock trading dashboard that shows:
- Current price of each stock
- Sum of prices in any range (for portfolio value)
- Minimum price in any range (for alerts)

**The naive approaches:**

**Approach 1: Compute on demand**
```java
int rangeSum(int[] arr, int left, int right) {
    int sum = 0;
    for (int i = left; i <= right; i++) {
        sum += arr[i];
    }
    return sum;
}
// Time: O(n) per query
// With 1 million queries per second, this is too slow!
```

**Approach 2: Precompute prefix sums**
```java
// Prefix sum array: prefixSum[i] = sum of arr[0..i-1]
int[] prefixSum = new int[n + 1];
for (int i = 0; i < n; i++) {
    prefixSum[i + 1] = prefixSum[i] + arr[i];
}

int rangeSum(int left, int right) {
    return prefixSum[right + 1] - prefixSum[left];
}
// Query: O(1) - Great!

void update(int index, int newValue) {
    // Must recompute all prefix sums from index onwards
    // Time: O(n) - Terrible for frequent updates!
}
```

### The Dilemma

| Approach | Query Time | Update Time |
|----------|------------|-------------|
| Naive | O(n) | O(1) |
| Prefix Sum | O(1) | O(n) |
| **Segment Tree** | **O(log n)** | **O(log n)** |
| **Fenwick Tree** | **O(log n)** | **O(log n)** |

### Real-World Pain Points

**Scenario 1: Real-Time Analytics Dashboard**
- 10 million data points
- 100,000 range queries per second
- Data updates every millisecond

**Scenario 2: Game Leaderboards**
- Find sum of scores in rank range [100, 200]
- Update player scores in real-time
- Need both operations to be fast

**Scenario 3: Database Query Optimization**
- Range sum queries on indexed columns
- Frequent inserts and updates
- B-trees handle this, but segment trees explain the concept

---

## 2️⃣ Intuition and Mental Model

### Segment Tree: The Tournament Bracket Analogy

Imagine a tennis tournament where you want to find the "best" player in any range.

**Players (leaves):**
```
[Alice, Bob, Carol, David, Eve, Frank, Grace, Henry]
   0     1     2      3     4     5      6      7
```

**Tournament structure:**
```
                    [Best of 0-7]
                   /             \
          [Best of 0-3]       [Best of 4-7]
           /       \           /       \
    [Best 0-1]  [Best 2-3]  [Best 4-5]  [Best 6-7]
      /   \       /   \       /   \       /   \
    [0]  [1]    [2]  [3]    [4]  [5]    [6]  [7]
```

**To find best in range [2, 5]:**
1. We need [Best 2-3] (already computed!)
2. We need [Best 4-5] (already computed!)
3. Compare them → Answer!

We didn't check all 4 players individually. We used precomputed results.

### Fenwick Tree: The Cumulative Responsibility Analogy

Imagine a company hierarchy where each manager is responsible for a specific range of employees.

**Employees numbered 1-8:**
```
Employee:  1    2    3    4    5    6    7    8
Salary:   [5]  [3]  [7]  [2]  [4]  [6]  [1]  [8]
```

**Managers responsible for cumulative sums:**
```
Manager 1: responsible for employee 1 only
Manager 2: responsible for employees 1-2
Manager 3: responsible for employee 3 only
Manager 4: responsible for employees 1-4
Manager 5: responsible for employee 5 only
Manager 6: responsible for employees 5-6
Manager 7: responsible for employee 7 only
Manager 8: responsible for employees 1-8
```

**To find sum of salaries 1-6:**
- Ask Manager 4 (knows 1-4): 5+3+7+2 = 17
- Ask Manager 6 (knows 5-6): 4+6 = 10
- Total: 27

Only 2 managers consulted, not 6 employees!

```
┌─────────────────────────────────────────────────────────────────┐
│                      KEY INSIGHT                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Both structures achieve O(log n) by:                           │
│                                                                  │
│  Segment Tree: Dividing the array into hierarchical segments    │
│                (like a binary tree over ranges)                 │
│                                                                  │
│  Fenwick Tree: Using binary representation to determine         │
│                which prefix sums each index is responsible for  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ Segment Tree: How It Works Internally

### Structure

A segment tree is a binary tree where:
- **Leaves**: Store individual array elements
- **Internal nodes**: Store aggregate of their children's ranges

For an array of size n:
- Tree has 2n - 1 nodes (n leaves + n-1 internal nodes)
- Height is ⌈log₂(n)⌉

### Building the Tree

```
Array: [1, 3, 5, 7, 9, 11, 13, 15]
       (indices 0-7)

Segment Tree (sum):

                        [64]              (sum of 0-7)
                       /    \
                   [16]      [48]         (sum of 0-3, 4-7)
                  /    \    /    \
                [4]   [12] [20]  [28]     (sum of 0-1, 2-3, 4-5, 6-7)
               / \    / \   / \   / \
              [1][3] [5][7][9][11][13][15] (individual elements)
```

### Array Representation

We store the tree in an array (like a heap):
- Root at index 1
- Left child of node i at index 2i
- Right child of node i at index 2i + 1
- Parent of node i at index i/2

```
Index:  1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
Value: [64][16][48][4][12][20][28][1] [3] [5] [7] [9][11][13][15]
        │   │   │   │   │   │   │   └──────────────────────────┘
        │   │   │   └───┴───┴───┴─ internal nodes              leaves
        └───┴───┴─ root and children
```

### Build Operation

```java
void build(int[] arr, int node, int start, int end) {
    if (start == end) {
        // Leaf node
        tree[node] = arr[start];
    } else {
        int mid = (start + end) / 2;
        // Build left child
        build(arr, 2 * node, start, mid);
        // Build right child
        build(arr, 2 * node + 1, mid + 1, end);
        // Internal node = aggregate of children
        tree[node] = tree[2 * node] + tree[2 * node + 1];
    }
}
```

### Query Operation

To query range [L, R]:

```
Query [2, 5] on array [1, 3, 5, 7, 9, 11, 13, 15]:

                        [64] (0-7)
                       /    \
                   [16]      [48] (4-7)
                  (0-3)     /    \
                  /    \  [20]   [28]
                [4]   [12](4-5)  (6-7)
               (0-1)  (2-3)
                      ↑
                      Need this!
                      
Step 1: At root (0-7), query (2-5)
        Range (0-7) partially overlaps (2-5)
        Go to both children

Step 2: At left child (0-3), query (2-5)
        Range (0-3) partially overlaps (2-5)
        Go to both children

Step 3: At (0-1), query (2-5)
        Range (0-1) is completely outside (2-5)
        Return 0

Step 4: At (2-3), query (2-5)
        Range (2-3) is completely inside (2-5)
        Return 12 (don't go deeper!)

Step 5: At right child (4-7), query (2-5)
        Range (4-7) partially overlaps (2-5)
        Go to both children

Step 6: At (4-5), query (2-5)
        Range (4-5) is completely inside (2-5)
        Return 20

Step 7: At (6-7), query (2-5)
        Range (6-7) is completely outside (2-5)
        Return 0

Result: 0 + 12 + 20 + 0 = 32
Verify: arr[2] + arr[3] + arr[4] + arr[5] = 5 + 7 + 9 + 11 = 32 ✓
```

### Update Operation

To update index i to new value:

```
Update index 3 to value 10 (was 7):

                        [64]
                       /    \
                   [16]      [48]
                  /    \    
                [4]   [12]  
                      / \   
                    [5] [7] ← Change to [10]

After update:
                        [67] (+3)
                       /    \
                   [19]      [48]
                  /    \    
                [4]   [15]  
                      / \   
                    [5] [10]

Only nodes on path from leaf to root are updated: O(log n)
```

---

## 4️⃣ Fenwick Tree (Binary Indexed Tree): How It Works

### The Key Insight: Binary Representation

Fenwick Tree uses a clever observation about binary numbers:

**Every positive integer can be represented as a sum of powers of 2.**

```
13 = 8 + 4 + 1 = 2³ + 2² + 2⁰
Binary: 1101
```

**The lowest set bit (LSB)** determines the range each index is responsible for:

```
Index (decimal) | Binary | LSB | Responsible for range
----------------|--------|-----|---------------------
1               | 0001   | 1   | [1, 1] (1 element)
2               | 0010   | 2   | [1, 2] (2 elements)
3               | 0011   | 1   | [3, 3] (1 element)
4               | 0100   | 4   | [1, 4] (4 elements)
5               | 0101   | 1   | [5, 5] (1 element)
6               | 0110   | 2   | [5, 6] (2 elements)
7               | 0111   | 1   | [7, 7] (1 element)
8               | 1000   | 8   | [1, 8] (8 elements)
```

### Structure

```
Array:     [_, 3, 2, 5, 1, 7, 4, 6, 8]  (1-indexed, ignore index 0)
            0  1  2  3  4  5  6  7  8

Fenwick:   [_, 3, 5, 5, 11, 7, 11, 6, 36]
            0  1  2  3   4  5   6  7   8

fenwick[1] = arr[1] = 3                    (range [1,1])
fenwick[2] = arr[1] + arr[2] = 3 + 2 = 5   (range [1,2])
fenwick[3] = arr[3] = 5                    (range [3,3])
fenwick[4] = arr[1..4] = 3+2+5+1 = 11      (range [1,4])
fenwick[5] = arr[5] = 7                    (range [5,5])
fenwick[6] = arr[5] + arr[6] = 7 + 4 = 11  (range [5,6])
fenwick[7] = arr[7] = 6                    (range [7,7])
fenwick[8] = arr[1..8] = 36                (range [1,8])
```

### Getting the Lowest Set Bit

```java
int lowbit(int x) {
    return x & (-x);
}

// Examples:
// lowbit(6)  = lowbit(0110) = 2 (0010)
// lowbit(12) = lowbit(1100) = 4 (0100)
// lowbit(7)  = lowbit(0111) = 1 (0001)
```

### Query Operation (Prefix Sum)

To get sum of [1, i], we add values while removing the lowest set bit:

```
Query prefix sum [1, 7]:

i = 7 (binary: 0111)
  sum += fenwick[7] = 6
  i = 7 - lowbit(7) = 7 - 1 = 6

i = 6 (binary: 0110)
  sum += fenwick[6] = 11
  i = 6 - lowbit(6) = 6 - 2 = 4

i = 4 (binary: 0100)
  sum += fenwick[4] = 11
  i = 4 - lowbit(4) = 4 - 4 = 0

i = 0, stop

Total: 6 + 11 + 11 = 28

Verify: arr[1..7] = 3 + 2 + 5 + 1 + 7 + 4 + 6 = 28 ✓
```

### Update Operation

To update index i, we add the delta while adding the lowest set bit:

```
Update index 3: add delta = 5

i = 3 (binary: 0011)
  fenwick[3] += 5
  i = 3 + lowbit(3) = 3 + 1 = 4

i = 4 (binary: 0100)
  fenwick[4] += 5
  i = 4 + lowbit(4) = 4 + 4 = 8

i = 8 (binary: 1000)
  fenwick[8] += 5
  i = 8 + lowbit(8) = 8 + 8 = 16

i = 16 > n, stop

Updated indices: 3, 4, 8 (all indices that include index 3 in their range)
```

### Range Sum Query

To get sum of [L, R]:
```java
int rangeSum(int L, int R) {
    return prefixSum(R) - prefixSum(L - 1);
}
```

---

## 5️⃣ Simulation: Complete Walkthrough

### Segment Tree Example

**Array:** [2, 1, 5, 3, 4, 2]

**Build:**
```
                    [17] (0-5)
                   /    \
              [8] (0-2)  [9] (3-5)
             /    \      /    \
        [3](0-1) [5](2) [7](3-4) [2](5)
        /   \           /   \
      [2]  [1]        [3]  [4]
      (0)  (1)        (3)  (4)
```

**Query sum [1, 4]:**
```
At (0-5): partially overlaps [1,4], go both ways
  At (0-2): partially overlaps [1,4]
    At (0-1): partially overlaps [1,4]
      At (0): outside [1,4] → return 0
      At (1): inside [1,4] → return 1
    At (2): inside [1,4] → return 5
  At (3-5): partially overlaps [1,4]
    At (3-4): inside [1,4] → return 7
    At (5): outside [1,4] → return 0

Total: 0 + 1 + 5 + 7 + 0 = 13
Verify: 1 + 5 + 3 + 4 = 13 ✓
```

**Update index 2 to 8 (was 5, delta = +3):**
```
Path: (2) → (0-2) → (0-5)

Before: [5] → [8] → [17]
After:  [8] → [11] → [20]
```

### Fenwick Tree Example

**Array:** [_, 2, 1, 5, 3, 4, 2] (1-indexed)

**Build Fenwick:**
```
Index:    1    2    3    4    5    6
Array:    2    1    5    3    4    2
Fenwick:  2    3    5   11    4    6

fenwick[1] = arr[1] = 2
fenwick[2] = arr[1] + arr[2] = 3
fenwick[3] = arr[3] = 5
fenwick[4] = arr[1..4] = 11
fenwick[5] = arr[5] = 4
fenwick[6] = arr[5] + arr[6] = 6
```

**Query prefix sum [1, 5]:**
```
i = 5 (binary: 101)
  sum += fenwick[5] = 4
  i = 5 - lowbit(5) = 5 - 1 = 4

i = 4 (binary: 100)
  sum += fenwick[4] = 11
  i = 4 - lowbit(4) = 4 - 4 = 0

Total: 4 + 11 = 15
Verify: 2 + 1 + 5 + 3 + 4 = 15 ✓
```

**Query range sum [2, 5]:**
```
prefixSum(5) - prefixSum(1) = 15 - 2 = 13
Verify: 1 + 5 + 3 + 4 = 13 ✓
```

---

## 6️⃣ Implementation in Java

### Segment Tree Implementation

```java
/**
 * Segment Tree for range sum queries and point updates.
 * Can be modified for min, max, GCD, etc.
 */
public class SegmentTree {
    
    private final int[] tree;
    private final int n;
    
    /**
     * Builds a segment tree from the input array.
     * Time: O(n)
     * Space: O(n)
     */
    public SegmentTree(int[] arr) {
        this.n = arr.length;
        // Tree size: 4n is safe upper bound
        this.tree = new int[4 * n];
        build(arr, 1, 0, n - 1);
    }
    
    /**
     * Recursively builds the tree.
     * 
     * @param node Current node index in tree array
     * @param start Start of current segment
     * @param end End of current segment
     */
    private void build(int[] arr, int node, int start, int end) {
        if (start == end) {
            // Leaf node
            tree[node] = arr[start];
        } else {
            int mid = (start + end) / 2;
            int leftChild = 2 * node;
            int rightChild = 2 * node + 1;
            
            build(arr, leftChild, start, mid);
            build(arr, rightChild, mid + 1, end);
            
            // Internal node stores sum of children
            tree[node] = tree[leftChild] + tree[rightChild];
        }
    }
    
    /**
     * Updates a single element.
     * Time: O(log n)
     */
    public void update(int index, int value) {
        update(1, 0, n - 1, index, value);
    }
    
    private void update(int node, int start, int end, int index, int value) {
        if (start == end) {
            // Leaf node
            tree[node] = value;
        } else {
            int mid = (start + end) / 2;
            if (index <= mid) {
                update(2 * node, start, mid, index, value);
            } else {
                update(2 * node + 1, mid + 1, end, index, value);
            }
            // Update internal node
            tree[node] = tree[2 * node] + tree[2 * node + 1];
        }
    }
    
    /**
     * Queries the sum of range [left, right].
     * Time: O(log n)
     */
    public int query(int left, int right) {
        return query(1, 0, n - 1, left, right);
    }
    
    private int query(int node, int start, int end, int left, int right) {
        // Case 1: Current segment is completely outside query range
        if (right < start || left > end) {
            return 0;  // Identity for sum
        }
        
        // Case 2: Current segment is completely inside query range
        if (left <= start && end <= right) {
            return tree[node];
        }
        
        // Case 3: Partial overlap, query both children
        int mid = (start + end) / 2;
        int leftSum = query(2 * node, start, mid, left, right);
        int rightSum = query(2 * node + 1, mid + 1, end, left, right);
        
        return leftSum + rightSum;
    }
}
```

### Segment Tree for Range Minimum Query (RMQ)

```java
/**
 * Segment Tree for Range Minimum Query.
 */
public class RMQSegmentTree {
    
    private final int[] tree;
    private final int n;
    private static final int INF = Integer.MAX_VALUE;
    
    public RMQSegmentTree(int[] arr) {
        this.n = arr.length;
        this.tree = new int[4 * n];
        Arrays.fill(tree, INF);
        build(arr, 1, 0, n - 1);
    }
    
    private void build(int[] arr, int node, int start, int end) {
        if (start == end) {
            tree[node] = arr[start];
        } else {
            int mid = (start + end) / 2;
            build(arr, 2 * node, start, mid);
            build(arr, 2 * node + 1, mid + 1, end);
            tree[node] = Math.min(tree[2 * node], tree[2 * node + 1]);
        }
    }
    
    public void update(int index, int value) {
        update(1, 0, n - 1, index, value);
    }
    
    private void update(int node, int start, int end, int index, int value) {
        if (start == end) {
            tree[node] = value;
        } else {
            int mid = (start + end) / 2;
            if (index <= mid) {
                update(2 * node, start, mid, index, value);
            } else {
                update(2 * node + 1, mid + 1, end, index, value);
            }
            tree[node] = Math.min(tree[2 * node], tree[2 * node + 1]);
        }
    }
    
    public int queryMin(int left, int right) {
        return queryMin(1, 0, n - 1, left, right);
    }
    
    private int queryMin(int node, int start, int end, int left, int right) {
        if (right < start || left > end) {
            return INF;  // Identity for min
        }
        if (left <= start && end <= right) {
            return tree[node];
        }
        int mid = (start + end) / 2;
        return Math.min(
            queryMin(2 * node, start, mid, left, right),
            queryMin(2 * node + 1, mid + 1, end, left, right)
        );
    }
}
```

### Fenwick Tree Implementation

```java
/**
 * Fenwick Tree (Binary Indexed Tree) for prefix sums and point updates.
 * Uses 1-based indexing internally.
 */
public class FenwickTree {
    
    private final int[] tree;
    private final int n;
    
    /**
     * Creates a Fenwick Tree of given size.
     * Time: O(n)
     */
    public FenwickTree(int n) {
        this.n = n;
        this.tree = new int[n + 1];  // 1-indexed
    }
    
    /**
     * Creates a Fenwick Tree from an array.
     * Time: O(n)
     */
    public FenwickTree(int[] arr) {
        this.n = arr.length;
        this.tree = new int[n + 1];
        
        // Build in O(n) using the property that each index
        // contributes to indices i + lowbit(i)
        for (int i = 0; i < n; i++) {
            add(i, arr[i]);
        }
    }
    
    /**
     * Returns the lowest set bit of x.
     * Example: lowbit(12) = lowbit(1100) = 4 (0100)
     */
    private int lowbit(int x) {
        return x & (-x);
    }
    
    /**
     * Adds delta to element at index (0-based).
     * Time: O(log n)
     */
    public void add(int index, int delta) {
        // Convert to 1-based
        index++;
        while (index <= n) {
            tree[index] += delta;
            index += lowbit(index);
        }
    }
    
    /**
     * Sets element at index to value (0-based).
     * Time: O(log n)
     */
    public void set(int index, int value) {
        int current = query(index, index);
        add(index, value - current);
    }
    
    /**
     * Returns sum of elements [0, index] (0-based, inclusive).
     * Time: O(log n)
     */
    public int prefixSum(int index) {
        // Convert to 1-based
        index++;
        int sum = 0;
        while (index > 0) {
            sum += tree[index];
            index -= lowbit(index);
        }
        return sum;
    }
    
    /**
     * Returns sum of elements [left, right] (0-based, inclusive).
     * Time: O(log n)
     */
    public int query(int left, int right) {
        if (left == 0) {
            return prefixSum(right);
        }
        return prefixSum(right) - prefixSum(left - 1);
    }
}
```

### Testing Both Implementations

```java
public class TreeTest {
    
    public static void main(String[] args) {
        int[] arr = {1, 3, 5, 7, 9, 11};
        
        System.out.println("=== Segment Tree ===");
        SegmentTree segTree = new SegmentTree(arr);
        
        System.out.println("Sum [1,4]: " + segTree.query(1, 4));  // 3+5+7+9 = 24
        System.out.println("Sum [0,5]: " + segTree.query(0, 5));  // 36
        
        segTree.update(2, 10);  // Change 5 to 10
        System.out.println("After update, Sum [1,4]: " + segTree.query(1, 4));  // 3+10+7+9 = 29
        
        System.out.println("\n=== Fenwick Tree ===");
        FenwickTree fenwick = new FenwickTree(arr);
        
        System.out.println("Sum [1,4]: " + fenwick.query(1, 4));  // 24
        System.out.println("Sum [0,5]: " + fenwick.query(0, 5));  // 36
        
        fenwick.set(2, 10);  // Change 5 to 10
        System.out.println("After update, Sum [1,4]: " + fenwick.query(1, 4));  // 29
        
        System.out.println("\n=== Performance Comparison ===");
        int n = 100000;
        int[] large = new int[n];
        Random rand = new Random();
        for (int i = 0; i < n; i++) large[i] = rand.nextInt(1000);
        
        long start = System.currentTimeMillis();
        SegmentTree st = new SegmentTree(large);
        for (int i = 0; i < 100000; i++) {
            st.query(rand.nextInt(n/2), n/2 + rand.nextInt(n/2));
        }
        System.out.println("Segment Tree: " + (System.currentTimeMillis() - start) + "ms");
        
        start = System.currentTimeMillis();
        FenwickTree ft = new FenwickTree(large);
        for (int i = 0; i < 100000; i++) {
            ft.query(rand.nextInt(n/2), n/2 + rand.nextInt(n/2));
        }
        System.out.println("Fenwick Tree: " + (System.currentTimeMillis() - start) + "ms");
    }
}
```

### Segment Tree with Lazy Propagation

```java
/**
 * Segment Tree with Lazy Propagation for range updates.
 * Supports: range add, range sum query.
 */
public class LazySegmentTree {
    
    private final long[] tree;
    private final long[] lazy;
    private final int n;
    
    public LazySegmentTree(int[] arr) {
        this.n = arr.length;
        this.tree = new long[4 * n];
        this.lazy = new long[4 * n];
        build(arr, 1, 0, n - 1);
    }
    
    private void build(int[] arr, int node, int start, int end) {
        if (start == end) {
            tree[node] = arr[start];
        } else {
            int mid = (start + end) / 2;
            build(arr, 2 * node, start, mid);
            build(arr, 2 * node + 1, mid + 1, end);
            tree[node] = tree[2 * node] + tree[2 * node + 1];
        }
    }
    
    /**
     * Pushes lazy value down to children.
     */
    private void pushDown(int node, int start, int end) {
        if (lazy[node] != 0) {
            int mid = (start + end) / 2;
            int leftChild = 2 * node;
            int rightChild = 2 * node + 1;
            
            // Update children's tree values
            tree[leftChild] += lazy[node] * (mid - start + 1);
            tree[rightChild] += lazy[node] * (end - mid);
            
            // Pass lazy value to children
            lazy[leftChild] += lazy[node];
            lazy[rightChild] += lazy[node];
            
            // Clear current lazy value
            lazy[node] = 0;
        }
    }
    
    /**
     * Adds value to all elements in range [left, right].
     * Time: O(log n)
     */
    public void rangeAdd(int left, int right, int value) {
        rangeAdd(1, 0, n - 1, left, right, value);
    }
    
    private void rangeAdd(int node, int start, int end, int left, int right, int value) {
        if (right < start || left > end) {
            return;
        }
        
        if (left <= start && end <= right) {
            // Current segment is completely inside update range
            tree[node] += (long) value * (end - start + 1);
            lazy[node] += value;
            return;
        }
        
        // Partial overlap, push down and recurse
        pushDown(node, start, end);
        
        int mid = (start + end) / 2;
        rangeAdd(2 * node, start, mid, left, right, value);
        rangeAdd(2 * node + 1, mid + 1, end, left, right, value);
        
        tree[node] = tree[2 * node] + tree[2 * node + 1];
    }
    
    /**
     * Queries sum of range [left, right].
     * Time: O(log n)
     */
    public long query(int left, int right) {
        return query(1, 0, n - 1, left, right);
    }
    
    private long query(int node, int start, int end, int left, int right) {
        if (right < start || left > end) {
            return 0;
        }
        
        if (left <= start && end <= right) {
            return tree[node];
        }
        
        pushDown(node, start, end);
        
        int mid = (start + end) / 2;
        return query(2 * node, start, mid, left, right) +
               query(2 * node + 1, mid + 1, end, left, right);
    }
}
```

---

## 7️⃣ Comparison: Segment Tree vs Fenwick Tree

| Aspect | Segment Tree | Fenwick Tree |
|--------|--------------|--------------|
| Space | 4n | n |
| Build time | O(n) | O(n) |
| Point update | O(log n) | O(log n) |
| Range query | O(log n) | O(log n) |
| Range update | O(log n) with lazy | Complex |
| Implementation | More complex | Simpler |
| Flexibility | Any associative operation | Mainly prefix sums |
| Cache efficiency | Worse | Better |

### When to Use Which

**Use Segment Tree when:**
- You need range updates (add X to all elements in [L, R])
- You need operations other than sum (min, max, GCD)
- You need to store additional information at nodes

**Use Fenwick Tree when:**
- You only need point updates and range sums
- Memory is a concern
- You want simpler code
- Performance is critical (better cache locality)

---

## 8️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Pitfalls

**1. Off-by-one errors in indexing**

```java
// BAD: Mixing 0-based and 1-based indexing
int sum = fenwick.query(left, right);  // Is this 0-based or 1-based?

// GOOD: Be consistent and document clearly
// This implementation uses 0-based indexing externally
```

**2. Integer overflow**

```java
// BAD: Sum can overflow int
int[] tree = new int[4 * n];

// GOOD: Use long for sums
long[] tree = new long[4 * n];
```

**3. Forgetting lazy propagation**

```java
// BAD: Query without pushing down lazy values
private long query(int node, int start, int end, int left, int right) {
    // Missing: pushDown(node, start, end);
    ...
}
```

**4. Wrong tree size**

```java
// BAD: Tree too small
int[] tree = new int[2 * n];  // Not enough for all cases!

// GOOD: Use 4n to be safe
int[] tree = new int[4 * n];
```

---

## 9️⃣ When NOT to Use These Structures

### Anti-Patterns

**1. No updates needed**
```java
// OVERKILL: If array never changes, use prefix sum array
// Query: O(1) instead of O(log n)
int[] prefix = new int[n + 1];
for (int i = 0; i < n; i++) prefix[i+1] = prefix[i] + arr[i];
int rangeSum = prefix[right+1] - prefix[left];
```

**2. Single element queries**
```java
// OVERKILL: Just use the array directly
int value = arr[index];  // O(1)
```

**3. Small arrays**
```java
// OVERKILL: For n < 1000, naive O(n) is fast enough
// The overhead of tree structures isn't worth it
```

---

## 🔟 Interview Follow-Up Questions with Answers

### L4 (Entry-Level) Questions

**Q1: What is a Segment Tree and when would you use it?**

**Answer**: A Segment Tree is a binary tree data structure that stores information about array segments. Each leaf stores an array element, and each internal node stores the aggregate (sum, min, max, etc.) of its children's ranges. It's used when you need both range queries and point updates in O(log n) time. Common use cases include range sum queries, range minimum queries, and finding the count of elements in a range.

**Q2: What's the difference between Segment Tree and Fenwick Tree?**

**Answer**: Both achieve O(log n) for queries and updates, but they differ in:
- **Space**: Fenwick uses n elements, Segment Tree uses 4n
- **Flexibility**: Segment Tree supports any associative operation; Fenwick is mainly for prefix sums
- **Range updates**: Segment Tree handles them naturally with lazy propagation; Fenwick requires tricks
- **Implementation**: Fenwick is simpler (about 10 lines of code)
- **Performance**: Fenwick has better cache locality

Choose Fenwick for simple sum queries, Segment Tree for complex operations.

**Q3: How does a Fenwick Tree achieve O(log n) operations?**

**Answer**: Fenwick Tree uses the binary representation of indices. Each index i is responsible for a range of size equal to its lowest set bit. For example, index 6 (binary 110) is responsible for 2 elements (indices 5-6). To query prefix sum, we add values while removing the lowest set bit, which takes at most log(n) steps. To update, we add delta while adding the lowest set bit. The key insight is that each element contributes to only O(log n) indices.

### L5 (Senior) Questions

**Q4: Explain lazy propagation in Segment Trees.**

**Answer**: Lazy propagation is an optimization for range updates. Instead of updating all affected nodes immediately, we store a "lazy" value at each node representing a pending update.

When we need to update range [L, R]:
1. If current segment is completely inside [L, R], update the node and store lazy value
2. Don't recurse into children yet

When we query or update later:
1. Before accessing children, "push down" the lazy value
2. Apply the lazy value to children and clear it from current node

This makes range updates O(log n) instead of O(n).

**Q5: How would you modify a Segment Tree to find the k-th smallest element in a range?**

**Answer**: Use a "merge sort tree" or "persistent segment tree":

```java
// Each node stores a sorted list of elements in its range
class Node {
    List<Integer> sorted;
}

// To find k-th smallest in [L, R]:
// 1. Binary search on the answer value X
// 2. For each X, count how many elements in [L, R] are <= X
// 3. Use segment tree to count in O(log² n)
// 4. Binary search finds k-th smallest in O(log n × log² n)

// Alternative: Use wavelet tree for O(log n) per query
```

### L6 (Staff) Questions

**Q6: Design a real-time analytics system using Segment Trees.**

**Answer**:

```
Use Case: Real-time dashboard showing metrics over time windows

Architecture:
1. Time-based Segment Tree:
   - Each leaf = 1 minute bucket
   - Internal nodes = aggregated metrics
   - Support: sum, min, max, percentiles

2. Multiple Trees:
   - One tree per metric (latency, throughput, errors)
   - One tree per dimension (per service, per region)

3. Rolling Window:
   - Circular buffer of trees
   - Each tree covers 1 hour
   - 24 trees for 24-hour history

4. Query Examples:
   - "Sum of requests in last 30 minutes" → O(log n)
   - "P99 latency in last hour" → O(log n) with augmented tree

5. Updates:
   - Metrics arrive via Kafka
   - Update appropriate leaf node
   - Propagate up the tree

6. Scaling:
   - Shard by metric/dimension
   - Each shard has its own tree
   - Aggregate across shards for global queries
```

---

## 1️⃣1️⃣ One Clean Mental Summary

Segment Trees and Fenwick Trees solve the problem of efficient range queries with updates. A Segment Tree is a binary tree where each node stores aggregate information about a range of the array, enabling O(log n) queries and updates by dividing the problem hierarchically. A Fenwick Tree (Binary Indexed Tree) achieves the same complexity using a clever encoding based on binary representation, where each index is responsible for a range determined by its lowest set bit. Use Segment Trees when you need flexibility (min, max, lazy propagation), and Fenwick Trees when you need simplicity and better performance for sum queries.

---

## Summary

Segment Trees and Fenwick Trees are essential for:
- **Range sum/min/max queries**: Analytics, leaderboards
- **Point updates with range queries**: Real-time dashboards
- **Competitive programming**: Many problems require these
- **Database internals**: Understanding query optimization

Key takeaways:
1. Both achieve O(log n) query and update
2. Segment Tree: More flexible, supports lazy propagation
3. Fenwick Tree: Simpler, less memory, better cache locality
4. Use prefix sums if no updates needed
5. Lazy propagation enables O(log n) range updates

