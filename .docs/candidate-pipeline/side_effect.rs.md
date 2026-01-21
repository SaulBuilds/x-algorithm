# side_effect.rs - Background Tasks

## File Location
`candidate-pipeline/side_effect.rs`

## Purpose
This file defines what a "Side Effect" is. Side effects are tasks that run after the main pipeline finishes but don't affect what posts are returned to you. They handle things like logging what posts were shown, updating caches, or recording analytics - important work that happens "on the side."

## Line-by-Line Explanation

```rust
use crate::util;
```
**Line 1**: Import helper functions from this package.

```rust
use std::any::type_name_of_val;
```
**Line 2**: Import the function that gets a value's type name.
- Used for logging to identify which side effect ran

```rust
use std::sync::Arc;
```
**Line 3**: Import `Arc` (Atomic Reference Counter).
- `Arc` = A smart pointer that allows multiple owners of the same data
- "Atomic" means it's safe to use across multiple threads
- Used to share data between the main thread and side effect threads

```rust
use tonic::async_trait;
```
**Line 4**: Import the async_trait helper.
- Enables defining traits with async functions

```rust
// A side-effect is an action run that doesn't affect the pipeline result from being returned
```
**Line 6**: Comment explaining what side effects are.
- Key insight: They don't change what gets returned to the user

```rust
#[derive(Clone)]
pub struct SideEffectInput<Q, C> {
    pub query: Arc<Q>,
    pub selected_candidates: Vec<C>,
}
```
**Lines 7-11**: Define the input structure for side effects.
- `#[derive(Clone)]` = Automatically implement cloning for this struct
- `pub struct SideEffectInput<Q, C>` = Public structure with generic types
- `pub query: Arc<Q>` = The query wrapped in Arc (shared ownership)
- `pub selected_candidates: Vec<C>` = The final list of selected posts

This structure contains everything a side effect needs to know:
- Who made the request (query)
- What posts were selected (candidates)

```rust
#[async_trait]
```
**Line 13**: Enable async functions in the trait below.

```rust
pub trait SideEffect<Q, C>: Send + Sync
```
**Line 14**: Define a public trait called `SideEffect`.
- `pub trait` = Public blueprint
- `SideEffect` = The name
- `<Q, C>` = Generic types (Q = Query, C = Candidate)
- `: Send + Sync` = Must be thread-safe

```rust
where
    Q: Clone + Send + Sync + 'static,
    C: Clone + Send + Sync + 'static,
```
**Lines 15-17**: Constraints on the generic types.
- Both Query and Candidate must be copyable and thread-safe

```rust
{
```
**Line 18**: Start of trait methods.

```rust
    /// Decide if this side-effect should be run
    fn enable(&self, _query: Arc<Q>) -> bool {
        true
    }
```
**Lines 19-22**: The `enable` method.
- `_query: Arc<Q>` = Takes an Arc-wrapped query (shared reference)
- Default: always enabled
- Can be overridden to only run in certain situations

Notice: The query is wrapped in `Arc`, not a regular reference. This is because side effects run in a separate thread.

```rust
    async fn run(&self, input: Arc<SideEffectInput<Q, C>>) -> Result<(), String>;
```
**Line 24**: The main `run` method.
- `async fn run` = Async function that can wait for things
- `input: Arc<SideEffectInput<Q, C>>` = Shared reference to the input data
- `-> Result<(), String>` = Returns:
  - Success: Nothing (empty tuple `()`)
  - Failure: An error message

No default implementation - every SideEffect MUST define what it does.

```rust
    fn name(&self) -> &'static str {
        util::short_type_name(type_name_of_val(self))
    }
```
**Lines 26-28**: The `name` method.
- Returns the side effect's name for logging
- Default implementation extracts the type name

```rust
}
```
**Line 29**: End of trait definition.

## Why Side Effects Use Arc

Side effects run in a separate background thread:

```
Main Thread                          Background Thread
     │                                      │
     │  Build feed                          │
     │      ↓                               │
     │  Return posts to user                │
     │      ↓                               │
     │  Spawn side effects ───────────────→ │
     │      ↓                               │  Run side effect 1
     │  DONE (user sees feed)               │  Run side effect 2
     │                                      │  Run side effect 3
                                            │      ↓
                                            │  DONE (logged, cached)
```

Because side effects run in a different thread, they can't use regular references (`&`). Instead, they use `Arc` which allows safe sharing between threads.

## What Side Effects Do

Side effects typically handle:

```
┌─────────────────────────────────────────────────────────────┐
│                     SIDE EFFECTS                             │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ CacheRequestInfoSideEffect                           │   │
│  │ - Save info about this request to a cache           │   │
│  │ - Used to avoid showing the same posts again        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ AnalyticsLoggingSideEffect                          │   │
│  │ - Record what posts were shown                      │   │
│  │ - Used for ML training and metrics                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ImpressionTrackingSideEffect                        │   │
│  │ - Note that these posts are about to be "served"    │   │
│  │ - Used to track served vs. seen posts               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Why Side Effects Don't Block

The key characteristic: **side effects don't delay the response to the user**.

Without side effects (blocking):
```
Request → [Process 100ms] → [Log 50ms] → Response
Total: 150ms
```

With side effects (non-blocking):
```
Request → [Process 100ms] → Response    (User sees this)
                    └→ [Log 50ms]       (Happens in background)
Total: 100ms (user only waits for processing)
```

## Example of a Real Side Effect

While this file only defines the blueprint, a real implementation might look like:

```rust
struct CacheRequestInfoSideEffect {
    cache: CacheClient,
}

impl SideEffect<Query, Candidate> for CacheRequestInfoSideEffect {
    async fn run(&self, input: Arc<SideEffectInput<Query, Candidate>>) -> Result<(), String> {
        // Get the post IDs that were selected
        let shown_post_ids: Vec<i64> = input
            .selected_candidates
            .iter()
            .map(|c| c.post_id)
            .collect();

        // Save to cache so we don't show these again soon
        self.cache.set(
            &format!("shown_posts:{}", input.query.user_id),
            &shown_post_ids,
            Duration::from_secs(3600),  // Expire in 1 hour
        ).await?;

        Ok(())
    }
}
```

Another example for analytics:

```rust
struct AnalyticsLoggingSideEffect {
    analytics: AnalyticsService,
}

impl SideEffect<Query, Candidate> for AnalyticsLoggingSideEffect {
    async fn run(&self, input: Arc<SideEffectInput<Query, Candidate>>) -> Result<(), String> {
        // Log each shown post for analytics
        self.analytics.log_feed_served(
            input.query.user_id,
            input.query.request_id.clone(),
            input.selected_candidates.iter().map(|c| FeedItem {
                post_id: c.post_id,
                score: c.final_score.unwrap_or(0.0),
                author_id: c.author_id,
                position: 0,  // Will be set elsewhere
            }).collect(),
        ).await?;

        Ok(())
    }
}
```

## Key Takeaways

1. **Side effects don't affect results** - They run after posts are selected
2. **They run in the background** - User doesn't wait for them
3. **They use Arc for thread safety** - Data is shared between threads
4. **Common uses**: Logging, caching, analytics, impression tracking
5. **Can fail silently** - Errors don't stop the feed from being returned
6. **Multiple run in parallel** - All enabled side effects execute together
