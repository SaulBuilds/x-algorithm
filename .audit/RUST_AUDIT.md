# Rust Code Audit

## 1. Performance Optimizations

### 1.1 Unnecessary Allocations & Clones

| File | Line | Issue | Severity |
|------|------|-------|----------|
| `candidate-pipeline/candidate_pipeline.rs` | 61 | Full Vec clone before filtering | HIGH |
| `candidate-pipeline/candidate_pipeline.rs` | 247 | Backup clone on every filter (only needed on error) | HIGH |
| `candidate-pipeline/candidate_pipeline.rs` | 81-82 | Cloning Arc and Vec for async side effects | MEDIUM |

**Example - Line 61**:
```rust
.filter(&hydrated_query, hydrated_candidates.clone())  // Unnecessary clone
```

**Fix**: Use references or iterator patterns instead of cloning.

### 1.2 Missing Pre-allocation

| File | Line | Issue |
|------|------|-------|
| `thunder/posts/post_store.rs` | 239 | `Vec::new()` without capacity hint |
| `thunder/posts/post_store.rs` | 431 | Same issue in trim loop |

**Fix**: Use `Vec::with_capacity(expected_size)`.

### 1.3 O(n) Lookups in Tight Loops

**File**: `home-mixer/filters/author_socialgraph_filter.rs:31-32`
```rust
let muted = viewer_muted_user_ids.contains(&author_id);   // O(n) Vec lookup
let blocked = viewer_blocked_user_ids.contains(&author_id);
```

**Fix**: Convert to `HashSet` for O(1) lookups when list > 10 items.

### 1.4 Lock Contention

**File**: `thunder/kafka/tweet_events_listener_v2.rs:183-185`
```rust
let mut consumer_lock = consumer.write().await;
consumer_lock.poll(batch_size).await  // Holding write lock across async
```

**Fix**: Minimize lock scope, only hold during actual mutation.

### 1.5 Missing Caching

**File**: `home-mixer/candidate_hydrators/gizmoduck_hydrator.rs:28-37`
- Multiple intermediate allocations for user ID conversions
- No request-scoped cache for user data

**File**: `home-mixer/filters/muted_keyword_filter.rs:38-40`
- Tokenizer created fresh every filter run
- Should cache tokenized muted keywords per session

---

## 2. Pipeline Parallelization

### 2.1 Sequential Filters That Could Run in Parallel

**File**: `candidate-pipeline/candidate_pipeline.rs:219-273`

Current design runs all filters sequentially. These filters are independent:
- `DropDuplicatesFilter`
- `AgeFilter`
- `SelfTweetFilter`

**Fix**: Group independent filters and run with `join_all()`.

### 2.2 Blocking Operations in Async Context

**File**: `thunder/posts/post_store.rs:417-420`
```rust
tokio::task::spawn_blocking(move || {
    let current_time = std::time::SystemTime::now()...
```

The entire trim operation blocks. `SystemTime::now()` is fast but iteration could be async-friendly.

### 2.3 Fire-and-Forget Side Effects

**File**: `candidate-pipeline/candidate_pipeline.rs:321`
```rust
tokio::spawn(async move {
    let _ = join_all(futures).await;  // No tracking, no timeout, no retry
});
```

**Issues**:
- Side effects run without completion tracking
- Cache writes could silently fail
- No circuit breaker pattern

---

## 3. Event Tracking & Observability Gaps

| Location | Missing Observability |
|----------|----------------------|
| `home-mixer/server.rs:50` | No Phoenix retrieval latency metrics |
| `candidate-pipeline/candidate_pipeline.rs:95-123` | Query hydration failures only logged, not metricked |
| `thunder/thunder_service.rs:279-314` | No per-filter exclusion reason tracking |

**Recommendation**: Add structured metrics at each pipeline stage:
- Source fetch latency + failure rate
- Per-filter exclusion counts
- Scorer latency distributions
- Side effect success/failure rates

---

## 4. Error Handling Issues

### 4.1 Swallowed Errors

| File | Line | Issue |
|------|------|-------|
| `candidate-pipeline/candidate_pipeline.rs` | 326 | `let _ = join_all(...)` discards errors |
| `thunder/kafka/tweet_events_listener_v2.rs` | 222 | JoinHandle result ignored |

### 4.2 Panics on Recoverable Errors

| File | Line | Issue |
|------|------|-------|
| `thunder/kafka/tweet_events_listener.rs` | 101-104 | `panic!` on Kafka failure |
| `thunder/kafka/tweet_events_listener_v2.rs` | 101-105 | `panic!` on thread exit |
| `thunder/posts/post_store.rs` | 475 | `.expect()` on spawn_blocking |
| `thunder/kafka/tweet_events_listener.rs` | 210 | `.unwrap()` on event deserialization |

**Impact**: Single partition failure brings down entire feeder.

### 4.3 Missing Retry Logic

| File | Line | Issue |
|------|------|-------|
| `home-mixer/sources/phoenix_source.rs` | 21-33 | Single attempt, no retry |
| `home-mixer/sources/thunder_source.rs` | 38-41 | No retry, no fallback |

**Fix**: Implement exponential backoff with jitter for transient failures.

---

## 5. Hardcoded Configuration

### 5.1 CRITICAL: Empty Kafka Topics

**File**: `thunder/kafka_utils.rs:15-19`
```rust
const TWEET_EVENT_TOPIC: &str = "";
const TWEET_EVENT_DEST: &str = "";
const IN_NETWORK_EVENTS_DEST: &str = "";
const IN_NETWORK_EVENTS_TOPIC: &str = "";
```

**Impact**: Thunder cannot connect to Kafka.

### 5.2 Other Hardcoded Values

| File | Line | Value |
|------|------|-------|
| `thunder/posts/post_store.rs` | 524 | 2-day default retention |
| `thunder/thunder_service.rs` | 56 | Zstd compression only |

---

## 6. Numerical Issues

### 6.1 Division by Zero Risk

**File**: `thunder/thunder_service.rs:121`
```rust
let posts_per_author = posts.len() as f64 / unique_author_count as f64;
```

No check for `unique_author_count == 0`.

### 6.2 NaN Score Handling

**File**: `candidate-pipeline/candidate_pipeline.rs:32`
```rust
.partial_cmp(&self.score(a)).unwrap_or(std::cmp::Ordering::Equal)
```

NaN scores compare as Equal, breaking sort stability.

---

## 7. Race Conditions

**File**: `thunder/posts/post_store.rs:103-112`
```rust
pub async fn finalize_init(&self) -> Result<()> {
    self.sort_all_user_posts().await;
    self.trim_old_posts().await;
    for entry in self.deleted_posts.iter() {
        self.posts.remove(entry.key());  // Race with concurrent reads
    }
```

Between `trim_old_posts()` and removal, requests can see stale data.

---

## Summary Table

| Category | Count | Critical | High | Medium |
|----------|-------|----------|------|--------|
| Performance | 12 | 0 | 3 | 9 |
| Parallelization | 4 | 0 | 2 | 2 |
| Error Handling | 8 | 1 | 4 | 3 |
| Configuration | 5 | 1 | 1 | 3 |
| Numerical | 2 | 0 | 0 | 2 |
| Race Conditions | 1 | 0 | 1 | 0 |
