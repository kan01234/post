---
layout: post
title: "System Design: Building a Fair and Scalable Gacha System for Mobile Games"
date: 2025-06-24
tags: system-design
categories: system-design
---

## 🧱 Objective

Design a Gacha (ガチャ) System for a mobile game that:

- Allows players to "roll" for random in-game items (e.g., characters, weapons)
- Ensures fairness, rarity rules, and rate-up events
- Can scale to millions of daily active users
- Is resilient, auditable, and safe from abuse

### ✅ Functional Requirements

- Support Single Roll and 10x Roll APIs
- Each roll returns an item (with rarity, metadata)
- Support Rate-Up Events (limited time boosted chances)
- Support Pity Counter (guaranteed SSR after X rolls)
- Track player history (what was rolled, when)
- Support virtual currency (gems, tickets) as payment
- Provide backend tools to configure drop rates

### ❌ Non-Functional Requirements

- High availability and low latency (<100ms per roll)
- Fairness (tamper-proof logic, prevent RNG manipulation)
- Scalability to millions of players
- Fraud detection and analytics
- Auditable (for player support/disputes)

## 🧩 High-Level Components

```
+------------------+
|    Game Client   |
+--------+---------+
         |
         v
+--------------------+          +-----------------------+
|     API Gateway    |--------->|    Gacha Service      |
+--------------------+          +----------+------------+
                                           |
                         +-----------------+-------------------+
                         |                                     |
              +----------v-----------+             +-----------v----------+
              |   RNG & Drop Engine  |             |  Player Inventory DB  |
              +----------+-----------+             +-----------+----------+
                         |                                     |
           +-------------v------------+         +--------------v-------------+
           |  Gacha Config Service    |         |   Transaction / Currency    |
           |  (rates, events, pity)   |         |   Service (Wallets)         |
           +--------------------------+         +-----------------------------+
```

### 🎲 What is RNG?

RNG stands for Random Number Generator.

In games (especially gacha games), it's the core of randomness — it's how the game decides what you get when you "pull" (e.g., which item or character).

Example:

- The game says: “You have a 1.5% chance to get a UR (Ultra Rare) character.”
- When you do a pull, the system uses RNG to generate a number between 0.0 and 1.0 (e.g., 0.0132).
- If that number is less than 0.015, you get the UR.

It’s like rolling a 100-sided die behind the scenes.

### 🧷 What is Pity?

Pity is a player-friendly mechanic that increases your chance of getting a rare item after multiple unlucky tries.

There are two main types:

| Type             | Description                                                                                                                                |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| 🧠 **Soft Pity** | Your chance increases slightly each time you don’t get the rare item. Example: UR starts at 1.5%, goes up by 0.1% per pull after 70 pulls. |
| 💎 **Hard Pity** | A guaranteed rare drop if you haven't gotten it after a fixed number of pulls (e.g., UR guaranteed at pull #90).                           |

Example:

| Pull Count | UR Chance               |
| ---------- | ----------------------- |
| 1          | 1.5%                    |
| 70         | 1.5%                    |
| 71         | 1.6%                    |
| 80         | 2.5%                    |
| 90         | 💥100% (guaranteed)\*\* |

This system is used in games like Genshin Impact, Fate/Grand Order, and others — it builds trust and reduces frustration for unlucky players.

### 🎲 Core Logic: RNG & Drop Engine

- Use pre-seeded cryptographically secure RNG
- Lookup Gacha Pool Configuration:
  - Define item pools by rarity (SSR, SR, R)
  - Define rates (e.g., SSR = 1.5%)
  - Support "rate-up" mechanics (SSR pool with boosted chances)

- Implement Pity Counter logic:
    - After X rolls, guarantee a specific rarity or item

- Return item from the selected pool

🧠 You may precompute drop tables per player or shard RNG seed per user/session for anti-cheating.

### 🗃️ Data Models

🎰 Gacha Config (in DB or Config Store)

| Field           | Type       | Description                      |
| --------------- | ---------- | -------------------------------- |
| gacha\_id       | UUID       | Unique Gacha pool                |
| rarity\_tiers   | JSON       | { SSR: 1.5%, SR: 12%, R: 86.5% } |
| items           | JSON       | List of items by rarity          |
| pity\_limit     | int        | Guaranteed SSR after N pulls     |
| rate\_up\_items | List<Item> | Limited-time boosted items       |

🧾 Roll History

| Field        | Type     |
| ------------ | -------- |
| player\_id   | UUID     |
| gacha\_id    | UUID     |
| item\_id     | UUID     |
| rarity       | String   |
| timestamp    | DateTime |
| is\_rate\_up | Boolean  |
| used\_pity   | Boolean  |

### ⚙️ Flow: Single Roll

Client sends POST /roll with gachaId and currency info

Gacha Service:

1. Validates request & user session
2. Checks currency via Transaction Service
3. Gets config for that gacha
4. Applies RNG + pity logic
5. Returns item
6. Writes roll result to History DB
7. Updates Player Inventory
8. Response: { item_id, rarity, used_pity }

### ⚡ Performance & Scale

| Component    | Strategy                             |
| ------------ | ------------------------------------ |
| Gacha Logic  | Stateless, horizontally scalable API |
| Config Store | Cached in memory, or in CDN / Redis  |
| Roll History | Write-heavy, use append-only logs    |
| Inventory    | Strong consistency, ideally in RDB   |
| RNG          | Use secure RNG, consider DRBG        |
| Wallet       | Pessimistic lock or atomic update    |

### 🔐 Fairness & Anti-Cheat

- Use server-side RNG (never client-side)
- Validate all requests against session/player ID
- Use signed drop results (e.g., HMAC for dispute resolution)
- Rate-limit API calls to avoid spam
- Store roll history immutably (for audits)

### 🔭 Observability & Analytics

| Metric                 | Use                                |
| ---------------------- | ---------------------------------- |
| Rolls per second       | Scale monitoring                   |
| SSR/SR drop rates      | Fairness and bug detection         |
| Pity counter triggered | Track behavior and user experience |
| Currency consumption   | Revenue analytics                  |
| Unique items rolled    | Event success metrics              |

### 🔁 Retry & Idempotency

- Use request_id or roll_id to prevent double roll
- Deduct currency before RNG generation
- Retry safe if using idempotency key + transaction store

### 🧱 Datastore Design

| Data               | Store                   | Why                            |
| ------------------ | ----------------------- | ------------------------------ |
| Gacha Configs      | Redis / S3              | Fast load, infrequent writes   |
| Roll History       | Append-only DB or Kafka | Easy replay, audit trail       |
| Inventory / Player | PostgreSQL              | ACID for inventory consistency |
| Wallet             | PostgreSQL              | Atomic balance updates         |
| Analytics          | Kafka + Druid           | Real-time insights             |

## 🎯 Fairness Problem

If each player rolls independently using RNG, then in theory, over enough rolls, the actual distribution should match the configured rates (e.g., 1% UR, 10% SR, etc.). But in practice:

- Some players may get very lucky, getting multiple URs.
- Others may get nothing, leading to dissatisfaction.
- If you want to enforce a hard global limit (e.g., 1 UR per 10,000 rolls globally), then pure RNG per player won’t work.

### 💡 Solution Options

✅ 1. Soft RNG + Pity System (Industry Standard)

Most games (e.g., Genshin, FGO, etc.):

- Do not try to enforce global fairness.
- Instead, they:
  - Use secure individual RNG per user
  - Add pity systems (guarantee after X rolls)
  - Use rate-up banners to increase chances temporarily

This simplifies scaling, ensures fairness per user, and is acceptable to players if the mechanics are transparent.

> 🎯 This is the most common and scalable way to handle gacha drop fairness.

---

❗️2. Global Rate Enforcement (Not Recommended for Realtime Gacha)

If you want to enforce a global UR quota, you'd need to:

- Maintain a global roll counter for each rarity
- Keep live distributed state of item counts and drops
- Block or delay drops if quota is exceeded

This is problematic:

- Needs strong consistency and coordination across shards
- Adds latency and makes the system non-scalable
- Can lead to poor user experience

Only some limited-time lottery events (not real-time gacha) might use this.

---

🧪 3. Pre-shuffled "Drop Bags" (Deterministic Fairness)

Used in competitive card games or small-scale events.

Mechanism:

- Pre-generate a "bag" with fixed number of items (e.g., 1 UR, 10 SR, 89 R = 100 rolls)
- Shuffle it, assign per player
- As user pulls, you draw from the bag

Pros:

- Guarantees drop rates within the bag
- Perfect fairness in small scope

Cons:

- Not suitable for infinite gacha
- Harder to implement with dynamic banners or rate-ups

---

🧠 Better Compromise: Per-Player Bounded RNG

Use a per-player strategy like:

- After X rolls:
  - Increase odds gradually (soft pity)
  - Or guarantee UR after max N rolls

This doesn’t enforce global fairness, but provides a bounded player experience, so no one is "eternally unlucky".

### 🔐 Summary: How to Ensure Fairness & Player Trust

| Technique                    | Global Rate Control | Player Experience | Scalable | Used In Practice  |
| ---------------------------- | ------------------- | ----------------- | -------- | ----------------- |
| Per-player RNG + Pity        | ❌                   | ✅ Excellent       | ✅ Yes    | ✅ Widely used     |
| Global Drop Rate Enforcement | ✅                   | ❌ Bad             | ❌ No     | ❌ Rare            |
| Pre-shuffled Drop Bags       | ✅ (per batch)       | ✅ Fair            | 🚫 No    | 🔶 Event use only |
| Per-player Bounded RNG       | ❌                   | ✅ Controlled      | ✅ Yes    | ✅ Often combined  |


## 🏁 TL;DR — Key Design Choices

| Component     | Choice                   | Justification                       |
| ------------- | ------------------------ | ----------------------------------- |
| RNG           | Secure RNG + server-only | Prevents manipulation               |
| Gacha config  | Cached + dynamic         | Event support, fast load            |
| Currency      | Wallet service (locked)  | Consistent balance and auditability |
| Inventory     | RDB                      | Reliable item tracking              |
| History       | Append log               | For audit, replays, user support    |
| Observability | Metrics + logs + alerts  | Operations visibility               |
