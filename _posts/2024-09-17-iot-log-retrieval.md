---
layout: post
title: "Remote Log Retrieval for Linux-Based Edge Devices: A System Design Approach"
date: 2024-09-15
tags: mqtt,system-design
---

## Remote Log Retrieval for Linux-Based Edge Devices: A System Design Approach

### Requrimetns

A Linux-powered edge device, operating in a remote setting, runs a program that generates periodic debug or error log messages. These logs are currently channeled through the system's default syslog mechanism.

The main features is allows an operator to remotely access log messages from the edge device though secure cloud server

To conserve network bandwidth and cloud storage, the device should only upload logs when explicitly requested by the operator, and only the specific logs that the operator needs.

### Assumption

#### Estimated Log Message Length

Let's make some estimations of the log message:

    Timestamp: 23 characters (yyyy-MM-dd HH:mm:ss.SSS)
    Log Level: 5 characters
    PID: Assuming a typical PID length of 5 digits
    Thread Name: 15 characters
    Logger Name: Let's assume an average of 20 characters
    Log Message: This is variable, but let's stick with our previous assumption of 100 bytes on average
    Exception Stack Trace: This is highly variable, but let's assume an average of 500 bytes when present (this can be adjusted based on your application's exception handling)
    Other characters and formatting: Let's add a buffer of 20 bytes for colons, spaces, and other formatting elements

#### Estimated Log Message / Day

Calculating the Average Log Message size:

    Log per day: 24 * 60 * 60 * 1000 (up to ms)
    Log message length with exception: 688 bytes
    Log message length without exception: 188 bytes
    Probability of exception: 10%

log message size = 86,400,000 * 10% * 688 bytes + 86,400,000 * 188 bytes = 20GB

#### Edge device can delivery the event in order

Ordered Log Delivery

A key assumption for the system design is that the edge device is capable of delivering log messages to the MQTT broker in the order they were generated. This implies that the device's logging mechanism or any intermediary components ensure that logs are published to the broker in their correct chronological sequence, even if there are network delays or temporary disconnections.

This assumption simplifies the overall architecture and eliminates the need for complex message reordering or buffering on the server-side. It allows us to leverage the inherent ordering guarantees of certain message queues (e.g., RabbitMQ with ordering enabled, Amazon SQS FIFO queues) to ensure that logs are processed and stored in the correct order on the cloud server.

#### Request Completion

Assume that every log retrieval request sent to the edge device can be successfully processed and completed by the device. This means that the device has sufficient resources (CPU, memory, storage) and network connectivity to retrieve, filter (if applicable), and upload all the relevant log data within a reasonable timeframe.

## System Design

![sa](/assets/2024-09-17/sa.png)

### Local Log Storage and Management on the Edge Device

#### File Structure

```
├── 2024-09-10/ 
    │   ├── 00/
    │   │   ├── 2024-09-10_00-00.json.gz 
    │   │   ├── 2024-09-10_00-00_1.json.gz
    │   │   ├── 2024-09-10_00-00_error.json.gz
    │   │   ├── 2024-09-10_00-01.json.gz
    │   │   ├── ...
    │   │   └── 2024-09-10_00-59.json.gz 
    │   ├── 01/
    │   │   ├── 2024-09-10_01-00.json
    │   │   ├── 2024-09-10_01-00_error.json
    │   │   ├── ... 
    │   └── ... 
    └── ... 
```

- Daily Directories: Logs are organized into directories named after the date (YYYY-MM-DD) they were generated.
- Hourly Subdirectories: Within each daily directory, there are subdirectories for each hour (00 to 23).
- Minutely Log Files: Inside the hourly directories, log files are created for each minute, following the naming convention: YYYY-MM-DD_HH-MM[_error].json.gz. The optional _error suffix denotes files containing error logs with exception stack traces.
- Sequence Numbers: If multiple log files are generated within the same minute (due to high volume or size limits), they are differentiated using sequence numbers (e.g., _1, _2, etc.).
- Compression: Log files are compressed using gzip (.gz extension), significantly reducing their storage footprint.

#### Estimated File Sizes

| Log Type   | Uncompressed Size (approx.) | Calculation                               | Compressed Size (gzip, approx.) |
|------------|-----------------------------|-------------------------------------------|---------------------------------|
| Non-Error  | 10.9 MB                     | 60,000 logs/min * 188 bytes/log / 1024^2 | 5.5 MB (50% compression)        |
| Error      | 4.13 MB                     | 60,000 logs/min * 10% * 500 bytes/log / 1024^2  | 2.1 MB (50% compression)        |

#### Log Rotation

Rotation Rules:

- Time-Based Rotation: Log files are rotated every minute, creating a new file for each new minute.
- Size-Based Rotation (Optional): The presence of sequence numbers suggests that log files might also be rotated if they exceed a certain size limit, even within the same minute.
- Compression: Compression is likely applied immediately after rotation to save storage space.
- Error Log Separation: Error logs are separated into distinct files, potentially allowing for different retention policies or handling compared to non-error logs.

#### Log Retrieval and Filtering Process

**Scenario: Retrieving logs between 10:01:00 and 10:05:00**

1. **Receive Log Request** 
   * The device receives a request specifying the desired time range (e.g., 10:01:00 - 10:05:00) and any filtering criteria (e.g., log level). We will dicusss how to receive this request later.

2. **Identify Relevant Files**
   * Based on the requested time range, the device identifies:
     * The relevant daily directory (e.g., `2024-09-10`)
     * The corresponding hourly subdirectory (`10`)
     * The log files within that directory whose timestamps fall within the range (`2024-09-10_10-01.json.gz` to `2024-09-10_10-05.json.gz`)
   * If filtering by log level is required, it further selects only the files with the `_error` suffix (if filtering for errors) or all files (if no filtering).

3. **Publish Log Chunks to MQTT**

    * The device publishes each log chunk to the appropriate MQTT topic, ensuring in-order delivery.
      * Each chunk's payload includes metadata like request_id, timestamp, chunk_sequence, and is_last_chunk.

4. **Handle Interruptions**

    * If the upload is interrupted, the progress checkpoint files indicate where to resume upon reconnection/restart.

5. **Completion and Cleanup**

    * Once all files are uploaded:
      * Delete the request JSON file.
      * Delete the temporary directory and its contents.

**Memory Usage in Typical case:**

The largest files would likely be the non-error log files, which are estimated to be around 10.9 MB (uncompressed) or 5.5 MB (compressed).
In this case, the maximum memory usage would be around 5.5 MB.

#### Drawback: Potential Transfer of Unnecessary Log Data

Retrieving entire log files based on minute boundaries can lead to transferring unnecessary data. Even if a specific time range within a minute is requested, the whole file is retrieved and processed, wasting bandwidth and server resources.

#### Potential Improvements

**Finer-Grained Log Rotation**

Concept: Decrease log rotation interval (e.g., to 30 seconds or 15 seconds) for smaller files and less extraneous data transfer.

Benefits:

- More precise log retrieval
- Potential performance improvement on resource-constrained devices

Considerations:
- Increased number of files, potentially impacting file system performance
- Adjustments needed to logrotate and possibly the application's logging


**In-File Filtering with Structured Logs**

Concept: Filter directly within log files on the device by parsing and extracting entries matching the requested criteria.
    
Benefits:
- Minimizes unnecessary data transfer
- Works with minutely or finer-grained rotation

Considerations:
- Adds complexity to device-side logic
- Might require more CPU and memory for parsing and filtering

### Communication bewteen Edge Device and Cloud Server

**Key takeaway**

The system uses MQTT for communication, ensures trust with TLS, and handles intermittent connectivity through retained messages, QoS, and persistent storage. It also employs a file-system-based approach with checkpoints to manage log retrieval and interruptions efficiently.

#### Protocol

MQTT is chosen for its lightweight nature, efficiency in handling intermittent connectivity, and publish-subscribe model suitable for edge device scenarios.

- Lightweight: Ideal for resource-constrained IoT devices with limited processing power and bandwidth.
- Efficient for Intermittent Connectivity: Handles scenarios where the device might lose connection or be offline periodically.
- Publish-Subscribe Model: Enables efficient communication between multiple devices and the server, allowing for selective message delivery based on topic subscriptions.

#### Establishing Trust with TLS:

Mutual TLS authentication is recommended to establish trust between the device and the server, ensuring secure and authenticated communication.

Both the device and the server use X.509 certificates to verify each other's identity during connection setup, preventing unauthorized access and man-in-the-middle attacks.

#### Authentication Methods

1. **OAuth 2.0/OpenID Connect**

* **Concept:** Leverages industry-standard protocols for delegated authorization and authentication. The device obtains an access token from an authorization server, which it then presents to the MQTT broker for authentication.
* **Advantages:** Strong authentication and fine-grained authorization capabilities, centralized identity management, supports various authentication flows and grant types
* **Considerations:** Adds complexity due to the need for an external authorization server, requires the device to interact with the authorization server for token acquisition and renewal

2. **Username/Password**

* **Concept:** The simplest authentication mechanism where the device provides a username and password during connection establishment. The broker verifies these credentials against a stored password file or database.
* **Advantages:** Easy to set up and configure, no external dependencies
* **Considerations:** Less secure compared to token-based or certificate-based authentication, passwords are transmitted in plaintext unless combined with TLS encryption, can be challenging to scale and manage in large deployments

**Comparison of Authentication Approaches**

| Feature        | OAuth 2.0/OpenID Connect | Username/Password | 
|----------------|---------------------------|--------------------|
| Security        | High                      | Low to Moderate    |
| Flexibility     | High (various flows/grants)| Low                |
| Scalability     | High (centralized identity)| High (with clustering) |
| Complexity      | High (external auth server)| Low                |
| Network Reliance| High (for token acquisition)| Low                |
| Suitability     | Large-scale, high-security | Small-scale, simple|

**Chosen Approach: Username/Password**

For this system, we'll utilize **username/password authentication** due to the intermittent network connectivity on the edge device. This approach minimizes reliance on network stability during the authentication process. We'll ensure robust security by combining it with TLS encryption and enforcing strong password policies on the device.

**Important Note:** If the system's security requirements change or the scale of deployment increases significantly, it's advisable to re-evaluate the authentication mechanism and potentially transition to a more robust approach like OAuth 2.0 or mutual TLS.

#### Private Tunnel (Optional):

While a private tunnel adds inherent security, TLS can be layered on top to provide defense-in-depth and granular access control.

Even within a private network, TLS ensures data confidentiality and integrity.

#### Handling Intermittent Connectivity and Device Downtime:

- MQTT's Publish-Subscribe Model: The server publishes log retrieval requests to specific topics, and the device subscribes to those topics. This allows the device to receive requests even if it was offline when the request was initially published.

- Retained Messages: The server can publish requests as retained messages, ensuring the device receives the latest request upon reconnection.

- Delivery manner: Using at-least-once or exactly-once delivery of messages, ensuring reliability even with intermittent connectivity.

- Persistent Storage: The device can store pending requests in persistent storage to handle scenarios where it's offline or loses power during processin

#### Handling Interruption on Process

```
/foo/bar/request/
├── request-1
├── request-2
├── request-1/
│   ├── 2024-09-10_05-00.json.gz
│   ├── 2024-09-10_06-00.json.gz
│   ├── 2024-09-10_07-00.json.gz
│   ├── 2024-09-12T10:05-00.trail
...
```

**1. Receive and Acknowledge Request**

* Device receives log retrieval request via MQTT.
* Creates request JSON file (`/foo/bar/request/request_<request_id>.json`).
* Publishes acknowledgement to the server.

**2. Identify and Copy Relevant Log Files**

* Identifies relevant daily and hourly directories based on time range.
* Selects log files (`.json.gz`) within the range and matching any log level filter.
* Copies selected files to a temporary request-specific directory.

**3. Process Each Copied File Sequentially**

* For each copied file:
    * Create or read progress checkpoint file (`.trail`).
    * If checkpoint exists, resume from the recorded position.
    * Upload entries in chunks, updating the checkpoint after each successful upload.

**4. Handle Interruptions**

* If interrupted, progress checkpoint files indicate where to resume upon reconnection/restart.

**5. Completion and Cleanup**

* Once all files are uploaded:
    * Delete the temporary directory and its contents.

Based on the requested time range, the device identifies the relevant daily and hourly directories in the file system (e.g., /var/log/iot_device/2024-09-10/10/).
It selects the log files (*.json.gz) within those directories whose timestamps fall within the requested range.
If filtering by log level is required, it further selects only the files with the _error

#### Sequence

<div class="mermaid">
sequenceDiagram
    participant e1 as Edge Device 1
    participant e2 as Edge Device 2
    participant b as Message Broker (Cloud Server)
    participant a as Log Service (API / UI)
    participant o as Operator

    e1 -->> b: subscribe topic /client/building-a/log-request/e1
    e2 -->> b: subscribe topic /client/building-a/log-request/e2
    e2 -->> e2: power off
    o -->> a: request log, from 10:05 to 10:08, device 1
    a -->> b: publish topic /client/building-a/log-request/e1, 10:05 - 10:08
    a -->> o: OK
    o -->> a: request log, from 11:05 - 11:08, device 2
    a -->> b: publish topic /client/building-a/log-request/e2, 11:05 - 11:08
    a -->> o: OK
    b -->> e1: publish request log, from 10:05 to 10:08, device 1
    e1 -->> e1: write request to local storage
    e1 -->> b: ack
    e1 -->> b: publish log by chunk, topic /client/building-a/log/e1, chunk, chunk_seq, is_last_chunk
    e2 -->> e2: power on
    b -->> e2: publish topic /client/building-a/e2, 11:05 - 11:08, (then same as the log request flow of device 1)
</div>

### Log Storage and Management on the Cloud Server

#### Log Request Data

let say we are having these meta data for the request

Log Request Metadata: This metadata is associated with each log retrieval request initiated by the operator. It includes information such as:

- request_id: A unique identifier for the request
- device_id: The identifier of the edge device
- start_time and end_time: The specified time range for log retrieval
- filter_criteria: Any applied filters (e.g., log level, keywords)
- status: The current status of the request (e.g., pending, in_progress, completed, error)
- requested_by: The identifier of the operator who initiated the request
- timestamp: Timestamp when the request was created
- Other optional metadata (e.g., completion time, error details)

#### Potential Database

Given the log request metadata's characteristics and the primary access pattern of querying by `request_id`, suitable database options include:

**1. Key-Value Store (KVS)**

* Examples: Redis, Amazon DynamoDB, Azure Cosmos DB (with key-value API)
* Advantages:
    * Optimized for fast key-value lookups (efficient `request_id` retrieval)
    * Built-in Time-To-Live (TTL) for automatic expiration of old data
    * Simple data model and implementation
    * Scalable for high read/write throughput

* Considerations:
    * Limited querying/filtering beyond key-based lookups; might need additional indexing or a hybrid approach for complex queries

**2. Relational Database (SQL)**

* Examples: PostgreSQL, MySQL, MariaDB
* Advantages:
    * Mature technology with extensive tooling and support
    * Strong ACID compliance for data integrity
    * Flexible querying and filtering using SQL

* Considerations:
    * Might have slightly higher overhead than KVS for simple lookups
    * Requires schema design and indexing for optimal performance

**3. Document Database**

* Examples: MongoDB, Couchbase, Amazon DocumentDB
* Advantages:
    * Flexible schema for semi-structured or evolving data
    * Handles a mix of structured and unstructured metadata
    * Scalable for large data volumes and concurrent requests

* Considerations:
    * Might not be as performant as KVS for key-value lookups
    * Querying/filtering can be more complex than SQL

**Recommendation:**

* **Primary Use Case: Retrieval by `request_id` and temporary data retention:**  KVS with TTL is a strong option due to its simplicity, efficiency, and automatic expiration.
* **Flexibility and Future Enhancements:** If you foresee complex queries, reporting, or evolving metadata structures, a relational database might be better.

#### Summary of Log Processing and Storage Architecture

**MQTT to Message Queue:**

The IoT device publishes log messages to the MQTT broker. The broker then forwards these messages to a message queue, ensuring reliable delivery and decoupling the device from the processing stage.

**Processor and Storage:**

The processor consumes messages from the queue, potentially processing them further. It then reassembles the log chunks into complete log files, maintaining their chronological order. These reassembled log files are then stored in either:

    S3: For cost-effective and scalable storage, especially for simpler log retrieval scenarios.
    Text Search Engine: For advanced search, filtering, and analytics capabilities.

#### S3 Folder Strcuture

```
s3://log-bucket/
├── requests/
│   ├── <request_UUID_1>/
│   │   ├── 2024-09-10_05-00.json.gz
│   │   └── 2024-09-10_06-00.json.gz 
│   └── ...
└── ...
```

In this structure:

- Each file within a request directory represents a processed log file, either containing non-error logs or error logs (with the _error suffix).

- The files can be either:

    * Compressed: using gzip (.json.gz) to save storage space
    * Uncompressed: (.json) if the subsequent analysis or retrieval tools require uncompressed text data

The choice between compressed or uncompressed storage depends on the trade-offs between storage costs, retrieval speed, and the capabilities of the tools used to access and analyze the logs.

#### Applying a Text Search Engine (e.g., Elasticsearch) (Optional)

Instead of (or in addition to) storing log chunks directly in S3, the processor can index them into a text search engine like Elasticsearch.

1. Parsing and Indexing: The processor parses each log chunk, extracts relevant fields (timestamp, log level, device ID, message, etc.), and indexes this structured data into Elasticsearch.

2. Querying: When the operator requests logs, the server translates the query criteria into an Elasticsearch query. Elasticsearch performs the search and filtering, returning the matching log entries.

#### Potential Processing and Filtering Tasks within the Processor (Optional)

* **Log Formatting and Normalization**
    * Standardize the format of log entries from different devices or sources
    * Convert timestamps to a unified format
    * Normalize log levels or other fields

* **Data Enrichment**
    * Add metadata or context to log entries:
        * Device location or geographic information
        * Device type or model
        * User or session information (if applicable)
        * Any other relevant contextual data

* **Filtering and Sanitization**
    * Filter out sensitive or personally identifiable information (PII)
    * Remove unnecessary or redundant log entries
    * Apply complex filtering based on keywords, patterns, or structured data

* **Aggregation and Summarization**
    * Aggregate or summarize log data to extract insights or trends
    * Calculate metrics or statistics
    * Generate reports or alerts based on patterns or thresholds

#### Determining Request Completion

* **Chunk-Based Upload with QoS:**
    * IoT device publishes log chunks to MQTT with QoS 1 or 2.
    * Chunk payload includes `is_last_chunk` flag to signal end of file upload.

* **Determining Completion:**
    * Processor tracks received chunks.
    * File upload complete when chunk with `is_last_chunk` set to `true` is received.
    * Request complete when all expected files are uploaded and processed.

* **Notification:**
    * Upon completion, processor notifies operator or publishes completion message to MQTT.

#### Accessing the Logs 

The operator can access the processed logs through the following methods:

* **API:**
   * A dedicated API can be provided to allow programmatic access to the logs. 
   * The operator can specify the request ID, time range, and any filtering criteria to retrieve the desired logs.
   * This approach is suitable for automated log analysis or integration with other systems.

* **S3 Commands:**
   * If the logs are stored in S3, the operator can use the AWS CLI or other S3 compatible tools to directly access and download the log files. 
   * They can use the `aws s3 cp` or `aws s3 sync` commands to download specific files or entire directories based on the request ID and file naming conventions.
   * This approach provides flexibility and direct access to the raw log files.

* **Web Interface (Optional):**
   * A user-friendly web interface can be developed to allow the operator to browse, search, and filter logs interactively.
   * The interface can provide visualizations, dashboards, and other tools to aid in log analysis and troubleshooting.

The choice of access method depends on the operator's preferences and the specific use case. The API offers programmatic access, S3 commands provide direct control, and a web interface offers a more user-friendly experience.