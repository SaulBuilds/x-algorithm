# selector.rs - Picking the Best Posts

## File Location
`candidate-pipeline/selector.rs`

## Purpose
This file defines what a "Selector" is. After posts have been scored, the Selector sorts them by score and picks the top ones. If you have 1,000 scored posts but only want to show 50, the Selector picks the best 50.

## Line-by-Line Explanation

```rust
use crate::util;
```
**Line 1**: Import helper functions from this package.

```rust
use std::any::type_name_of_val;
```
**Line 2**: Import the function that gets a value's type name.
- Used for logging to identify the selector

```rust
pub trait Selector<Q, C>: Send + Sync
```
**Line 4**: Define a public trait called `Selector`.
- `pub trait` = Public blueprint
- `Selector` = The name
- `<Q, C>` = Generic types (Q = Query, C = Candidate)
- `: Send + Sync` = Must be thread-safe

Note: This trait is not `async` - selection is a synchronous operation (just sorting and cutting).

```rust
where
    Q: Clone + Send + Sync + 'static,
    C: Clone + Send + Sync + 'static,
```
**Lines 5-7**: Constraints on the generic types.
- Both Query and Candidate must be copyable and thread-safe

```rust
{
```
**Line 8**: Start of trait methods.

```rust
    /// Default selection: sort and truncate based on provided configs
    fn select(&self, _query: &Q, candidates: Vec<C>) -> Vec<C> {
        let mut sorted = self.sort(candidates);
        if let Some(limit) = self.size() {
            sorted.truncate(limit);
        }
        sorted
    }
```
**Lines 9-16**: The main `select` method.
- `fn select` = A synchronous function (no `async`)
- `_query: &Q` = Reference to query (underscore means unused in default impl)
- `candidates: Vec<C>` = Takes ownership of the candidate list
- `-> Vec<C>` = Returns the selected candidates

The default implementation:
1. `self.sort(candidates)` = Sort candidates by score
2. `self.size()` = Get the maximum number of posts to return
3. `sorted.truncate(limit)` = Cut the list to that size
4. `sorted` = Return the trimmed list

```rust
    /// Decide if this selector should run for the given query
    fn enable(&self, _query: &Q) -> bool {
        true
    }
```
**Lines 18-21**: The `enable` method.
- Decides if this selector should run
- Default: always enabled
- Can be overridden for conditional selection

```rust
    /// Extract the score from a candidate to use for sorting.
    fn score(&self, candidate: &C) -> f64;
```
**Lines 23-24**: The `score` method.
- `candidate: &C` = Reference to a candidate
- `-> f64` = Returns the score as a 64-bit floating point number

No default implementation - every Selector MUST define how to extract scores from candidates.

```rust
    /// Sort candidates by their scores in descending order.
    fn sort(&self, candidates: Vec<C>) -> Vec<C> {
        let mut sorted = candidates;
        sorted.sort_by(|a, b| {
            self.score(b)
                .partial_cmp(&self.score(a))
                .unwrap_or(std::cmp::Ordering::Equal)
        });
        sorted
    }
```
**Lines 26-35**: The `sort` method.
- `candidates: Vec<C>` = Takes ownership of candidates
- `-> Vec<C>` = Returns sorted candidates

The implementation:
1. `let mut sorted = candidates` = Make the list mutable
2. `sorted.sort_by(...)` = Sort using a comparison function
3. `self.score(b).partial_cmp(&self.score(a))` = Compare scores
   - Note: `b` comes before `a` - this gives descending order (highest first)
4. `.unwrap_or(std::cmp::Ordering::Equal)` = If scores can't be compared (e.g., NaN), treat as equal
5. `sorted` = Return the sorted list

```rust
    /// Optionally provide a size to select. Defaults to no truncation if not overridden.
    fn size(&self) -> Option<usize> {
        None
    }
```
**Lines 37-40**: The `size` method.
- `-> Option<usize>` = Returns either a size limit or nothing
- `None` = Default is no limit (return all candidates)

Implementations can override this to set a maximum number of results.

```rust
    fn name(&self) -> &'static str {
        util::short_type_name(type_name_of_val(self))
    }
```
**Lines 42-44**: The `name` method.
- Returns the selector's name for logging
- Default implementation extracts the type name

```rust
}
```
**Line 45**: End of trait definition.

## How Selection Works

```
Scored Candidates (unsorted)              After Selection
┌────────────────────────────┐           ┌─────────────────┐
│ Post A: score 0.72         │           │ 1. Post C: 0.92 │
│ Post B: score 0.45         │   ───→    │ 2. Post A: 0.72 │
│ Post C: score 0.92         │  (select  │ 3. Post D: 0.68 │
│ Post D: score 0.68         │   top 5)  │ 4. Post F: 0.61 │
│ Post E: score 0.33         │           │ 5. Post B: 0.45 │
│ Post F: score 0.61         │           └─────────────────┘
│ Post G: score 0.28         │
└────────────────────────────┘
       (7 posts)                              (5 posts)
```

## Why Selection is Separate from Scoring

You might wonder why we don't just sort during scoring. The separation provides:

1. **Flexibility**: Different selectors can use different scores from the same candidates
2. **Customization**: Selection logic might be more complex than just "top N by score"
3. **Clarity**: Each component does one thing well

## Example of a Real Selector

While this file only defines the blueprint, a real implementation might look like:

```rust
struct TopKScoreSelector {
    limit: usize,  // How many to select
}

impl Selector<Query, Candidate> for TopKScoreSelector {
    fn score(&self, candidate: &Candidate) -> f64 {
        // Use the final combined score
        candidate.final_score.unwrap_or(0.0)
    }

    fn size(&self) -> Option<usize> {
        Some(self.limit)
    }
}
```

A more complex selector with diversity logic:

```rust
struct DiverseTopKSelector {
    limit: usize,
    max_per_author: usize,  // Don't show more than N posts from same author
}

impl Selector<Query, Candidate> for DiverseTopKSelector {
    fn score(&self, candidate: &Candidate) -> f64 {
        candidate.final_score.unwrap_or(0.0)
    }

    fn select(&self, _query: &Query, candidates: Vec<Candidate>) -> Vec<Candidate> {
        let sorted = self.sort(candidates);

        // Custom selection: limit posts per author
        let mut selected = Vec::new();
        let mut author_counts: HashMap<i64, usize> = HashMap::new();

        for candidate in sorted {
            let count = author_counts.entry(candidate.author_id).or_insert(0);
            if *count < self.max_per_author {
                selected.push(candidate);
                *count += 1;
            }
            if selected.len() >= self.limit {
                break;
            }
        }

        selected
    }

    fn size(&self) -> Option<usize> {
        Some(self.limit)
    }
}
```

## The Sorting Algorithm

The sort uses Rust's built-in sorting, which is:
- **Stable**: Posts with equal scores stay in their original relative order
- **Efficient**: O(n log n) time complexity
- **Descending**: Higher scores come first (notice `b` compared before `a`)

## Handling Edge Cases

The code handles:
- **NaN scores**: Treated as equal to other NaN values
- **No size limit**: All candidates are returned (just sorted)
- **Empty input**: Returns an empty list

## Key Takeaways

1. **Selectors pick winners** - They choose which posts make it into your feed
2. **They sort by score** - Highest scores first (descending order)
3. **They can limit count** - The `size()` method controls how many to return
4. **Selection is synchronous** - No async needed, just sorting
5. **The `score()` method extracts values** - Implementations define which score to use
6. **Can be customized** - Override `select()` for complex selection logic (e.g., diversity)
7. **Default behavior is simple** - Sort then truncate
