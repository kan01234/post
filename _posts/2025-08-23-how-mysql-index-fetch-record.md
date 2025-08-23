---
layout: post
title: " How MySQL (InnoDB) Fetches Records Using Indexes"
date: 2025-08-23
tags: mysql, performance, data-structures, innodb
categories: datastore
---

When we write a query like:

```sql
SELECT * FROM users WHERE id = 123;
```

MySQL (with the InnoDB storage engine) doesn’t just “magically” find the row.
Under the hood, it performs a well-optimized journey: navigating a B+Tree index, diving into a file page, using binary search with a page directory, and finally walking a linked list of records to reach the exact row.

In this post, we’ll peel back the layers of how InnoDB index lookups really work.
By the end, you’ll understand not just that indexes speed up queries, but how MySQL finds your data so efficiently.

---

## 1. The B+Tree: Index as a Navigational Map

InnoDB stores indexes as a B+Tree. Each node is a 16KB page, and the tree ensures that:

- Internal nodes (non-leaf pages) hold only keys and pointers to child pages.
- Leaf nodes contain the actual index entries. For a clustered index (usually the PRIMARY KEY), the leaf entries also contain the full row data.

Think of the B+Tree as a map that narrows down your search step by step.

```
          [50 | 100 | 150]   <-- Root (internal node)
          /      |       \
   [10..49]   [51..99]   [101..199]   <-- Child nodes (leaf pages)
```

If you search for id=123, the traversal goes:

- Root: 123 is greater than 100, less than 150 → go right branch.
- Leaf: within page [101..199], find exact key 123.

---

## 2. The File Page: Reading from Disk or Buffer Pool

Each node in the B+Tree corresponds to a 16KB file page inside InnoDB’s tablespace. When InnoDB traverses the B+Tree:

1. It first checks the Buffer Pool (in-memory cache).
2. If the page isn’t cached, it issues a random read from the .ibd file (tablespace file on disk).

This is why index access patterns are crucial: well-designed indexes reduce unnecessary random I/O.

---

## 3. Key Linked List: Searching Inside the Page

Once InnoDB lands on the correct leaf page, it doesn’t immediately know where your key is. Each page maintains its entries in a sorted linked list by key.

```
Page [101..199]
 ┌─────────────────────────────────┐
 │ 101 -> 110 -> 123 -> 150 -> 180 │
 └─────────────────────────────────┘
```

To optimize, InnoDB uses a page directory (slots at the end of the page) to speed up binary search instead of scanning the list one by one.

So the search process inside a page is:

- Binary search on the page directory → find the slot near your key.
- Iterate forward in the linked list → fetch the exact record.

---

## 4. Record Fetch: Finding the Row Data

Finally, when the matching key is found:

- For a clustered index (PRIMARY KEY):
  The record itself (all columns) is stored in the leaf page entry. The search stops here.

- For a secondary index:
  The leaf entry stores the secondary key + a pointer (the PRIMARY KEY). InnoDB must then do another lookup in the clustered index to fetch the full row.

So, fetching a record from a secondary index is effectively:

Secondary B+Tree lookup → Clustered B+Tree lookup → Row data.

## 5. Page Layout: Binary Search + Linked List

To tie it together, here’s how a single leaf page is structured:

```
Leaf Page (16KB)

+-------------------------------+
| Infimum record                |
| Record: id=101                |
| Record: id=110                |
| Record: id=123                |  <-- linked list of user records
| Record: id=150                |
| Record: id=180                |
| Supremum record               |
+-------------------------------+
| Free space (for inserts)      |
+-------------------------------+
| Page Directory (array of slots)
|   slot0 -> points to 101
|   slot1 -> points to 123
|   slot2 -> points to 180
+-------------------------------+
```

Lookup process inside the page:

1. Do a binary search on the page directory.
2. Jump to the nearest record in the user record linked list.
3. Traverse the list until you reach the exact match.

---

## Visualization: The Full Path

Here’s the whole journey put together:

```
Query: SELECT * FROM users WHERE id = 123;

   [B+Tree Root Page]
            │
            ▼
   [Leaf Page containing 123]
            │
            ▼
   [Page Directory → Linked List]
            │
            ▼
   [Record: id=123, name="Alice", age=30]
```

For a secondary index:

```
[B+Tree (Secondary Index)] → find key "email=alice@example.com"
             │
             ▼
   [Pointer to Primary Key (id=123)]
             │
             ▼
   [Clustered Index B+Tree (PRIMARY KEY)]
             │
             ▼
   [Record: id=123, name="Alice", age=30]
```

---

## Summary

To recap, here’s how InnoDB uses indexes to fetch an exact record:

1. B+Tree traversal

- Binary search inside internal nodes.
- Follow pointers down until the correct leaf page is found.

2. Inside the page

- Use binary search on the page directory to jump close to the target.
- Follow the linked list of records for the “last mile” search.

3. Fetch record

- Clustered index → record is here.
- Secondary index → get primary key, then fetch from clustered index.

✅ In short:
Binary search (B+Tree) → Binary search (Page Directory) → Linked List iteration → Record

---

Key Takeaways

- Primary key lookups are fastest.
- Secondary index lookups involve one extra step.
- I/O efficiency (buffer pool hits vs disk reads) directly impacts query latency.

Understanding this flow explains why good index design is critical for query performance.