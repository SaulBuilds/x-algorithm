# Sprint 0: Critical Fixes

**Priority**: P0 - Production Blockers
**Status**: PLANNED

## Overview

These fixes must be applied before the system can function correctly. Without them, the model cannot learn and the pipeline will fail.

---

## Issue 1: Zero-Initialized Weights (CRITICAL)

### Problem
All `Linear` layers and `RMSNorm` scales are initialized to 0, causing the model to output zeros.

### Files
- `phoenix/grok.py:148-149` (Linear)
- `phoenix/grok.py:176-180` (RMSNorm)

### Fix

```python
# grok.py:148-149 - Linear class
# BEFORE:
w = hk.get_parameter(
    "w", [input_size, output_size], jnp.float32, init=hk.initializers.Constant(0)
)

# AFTER:
w = hk.get_parameter(
    "w", [input_size, output_size], jnp.float32,
    init=hk.initializers.VarianceScaling(1.0, "fan_avg", "truncated_normal")
)
```

```python
# grok.py:176-180 - RMSNorm class
# BEFORE:
scale = hk.get_parameter(
    "scale",
    param_shape,
    dtype=jnp.float32,
    init=hk.initializers.Constant(0),
)

# AFTER:
scale = hk.get_parameter(
    "scale",
    param_shape,
    dtype=jnp.float32,
    init=hk.initializers.Constant(1.0),  # Scale should be 1, not 0
)
```

### Validation
```python
# Test that model produces non-zero outputs
def test_model_not_dead():
    model = RecsysModel(config)
    batch = create_test_batch()
    output = model(batch)
    assert jnp.any(output.logits != 0), "Model outputs all zeros - initialization bug"
```

---

## Issue 2: Empty Kafka Topic Constants (CRITICAL)

### Problem
Kafka topic constants are empty strings, causing Thunder to fail connection.

### File
`thunder/kafka_utils.rs:15-19`

### Current Code
```rust
const TWEET_EVENT_TOPIC: &str = "";
const TWEET_EVENT_DEST: &str = "";
const IN_NETWORK_EVENTS_DEST: &str = "";
const IN_NETWORK_EVENTS_TOPIC: &str = "";
```

### Fix Option A: Environment Variables
```rust
use std::env;

lazy_static! {
    static ref TWEET_EVENT_TOPIC: String = env::var("TWEET_EVENT_TOPIC")
        .expect("TWEET_EVENT_TOPIC must be set");
    static ref TWEET_EVENT_DEST: String = env::var("TWEET_EVENT_DEST")
        .expect("TWEET_EVENT_DEST must be set");
    static ref IN_NETWORK_EVENTS_TOPIC: String = env::var("IN_NETWORK_EVENTS_TOPIC")
        .expect("IN_NETWORK_EVENTS_TOPIC must be set");
    static ref IN_NETWORK_EVENTS_DEST: String = env::var("IN_NETWORK_EVENTS_DEST")
        .expect("IN_NETWORK_EVENTS_DEST must be set");
}
```

### Fix Option B: CLI Arguments
```rust
#[derive(Parser)]
struct Args {
    #[arg(long, env = "TWEET_EVENT_TOPIC")]
    tweet_event_topic: String,

    #[arg(long, env = "TWEET_EVENT_DEST")]
    tweet_event_dest: String,

    #[arg(long, env = "IN_NETWORK_EVENTS_TOPIC")]
    in_network_events_topic: String,

    #[arg(long, env = "IN_NETWORK_EVENTS_DEST")]
    in_network_events_dest: String,
}
```

---

## Issue 3: Input Validation (HIGH)

### Problem
No bounds checking on API inputs allows DoS and model poisoning.

### Files
- `home-mixer/server.rs:33-35`
- `phoenix/recsys_model.py:349, 384-395`

### Rust Fix (server.rs)
```rust
const MAX_SEEN_IDS: usize = 10_000;
const MAX_SERVED_IDS: usize = 1_000;
const MAX_BLOOM_FILTER_SIZE: usize = 100_000;

pub async fn get_scored_posts(
    &self,
    request: Request<ScoredPostsQuery>,
) -> Result<Response<ScoredPostsResponse>, Status> {
    let query = request.into_inner();

    // Validate viewer_id
    if query.viewer_id == 0 {
        return Err(Status::invalid_argument("viewer_id cannot be 0"));
    }

    // Validate input sizes
    if query.seen_ids.len() > MAX_SEEN_IDS {
        return Err(Status::invalid_argument(format!(
            "seen_ids exceeds maximum of {}", MAX_SEEN_IDS
        )));
    }

    if query.served_ids.len() > MAX_SERVED_IDS {
        return Err(Status::invalid_argument(format!(
            "served_ids exceeds maximum of {}", MAX_SERVED_IDS
        )));
    }

    // Continue with validated inputs...
}
```

### Python Fix (recsys_model.py)
```python
def validate_batch(batch: RecsysBatch, config: RecsysModelConfig) -> None:
    """Validate batch inputs are within expected bounds."""
    # Check hash bounds
    if jnp.any(batch.user_hashes < 0):
        raise ValueError("user_hashes contains negative values")
    if jnp.any(batch.user_hashes >= config.num_user_embeddings):
        raise ValueError("user_hashes exceeds embedding table size")

    # Check action bounds
    if jnp.any(batch.history_actions < 0) or jnp.any(batch.history_actions > 1):
        raise ValueError("history_actions must be in [0, 1]")

    # Check product surface bounds
    if jnp.any(batch.history_product_surface < 0):
        raise ValueError("product_surface contains negative values")
    if jnp.any(batch.history_product_surface >= config.product_surface_vocab_size):
        raise ValueError("product_surface exceeds vocab size")
```

---

## Issue 4: Panic on Recoverable Errors (HIGH)

### Problem
Panics on Kafka failures kill entire service instead of graceful degradation.

### Files
- `thunder/kafka/tweet_events_listener.rs:101-104`
- `thunder/kafka/tweet_events_listener_v2.rs:101-105`

### Fix
```rust
// BEFORE:
panic!(
    "Tweet events processing thread {} exited unexpectedly: {:#}",
    thread_id, e
);

// AFTER:
error!(
    "Tweet events processing thread {} exited unexpectedly: {:#}. Attempting restart...",
    thread_id, e
);

// Attempt restart with backoff
let backoff = ExponentialBackoff::default();
retry(backoff, || {
    self.start_processing_thread(thread_id)
}).await.map_err(|e| {
    error!("Failed to restart thread {} after retries: {:#}", thread_id, e);
    // Mark service as unhealthy but don't panic
    self.health_check.set_unhealthy();
    e
})?;
```

---

## Issue 5: Division by Zero (MEDIUM)

### Problem
No check for zero unique authors before division.

### File
`thunder/thunder_service.rs:121`

### Fix
```rust
// BEFORE:
let posts_per_author = posts.len() as f64 / unique_author_count as f64;

// AFTER:
let posts_per_author = if unique_author_count > 0 {
    posts.len() as f64 / unique_author_count as f64
} else {
    0.0
};
```

---

## Validation Checklist

- [ ] Phoenix model produces non-zero outputs after weight initialization fix
- [ ] Thunder connects to Kafka with configured topics
- [ ] API rejects oversized inputs with appropriate error messages
- [ ] Python batch validation catches out-of-bounds hashes
- [ ] Service survives Kafka partition failure without crashing
- [ ] No division by zero in metrics calculation

---

## Rollout Plan

1. **Dev Environment**: Apply all fixes, run full test suite
2. **Staging**: Deploy with monitoring, verify Kafka connection
3. **Canary**: 1% traffic, monitor for errors
4. **Production**: Gradual rollout with kill switch
