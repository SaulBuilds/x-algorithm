# query_hydrator.rs - Adding Details About the User

## File Location
`candidate-pipeline/query_hydrator.rs`

## Purpose
This file defines what a "Query Hydrator" is. While regular hydrators add data to posts, query hydrators add data to the user's request. Before finding posts, the system needs to know about you - your preferences, history, blocked accounts, etc. Query hydrators gather this information.

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
#[async_trait]
```
**Line 6**: Enable async functions in the trait below.

```rust
pub trait QueryHydrator<Q>: Any + Send + Sync
```
**Line 7**: Define a public trait called `QueryHydrator`.
- `pub trait` = Public blueprint
- `QueryHydrator` = The name
- `<Q>` = Generic type (Q = Query type only - no Candidate here!)
- `: Any + Send + Sync` = Must be thread-safe and inspectable

Notice: Unlike other traits, this only has `<Q>`, not `<Q, C>`. Query hydrators work on the query itself, not on candidates.

```rust
where
    Q: Clone + Send + Sync + 'static,
```
**Lines 8-9**: Constraints on the Query type.
- Must be copyable and thread-safe

```rust
{
```
**Line 10**: Start of trait methods.

```rust
    /// Decide if this query hydrator should run for the given query
    fn enable(&self, _query: &Q) -> bool {
        true
    }
```
**Lines 11-14**: The `enable` method.
- Decides if this hydrator should run
- Default: always enabled
- Can be overridden for conditional hydration

```rust
    /// Hydrate the query by performing async operations.
    /// Returns a new query with this hydrator's fields populated.
    async fn hydrate(&self, query: &Q) -> Result<Q, String>;
```
**Lines 16-18**: The main `hydrate` method.
- `async fn hydrate` = Async function that can wait for network calls
- `query: &Q` = Reference to the current query
- `-> Result<Q, String>` = Returns:
  - Success: A new query with additional data
  - Failure: An error message

This method takes the existing query and returns a new version with more information filled in.

```rust
    /// Update the query with the hydrated fields.
    /// Only the fields this hydrator is responsible for should be copied.
    fn update(&self, query: &mut Q, hydrated: Q);
```
**Lines 20-22**: The `update` method.
- `query: &mut Q` = A mutable reference to the original query
- `hydrated: Q` = The hydrated version with new data
- Copies specific fields from hydrated to query

No default implementation - every QueryHydrator MUST define this.

```rust
    fn name(&self) -> &'static str {
        util::short_type_name(type_name_of_val(self))
    }
```
**Lines 24-26**: The `name` method.
- Returns the hydrator's name for logging
- Default implementation extracts the type name

```rust
}
```
**Line 27**: End of trait definition.

## When Query Hydration Happens

Query hydration is the **first step** in the pipeline:

```
User opens X app
       ↓
┌─────────────────────────────────────────────────────────────┐
│ Initial Query (minimal info)                                 │
│ {                                                            │
│   user_id: 12345,                                           │
│   request_id: "abc-123",                                    │
│   device_type: "mobile",                                    │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
       ↓
       ↓  (Query Hydrators run in parallel)
       ↓
┌─────────────────────────────────────────────────────────────┐
│ Hydrated Query (full user context)                          │
│ {                                                            │
│   user_id: 12345,                                           │
│   request_id: "abc-123",                                    │
│   device_type: "mobile",                                    │
│                                                              │
│   // Added by UserFeaturesQueryHydrator:                    │
│   user_features: [interests, language, location, ...],      │
│                                                              │
│   // Added by UserActionSeqQueryHydrator:                   │
│   recent_actions: [liked post A, replied to B, ...],        │
│                                                              │
│   // Added by BlockedUsersQueryHydrator:                    │
│   blocked_users: [999, 888, 777],                           │
│                                                              │
│   // Added by MutedKeywordsQueryHydrator:                   │
│   muted_keywords: ["politics", "sports"],                   │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
       ↓
Now the pipeline knows everything about the user
and can find/filter/score posts appropriately
```

## Why Query Hydrators Run First

The enriched query is needed throughout the pipeline:

1. **Sources** use it to know whose posts to fetch
2. **Filters** use it to know what to remove (blocked users, muted keywords)
3. **Scorers** use it to personalize scores (user interests affect relevance)

Without query hydration, the pipeline would only know the user's ID - not enough to build a personalized feed.

## Example of a Real Query Hydrator

While this file only defines the blueprint, a real implementation might look like:

```rust
struct UserFeaturesQueryHydrator {
    user_service: UserService,
}

impl QueryHydrator<Query> for UserFeaturesQueryHydrator {
    async fn hydrate(&self, query: &Query) -> Result<Query, String> {
        // Fetch user features from the user service
        let features = self.user_service
            .get_features(query.user_id)
            .await?;

        // Create a new query with features added
        let mut hydrated = query.clone();
        hydrated.user_features = Some(features);
        Ok(hydrated)
    }

    fn update(&self, query: &mut Query, hydrated: Query) {
        // Only update the user_features field
        query.user_features = hydrated.user_features;
    }
}
```

Another example for user's recent activity:

```rust
struct UserActionSeqQueryHydrator {
    action_log: ActionLogService,
}

impl QueryHydrator<Query> for UserActionSeqQueryHydrator {
    async fn hydrate(&self, query: &Query) -> Result<Query, String> {
        // Get the user's last 100 actions
        let actions = self.action_log
            .get_recent_actions(query.user_id, 100)
            .await?;

        let mut hydrated = query.clone();
        hydrated.user_action_sequence = Some(actions);
        Ok(hydrated)
    }

    fn update(&self, query: &mut Query, hydrated: Query) {
        // Only update the action sequence field
        query.user_action_sequence = hydrated.user_action_sequence;
    }
}
```

## Difference from Regular Hydrators

| Query Hydrators | Regular Hydrators |
|-----------------|-------------------|
| Enrich the **user/request** | Enrich **posts** |
| Run **once** at the start | Run **once per post** |
| Only need `<Q>` type | Need `<Q, C>` types |
| Result affects entire pipeline | Result affects that post only |

## Key Takeaways

1. **Query hydrators gather user information** - They learn about who is making the request
2. **They run first** - Before any posts are fetched
3. **They run in parallel** - Multiple hydrators gather data simultaneously
4. **Each hydrator owns specific fields** - The `update` method only copies its own data
5. **The enriched query travels through the pipeline** - All other components can use this data
6. **Can be async** - Often need to call external services (user database, action logs)
