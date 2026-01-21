# hydrator.rs - Adding Details to Posts

## File Location
`candidate-pipeline/hydrator.rs`

## Purpose
This file defines what a "Hydrator" is. Hydration means adding water to something dry - in this context, it means adding data to posts that only have basic information. A post might start with just an ID, and hydrators add the author info, engagement counts, media details, and more.

## Line-by-Line Explanation

```rust
use crate::util;
```
**Line 1**: Import the `util` module from this package.
- `crate` = This package
- `util` = Helper functions

```rust
use std::any::{Any, type_name_of_val};
```
**Line 2**: Import tools for working with types at runtime.
- `Any` = A type that can represent any other type
- `type_name_of_val` = Gets the name of a value's type

```rust
use tonic::async_trait;
```
**Line 3**: Import the async_trait helper.
- Allows defining traits with async functions

```rust
// Hydrators run in parallel and update candidate fields
```
**Line 5**: A comment explaining what hydrators do.
- "run in parallel" = Multiple hydrators work at the same time
- "update candidate fields" = Add data to posts

```rust
#[async_trait]
```
**Line 6**: Enable async functions in the trait below.

```rust
pub trait Hydrator<Q, C>: Any + Send + Sync
```
**Line 7**: Define a public trait called `Hydrator`.
- `pub trait` = Public blueprint
- `Hydrator` = The name
- `<Q, C>` = Generic types (Q = Query, C = Candidate)
- `: Any + Send + Sync` = Must be thread-safe and inspectable

```rust
where
    Q: Clone + Send + Sync + 'static,
    C: Clone + Send + Sync + 'static,
```
**Lines 8-10**: Constraints on the generic types.
- Both Query and Candidate must be copyable and thread-safe

```rust
{
```
**Line 11**: Start of trait methods.

```rust
    /// Decide if this hydrator should run for the given query
    fn enable(&self, _query: &Q) -> bool {
        true
    }
```
**Lines 12-15**: The `enable` method.
- Decides if this hydrator should run
- Default: always enabled (returns `true`)
- Implementations can override to be selective

```rust
    /// Hydrate candidates by performing async operations.
    /// Returns candidates with this hydrator's fields populated.
    ///
    /// IMPORTANT: The returned vector must have the same candidates in the same order as the input.
    /// Dropping candidates in a hydrator is not allowed - use a filter stage instead.
    async fn hydrate(&self, query: &Q, candidates: &[C]) -> Result<Vec<C>, String>;
```
**Lines 17-22**: The main `hydrate` method.
- `async fn hydrate` = Async function that can wait for network calls
- `query: &Q` = Reference to user/request info
- `candidates: &[C]` = Reference to a slice (list) of candidates
- `-> Result<Vec<C>, String>` = Returns success with candidates, or an error

**IMPORTANT RULE**: The returned list must:
- Have the same number of items as the input
- Have items in the same order
- NOT remove any candidates (that's what filters are for)

```rust
    /// Update a single candidate with the hydrated fields.
    /// Only the fields this hydrator is responsible for should be copied.
    fn update(&self, candidate: &mut C, hydrated: C);
```
**Lines 24-26**: The `update` method.
- `candidate: &mut C` = A mutable reference to the original candidate
- `hydrated: C` = The hydrated version with new data
- This copies specific fields from hydrated to candidate
- Each hydrator only updates the fields it's responsible for

This has no default implementation - every Hydrator MUST define how to update candidates.

```rust
    /// Update all candidates with the hydrated fields from `hydrated`.
    /// Default implementation iterates and calls `update` for each pair.
    fn update_all(&self, candidates: &mut [C], hydrated: Vec<C>) {
        for (c, h) in candidates.iter_mut().zip(hydrated) {
            self.update(c, h);
        }
    }
```
**Lines 28-34**: The `update_all` method.
- `candidates: &mut [C]` = Mutable reference to all original candidates
- `hydrated: Vec<C>` = All hydrated candidates
- Default implementation:
  - `candidates.iter_mut()` = Iterate over original candidates with mutation allowed
  - `.zip(hydrated)` = Pair each original with its hydrated version
  - `self.update(c, h)` = Call update on each pair

This provides a default way to update all candidates at once.

```rust
    fn name(&self) -> &'static str {
        util::short_type_name(type_name_of_val(self))
    }
```
**Lines 36-38**: The `name` method.
- Returns the hydrator's name for logging
- Default implementation extracts the type name automatically

```rust
}
```
**Line 39**: End of trait definition.

## How Hydrators Work in the Pipeline

```
Posts (ID only)           Hydrators (parallel)           Enriched Posts
┌───────────┐     ┌──────────────────────────────┐     ┌───────────────┐
│ Post 123  │ ──→ │ AuthorHydrator               │ ──→ │ Post 123      │
│           │     │ (adds author name, avatar)   │     │ + author info │
└───────────┘     │                              │     │ + engagement  │
                  │ EngagementHydrator           │     │ + media       │
                  │ (adds like/repost counts)    │     │ + ...         │
                  │                              │     └───────────────┘
                  │ MediaHydrator                │
                  │ (adds image/video details)   │
                  └──────────────────────────────┘
```

## Why "Must Not Remove Candidates"

The rule about not removing candidates is important for two reasons:

1. **Parallel execution**: All hydrators run at the same time on the same candidates. If one hydrator removed a candidate, others would have inconsistent data.

2. **Separation of concerns**: Hydrators add data. Filters remove candidates. Keeping these separate makes the code easier to understand and debug.

## Example of a Real Hydrator

While this file only defines the blueprint, a real implementation might look like:

```rust
struct AuthorHydrator {
    user_service: UserService,
}

impl Hydrator<Query, Candidate> for AuthorHydrator {
    async fn hydrate(&self, _query: &Query, candidates: &[Candidate]) -> Result<Vec<Candidate>, String> {
        // Get all unique author IDs
        let author_ids: Vec<_> = candidates.iter().map(|c| c.author_id).collect();

        // Fetch author info in one batch call
        let authors = self.user_service.get_users(author_ids).await?;

        // Return candidates with author info added
        Ok(candidates.iter().map(|c| {
            let mut new = c.clone();
            if let Some(author) = authors.get(&c.author_id) {
                new.author_name = Some(author.name.clone());
                new.author_avatar = Some(author.avatar.clone());
            }
            new
        }).collect())
    }

    fn update(&self, candidate: &mut Candidate, hydrated: Candidate) {
        // Only update author fields
        candidate.author_name = hydrated.author_name;
        candidate.author_avatar = hydrated.author_avatar;
    }
}
```

## Key Takeaways

1. **Hydrators add data to posts** - They don't filter or score, just enrich
2. **They run in parallel** - All hydrators work simultaneously for speed
3. **They must preserve order and count** - Input and output must match exactly
4. **Each hydrator owns specific fields** - The `update` method only copies its own fields
5. **The pattern is: hydrate then merge** - Compute new values, then merge into originals
