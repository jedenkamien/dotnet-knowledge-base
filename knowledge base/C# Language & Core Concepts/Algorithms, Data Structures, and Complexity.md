# Algorithms, Data Structures, and Complexity

## Purpose

Algorithms and data structures are primarily about choosing the right trade-off between:

- Execution time
- Memory usage
- Code complexity

In interviews, the most important skill is usually not remembering obscure algorithms, but being able to:

1. Analyze complexity.
2. Choose appropriate data structures.
3. Explain trade-offs.

---

# Complexity Analysis

## Big-O Notation

Big-O describes how runtime or memory usage grows as input size `n` increases.

Common complexities:

```text
O(1)      Constant
O(log n)  Logarithmic
O(n)      Linear
O(n log n)
O(n²)
O(2ⁿ)
O(n!)
```

Performance ranking:

```text
O(1)
↓
O(log n)
↓
O(n)
↓
O(n log n)
↓
O(n²)
↓
O(2ⁿ)
↓
O(n!)
```

---

## Time vs Space Complexity

Algorithms often trade memory for speed.

Example:

- Duplicate detection with `HashSet<T>`
  - Time: O(n)
  - Space: O(n)

Without additional memory:

- Time: O(n²)
- Space: O(1)

Interview question:

> Can you make it faster?
>
> Can you make it use less memory?

Usually improving one worsens the other.

---

## Common Complexity Traps

### Two consecutive loops

```csharp
for (...)
for (...)
```

Complexity:

```text
O(n)
```

not O(n²).

Reason:

```text
n + n = 2n
```

Constants are ignored.

### Nested loops

```csharp
for (...)
    for (...)
```

Usually:

```text
O(n²)
```

### Halving the problem

```csharp
while (n > 1)
    n /= 2;
```

Complexity:

```text
O(log n)
```

---

# Arrays and Lists

## Array

Characteristics:

- Contiguous memory
- Excellent cache locality
- Fast indexing
- Expensive insertions/removals in the middle

| Operation | Complexity |
|------------|------------|
| Index access | O(1) |
| Search | O(n) |
| Insert middle | O(n) |
| Remove middle | O(n) |

---

## List<T>

Internally implemented as a dynamic array.

### Add()

Normally:

```text
O(1)
```

but occasionally:

1. New larger array allocated
2. Existing items copied
3. New item added

That resize operation is:

```text
O(n)
```

Therefore:

```text
List<T>.Add() = O(1) amortized
```

### Insert()

```csharp
list.Insert(0, value);
```

Complexity:

```text
O(n)
```

because all elements must be shifted.

---

# Amortized Complexity

Amortized complexity means:

> The average cost of an operation when occasional expensive operations are spread across many cheap operations.

For `List<T>.Add()`:

```text
Most adds = O(1)
Rare resize = O(n)
Average over many calls ≈ O(1)
```

Interview point:

Amortized complexity is neither best case nor worst case. It is the average cost across repeated operations.

---

# Linked Lists

Each node stores:

- Value
- Reference to next node
- Optionally previous node

Advantages:

- Cheap insertion/removal when node is already known

Disadvantages:

- No efficient indexing
- Additional memory for references
- Poor cache locality

| Operation | Complexity |
|------------|------------|
| Search | O(n) |
| Insert known node | O(1) |
| Remove known node | O(1) |

Important interview insight:

Linked lists are not automatically faster than arrays. Modern CPUs often favor arrays because contiguous memory greatly improves cache performance.

---

# Stack

LIFO (Last In, First Out)

Operations:

```csharp
Push()
Pop()
Peek()
```

Complexity:

```text
O(1)
```

Common uses:

- Function call stack
- Undo functionality
- Depth-First Search (DFS)

---

# Queue

FIFO (First In, First Out)

Operations:

```csharp
Enqueue()
Dequeue()
Peek()
```

Complexity:

```text
O(1)
```

Common uses:

- Message processing
- Scheduling
- Breadth-First Search (BFS)

---

# Hash Tables

.NET implementations:

```csharp
Dictionary<TKey, TValue>
HashSet<T>
```

Average complexity:

| Operation | Complexity |
|------------|------------|
| Insert | O(1) |
| Lookup | O(1) |
| Remove | O(1) |

### Collisions

Multiple keys can land in the same bucket.

Worst-case complexity becomes:

```text
O(n)
```

Important interview point:

Dictionary lookup is O(1) on average, not guaranteed O(1).

---

# Trees

## What Is a Tree?

A hierarchical structure where nodes may have children.

Examples:

- Folder structures
- HTML DOM
- XML
- JSON
- Organization charts
- Database indexes

Unlike arrays, trees represent relationships naturally.

---

# Binary Search Tree (BST)

Property:

```text
Left subtree  < Node
Right subtree > Node
```

Example:

```text
        10
       /  \
      5   15
     / \  / \
    2  8 12 20
```

---

## What Problem Does BST Solve?

Efficient searching while preserving sorted order.

Balanced BST:

```text
Search  O(log n)
Insert  O(log n)
Remove  O(log n)
```

A search repeatedly removes half of the remaining possibilities.

Example:

```text
Search 12:

10 -> right
15 -> left
12 found
```

---

## Degenerate BST

Insertion order may produce:

```text
1
 \
  2
   \
    3
     \
      4
```

Now the tree behaves like a linked list.

Complexity degrades to:

```text
O(n)
```

Interview point:

BST is only O(log n) when sufficiently balanced.

---

## BST vs Dictionary

Dictionary:

- Faster lookups (average O(1))
- No ordering

BST:

- Keeps values sorted
- Supports range queries
- Supports "next larger" / "next smaller" operations

Examples:

```text
Find values between 100 and 200
Find smallest value
Find largest value
```

These are natural tree operations.

---

# Binary Search

Requirement:

```text
Data must be sorted.
```

Process:

1. Check middle element.
2. Discard half of remaining values.
3. Repeat.

Complexity:

```text
O(log n)
```

Classic interview trap:

Binary search cannot be applied efficiently to unsorted data.

---

# Sorting Algorithms

## QuickSort

Process:

1. Choose pivot.
2. Move smaller items left.
3. Move larger items right.
4. Recursively sort partitions.

Average complexity:

```text
O(n log n)
```

Reason:

- Partition work per level: O(n)
- Number of levels: log n

Worst case:

```text
O(n²)
```

Occurs when partitions are extremely unbalanced.

Example:

```text
1 2 3 4 5
```

with poor pivot selection.

Strengths:

- Excellent average performance
- Very cache-friendly
- Widely used in practice

---

## MergeSort

Process:

1. Repeatedly divide collection in half.
2. Sort smaller pieces.
3. Merge them back together.

Complexity:

```text
O(n log n)
```

Guaranteed.

Additional memory:

```text
O(n)
```

because temporary storage is required during merging.

Strength:

Predictable performance.

---

## HeapSort

Uses a heap (priority queue).

Repeatedly:

```text
Extract largest (or smallest) element
```

Complexity:

```text
O(n log n)
```

Reason:

- Extraction = O(log n)
- Performed n times

Additional memory:

```text
O(1)
```

---

# Heap

A tree-like structure optimized for retrieving the highest- or lowest-priority element.

.NET:

```csharp
PriorityQueue<TElement, TPriority>
```

Complexities:

| Operation | Complexity |
|------------|------------|
| Peek | O(1) |
| Insert | O(log n) |
| Extract | O(log n) |

Typical use cases:

- Scheduling
- Priority processing
- Dijkstra's algorithm
- Top-K problems

---

# Recursion

A function calling itself.

Common uses:

- Tree traversal
- DFS
- Divide-and-conquer algorithms

Important consideration:

Every recursive call consumes stack space.

Possible problem:

```text
StackOverflowException
```

for very deep recursion.

Complexity analysis must consider:

- Execution time
- Additional stack memory

---

# Graph Traversal

## Depth-First Search (DFS)

Idea:

> Follow one path as deeply as possible before backtracking.

Typical implementation:

- Stack
- Recursion

Traversal example:

```text
A B D E C F G
```

Characteristics:

- Goes deep first
- Usually lower memory consumption
- Good for exhaustive exploration

---

## Breadth-First Search (BFS)

Idea:

> Visit all neighbors before moving deeper.

Typical implementation:

- Queue

Traversal example:

```text
A B C D E F G
```

Characteristics:

- Level-by-level traversal
- Explores outward uniformly

Key interview fact:

BFS finds the shortest path in an unweighted graph.

---

# Important Interview Facts to Remember

1. `List<T>.Add()` is O(1) amortized, not strictly O(1).
2. `Dictionary<TKey,TValue>` is O(1) on average, O(n) in worst case.
3. A BST can degrade to O(n) if unbalanced.
4. Binary search requires sorted data.
5. Linked lists often perform worse in practice than arrays due to cache locality.
6. DFS typically uses a stack; BFS uses a queue.
7. BFS finds shortest paths in unweighted graphs.
8. QuickSort is usually fast because partitions are typically balanced.
9. MergeSort guarantees O(n log n) but requires O(n) extra memory.
10. Understanding trade-offs is usually more important than memorizing algorithms.