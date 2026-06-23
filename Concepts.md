# Platform Concepts

## Redis vs Memcached

| Feature | Memcached | Redis |
|---|---|---|
| **Primary Use** | Ephemeral, simple key-value caching. | Complex caching, messaging, session management, and real-time data. |
| **Data Types** | Strings and integers only. | Strings, Hashes, Lists, Sets, Sorted Sets, Bitmaps, and Geospatial indexes. |
| **Performance** | Higher read/write throughput for basic blobs of data. | Excellent read/write performance with low latency; may experience slightly more overhead due to advanced features. |
| **Data Persistence** | None — data is lost on restart. | Optional but robust persistence via RDB snapshots and AOF logs. |
| **Architecture** | Multi-threaded; scales across multiple CPU cores. | Single-threaded core (for most commands); supports I/O threading and clustering. |


