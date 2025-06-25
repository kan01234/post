---
layout: post
title: "System Design: How to Build a URL Shortening Service at Scale"
date: 2025-06-23
tags: system-design
categories: system-design
---

## 🔧 Goal: Design a URL Shortening Service

> Like Bitly: Given a long URL, generate a short unique alias (e.g., bit.ly/abc123) and redirect users when they visit that short link.

### 🧱 Functional Requirements

- Shorten a given URL (e.g., POST /shorten) → returns short code
- Redirect to original URL when short code is accessed (GET /abc123)
- Custom alias support (optional)
- Expiry or analytics support (optional)

### ❌ Non-Functional Requirements

- High availability (reads are critical)
- Low latency redirect
- Scalable to billions of URLs
- Eventually consistent analytics (optional)

## 🧩 High-Level Components

```
+-------------+       +------------------+       +--------------------+
|   Clients   | <---> |   API Gateway    | <---> |   URL Shorten API  |
+-------------+       +------------------+       +---------+----------+
                                                            |
                                                            v
                                              +-------------+-------------+
                                              |     Storage System        |
                                              |  (DB + Cache + Indexes)   |
                                              +---------------------------+
```

## 🔑 Core Design Decision: How to Generate Short URL?

Option A: Base62 Encoding of ID

- Store the long URL in DB with auto-increment ID
- Encode ID to short string using [a-zA-Z0-9] (62 chars)
    - ID 125 → cb
- Fast, unique, simple
- Needs centralized ID generator (single point of contention)

Option B: Random String

- Generate random 6-10 character string (check for collision)
- Needs retry or bloom filter
- No ordering, harder to predict

Option C: Hash (MD5/SHA1)

- Hash the long URL and truncate
- Collision possible → store and check
- Deterministic (same URL → same code)

👉 Most systems use Base62 of an integer ID or random + uniqueness check.

### 🗃️ Database Schema

```
Table: urls
- id (BIGINT PK)
- short_code (VARCHAR UNIQUE)
- long_url (TEXT)
- created_at
- expiration_time (optional)
- owner_id (optional)
```

### ⚙️ Flow: Shortening a URL (POST)

1. Client calls POST /shorten with long URL
2. Server generates a short code:
  2.1 Either by inserting and getting auto-increment ID, then encode
  2.2 Or generate random string + check uniqueness
3. Store (short_code, long_url) in DB
4. Return short_url = domain/short_code

### ⚡ Flow: Redirect (GET /abc123)

1. Client requests GET /abc123
2. Lookup short code:
  2.1 First check cache (Redis)
  2.2 If not found, hit DB
3. If found, redirect (HTTP 301 or 302); Else, return 404

### 🧠 Optimization

#### Caching

- Cache short_code → long_url (e.g., in Redis)
- Cache hot or frequently accessed URLs
- Use TTL for cache eviction

#### 📈 Analytics (Optional)

- Log redirects asynchronously (fire-and-forget)
- Count clicks, geolocation, referrers
- Store in Kafka + sink to data warehouse (e.g., BigQuery)

### 🧮 Scale Considerations

| Component | Strategy                           |
| --------- | ---------------------------------- |
| DB Writes | Sharded ID generator, UUID         |
| DB Reads  | Cache layer, CDN                   |
| API Layer | Stateless, horizontally scaled     |
| Storage   | Use consistent hashing or sharding |
| Backup    | Replicate data, versioning         |

### 🚨 Failure & Recovery

- Retry DB writes (w/ backoff)
- Cache fallback to DB
- Monitor for cache misses, high 404 rates

### Trade-offs

| Choice       | Pros             | Cons                            |
| ------------ | ---------------- | ------------------------------- |
| Base62 ID    | Compact, ordered | Global ID = bottleneck          |
| Random codes | No dependency    | Needs collision detection       |
| Hashing      | Deterministic    | Collisions + long to short loss |
| Cache layer  | Super fast reads | Stale data risk                 |

## 🔨 What Datastores to Use?

### 🧠 What Are We Storing?

1. 🔗 URL Mappings

- short_code → long_url
- Metadata: created_at, expiration_time, maybe owner_id
- Must support:
  - Fast read (GET /abc123)
  - Idempotent write (create/update)
  - Uniqueness (short_code must be unique)

2. 📈 Analytics (Optional)

- Clicks: who clicked, when, from where
- Needs high write throughput
- Not latency-critical, but good for batch or streaming

3. ⚡ Caching

- Hot short_codes (frequently accessed)
- Can be safely evicted
- Needs millisecond latency

### Datastores to Use

🧮 Primary: Relational Database (RDBMS)

✅ Best for: URL mappings

- Use: RDB (PostgreSQL / MySQL)

- Reason:

  - Strong consistency (no duplicate short_codes)
  - Indexing on short_code for fast lookup
  - Easy to query/report (filter by user, time, etc.)
  - Supports transactions for atomic insert/update

Pros:

- Mature ecosystem
- Easy to back up, migrate, replicate
- Strong integrity constraints

Cons:

- Doesn’t scale writes horizontally (without effort)
- Global auto-increment ID can become a bottleneck (solution: sharded ID or Snowflake-style ID)

---

💨 Cache Layer: KVS (Redis)

✅ Best for: Low-latency reads

- Store: short_code → long_url
- Set with TTL (time to live)
- If not found, fallback to DB and update cache

Why KVS (Redis)?

- In-memory = very fast (~1ms)
- Simple key-value lookup
- Supports expiration (great for temp URLs)

--- 

📊 Analytics Store: Append-Only Event System + Columnar DB

✅ Best for: Tracking clicks and usage

- Event queue: Kafka / Kinesis
  - Push redirect events to queue (non-blocking)
- Sink to data warehouse:
  - BigQuery, Redshift, or Apache Druid
- Or store raw in:
  - S3 + Athena or Hadoop (if large scale)

Why?

- Decouples analytics from main flow
- Horizontal scalability for billions of events
- Query performance (for dashboards, fraud detection, etc.)

---

🔐 Optional: NoSQL (Only If Needed)

For extreme scale, consider NoSQL alternatives:
A. Cassandra / DynamoDB

- Used if you're at global scale (e.g., millions of writes/sec)
- Trade-offs:

    - High throughput
    - Tunable consistency
    - Eventual consistency for some reads
    - Harder joins/queries

💡 Only introduce NoSQL when:

- You outgrow RDBMS vertical scaling
- Your schema is simple (like just key-value)
- You require geo-distributed writes

### 🧠 Storage Summary Table

| Data                    | Store              | Why                                     |
| ----------------------- | ------------------ | --------------------------------------- |
| Short URL mappings      | PostgreSQL         | Relational integrity, easy indexing     |
| Hot URL cache           | Redis              | Sub-ms lookup, TTL support              |
| Click analytics         | Kafka + BQ         | Async, scalable, columnar analytics     |
| Extreme key-value scale | DynamoDB/Cassandra | Only if millions of writes/sec required |

### 🏁 TL;DR — Recommended Stack

For 99% of teams, this is enough:

- PostgreSQL: Source of truth
- Redis: Fast lookup for short_code
- Kafka → BigQuery: Event log and analytics
