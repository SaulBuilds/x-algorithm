# filter.rs - Removing Unwanted Posts

## File Location
`candidate-pipeline/filter.rs`

## Purpose
This file defines what a "Filter" is. Filters examine posts and decide which ones should continue through the pipeline and which ones should be removed. Reasons to remove a post might include: you've already seen it, you've blocked the author, it contains muted keywords, or it violates content policies.

## Line-by-Line Explanation

```rust
use std::any::{Any, type_name_of_val};
```
**Line 1**: Import tools for working with types at runtime.
- `Any` = A type that can represent any other type
- `type_name_of_val` = Gets the name of a value's type

```rust
use tonic::async_trait;
```
**Line 2**: Import the async_trait helper from tonic.
- Enables defining traits with async functions

```rust
use crate::util;
```
**Line 4**: Import helper functions from this package.

```rust
pub struct FilterResult<C> {
    pub kept: Vec<C>,
    pub removed: Vec<C>,
}
```
**Lines 6-9**: Define a structure to hold filter results.
- `pub struct FilterResult<C>` = A public structure with generic type C (Candidate)
- `pub kept: Vec<C>` = A public vector of candidates that passed the filter
- `pub removed: Vec<C>` = A public vector of candidates that failed the filter

This structure clearly separates the two outcomes: posts we keep and posts we remove.

```rust
/// Filters run sequentially and partition candidates into kept and removed sets
```
**Line 11**: Documentation explaining filter behavior.
- "run sequentially" = One filter runs, then the next, etc. (not parallel like hydrators)
- "partition" = Divide into two groups

```rust
#[async_trait]
```
**Line 12**: Enable async functions in the trait below.

```rust
pub trait Filter<Q, C>: Any + Send + Sync
```
**Line 13**: Define a public trait called `Filter`.
- `pub trait` = Public blueprint
- `Filter` = The name
- `<Q, C>` = Generic types (Q = Query, C = Candidate)
- `: Any + Send + Sync` = Must be thread-safe and inspectable

```rust
where
    Q: Clone + Send + Sync + 'static,
    C: Clone + Send + Sync + 'static,
```
**Lines 14-16**: Constraints on the generic types.
- Both Query and Candidate must be copyable and thread-safe

```rust
{
```
**Line 17**: Start of trait methods.

```rust
    /// Decide if this filter should run for the given query
    fn enable(&self, _query: &Q) -> bool {
        true
    }
```
**Lines 18-21**: The `enable` method.
- Decides if this filter should run for this request
- Default: always enabled (returns `true`)
- Can be overridden for conditional filtering

```rust
    /// Filter candidates by evaluating each against some criteria.
    /// Returns a FilterResult containing kept candidates (which continue to the next stage)
    /// and removed candidates (which are excluded from further processing).
    async fn filter(&self, query: &Q, candidates: Vec<C>) -> Result<FilterResult<C>, String>;
```
**Lines 23-26**: The main `filter` method.
- `async fn filter` = Async function that can wait for network calls
- `query: &Q` = Reference to user/request info
- `candidates: Vec<C>` = The list of candidates to filter (takes ownership)
- `-> Result<FilterResult<C>, String>` = Returns:
  - Success: A FilterResult with kept and removed lists
  - Failure: An error message

Note: This method takes ownership of `candidates` (no `&`), meaning the filter gets to decide what happens to each post.

```rust
    /// Returns a stable name for logging/metrics.
    fn name(&self) -> &'static str {
        util::short_type_name(type_name_of_val(self))
    }
```
**Lines 28-31**: The `name` method.
- Returns the filter's name for logging
- Default implementation extracts the type name automatically

```rust
}
```
**Line 32**: End of trait definition.

## Why Filters Run Sequentially

Unlike hydrators that run in parallel, filters run one at a time. Here's why:

```
               Candidates: [A, B, C, D, E]
                        ↓
┌───────────────────────────────────────────────┐
│ Filter 1: Remove Already Seen                  │
│ Input:  [A, B, C, D, E]                        │
│ Output: kept=[A, C, E], removed=[B, D]         │
└───────────────────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────┐
│ Filter 2: Remove Blocked Authors               │
│ Input:  [A, C, E]     ← Only gets what passed │
│ Output: kept=[A, E], removed=[C]               │
└───────────────────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────┐
│ Filter 3: Remove Muted Keywords                │
│ Input:  [A, E]                                 │
│ Output: kept=[A, E], removed=[]                │
└───────────────────────────────────────────────┘
                        ↓
               Final: [A, E]
```

Each filter only processes what the previous filter kept. This is efficient because:
1. Later filters process fewer items
2. If something is already removed, we don't waste time checking it again

## The FilterResult Structure

The FilterResult serves two purposes:

1. **Pipeline continuation**: The `kept` list goes to the next stage
2. **Transparency**: The `removed` list can be logged for debugging or returned to the user

This is different from just returning the kept posts - we explicitly track what was removed and why.

## Example of a Real Filter

While this file only defines the blueprint, a real implementation might look like:

```rust
struct BlockedAuthorsFilter {
    // No fields needed - uses query data
}

impl Filter<Query, Candidate> for BlockedAuthorsFilter {
    async fn filter(&self, query: &Query, candidates: Vec<Candidate>) -> Result<FilterResult<Candidate>, String> {
        let blocked_authors = &query.user_blocked_authors;

        let (kept, removed): (Vec<_>, Vec<_>) = candidates
            .into_iter()
            .partition(|c| !blocked_authors.contains(&c.author_id));

        Ok(FilterResult { kept, removed })
    }
}
```

Another example with async operations:

```rust
struct ContentPolicyFilter {
    policy_service: PolicyService,
}

impl Filter<Query, Candidate> for ContentPolicyFilter {
    async fn filter(&self, query: &Query, candidates: Vec<Candidate>) -> Result<FilterResult<Candidate>, String> {
        // Batch check all posts against content policies
        let post_ids: Vec<_> = candidates.iter().map(|c| c.post_id).collect();
        let violations = self.policy_service.check_posts(post_ids).await?;

        let (kept, removed): (Vec<_>, Vec<_>) = candidates
            .into_iter()
            .partition(|c| !violations.contains(&c.post_id));

        Ok(FilterResult { kept, removed })
    }
}
```

## Key Takeaways

1. **Filters remove posts** - Unlike hydrators, they can reduce the candidate count
2. **They run sequentially** - One after another, each processing what the previous kept
3. **FilterResult tracks both outcomes** - We know what was kept and what was removed
4. **The filter takes ownership** - It receives the full list and partitions it
5. **Can be async** - Filters might need to call external services (content policy, etc.)
6. **Transparency for debugging** - Tracking removed items helps understand why posts didn't appear
