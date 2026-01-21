# scorer.rs - Giving Posts a Score

## File Location
`candidate-pipeline/scorer.rs`

## Purpose
This file defines what a "Scorer" is. Scorers calculate how interesting or relevant each post is to the user. The score determines the post's position in your feed - higher scored posts appear first.

## Line-by-Line Explanation

```rust
use crate::util;
```
**Line 1**: Import helper functions from this package.

```rust
use std::any::type_name_of_val;
```
**Line 2**: Import the function that gets a value's type name.
- Used for logging to identify which scorer is running

```rust
use tonic::async_trait;
```
**Line 3**: Import the async_trait helper.
- Enables defining traits with async functions

```rust
/// Scorers update candidate fields (like a score field) and run sequentially
```
**Line 5**: Documentation comment explaining scorers.
- "update candidate fields" = Add or modify a score value on posts
- "run sequentially" = One scorer runs, then the next

```rust
#[async_trait]
```
**Line 6**: Enable async functions in the trait below.

```rust
pub trait Scorer<Q, C>: Send + Sync
```
**Line 7**: Define a public trait called `Scorer`.
- `pub trait` = Public blueprint
- `Scorer` = The name
- `<Q, C>` = Generic types (Q = Query, C = Candidate)
- `: Send + Sync` = Must be thread-safe

Note: Unlike other traits, this doesn't require `Any`. It just needs thread safety.

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
    /// Decide if this scorer should run for the given query
    fn enable(&self, _query: &Q) -> bool {
        true
    }
```
**Lines 12-15**: The `enable` method.
- Decides if this scorer should run
- Default: always enabled (returns `true`)
- Can be overridden for conditional scoring

```rust
    /// Score candidates by performing async operations.
    /// Returns candidates with this scorer's fields populated.
    ///
    /// IMPORTANT: The returned vector must have the same candidates in the same order as the input.
    /// Dropping candidates in a scorer is not allowed - use a filter stage instead.
    async fn score(&self, query: &Q, candidates: &[C]) -> Result<Vec<C>, String>;
```
**Lines 17-22**: The main `score` method.
- `async fn score` = Async function that can wait (e.g., for ML model calls)
- `query: &Q` = Reference to user/request info
- `candidates: &[C]` = Reference to a slice of candidates
- `-> Result<Vec<C>, String>` = Returns:
  - Success: Candidates with scores added
  - Failure: An error message

**IMPORTANT RULE**: Same as hydrators:
- The returned list must have the same number of items
- Items must be in the same order
- Scorers cannot remove candidates (use filters for that)

```rust
    /// Update a single candidate with the scored fields.
    /// Only the fields this scorer is responsible for should be copied.
    fn update(&self, candidate: &mut C, scored: C);
```
**Lines 24-26**: The `update` method.
- `candidate: &mut C` = A mutable reference to the original candidate
- `scored: C` = The scored version with new data
- Copies score fields from scored to candidate

No default implementation - every Scorer MUST define this.

```rust
    /// Update all candidates with the scored fields from `scored`.
    /// Default implementation iterates and calls `update` for each pair.
    fn update_all(&self, candidates: &mut [C], scored: Vec<C>) {
        for (c, s) in candidates.iter_mut().zip(scored) {
            self.update(c, s);
        }
    }
```
**Lines 28-34**: The `update_all` method.
- `candidates: &mut [C]` = Mutable reference to all original candidates
- `scored: Vec<C>` = All scored candidates
- Default implementation:
  - `candidates.iter_mut()` = Iterate over candidates with mutation allowed
  - `.zip(scored)` = Pair each original with its scored version
  - `self.update(c, s)` = Call update on each pair

```rust
    fn name(&self) -> &'static str {
        util::short_type_name(type_name_of_val(self))
    }
```
**Lines 36-38**: The `name` method.
- Returns the scorer's name for logging
- Default implementation extracts the type name

```rust
}
```
**Line 39**: End of trait definition.

## Why Scorers Run Sequentially

Scorers run one at a time because each might depend on previous scores:

```
                Candidates: [A, B, C]
                        ↓
┌───────────────────────────────────────────────┐
│ Scorer 1: Phoenix ML Score                     │
│ A.ml_score = 0.85                              │
│ B.ml_score = 0.72                              │
│ C.ml_score = 0.91                              │
└───────────────────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────┐
│ Scorer 2: Author Diversity Score               │
│ (May use ml_score as input)                    │
│ A.diversity_score = 0.9                        │
│ B.diversity_score = 0.7                        │
│ C.diversity_score = 0.95                       │
└───────────────────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────┐
│ Scorer 3: Weighted Combined Score              │
│ A.final_score = 0.85 * 0.8 + 0.9 * 0.2 = 0.86 │
│ B.final_score = 0.72 * 0.8 + 0.7 * 0.2 = 0.72 │
│ C.final_score = 0.91 * 0.8 + 0.95 * 0.2 = 0.92│
└───────────────────────────────────────────────┘
                        ↓
          Final Ranking: C, A, B
```

## How Scoring Works

```
┌────────────────────────────────────────────────────────────┐
│                     USER REQUEST                            │
│  User: @john_doe                                           │
│  Recent activity: liked tech posts, muted politics         │
└────────────────────────────────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────────┐
│                    SCORING ENGINE                           │
│                                                             │
│  For each post, calculate probabilities:                   │
│                                                             │
│  Post A: "New JavaScript framework released"               │
│  ├─ P(like) = 0.72        (72% chance user likes)          │
│  ├─ P(reply) = 0.23       (23% chance user replies)        │
│  ├─ P(repost) = 0.45      (45% chance user reposts)        │
│  ├─ P(dwell) = 0.89       (89% chance user reads it)       │
│  └─ Final Score = weighted_combination = 0.67              │
│                                                             │
│  Post B: "Local election results"                          │
│  ├─ P(like) = 0.15                                         │
│  ├─ P(reply) = 0.05                                        │
│  ├─ P(repost) = 0.02                                       │
│  ├─ P(dwell) = 0.30                                        │
│  └─ Final Score = 0.18                                     │
└────────────────────────────────────────────────────────────┘
                           ↓
              Post A appears before Post B
```

## Example of a Real Scorer

While this file only defines the blueprint, a real implementation might look like:

```rust
struct PhoenixScorer {
    phoenix_client: PhoenixClient,  // Connection to Phoenix ML service
}

impl Scorer<Query, Candidate> for PhoenixScorer {
    async fn score(&self, query: &Query, candidates: &[Candidate]) -> Result<Vec<Candidate>, String> {
        // Send all candidates to Phoenix for ML scoring
        let scores = self.phoenix_client.predict(
            &query.user_features,
            &query.user_history,
            candidates
        ).await?;

        // Create new candidates with scores attached
        Ok(candidates.iter().zip(scores).map(|(c, score)| {
            let mut new = c.clone();
            new.phoenix_score = Some(score);
            new
        }).collect())
    }

    fn update(&self, candidate: &mut Candidate, scored: Candidate) {
        // Only update the phoenix_score field
        candidate.phoenix_score = scored.phoenix_score;
    }
}
```

## Multiple Score Types

A candidate might have multiple scores from different scorers:

```rust
struct Candidate {
    post_id: i64,

    // Different scorers add different scores
    phoenix_score: Option<f64>,      // From PhoenixScorer
    diversity_score: Option<f64>,     // From AuthorDiversityScorer
    recency_score: Option<f64>,       // From RecencyScorer
    final_score: Option<f64>,         // From WeightedScorer (combines others)
}
```

## Key Takeaways

1. **Scorers calculate relevance** - They determine how interesting each post is
2. **They run sequentially** - Later scorers can use earlier scores
3. **They must preserve order and count** - Input and output must match exactly
4. **Each scorer owns specific fields** - The `update` method only copies its own score fields
5. **Scores drive ranking** - The final score determines position in your feed
6. **Can be async** - Scoring often requires calling ML models over the network
7. **Multiple scores combine** - Posts can have many scores that get combined into a final score
