---
layout: post
title: "How We Solved a High Memory Usage Issue in Our Apache Camel Batch Pipeline"
date: 2025-5-10
tags: apache-camel spring-boot spring-data-jpa batch-processing csv-export memory-optimization java streaming performance mysql backend
categories: best-practices
---

# 🔧 How We Solved a High Memory Usage Issue in Our Apache Camel Batch Pipeline

Recently, we encountered and resolved a **high memory usage problem** in our Apache Camel application, which processes **large batches of records** for CSV export. This post breaks down the problem, root causes, and the steps we took to improve performance and stability.

---

## 💥 The Problem: Memory Spikes During Batch CSV Export

Our application needed to **export a large number of database records** to a CSV file using **Apache Camel**. However, during execution, the application showed **severe memory pressure**, often leading to GC thrashing or even `OutOfMemoryError` in high-load environments.

After profiling the app, we identified two major bottlenecks:

1. **Spring Data JPA Query Loaded All Records Into Memory**
   - The SQL query used a `UNION` and an `ORDER BY`, which prevented **pagination** or **streaming** at the JDBC level.
   - As a result, the entire result set was loaded into memory at once.

2. **Apache Camel Was Not Streaming Writes**
   - We were using Camel’s `marshal().csv()` and `to("file://...")` to write CSV output.
   - However, by default, Camel buffers the full message before writing, which added more pressure on heap memory.

---

## 🔍 Root Causes

### 1. Query Design Prevented Streaming

The query used:

```sql
(SELECT ... FROM A ...) 
UNION 
(SELECT ... FROM B ...)
ORDER BY timestamp;
```

This design forced the database to materialize the entire result set, preventing any form of lazy fetching or cursor-based streaming. This meant Spring Data had no choice but to load everything at once.

### 2. Camel’s Non-Streaming Default Behavior

Camel’s CSV marshalling and file writing behavior assumes the whole message is available and does not stream row-by-row unless explicitly told to do so.

## ✅ The Fix: Query Rewrite + Streaming Pipeline

1. Refactored the Query

    Removed the UNION. We realized the combined data could be retrieved through a single, more general query using a discriminator column.

    This change allowed Spring Data to use streaming query execution.

2. Enabled Streaming in Spring Data

Instead of returning List<T>, we used:

```
@Query("SELECT e FROM Entity e")
Stream<MyEntity> streamAll();
```

Combined with:

```
@Modifying(clearAutomatically = true)
@Transactional(readOnly = true)
```

This enabled MySQL streaming result sets through Spring Data JPA (using ResultSet.TYPE_FORWARD_ONLY and fetchSize=Integer.MIN_VALUE under the hood).

> ⚠️ Note: Avoid using offset-based pagination in large datasets — it leads to performance degradation and doesn’t scale well in production.

### 3. Buffered CSV Writing in Apache Camel

We changed the route from:

```
from("direct:start")
    .marshal().csv()
    .to("file://output");
```

To a custom streaming approach:

```
from("direct:start")
    .split(body()).streaming()
    .process(exchange -> {
        MyEntity entity = exchange.getIn().getBody(MyEntity.class);
        String line = convertToCsv(entity);
        Files.write(outputPath, (line + "\n").getBytes(), StandardOpenOption.APPEND);
    });
```

This way, each line is written immediately, avoiding large in-memory CSV representations.

## 📈 Results

- Heap usage dropped significantly (from 32 GBs to 4 GBs).

- GC frequency and duration reduced by over 80%.

- The system can now handle millions of records without OOM risk.

- Our batch job runtime decreased by 35% thanks to reduced memory pressure.

## 🧠 Lessons Learned

- Review SQL queries carefully; set-based operations like UNION + ORDER BY can kill streamability.

- Use streaming where possible in both your data source (JPA/JDBC) and data sink (Camel/file output).

- Apache Camel is powerful, but you must explicitly configure it to process large datasets efficiently.

## ✅ Related Tips

- Prefer cursor-based iteration (Stream<T>) over loading full lists in Spring Data.

- Don’t rely on offset/limit pagination for high-volume jobs — use keyset pagination or streaming.

- Always profile and monitor heap usage before deploying batch jobs in production.

## 💭 Final Thought

Of course, there are many other architectural solutions to handle large-scale batch processing — like pre-calculating results, splitting data by partition, or offloading processing to distributed systems. But those approaches often require more time, complexity, and effort to develop and maintain.

For us, optimizing the existing flow with streaming, query refinement, and buffered writing provided the most practical balance between performance and development cost.
