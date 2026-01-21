# Thunder Module

## What This Module Does

Thunder is X's **in-memory post store**. It keeps recent posts in memory so they can be retrieved quickly. When Home Mixer needs posts from accounts you follow, Thunder provides them almost instantly because they're already in memory - no need to query a slow database.

Think of Thunder as a high-speed cache of recent posts, organized by author.

## How Thunder Fits in the System

```
┌─────────────────────────────────────────────────────────────────────┐
│                           KAFKA                                     │
│                    (Real-time post stream)                          │
│                                                                     │
│  New Post → Topic → Thunder receives it → Stores in memory          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          THUNDER                                     │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                        PostStore                             │   │
│  │                                                              │   │
│  │  Author 123: [Post A, Post B, Post C]                       │   │
│  │  Author 456: [Post D, Post E]                               │   │
│  │  Author 789: [Post F, Post G, Post H, Post I]               │   │
│  │  ...                                                         │   │
│  │                                                              │   │
│  │  (Millions of posts organized by author)                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ "Get posts from users [123, 456, 789]"
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        HOME MIXER                                    │
│                                                                     │
│  Thunder Source requests posts → Receives ~200 recent posts        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## The Files in This Module

### Core Files

| File | Purpose |
|------|---------|
| `lib.rs` | Module entry point - lists all submodules |
| `main.rs` | Starts the Thunder server |
| `thunder_service.rs` | Handles gRPC requests for posts |

### Data Storage

| File | Purpose |
|------|---------|
| `posts/post_store.rs` | In-memory storage of posts by author |
| `posts/mod.rs` | Posts module entry point |
| `deserializer.rs` | Converts Kafka messages into posts |
| `schema.rs` | Defines the data structures for posts |

### Kafka Integration

| File | Purpose |
|------|---------|
| `kafka_utils.rs` | Utilities for Kafka connection |
| `kafka/mod.rs` | Kafka module entry point |
| `kafka/tweet_events_listener.rs` | Listens for new posts from Kafka |
| `kafka/tweet_events_listener_v2.rs` | Updated listener implementation |
| `kafka/utils.rs` | Kafka helper functions |

### Support Files

| File | Purpose |
|------|---------|
| `strato_client.rs` | Fetches user's following list |
| `config.rs` | Configuration constants |
| `args.rs` | Command-line argument definitions |
| `metrics.rs` | Performance monitoring |
| `o2.rs` | Internal tooling integration |

## How Thunder Works

### 1. Receiving Posts (Kafka Listener)

```
Kafka Topic (tweet events)
          │
          ▼
┌─────────────────────────────┐
│  Kafka Listener             │
│                             │
│  1. Receive message         │
│  2. Deserialize to Post     │
│  3. Store in PostStore      │
└─────────────────────────────┘
          │
          ▼
┌─────────────────────────────┐
│  PostStore                  │
│                             │
│  Index by author_id         │
│  Keep only recent posts     │
│  Auto-trim old posts        │
└─────────────────────────────┘
```

### 2. Serving Posts (gRPC Service)

```
Home Mixer Request
"Get posts from users [A, B, C], exclude posts [1, 2, 3]"
          │
          ▼
┌─────────────────────────────┐
│  ThunderService             │
│                             │
│  1. Check rate limits       │
│  2. Look up following list  │
│  3. Query PostStore         │
│  4. Filter excluded posts   │
│  5. Sort by recency         │
│  6. Return top N            │
└─────────────────────────────┘
          │
          ▼
Response: [Post D, Post F, Post A, ...]
```

### 3. Memory Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                        PostStore Memory                              │
│                                                                     │
│  Retention Period: 48 hours                                        │
│                                                                     │
│  Every 2 minutes:                                                  │
│  ├─ Scan all posts                                                 │
│  ├─ Remove posts older than 48 hours                               │
│  └─ Free memory                                                    │
│                                                                     │
│  Before:                    After trim:                            │
│  [Post 1 - 72h ago] ───→   (deleted)                              │
│  [Post 2 - 50h ago] ───→   (deleted)                              │
│  [Post 3 - 24h ago] ───→   [Post 3 - 24h ago]                     │
│  [Post 4 - 1h ago]  ───→   [Post 4 - 1h ago]                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Key Concepts

### PostStore

The central data structure. Stores posts indexed by author_id for fast lookup.

```rust
// Conceptually:
PostStore {
    posts_by_author: HashMap<AuthorId, Vec<Post>>,
    retention_seconds: u64,  // How long to keep posts (e.g., 48 hours)
}
```

### LightPost

A lightweight representation of a post for network transfer:

```rust
LightPost {
    tweet_id: u64,        // Post ID
    author_id: u64,       // Who wrote it
    created_at: i64,      // Timestamp
    is_reply: bool,       // Is this a reply?
    is_video: bool,       // Does it have video?
    // ... minimal fields for efficiency
}
```

### Semaphore Rate Limiting

Thunder uses semaphores to prevent overload:

```rust
// Only N requests can run at once
let permit = semaphore.try_acquire();
if permit.is_err() {
    return Err("Server at capacity, please retry");
}
// Process request...
// Permit is automatically released when done
```

## Request Flow Example

```
1. Home Mixer calls Thunder:
   GetInNetworkPostsRequest {
       user_id: 12345,
       following_user_ids: [100, 200, 300, ...],  // 500 followed accounts
       exclude_tweet_ids: [999, 888, 777, ...],   // Already seen posts
       max_results: 200,
   }

2. Thunder processes:
   a. Acquire semaphore permit (rate limiting)
   b. If following_user_ids empty, fetch from Strato
   c. Query PostStore for posts from those authors
   d. Filter out excluded posts
   e. Sort by created_at (newest first)
   f. Take top 200

3. Thunder returns:
   GetInNetworkPostsResponse {
       posts: [
           LightPost { tweet_id: 456, author_id: 100, created_at: 1705100000 },
           LightPost { tweet_id: 457, author_id: 200, created_at: 1705099000 },
           // ... 198 more posts
       ]
   }
```

## Metrics Collected

Thunder tracks extensive metrics:

| Metric | What It Measures |
|--------|------------------|
| `GET_IN_NETWORK_POSTS_COUNT` | Posts returned per request |
| `GET_IN_NETWORK_POSTS_DURATION` | Total request latency |
| `GET_IN_NETWORK_POSTS_FOLLOWING_SIZE` | Following list size |
| `GET_IN_NETWORK_POSTS_FOUND_FRESHNESS_SECONDS` | Age of newest post |
| `GET_IN_NETWORK_POSTS_FOUND_UNIQUE_AUTHORS` | Author diversity |
| `IN_FLIGHT_REQUESTS` | Current concurrent requests |
| `REJECTED_REQUESTS` | Requests rejected due to capacity |

## Why Thunder Exists

Without Thunder, every feed request would need to:
1. Query a database for all followed users' posts
2. Wait for database I/O
3. Process potentially millions of posts

With Thunder:
1. Posts are already in memory
2. Lookup is nearly instant (microseconds, not milliseconds)
3. Pre-organized by author for efficient querying

This makes the "in-network" portion of your feed fast to generate.

## Reading Order

For best understanding, read the files in this order:

1. `lib.rs` - See what's available
2. `main.rs` - How the server starts
3. `posts/post_store.rs` - How posts are stored
4. `thunder_service.rs` - How requests are handled
5. `kafka/tweet_events_listener.rs` - How posts arrive
6. `deserializer.rs` - How posts are parsed
