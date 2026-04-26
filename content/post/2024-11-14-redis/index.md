+++
author = "yuhao"
title = "Redis Essentials: Features, Setup, and Operations"
date = "2024-11-14"
description = "Redis is a distributed in-memory NoSQL database supporting persistence. It offers rich data structures including strings, lists, sets, sorted sets, and hashes."
tags = [
    "Database",
    "Redis",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
## Introduction


Redis is a distributed in-memory NoSQL database supporting persistence. It offers rich data structures including strings, lists, sets, sorted sets, and hashes.

**Key Features**:

- Memory-based with optional disk persistence
- Distributed architecture support
- Multi-data structure support
- Pub/Sub messaging system

Official Documentation: [http://redisdoc.com](http://redisdoc.com)

---

## Use Cases

1. **Session Storage**: Persistent login sessions
2. **Rankings/Counters**: Real-time leaderboards, article view counters
3. **Message Queues**: Task queue systems (e.g., Celery integration)
4. **Real-time Metrics**: Active user tracking
5. **Data Caching**: Frequently accessed data (e.g., forum sections)
6. **Social Features**: Friend relationships (used by Weibo)
7. **Pub/Sub Systems**: Chat applications

---

## Redis vs Memcached

| Feature                | Memcached               | Redis                          |
|------------------------|-------------------------|---------------------------------|
| Storage Type           | Pure In-Memory          | Memory + Disk Sync             |
| Data Types             | Fixed Value Types       | Multiple Structures             |
| Virtual Memory          | ❌ Not Supported         | ✔️ Supported                   |
| Data Persistence        | ❌ No                   | ✔️ Dump File Backup            |
| Disaster Recovery       | ❌ No                   | ✔️ Memory Restoration          |
| Pub/Sub                | ❌ No                   | ✔️ Supported                   |

---

## Installation Guides

### Windows Systems

1. Download from [Microsoft Archive](https://github.com/MicrosoftArchive/redis/releases)
2. Run:
   ```cmd
   redis-server.exe redis.windows.conf
   ```
3. Connect:
   ```cmd
   redis-cli
   ```

### Ubuntu Systems

1. Install:
   ```bash
   sudo apt-get install redis-server
   ```
2. Start/Stop:
   ```bash
   sudo service redis-server start
   sudo service redis-server stop
   ```

---

## Remote Access Configuration

Modify `redis.conf`:

```conf
bind 0.0.0.0  # Allow all network interfaces
```

---

## Core Operations

### Basic Commands

```redis
SET key value          # Add key-value pair
DEL key               # Delete key
EXPIRE key 60         # Set 60-second expiration
TTL key               # Check remaining expiration
KEYS *                # List all keys
```

### List Operations

```redis
LPUSH list1 "item1"    # Add to list head
RPUSH list1 "item2"   # Add to list tail
LRANGE list1 0 -1     # Get all elements
LREM list1 2 "value"  # Remove two occurrences
LLEN list1            # Get list length
```

### Set Operations

```redis
SADD set1 "A" "B"      # Add elements
SMEMBERS set1          # View all elements
SREM set1 "A"          # Remove element
SINTER set1 set2       # Find intersections
```

### Hash Operations

```redis
HSET user:1001 name "John"  # Create hash
HGET user:1001 name        # Get value
HGETALL user:1001         # Get all fields
HDEL user:1001 email      # Delete field
```

---

## Advanced Features

### Transactions

```redis
MULTI                  # Start transaction
SET balance 100
SET credit 50
EXEC                   # Commit transaction
DISCARD                # Cancel transaction
WATCH key1             # Monitor key changes
```

### Pub/Sub System

```redis
PUBLISH news "Update"  # Send message to channel
SUBSCRIBE news         # Receive messages
```

---

## Persistence Configuration

Enable in `redis.conf`:

```conf
save 900 1     # Save if 1+ changes in 15min
save 300 10    # Save if 10+ changes in 5min
```

---

## Best Practices

1. Use connection pooling for high traffic
2. Monitor memory usage with `INFO MEMORY`
3. Implement proper key expiration policies
4. Use AOF (Append-Only File) for critical data
5. Regularly backup RDB snapshots

Command Reference:

```bash
redis-cli INFO SERVER   # View server metrics
BGSAVE                  # Create background snapshot
```

