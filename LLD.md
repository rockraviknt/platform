what operations does the system support? 
what can go wrong? 
what's in scope?
What scale are we targeting? Dozens of files, or potentially thousands?
what might we extend later?



Implementation:
Define the core logic - The happy path when everything works correctly
Handle edge cases - Invalid inputs, boundary conditions, operations that can fail
Ask focused questions to define scope, constraints, and success criteria.

Ensure immutability through constructor
private final or static

How do you prevent overselling when orders are in progress in real production systems?

How would you handle double booking?
If contention on a single showtime is genuinely your bottleneck

How would you handle temporary seat holds during checkout?
When a user selects seats, we reserve them for a limited time (say, 5 minutes). During that window, those seats appear as unavailable to everyone else. If the user completes payment within the window, the hold converts into a confirmed reservation. If they abandon checkout or the timer expires, the seats release back into the pool.


Yes, immutable objects(deep copy) are inherently thread-safe. Because their internal state cannot be modified after creation, it is impossible for multiple threads to conflict or cause data corruption while reading them.Why Immutability Guarantees Thread SafetyNo Race Conditions: 
Race conditions occur when one thread writes data while another reads or writes it. With immutable objects, no thread can perform write operations.No Synchronization Needed: You do not need to use locks, synchronized blocks, or complex concurrency utilities to read immutable data.Safe Sharing: Multiple threads can freely share references to the same instance without risk. Common standard library examples include Java's String and Integer.


 should throw exceptions. Use specific exception types so callers can handle different failure modes appropriately."



 Code Review 
 Testing approaches
 Databse optimization (N+1) queries
 Kafka
 Monitoring metrices
 Communication protocols
 Authentication and Authorization
 Security

 HTTP/2 multiplexing,bidirectional streaming.real-time communication systems
 gRPC is a modern, high-performance Remote Procedure Call (RPC) framework developed by Google. It enables efficient communication between client and server applications across different environments. In this article, we’ll walk through building a simple gRPC client and server application using Spring Boot 3. By the end, you’ll have a working example of a gRPC service that allows a client to send a greeting request to a server and receive a response.

What is gRPC and Protobuf?
gRPC is an open-source RPC framework that uses Protocol Buffers (Protobuf) as its interface definition language (IDL) and message format. It is built on top of HTTP/2, which provides features like multiplexing, header compression, and bidirectional streaming. gRPC is widely used in microservices architectures, real-time communication systems, and cloud-native applications due to its efficiency, cross-platform support, and strongly-typed contracts.

Protobuf is like a secret code that helps computers talk to each other using very short and simple messages. Imagine you and your friend make a list of lunch items, and instead of saying the full names like “sandwich” or “banana,” you use numbers like 1 for sandwich and 2 for banana. So, instead of saying a long sentence, you just say “1, 2, 3.” It’s much faster and takes up less space. That’s what Protobuf does — it turns big messages into small codes that are easy and quick for computers to understand and send

gRPC is like a super-fast mailman for computers. It helps two apps talk to each other quickly and clearly, like using walkie-talkies instead of slow letters. It knows where to deliver the message and what to do with it.



Rate limiting prevents abuse by restricting how many requests a client can make in a given time frame.
retry with exponential backoff


Modern SQL databases scale with techniques like read replicas, sharding, connection pooling, and caching. 

Choosing DB
Schema flexibility

DynamoDB (Key value store)
Unpredictable Traffic & ultra Low latency for query -

Document Database - Rapidly Changing Schema, rapidly evolving applications do not know structure, highly nested structures, when different records have vastly different structures even they are same entity. MongoDB, Firestore, and CouchDB.


Wide-Column Databases: organize data into column families where rows can have different sets of columns. They're optimized for massive write-heavy workloads and time-series data. Cassandra(Amazon Keyspaces) and HBase.
 When you have enormous write volumes, time-series data, or analytics workloads where you primarily append data and run aggregations. Think telemetry, event logging, or IoT sensor data.


 Choosing a DB
 -Data volume:- determines where your data can physically live
 -Access patterns:- How will your data be queried?, Structured SQL with JOINs
 -Consistency requirements:- determine how tightly coupled your data can be. ACID compliance
 Semistructured or unstructured data
 single-digit millisecond latency for real-time tracking, gaming leaderboards, 
 Rapid Prototyping: Your product is changing rapidly every week, its schema not fixed



PostgreSQL:- can batch multiple transactions before flushing to disk 
 Indexes has associated cost:
  Every index we create requires additional disk space, sometimes nearly as much as the original data.
Write performance takes a hit too. When we insert a new row or update an existing one, the database must update not just the main table, but every index on it. With multiple indexes, a single write operation can trigger several disk writes.


100 M DAU, 20 RPS => 2B
DB read - 50 - 60 ms
Caching - single digit ms latency

500 million rows, each of 5kb => 2.5 TB of data
Single instance DB capacity: 20K WPS
Vertically scalablity => 50K WPS & 256 TB (Aurora)
TB data
Sharding 
first identify shard key, then how to distribute data?
Shard key-> high cardinality(many unique values, gives plenty room to distribute data), Evenly distribution across shards, Align with query access patterns.
Range based sharding- SaaS application where each client has range based userID's, in multi tenant systems
Challenges with Sharding:
uneven load, queries that span shards, and maintaining consistency across databases
Atomicity -> transaction span across multiple shards
use 2phase commit, avoid cross shard transactions
Saga pattern- Orchestration, chronlogical, compensating actions



Metrices: CPU & Memory usage, query latency, request voulume,

Scale Numbers:
S3 -> Peta bytes
Caching scalability need- reached to tera bytes scale, 100k WPS
Database-> Single instance Arora can store upto 256TB, Up to 50k WPS, 5-20k concurrent connections
When to propose Sharding:
-if DB growing upto 50TB storage and 10K WPS then we can go with sharding
-Geographic Distribution: Cross-region replication or distribution needs
-Backup/Recovery: Backup windows that stretch into hours or become operationally impractical

Kafka: Decouple, streaming/process Millions of message per second with Single digit ms latency, can store month of data, Message Size: 1KB-10MB
Throughput: Nearing 800k messages/second per broker
Partition Count: Approaching 200k per cluster

AWS SQS- Message size 1MB, high throughput, retentention upto 14 days
Maximum Visibility Timeout - 30 sec - 12 hours

RabbitMQ - more controll, cheap, management, operational burden, Message size upto 16MB
Maximum Visibility Timeout = heartbeat interval

Load balancers timeout: 50-60 sec

Async processing: Fast user response times, Independent scalling, Fault isolation, Better resource utilization by using CPU/Memory optimized workers or commodity hardware, idempotency
Retry with exponential backoff, if all exhausted then send to DLQ, you can enable auto redrive for DLQs 
Autoscale workers based on queue depth(Approximate number of messages)
Change data capture(CDC)

Clean up incomplete Multipart upload

Transaction isolation levels: how much transaction can see others T's inflight work.
Read uncommitted - High performance(Rarely used)
Read Committed-Postgres
Repeatable Reads - Mysql
Serializable- if conflict then other tranasaction to rollback and retry, this has cost

Conditional writes:- counter decrement, a status flip, or claiming a row, simple, no locking
Pessimistic locking- when contention is high
Optimistic concurrency control:- Better performance and no blocking when conflicts are infrequent/ rare
Isolation levels:- write skew, At least one doctor must be on call at all times.no single row to lock
Distributed locks:-reserving a seat during checkout
Combine Distributed lock with Optimistic locking: Redis lock expires too early due to network lag, the DB lock is your final line of defense


Read scalability:
Read: write ratio -> 10:1, 100:1
1. indexing, Denormalization
2. Vertically scale, Read replicas, horizontally scale DB
3. Introduce caching layer: cache invalidation, cache eviction

Backup and recovery:
RPO, RTO

Writing scalability:
1.Vertical Scaling and Database Choices
2.Sharding and Partitioning
3.Handling Bursts with Queues and Load Shedding
4.Batching and Hierarchical Aggregation


Apache Flink or Spark Streaming
