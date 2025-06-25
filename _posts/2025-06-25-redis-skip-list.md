---
layout: post
title: "How Redis Achieves O(1) Operations — And When It Doesn't"
date: 2025-06-25
tags: redis, performance, data-structures
categories: datastore
---

Redis is renowned for its blazing-fast performance, often delivering sub-millisecond responses. A common claim is that many Redis operations run in **O(1)** time. But what exactly does that mean—and when does it not hold true?

Let’s break it down.

---

## ⛷️ What Is a Skip List?

A Skip List is a layered, ordered data structure built on top of a linked list that allows O(log n) time complexity for:

- Search
- Insert
- Delete

It achieves this by maintaining multiple levels of forward pointers that "skip" over elements.

### Skip List Basics

- Think of it as a multi-level linked list.
- The bottom layer is a standard sorted linked list.
- Each higher layer acts as an “express lane” that skips over multiple nodes.
- On average, the number of layers is `O(log n)`.

### 🔗 Structure Overview

- The bottom level is a fully linked list of all elements.
- Each higher level contains a subset of the elements.
- Level-k can skip over elements present in lower levels.

```
Level 3:            ──────────────> F ───────────>
Level 2:        ─────> B ──────────> F ───────────>
Level 1:    A ─────> B ─────> D ─────> F ─────> H ─>
Level 0: A → B → C → D → E → F → G → H → NULL
```

### 🎲 How Are These Levels Formed?

- When inserting a new node, you randomly assign it a level L.
- Each level is added with probability p (commonly p = 0.5).
- This creates a geometric distribution of levels:
  - ~50% of nodes appear in level 1
  - ~25% in level 2
  - ~12.5% in level 3
  - ... and so on

So only a few nodes appear in the top levels.

### 🔍 How Search Works in O(log n)

To search for a key:

1. Start from the highest level of the head node.
2. Move forward until you reach a node whose next value is greater than the target.
3. Drop down a level and continue.
4. Repeat until you reach level 0.

Each drop cuts the remaining search space approximately in half, just like binary search.

This gives an expected search time of O(log n).

### 📈 Mathematical Intuition

If:
- The probability of a node appearing at level i is p^i (e.g., 0.5, 0.25, 0.125...)
- The expected number of nodes at level i is n * p^i
- The maximum level is about log₁/p(n)

Then:
- The search path passes through at most O(log n) levels
- At each level, you move forward a constant number of steps (expected)

So total operations = O(log n) on average

### ✅ Why Insert/Delete Are Also O(log n)

- Both operations start with a search (O(log n))
- Once found, you:
  - Insert new forward pointers up to its random level
  - Or update forward pointers to remove a node

These adjustments are done in parallel across O(log n) levels → so also O(log n).

### ⏱️ Time & Space Complexity

| Operation | Time Complexity   | Explanation                     |
| --------- | ----------------- | ------------------------------- |
| Search    | `O(log n)`        | Because it skips many elements  |
| Insert    | `O(log n)`        | Random level assignment         |
| Delete    | `O(log n)`        | Same as search + pointer update |
| Space     | `O(n)` (expected) | Multiple pointers per node      |

### ✅ When to Use Skip Lists

- You need sorted access (e.g., leaderboards, price ranges)
- You want logarithmic performance with simpler implementation
- You don’t need the complexity of trees (rotations, rebalancing)
- You’re working in systems where predictable insert/delete latency is more important than worst-case guarantees

## 📦 Why Redis Uses Skip Lists

- In Redis, skip lists are used in:
  - Sorted Sets (ZSET)
  - Skip list maintains score order
  - Hash map provides O(1) lookups
- Provides:
  - Fast range queries (e.g., top 10, score between A and B)
  -  Efficient insert/delete in sorted order

**Redis Choice: Skip List vs. Binary Tree**

| Skip List                                    | Binary Search Tree (e.g., AVL/Red-Black) |
| -------------------------------------------- | ---------------------------------------- |
| Simple pointer manipulation                  | Needs rotations                          |
| Fast and easy to implement                   | More complex balancing logic             |
| Probabilistic but fast enough                | Deterministic guarantees                 |
| Redis is single-threaded → simpler is better | Tree might be overkill                   |

### ⚡ What is O(1) in Redis?

O(1), or **constant time**, means the time required to perform an operation **does not grow** with the size of the data. Redis achieves this efficiency by leveraging **in-memory data structures** and optimized algorithms.

#### ✅ Examples of O(1) Operations

| Command                  | Complexity | Description                              |
|--------------------------|------------|------------------------------------------|
| `GET`, `SET`             | O(1)       | Fetch or store a string value            |
| `HGET`, `HSET`           | O(1)       | Access or update a hash field            |
| `LPUSH`, `RPUSH`         | O(1)       | Insert at the head or tail of a list     |
| `LPOP`, `RPOP`           | O(1)       | Remove and return from list ends         |
| `SADD`, `SREM`, `SISMEMBER` | O(1)   | Set operations using internal hash table |

These operations are fast because Redis uses **hash tables** or **linked lists** under the hood.

---

### ❗ When Redis is NOT O(1)

Some Redis data structures provide powerful features like ordering or scoring. These require more complex internal mechanisms such as **skip lists** or **dictionaries with binary heaps**, which come with higher time complexity.

### Examples of Non-O(1) Operations

| Command                    | Complexity          | Notes                                     |
|----------------------------|---------------------|-------------------------------------------|
| `ZADD`, `ZREM`             | O(log n)            | Sorted sets use skip lists for ordering   |
| `ZRANK`, `ZRANGE`          | O(log n + m)        | Sorted retrieval based on score or rank   |
| `LRANGE`, `LTRIM`          | O(n)                | List slicing operations                   |
| `SCAN`, `SSCAN`, `ZSCAN`   | O(1) per call, O(n) total | Iterative over full collection       |
| `SUNION`, `SINTER`, `SDIFF`| O(n)                | Set math depends on input size            |

> 🔎 Skip lists are the reason why `ZSET` operations (like `ZADD`, `ZRANK`, etc.) are `O(log n)`. Redis uses them to maintain ordered data efficiently.

---

### 💡 Why O(1) Isn’t Always the Goal

While O(1) is desirable, it’s not always feasible when advanced querying or ordering is needed. Redis optimizes for **practical speed** and **memory efficiency**, even when operations aren’t strictly constant time.

---

## 🧠 Summary

| Operation Type          | Example Commands          | Complexity     |
|-------------------------|---------------------------|----------------|
| Key-Value Lookup        | `GET`, `SET`              | O(1)           |
| Hash Field Access       | `HGET`, `HSET`            | O(1)           |
| List Insertion/Removal  | `LPUSH`, `RPOP`           | O(1)           |
| Sorted Set Management   | `ZADD`, `ZRANGE`          | O(log n)       |
| Set Operations          | `SINTER`, `SUNION`        | O(n)           |

Redis delivers impressive performance through smart data structure design. But as engineers, we should always know **what's under the hood** to choose the right tools for the job.

---

## 🔧 TL;DR

- ✅ **O(1)**: For strings, hashes, lists, and sets (basic operations)
- ❗ **O(log n)**: For ordered operations using sorted sets (ZSET)
- 📉 **O(n)**: For bulk operations like `LRANGE`, `SCAN`, `SUNION`

Understanding Redis complexity lets you **design better systems** and **optimize performance** where it matters.

---

✍️ *Got questions about Redis internals? Or want to dive deeper into skip lists and memory usage? Let’s connect in the next post.*
