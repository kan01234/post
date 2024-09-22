---
layout: post
title: "Understanding Consistent Hashing: From Basics to Real-World Use Cases"
date: 2024-09-22
tags: system-design
---

## Understanding Consistent Hashing: From Basics to Real-World Use Cases

To achieve horizontal scaling, it's crucial to distribute requests and data efficiently and evenly across servers. A key challenge arises when adding or removing servers, which can disrupt the balance and lead to costly data movement. Consistent hashing is a powerful technique designed to address this issue. But before diving into how it works, let’s first take a closer look at the problem.

### Introduction to Hashing

In computing, hashing is a technique used to efficiently map data (such as a key) to a fixed location in memory or on disk. This process is carried out using a hash function, which takes an input (the key) and produces a fixed-size hash value, often represented as an integer. The goal of hashing is to distribute data uniformly across a set of available storage locations.

#### Traditional Hashing Example

In a traditional system, hashing helps quickly locate and retrieve data. For instance, a simple hash function could map a user ID to a particular server in a system of five servers. Using a modulo operation like hash(user_id) % 5, we can easily determine which server should handle the request.

| **User ID** | **Hash Value (Simulated)** | **Hash Value % 5 (Server)** |
|-------------|----------------------------|-----------------------------|
| 1001        | 5643                       | 5643 % 5 = 3 (Server 3)     |
| 2005        | 7932                       | 7932 % 5 = 2 (Server 2)     |
| 3021        | 10123                      | 10123 % 5 = 3 (Server 3)    |
| 4987        | 16489                      | 16489 % 5 = 4 (Server 4)    |
| 5763        | 18642                      | 18642 % 5 = 2 (Server 2)    |

This means:
- User 1001 and User 3021 are mapped to **Server 3**
- User 2005 and User 5763 are mapped to **Server 2**

#### The Problem with Traditional Hashing

There’s a fundamental problem with traditional hashing when applied to distributed systems: scalability. Distributed systems often involve multiple servers or nodes that may need to be dynamically added or removed due to scaling requirements or failures.

1. Node Addition/Removal: When new nodes are added or removed, the entire hash distribution changes. For example, if we add a new server, the hash function’s modulo calculation (hash(key) % number_of_nodes) no longer holds, and almost every piece of data must be reassigned to a different node.

2. Data Movement: As a result, significant amounts of data must be redistributed across nodes, which can lead to performance degradation, system downtime, and increased operational costs.

| **User ID** | **Hash Value (Simulated)** | **Hash Value % 5 (Server)** | **Hash Value % 6 (Server)** |
|-------------|----------------------------|-----------------------------|-----------------------------|
| 1001        | 5643                       | 5643 % 5 = 3 (Server 3)     | 5643 % 6 = 3 (Server 3)     |
| 2005        | 7932                       | 7932 % 5 = 2 (Server 2)     | 7932 % 6 = 0 (Server 0)     |
| 3021        | 10123                      | 10123 % 5 = 3 (Server 3)    | 10123 % 6 = 1 (Server 1)    |
| 4987        | 16489                      | 16489 % 5 = 4 (Server 4)    | 16489 % 6 = 1 (Server 1)    |
| 5763        | 18642                      | 18642 % 5 = 2 (Server 2)    | 18642 % 6 = 3 (Server 3)    |

### Introduction Consistent Hashing

Consistent hashing is a distributed hashing technique that addresses the key challenges faced by traditional hashing when nodes are added or removed. It achieves this by using a ring (or circle) structure where both data (keys) and nodes (servers) are placed based on their hash values.

#### How Consistent Hashing Works

1. The Hash Ring: Imagine a circle (or ring) where all possible hash values are arranged in a clockwise direction. Each node in the system is assigned a position on this ring, determined by applying a hash function to the node's identifier (like its IP address or server ID).

2. Key Placement: Each data key is also assigned a position on the ring using the same hash function. A key is mapped to the first node that appears in a clockwise direction from its position. This ensures that each node is responsible for a specific range of keys.

![hash-ring](/post/assets/2024-09-22/ring.svg)

3. Minimal Data Movement: The primary advantage of consistent hashing is that when a node is added or removed, only a small portion of the keys need to be reassigned. This contrasts sharply with traditional hashing, where every key might need to be moved. For example, if a node is added to the ring, only the keys that would have been handled by the neighboring node (clockwise) are reassigned to the new node.

![hash-ring](/post/assets/2024-09-22/ring-add.svg)

#### Issues in the Basic Approach of Consistent Hashing

The term "consistent hashing" was introduced by David Karger et al. at MIT, and the basic idea are:

1. **Map servers and keys onto the ring** using a uniformly distributed hash function.
2. **Locate a server for a key** by going clockwise from the key's position until the first server on the ring is found.

However, two main problems arise with this approach:

##### 1. Imbalanced Partitions:

In a real system, it's impossible to ensure that each server gets an equal-sized partition of the ring. Each partition is the hash space between two adjacent servers. As servers are added or removed, the size of these partitions can become highly uneven, meaning:

- **Some servers may be assigned very small partitions** (responsible for only a small portion of keys).
- **Others may get very large partitions**, causing them to handle much more data and requests.

This imbalance can lead to inefficiency, as some servers may be overburdened while others are underutilized.

![hash-ring](/post/assets/2024-09-22/ring-remove.svg)

if node 3 is removed, node 4 will get much more data than other nodes

##### 2. Hotspots and Load Imbalance:

Because partitions vary in size, it’s likely that some servers will experience much heavier loads than others. For instance, a server with a larger partition will receive more requests, which could cause performance bottlenecks. This can become particularly problematic as new servers are added or removed, leading to frequent rebalancing.

#### Addressing the Challenge

To address these issues, one common solution is to use **virtual nodes** or **replicas**. By assigning multiple virtual nodes to each server, the partitions can be split more evenly across the ring. This ensures a more balanced distribution of keys and prevents any single server from being overloaded.

![hash-ring](/post/assets/2024-09-22/ring-vn.svg)

Assume we have 3 physical nodes, and for each node, we create 3 virtual nodes. In a real-world scenario, the number of virtual nodes would typically be much larger, but for simplicity, we’ll use 3 here.

Instead of directly mapping to physical nodes like N1, N2, and N3, we now divide the hash ring into 9 partitions represented by the virtual nodes: N1_0, N1_1, N1_2, N2_0, N2_1, N2_2, N3_0, N3_1, N3_2. Each virtual node corresponds to a different partition on the hash ring.

With this setup, each physical node is responsible for multiple smaller partitions (represented by its virtual nodes), which helps distribute the load more evenly across all nodes in the system. This ensures that if a node joins or leaves the system, only the keys associated with its virtual nodes need to be reassigned, minimizing disruption and improving scalability.

**How Virtual Nodes Solve the Problem**

1. Balanced Load Distribution:
- In the basic approach, the physical server's position on the ring determines the size of the partition it manages, which can result in uneven distribution.
- With virtual nodes, the ring is populated with multiple points for each server. Since these points are spread out, each server ends up managing several smaller, more equally distributed partitions. As a result, even if one server handles a slightly larger range of keys, its multiple virtual nodes ensure that it doesn’t get overloaded.

2. Minimizing Data Movement:
- When a new physical server is added, it’s assigned its own set of virtual nodes. Since the virtual nodes are spread out across the ring, only a small subset of keys needs to be redistributed to the new server, rather than a large chunk, minimizing disruption.
- The same happens when a server is removed. Only the keys mapped to that server’s virtual nodes are redistributed to the remaining nodes.

3. Fine-Grained Control:
- Virtual nodes give the system fine-grained control over balancing the load. For example, if a server is more powerful, you can assign it more virtual nodes, allowing it to handle a larger proportion of the data and traffic. Conversely, less powerful servers can have fewer virtual nodes.

### Drawbacks of Consistent Hashing

**1. Imperfect Load Balancing**

Even with virtual nodes, consistent hashing doesn’t always guarantee a perfectly balanced load. Some servers may still end up handling more data or requests than others, especially if certain keys are accessed more frequently (hotspots). For instance, in a system where some resources (e.g., popular videos or viral posts) are much more in demand, consistent hashing may not fully mitigate the issue.

**2. Increased System Complexity:**

Managing virtual nodes introduces overhead, particularly in large-scale systems with numerous servers. Adding more virtual nodes to improve balance increases operational complexity and makes it more difficult to debug and manage the system.

**3. Cold Start Problem:**

When new servers are added (e.g., during scaling events), they may initially have no data assigned, leading to a cold start issue. This server will need to gradually pick up load, which may create inefficiencies or delays. Over time, consistent hashing will rebalance, but the initial load distribution can be less efficient.

**4. Snowball Effect:**

If a server failure or removal occurs in an already uneven system, the redistribution of load can cause other servers to become overloaded. This “snowball effect” can lead to cascading failures as more and more load is shifted to fewer servers, potentially disrupting the entire system.

**5. Hash Function Dependency:**

The effectiveness of consistent hashing depends heavily on the quality of the chosen hash function. Poorly designed or non-uniform hash functions can lead to uneven distribution of data, which defeats the purpose of using consistent hashing in the first place.

### Efficiency vs. Uniformity 

**Efficiency:**

- Consistent hashing minimizes data movement when nodes are added or removed, leading to efficient scaling with minimal disruption.
- It helps avoid the costly reallocation of keys that traditional hashing methods would require.
- However, ensuring this efficiency introduces complexity in managing virtual nodes, load balancing, and maintaining redundancy.

**Uniformity:**

- While consistent hashing distributes keys more evenly across nodes compared to traditional methods, achieving perfect uniformity is difficult.
- Some nodes may end up with more load than others (hotspots), especially if virtual nodes or load balancing techniques aren’t carefully applied.
- The challenge is balancing this distribution to ensure uniform performance without overloading specific nodes.

### Conclusion

The key benefits of consistent hashing include:

- **Minimal Data Redistribution**: When servers are added or removed, only a small subset of keys needs to be reassigned, which helps avoid large-scale system disruptions.
- **Horizontal Scalability**: Consistent hashing enables easy horizontal scaling since the data is evenly distributed across the nodes, preventing bottlenecks.
- **Hotspot Mitigation**: It helps to mitigate the hotspot issue, where certain shards might receive excessive access. For example, popular keys like data for Katy Perry, Justin Bieber, and Lady Gaga might overwhelm a single shard, but consistent hashing ensures they are distributed more evenly.

**Consistent hashing** is a core part of the infrastructure for many large-scale distributed systems, including:

- **Amazon’s DynamoDB** for partitioning data across clusters.
- **Apache Cassandra** for managing distributed databases.
- **Discord** for balancing load across their messaging system.
- **Akamai’s content delivery network (CDN)** for efficient content distribution.

By leveraging consistent hashing, these systems can scale effectively, distribute data evenly, and maintain high availability, even in the face of frequent changes to the system topology.
