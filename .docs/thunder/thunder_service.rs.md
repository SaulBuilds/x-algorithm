# thunder_service.rs - Handling Post Requests

## File Location
`thunder/thunder_service.rs`

## Purpose
This file handles incoming requests for posts from followed accounts. When Home Mixer needs "in-network" posts (posts from accounts you follow), it calls Thunder, and this service provides them.

## Line-by-Line Explanation

### Imports (Lines 1-27)

```rust
use lazy_static::lazy_static;
```
**Line 1**: For creating global static variables.

```rust
use log::{debug, info, warn};
```
**Line 2**: Logging functions for different severity levels.

```rust
use std::cmp::Reverse;
```
**Line 3**: For sorting in reverse order (newest first).

```rust
use std::collections::HashSet;
```
**Line 4**: For efficient membership checking (exclude lists).

```rust
use tokio::sync::Semaphore;
```
**Line 7**: For rate limiting concurrent requests.

```rust
use xai_thunder_proto::{
    GetInNetworkPostsRequest, GetInNetworkPostsResponse, LightPost,
    in_network_posts_service_server::{InNetworkPostsService, InNetworkPostsServiceServer},
};
```
**Lines 10-13**: Protocol buffer types for requests and responses.

```rust
use crate::config::{
    MAX_INPUT_LIST_SIZE, MAX_POSTS_TO_RETURN, MAX_VIDEOS_TO_RETURN,
};
```
**Lines 15-17**: Configuration constants (limits on input/output sizes).

### The Service Structure (Lines 29-36)

```rust
pub struct ThunderServiceImpl {
    /// PostStore for retrieving posts by user ID
    post_store: Arc<PostStore>,
    /// StratoClient for fetching following lists when not provided
    strato_client: Arc<StratoClient>,
    /// Semaphore to limit concurrent requests and prevent overload
    request_semaphore: Arc<Semaphore>,
}
```
The service holds three components:
- `post_store` = Where posts are stored in memory
- `strato_client` = For fetching following lists
- `request_semaphore` = Limits how many requests can run at once

### Constructor (Lines 38-53)

```rust
pub fn new(
    post_store: Arc<PostStore>,
    strato_client: Arc<StratoClient>,
    max_concurrent_requests: usize,
) -> Self {
    Self {
        post_store,
        strato_client,
        request_semaphore: Arc::new(Semaphore::new(max_concurrent_requests)),
    }
}
```
Creates the service with a semaphore limiting concurrent requests.

### Server Creation (Lines 55-60)

```rust
pub fn server(self) -> InNetworkPostsServiceServer<Self> {
    InNetworkPostsServiceServer::new(self)
        .accept_compressed(tonic::codec::CompressionEncoding::Zstd)
        .send_compressed(tonic::codec::CompressionEncoding::Zstd)
}
```
Wraps the service in a gRPC server with Zstd compression.

### Statistics Analysis (Lines 62-148)

```rust
fn analyze_and_report_post_statistics(posts: &[LightPost], stage: &str) {
```
Analyzes posts and reports metrics. Calculates:
- Time since most recent post (freshness)
- Time range covered by posts
- Ratio of replies vs. original posts
- Number of unique authors
- Posts per author

### The Main Request Handler (Lines 151-331)

```rust
async fn get_in_network_posts(
    &self,
    request: Request<GetInNetworkPostsRequest>,
) -> Result<Response<GetInNetworkPostsResponse>, Status> {
```
**Lines 154-157**: Method signature for handling requests.

#### Rate Limiting (Lines 158-180)

```rust
let _permit = match self.request_semaphore.try_acquire() {
    Ok(permit) => {
        IN_FLIGHT_REQUESTS.inc();
        permit
    }
    Err(_) => {
        REJECTED_REQUESTS.inc();
        return Err(Status::resource_exhausted(
            "Server at capacity, please retry",
        ));
    }
};
```
**Lines 160-171**: Try to get a "permit" to process this request.
- If semaphore has capacity: proceed and increment in-flight counter
- If at capacity: reject immediately with "resource exhausted" error

```rust
struct InFlightGuard;
impl Drop for InFlightGuard {
    fn drop(&mut self) {
        IN_FLIGHT_REQUESTS.dec();
    }
}
let _in_flight_guard = InFlightGuard;
```
**Lines 174-180**: RAII guard to decrement counter when request completes.
- When `_in_flight_guard` goes out of scope (request done), counter decreases
- Works even if the function returns early due to an error

#### Request Parsing (Lines 182-194)

```rust
let req = request.into_inner();

if req.debug {
    info!(
        "Received GetInNetworkPosts request: user_id={}, following_count={}, exclude_tweet_ids={}",
        req.user_id,
        req.following_user_ids.len(),
        req.exclude_tweet_ids.len(),
    );
}
```
Extract the request and optionally log it (if debug mode).

#### Following List Handling (Lines 196-229)

```rust
let following_user_ids = if req.following_user_ids.is_empty() && req.debug {
    // Fetch from Strato if not provided
    match self.strato_client
        .fetch_following_list(req.user_id as i64, MAX_INPUT_LIST_SIZE as i32)
        .await
    {
        Ok(following_list) => {
            following_list.into_iter().map(|id| id as u64).collect()
        }
        Err(e) => {
            return Err(Status::internal(format!(
                "Failed to fetch following list: {}", e
            )));
        }
    }
} else {
    req.following_user_ids
};
```
If no following list was provided, fetch it from Strato (another service that stores who follows whom).

#### Limiting Input Sizes (Lines 248-272)

```rust
let following_count = following_user_ids.len();
if following_count > MAX_INPUT_LIST_SIZE {
    warn!(
        "Limiting following_user_ids from {} to {} entries for user {}",
        following_count, MAX_INPUT_LIST_SIZE, req.user_id
    );
}
let following_user_ids: Vec<u64> = following_user_ids
    .into_iter()
    .take(MAX_INPUT_LIST_SIZE)
    .collect();
```
Cap the input lists to prevent excessive processing.

#### Post Retrieval (Lines 274-314)

```rust
let proto_posts = tokio::task::spawn_blocking(move || {
    // Create exclude set for efficient filtering
    let exclude_tweet_ids: HashSet<i64> =
        exclude_tweet_ids.iter().map(|&id| id as i64).collect();

    // Fetch all posts for the followed users
    let all_posts: Vec<LightPost> = if req.is_video_request {
        post_store.get_videos_by_users(
            &following_user_ids,
            &exclude_tweet_ids,
            start_time,
            request_user_id,
        )
    } else {
        post_store.get_all_posts_by_users(
            &following_user_ids,
            &exclude_tweet_ids,
            start_time,
            request_user_id,
        )
    };

    // Analyze and score posts
    ThunderServiceImpl::analyze_and_report_post_statistics(&all_posts, "retrieved");
    let scored_posts = score_recent(all_posts, max_results);
    ThunderServiceImpl::analyze_and_report_post_statistics(&scored_posts, "scored");

    scored_posts
})
.await
.map_err(|e| Status::internal(format!("Failed to process posts: {}", e)))?;
```

**Key operations:**
1. `spawn_blocking` - Run CPU-intensive work on a separate thread
2. Convert exclude list to HashSet for O(1) lookup
3. Query PostStore for posts from followed users
4. Either get regular posts or video posts (based on request type)
5. Filter out excluded posts
6. Analyze statistics
7. Sort and limit results

#### Response (Lines 316-329)

```rust
GET_IN_NETWORK_POSTS_COUNT.observe(proto_posts.len() as f64);

let response = GetInNetworkPostsResponse { posts: proto_posts };

Ok(Response::new(response))
```
Record metrics and return the response.

### Scoring Function (Lines 333-339)

```rust
fn score_recent(mut light_posts: Vec<LightPost>, max_results: usize) -> Vec<LightPost> {
    light_posts.sort_unstable_by_key(|post| Reverse(post.created_at));
    light_posts.into_iter().take(max_results).collect()
}
```
Simple scoring: sort by `created_at` descending (newest first), take top N.

## Request Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INCOMING REQUEST                              │
│                                                                     │
│  GetInNetworkPostsRequest {                                        │
│    user_id: 12345,                                                 │
│    following_user_ids: [100, 200, 300, ...],                       │
│    exclude_tweet_ids: [999, 888, ...],                             │
│    max_results: 200,                                               │
│    is_video_request: false,                                        │
│  }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      RATE LIMITING                                   │
│                                                                     │
│  Semaphore permits available?                                      │
│  ├─ YES → Acquire permit, continue                                 │
│  └─ NO  → Return "resource exhausted" error                        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    INPUT VALIDATION                                  │
│                                                                     │
│  - If following_user_ids empty → Fetch from Strato                 │
│  - Limit lists to MAX_INPUT_LIST_SIZE                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     POST RETRIEVAL                                   │
│                    (spawn_blocking)                                  │
│                                                                     │
│  1. Build exclude HashSet                                          │
│  2. Query PostStore for each followed user                         │
│  3. Filter out excluded posts                                      │
│  4. Collect all posts                                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        SCORING                                       │
│                                                                     │
│  Sort by created_at (newest first)                                 │
│  Take top max_results                                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        RESPONSE                                      │
│                                                                     │
│  GetInNetworkPostsResponse {                                       │
│    posts: [                                                        │
│      LightPost { tweet_id: 456, author_id: 100, ... },            │
│      LightPost { tweet_id: 457, author_id: 200, ... },            │
│      ... (up to 200 posts)                                         │
│    ]                                                                │
│  }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

## Rate Limiting Explained

```
Server Capacity: 100 concurrent requests

Time 0:   [Permit 1] [Permit 2] ... [Permit 100] available
          └─────────────────────────────────────┘
                    100 permits

Request A arrives: Takes permit → 99 remaining
Request B arrives: Takes permit → 98 remaining
...
Request 100 arrives: Takes permit → 0 remaining

Request 101 arrives: No permits → REJECTED "resource exhausted"

Request A completes: Releases permit → 1 available
Request 101 retries: Takes permit → 0 remaining
```

## Why spawn_blocking?

```rust
let proto_posts = tokio::task::spawn_blocking(move || {
    // CPU-intensive work
}).await?;
```

Tokio's async runtime is designed for I/O operations. CPU-intensive work (like iterating over millions of posts) would block other tasks. `spawn_blocking` moves that work to a separate thread pool designed for blocking operations.

## Key Takeaways

1. **Rate limiting with semaphores** - Prevents server overload
2. **RAII guards** - Automatic cleanup when requests complete
3. **spawn_blocking** - Keeps async runtime responsive
4. **HashSet for excludes** - O(1) lookup instead of O(n) linear search
5. **Simple scoring** - Just sort by recency (ML scoring happens later in Home Mixer)
6. **Metrics everywhere** - Comprehensive observability
7. **Graceful degradation** - Falls back to Strato if following list not provided
