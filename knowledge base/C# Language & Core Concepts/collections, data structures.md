# Data Structures (Interview-Oriented .NET Notes)

## Overview

A strong interview answer should cover:

- When to use a structure
- Typical operation complexities
- Basic internal implementation
- Trade-offs and edge cases
- Real-world .NET usage

---

# Arrays (`T[]`)

## Characteristics

- Fixed size after creation
- Contiguous memory layout
- Fastest random access structure
- Cache-friendly

## Complexity

- Access by index: **O(1)**
- Search (unsorted): **O(n)**
- Insert/Delete in middle: **O(n)**

## Why is indexing O(1)?

The address can be calculated directly:

```
address = baseAddress + index * elementSize
```

No traversal is required.

## Interview Notes

- Arrays are the foundation of many other collections.
- Large arrays (> ~85 KB) are typically allocated on the Large Object Heap (LOH).

---

# List<T>

## What it is

A dynamically-sized array.

Internally:

```csharp
private T[] _items;
```

## Complexity

- Index access: **O(1)**
- Add at end: **O(1) amortized**
- Insert/Delete in middle: **O(n)**
- Search: **O(n)**

## Amortized O(1)

When capacity is exceeded:

1. Larger array is allocated
2. Existing elements are copied
3. Old array is discarded

Resizing is infrequent, so average insertion cost remains O(1).

## Capacity vs Count

```csharp
list.Count
list.Capacity
```

- Count = actual elements
- Capacity = allocated storage

Pre-sizing can reduce reallocations.

## Common Pitfall

```csharp
foreach (var item in list)
{
    list.Add(item);
}
```

Throws:

```
InvalidOperationException
```

because the collection is modified during enumeration.

---

# HashSet<T>

## Purpose

Store unique values and perform fast membership checks.

## Complexity

- Add: **O(1) average**
- Remove: **O(1) average**
- Contains: **O(1) average**

## Internals

Implemented using a hash table.

```
Value
  ↓
GetHashCode()
  ↓
Bucket
```

## Hash Collisions

Multiple values may map to the same bucket:

```
A -> bucket 5
B -> bucket 5
C -> bucket 5
```

Too many collisions can degrade performance toward O(n).

## Important Interview Topic

For custom classes, `Equals()` and `GetHashCode()` must be implemented consistently.

Otherwise logically identical objects may be treated as different values.

---

# Dictionary<TKey, TValue>

## Purpose

Fast lookup by key.

## Complexity

- Lookup: **O(1) average**
- Insert: **O(1) average**
- Remove: **O(1) average**

## Internals

Also based on a hash table.

```
Key
 ↓
HashCode
 ↓
Bucket
 ↓
Stored Entry
```

## Best Practice

Prefer:

```csharp
dictionary.TryGetValue(key, out var value);
```

over:

```csharp
if (dictionary.ContainsKey(key))
{
    value = dictionary[key];
}
```

The latter performs two lookups.

## Pitfall

Avoid mutable keys.

Changing data that affects the hash code after insertion can break lookups.

---

# Stack<T>

## Principle

LIFO (Last In, First Out)

```
3 ← top
2
1
```

## Operations

```csharp
Push()
Pop()
Peek()
```

## Complexity

All major operations: **O(1)**

## Common Uses

- DFS
- Undo/Redo
- Expression evaluation
- Recursion support

## Interview Fact

Method calls in .NET use the **call stack**.

---

# Queue<T>

## Principle

FIFO (First In, First Out)

```
1 2 3
↑
front
```

## Operations

```csharp
Enqueue()
Dequeue()
Peek()
```

## Complexity

All major operations: **O(1)**

## Common Uses

- Task scheduling
- Producer/Consumer scenarios
- Message processing
- BFS

---

# LinkedList<T>

## Structure

.NET implements a doubly-linked list.

```
A ⇄ B ⇄ C ⇄ D
```

Each node contains:

- Value
- Next
- Previous

## Complexity

- Access by position: **O(n)**
- Search: **O(n)**
- Insert after known node: **O(1)**
- Remove known node: **O(1)**

## Common Interview Trap

"LinkedList insertions are O(1)" is only true if you already have the node reference.

```csharp
linkedList.AddAfter(node, value);
```

If you must first search for the node:

```
Search O(n)
Insert O(1)
Total O(n)
```

## Practical Reality

`List<T>` is often faster despite O(n) insertions because:

- Better cache locality
- Fewer allocations
- Contiguous memory

---

# Trees

## Basic Terminology

- Root
- Parent
- Child
- Leaf
- Depth
- Height

## Depth vs Height

Example:

```
      A
     / \
    B   C
       /
      D
```

Depth:

```
A = 0
B = 1
C = 1
D = 2
```

Height:

```
B = 0
D = 0
C = 1
A = 2
```

### Rule

- Depth = distance from root to node
- Height = distance from node to deepest descendant leaf

The tree height is the height of the root.

---

# Binary Search Tree (BST)

## Ordering Rule

```
Left < Parent < Right
```

Example:

```
      10
     /  \
    5   15
```

## Complexity (Balanced BST)

- Search: **O(log n)**
- Insert: **O(log n)**
- Delete: **O(log n)**

## Why Can BST Become O(n)?

The ordering rule does not guarantee balance.

This is still a valid BST:

```
1
 \
  2
   \
    3
     \
      4
```

Every value is greater than its parent, so the BST rule is satisfied.

However it behaves like a linked list, making search O(n).

## Key Interview Distinction

- BST property = correct ordering
- Balance = good performance

These are separate concerns.

---

# Red-Black Trees

A self-balancing Binary Search Tree.

Purpose:

- Keep height roughly balanced
- Preserve O(log n) operations

Used internally by:

- `SortedDictionary<TKey,TValue>`
- `SortedSet<T>`

---

# B-Trees and B+ Trees

## B-Tree

Balanced tree where nodes can have many children.

Optimized for:

- Databases
- File systems
- Disk/page access

## B+ Tree

Variant of B-Tree where:

- Actual data is stored only in leaf nodes
- Leaf nodes are linked together

Particularly efficient for:

- Range queries
- Database indexes

## Interview Shortcut

- Red-Black Tree → in-memory sorted collections
- B-Tree / B+ Tree → database and filesystem indexes

---

# Real-World Uses of Trees

- Database indexes (B-Trees/B+ Trees)
- `SortedDictionary` and `SortedSet`
- File systems
- Browser DOM
- Compiler syntax trees (ASTs)

---

# Graphs

## Definition

A graph consists of:

- Vertices (nodes)
- Edges (connections)

Unlike trees:

- Cycles are allowed
- Multiple paths may exist

Example:

```
A --- B
|     |
|     |
C --- D
```

## Types

### Directed Graph

```
A → B
```

### Undirected Graph

```
A — B
```

### Weighted Graph

```
A --5--> B
```

Weights may represent:

- Cost
- Time
- Distance

---

# DFS (Depth-First Search)

Explores as deeply as possible before backtracking.

Uses:

- Stack
- Or recursion (call stack)

Example order:

```
A, B, D, E, C
```

## Why Stack?

The most recently discovered node is processed next.

LIFO behavior naturally matches DFS.

---

# BFS (Breadth-First Search)

Explores level by level.

Example order:

```
A, B, C, D, E
```

## Why Queue?

The earliest discovered node is processed next.

FIFO behavior naturally matches BFS.

## Interview Fact

BFS finds the shortest path in an **unweighted graph**.

---

# Complexity Cheat Sheet

| Structure | Lookup | Insert | Delete |
|------------|----------|----------|----------|
| Array | O(1) index | O(n) | O(n) |
| List<T> | O(1) index | O(1)* end | O(n) |
| HashSet<T> | O(1) avg | O(1) avg | O(1) avg |
| Dictionary<TKey,TValue> | O(1) avg | O(1) avg | O(1) avg |
| Stack<T> | O(1) | O(1) | O(1) |
| Queue<T> | O(1) | O(1) | O(1) |
| LinkedList<T> | O(n) | O(1)** | O(1)** |
| Balanced BST | O(log n) | O(log n) | O(log n) |

\* amortized  
\** if the node reference is already known

---

# High-Value Interview Questions

1. Why is `List<T>.Add()` amortized O(1)?
2. How does a hash table work?
3. What causes hash collisions?
4. Why must `Equals()` and `GetHashCode()` be consistent?
5. HashSet vs Dictionary?
6. When would you choose HashSet over List?
7. Why is LinkedList rarely used?
8. Why can a BST degrade to O(n)?
9. Why does DFS use a stack?
10. Why does BFS use a queue?
11. Difference between tree height and node depth?
12. When are B-Trees/B+ Trees preferred over BSTs?