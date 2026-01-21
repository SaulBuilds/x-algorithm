# candidate_pipeline.rs - The Main Orchestrator

## File Location
`candidate-pipeline/candidate_pipeline.rs`

## Purpose
This is the heart of the recommendation system. It defines the `CandidatePipeline` trait - the complete blueprint for how posts flow from retrieval to your feed. All the components we've discussed (sources, hydrators, filters, scorers, selectors, side effects) come together here.

## Line-by-Line Explanation

### Imports (Lines 1-11)

```rust
use crate::filter::Filter;
```
**Line 1**: Import the Filter trait from this package.

```rust
use crate::hydrator::Hydrator;
```
**Line 2**: Import the Hydrator trait.

```rust
use crate::query_hydrator::QueryHydrator;
```
**Line 3**: Import the QueryHydrator trait.

```rust
use crate::scorer::Scorer;
```
**Line 4**: Import the Scorer trait.

```rust
use crate::selector::Selector;
```
**Line 5**: Import the Selector trait.

```rust
use crate::side_effect::{SideEffect, SideEffectInput};
```
**Line 6**: Import the SideEffect trait and its input structure.

```rust
use crate::source::Source;
```
**Line 7**: Import the Source trait.

```rust
use futures::future::join_all;
```
**Line 8**: Import `join_all` from the futures library.
- `join_all` = Run multiple async tasks in parallel and wait for all to finish
- Essential for running multiple sources/hydrators simultaneously

```rust
use log::{error, info, warn};
```
**Line 9**: Import logging functions.
- `error` = Log critical errors
- `info` = Log normal information
- `warn` = Log warnings (something unexpected but recoverable)

```rust
use std::sync::Arc;
```
**Line 10**: Import Arc for shared ownership.
- Used for sharing data between threads (especially for side effects)

```rust
use tonic::async_trait;
```
**Line 11**: Import async_trait for async methods in traits.

### Pipeline Stage Enum (Lines 13-22)

```rust
#[derive(Copy, Clone, Debug)]
pub enum PipelineStage {
    QueryHydrator,
    Source,
    Hydrator,
    PostSelectionHydrator,
    Filter,
    PostSelectionFilter,
    Scorer,
}
```
**Lines 13-22**: Define an enumeration of pipeline stages.
- `#[derive(Copy, Clone, Debug)]` = Automatically implement copy, clone, and debug printing
- `pub enum PipelineStage` = A public enum listing all stages
- Each variant represents a stage in the pipeline

Used for logging to identify which stage an error occurred in.

### Pipeline Result Structure (Lines 24-29)

```rust
pub struct PipelineResult<Q, C> {
    pub retrieved_candidates: Vec<C>,
    pub filtered_candidates: Vec<C>,
    pub selected_candidates: Vec<C>,
    pub query: Arc<Q>,
}
```
**Lines 24-29**: Define what the pipeline returns.
- `retrieved_candidates` = All posts that were initially fetched
- `filtered_candidates` = Posts that were removed by filters
- `selected_candidates` = The final posts to show (THE FEED)
- `query` = The hydrated query (with user info)

This structure provides transparency - you can see what was considered, what was removed, and what was selected.

### Request ID Trait (Lines 31-34)

```rust
/// Provides a stable request identifier for logging/tracing.
pub trait HasRequestId {
    fn request_id(&self) -> &str;
}
```
**Lines 31-34**: Define a trait for getting a request ID.
- Any query type must be able to provide a unique request ID
- Used for tracing and debugging - every log line includes this ID

### The Main CandidatePipeline Trait (Lines 36-51)

```rust
#[async_trait]
pub trait CandidatePipeline<Q, C>: Send + Sync
where
    Q: HasRequestId + Clone + Send + Sync + 'static,
    C: Clone + Send + Sync + 'static,
{
```
**Lines 36-41**: Start of the CandidatePipeline trait.
- `Q: HasRequestId + ...` = Query must have a request ID and be thread-safe
- `C: Clone + Send + Sync + 'static` = Candidate must be copyable and thread-safe

```rust
    fn query_hydrators(&self) -> &[Box<dyn QueryHydrator<Q>>];
```
**Line 42**: Return the list of query hydrators.
- `&[Box<dyn ...>]` = A slice of boxed (heap-allocated) dynamic trait objects
- This allows different implementations of QueryHydrator in the same list

```rust
    fn sources(&self) -> &[Box<dyn Source<Q, C>>];
```
**Line 43**: Return the list of sources.

```rust
    fn hydrators(&self) -> &[Box<dyn Hydrator<Q, C>>];
```
**Line 44**: Return the list of candidate hydrators.

```rust
    fn filters(&self) -> &[Box<dyn Filter<Q, C>>];
```
**Line 45**: Return the list of filters.

```rust
    fn scorers(&self) -> &[Box<dyn Scorer<Q, C>>];
```
**Line 46**: Return the list of scorers.

```rust
    fn selector(&self) -> &dyn Selector<Q, C>;
```
**Line 47**: Return the selector (just one, not a list).

```rust
    fn post_selection_hydrators(&self) -> &[Box<dyn Hydrator<Q, C>>];
```
**Line 48**: Return hydrators that run after selection.

```rust
    fn post_selection_filters(&self) -> &[Box<dyn Filter<Q, C>>];
```
**Line 49**: Return filters that run after selection.

```rust
    fn side_effects(&self) -> Arc<Vec<Box<dyn SideEffect<Q, C>>>>;
```
**Line 50**: Return side effects (wrapped in Arc for thread sharing).

```rust
    fn result_size(&self) -> usize;
```
**Line 51**: Return the maximum number of results to return.

### The Execute Method (Lines 53-92)

This is the core method that runs the entire pipeline:

```rust
    async fn execute(&self, query: Q) -> PipelineResult<Q, C> {
```
**Line 53**: Start of the execute method.
- Takes a query, returns a PipelineResult
- This is what gets called when a user opens their feed

```rust
        let hydrated_query = self.hydrate_query(query).await;
```
**Line 54**: **Step 1** - Hydrate the query (gather user info).

```rust
        let candidates = self.fetch_candidates(&hydrated_query).await;
```
**Line 56**: **Step 2** - Fetch candidate posts from all sources.

```rust
        let hydrated_candidates = self.hydrate(&hydrated_query, candidates).await;
```
**Line 58**: **Step 3** - Add details to all candidates.

```rust
        let (kept_candidates, mut filtered_candidates) = self
            .filter(&hydrated_query, hydrated_candidates.clone())
            .await;
```
**Lines 60-62**: **Step 4** - Filter out unwanted posts.
- Returns two lists: kept and removed

```rust
        let scored_candidates = self.score(&hydrated_query, kept_candidates).await;
```
**Line 64**: **Step 5** - Score the remaining candidates.

```rust
        let selected_candidates = self.select(&hydrated_query, scored_candidates);
```
**Line 66**: **Step 6** - Select the top candidates.

```rust
        let post_selection_hydrated_candidates = self
            .hydrate_post_selection(&hydrated_query, selected_candidates)
            .await;
```
**Lines 68-70**: **Step 7** - Additional hydration on selected posts only.

```rust
        let (mut final_candidates, post_selection_filtered_candidates) = self
            .filter_post_selection(&hydrated_query, post_selection_hydrated_candidates)
            .await;
        filtered_candidates.extend(post_selection_filtered_candidates);
```
**Lines 72-75**: **Step 8** - Final filtering on selected posts.

```rust
        final_candidates.truncate(self.result_size());
```
**Line 77**: **Step 9** - Ensure we don't return more than result_size.

```rust
        let arc_hydrated_query = Arc::new(hydrated_query);
        let input = Arc::new(SideEffectInput {
            query: arc_hydrated_query.clone(),
            selected_candidates: final_candidates.clone(),
        });
        self.run_side_effects(input);
```
**Lines 79-84**: **Step 10** - Kick off side effects in the background.

```rust
        PipelineResult {
            retrieved_candidates: hydrated_candidates,
            filtered_candidates,
            selected_candidates: final_candidates,
            query: arc_hydrated_query,
        }
    }
```
**Lines 86-92**: Return the complete result.

### Query Hydration Method (Lines 94-123)

```rust
    /// Run all query hydrators in parallel and merge results into the query.
    async fn hydrate_query(&self, query: Q) -> Q {
```
**Lines 94-95**: Start of query hydration.

```rust
        let request_id = query.request_id().to_string();
```
**Line 96**: Get the request ID for logging.

```rust
        let hydrators: Vec<_> = self
            .query_hydrators()
            .iter()
            .filter(|h| h.enable(&query))
            .collect();
```
**Lines 97-101**: Get only the enabled hydrators.
- `filter(|h| h.enable(&query))` = Keep only hydrators that want to run

```rust
        let hydrate_futures = hydrators.iter().map(|h| h.hydrate(&query));
        let results = join_all(hydrate_futures).await;
```
**Lines 102-103**: Run all hydrators in parallel.
- `map(|h| h.hydrate(&query))` = Create a future for each hydrator
- `join_all(...).await` = Run all futures simultaneously and wait for all

```rust
        let mut hydrated_query = query;
        for (hydrator, result) in hydrators.iter().zip(results) {
            match result {
                Ok(hydrated) => {
                    hydrator.update(&mut hydrated_query, hydrated);
                }
                Err(err) => {
                    error!(
                        "request_id={} stage={:?} component={} failed: {}",
                        request_id,
                        PipelineStage::QueryHydrator,
                        hydrator.name(),
                        err
                    );
                }
            }
        }
        hydrated_query
    }
```
**Lines 105-123**: Merge results from all hydrators.
- Loop through each hydrator's result
- If successful, update the query with new data
- If failed, log the error but continue (graceful degradation)

### Fetch Candidates Method (Lines 125-157)

```rust
    /// Run all candidate sources in parallel and collect results.
    async fn fetch_candidates(&self, query: &Q) -> Vec<C> {
```
**Lines 125-126**: Start of candidate fetching.

The logic is similar to query hydration:
1. Get enabled sources
2. Run all in parallel
3. Collect all results into one list
4. Log any errors

```rust
        collected.append(&mut candidates);
```
**Line 143**: Merge candidates from each source into one combined list.

### Hydration Methods (Lines 159-217)

```rust
    async fn hydrate(&self, query: &Q, candidates: Vec<C>) -> Vec<C> {
        self.run_hydrators(query, candidates, self.hydrators(), PipelineStage::Hydrator)
            .await
    }
```
**Lines 159-163**: Call the shared helper for regular hydration.

```rust
    async fn hydrate_post_selection(&self, query: &Q, candidates: Vec<C>) -> Vec<C> {
        self.run_hydrators(
            query,
            candidates,
            self.post_selection_hydrators(),
            PipelineStage::PostSelectionHydrator,
        )
        .await
    }
```
**Lines 165-174**: Call the shared helper for post-selection hydration.

```rust
    async fn run_hydrators(
        &self,
        query: &Q,
        mut candidates: Vec<C>,
        hydrators: &[Box<dyn Hydrator<Q, C>>],
        stage: PipelineStage,
    ) -> Vec<C> {
```
**Lines 176-183**: The shared hydrator runner.

Key logic:
- Filter to enabled hydrators
- Run all in parallel
- Merge results back
- Validate that counts match (hydrators must not drop candidates)

```rust
                    if hydrated.len() == expected_len {
                        hydrator.update_all(&mut candidates, hydrated);
                    } else {
                        warn!(
                            "request_id={} stage={:?} component={} skipped: length_mismatch expected={} got={}",
```
**Lines 192-202**: Safety check - if a hydrator returns the wrong number of candidates, skip it and log a warning.

### Filter Methods (Lines 219-273)

```rust
    async fn filter(&self, query: &Q, candidates: Vec<C>) -> (Vec<C>, Vec<C>) {
```
**Line 220**: Run filters, return (kept, removed).

```rust
    async fn run_filters(
        &self,
        query: &Q,
        mut candidates: Vec<C>,
        filters: &[Box<dyn Filter<Q, C>>],
        stage: PipelineStage,
    ) -> (Vec<C>, Vec<C>) {
```
**Lines 237-243**: The shared filter runner.

Key difference from hydrators: **filters run sequentially**, not in parallel.

```rust
        for filter in filters.iter().filter(|f| f.enable(query)) {
            let backup = candidates.clone();
            match filter.filter(query, candidates).await {
                Ok(result) => {
                    candidates = result.kept;
                    all_removed.extend(result.removed);
                }
                Err(err) => {
                    ...
                    candidates = backup;
                }
            }
        }
```
**Lines 246-264**: Run each filter one at a time.
- If a filter succeeds, update candidates to only the kept ones
- If a filter fails, restore from backup and continue

### Score Method (Lines 275-307)

```rust
    async fn score(&self, query: &Q, mut candidates: Vec<C>) -> Vec<C> {
```
**Line 276**: Score candidates.

Scorers run **sequentially** (like filters), because later scorers might use earlier scores.

### Select Method (Lines 309-316)

```rust
    fn select(&self, query: &Q, candidates: Vec<C>) -> Vec<C> {
        if self.selector().enable(query) {
            self.selector().select(query, candidates)
        } else {
            candidates
        }
    }
```
**Lines 310-316**: Run the selector if enabled.
- Note: `fn` not `async fn` - selection is synchronous

### Side Effects Method (Lines 318-328)

```rust
    fn run_side_effects(&self, input: Arc<SideEffectInput<Q, C>>) {
        let side_effects = self.side_effects();
        tokio::spawn(async move {
            let futures = side_effects
                .iter()
                .filter(|se| se.enable(input.query.clone()))
                .map(|se| se.run(input.clone()));
            let _ = join_all(futures).await;
        });
    }
```
**Lines 319-328**: Run side effects in the background.
- `tokio::spawn` = Start a new async task that runs independently
- `async move` = The closure takes ownership of its captured variables
- `let _ = join_all(...).await` = Run all side effects in parallel, ignore results

## The Complete Pipeline Flow

```
USER REQUEST
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│ 1. QUERY HYDRATION (parallel)                                   │
│    UserFeaturesHydrator ─┐                                     │
│    ActionSeqHydrator ────┼──→ Hydrated Query                   │
│    BlockedUsersHydrator ─┘                                     │
└────────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│ 2. FETCH CANDIDATES (parallel)                                  │
│    ThunderSource ─┐                                            │
│    PhoenixSource ─┼──→ ~1000+ Candidates                       │
│    OtherSource ───┘                                            │
└────────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│ 3. HYDRATE CANDIDATES (parallel)                                │
│    AuthorHydrator ─┐                                           │
│    MediaHydrator ──┼──→ Enriched Candidates                    │
│    StatsHydrator ──┘                                           │
└────────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│ 4. FILTER (sequential)                                          │
│    BlockedAuthorFilter ──→ MutedKeywordFilter ──→              │
│    ──→ PreviouslySeenFilter ──→ ... ──→ ~500 Candidates       │
└────────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│ 5. SCORE (sequential)                                           │
│    PhoenixScorer ──→ DiversityScorer ──→                       │
│    ──→ WeightedScorer ──→ Scored Candidates                    │
└────────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│ 6. SELECT                                                       │
│    Sort by score, take top 50 ──→ 50 Selected Candidates       │
└────────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│ 7. POST-SELECTION HYDRATION (parallel)                          │
│    Add expensive details only for final 50                     │
└────────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│ 8. POST-SELECTION FILTER (sequential)                           │
│    Final safety checks ──→ Final Candidates                    │
└────────────────────────────────────────────────────────────────┘
     │
     ├──────────────────────────────────────────────┐
     ▼                                              ▼
YOUR FEED                               SIDE EFFECTS (background)
(~50 posts)                             Logging, Caching, Analytics
```

## Key Takeaways

1. **This is the orchestrator** - It coordinates all other components
2. **Stages run in specific order** - Query → Fetch → Hydrate → Filter → Score → Select
3. **Parallel when possible** - Hydrators and sources run in parallel for speed
4. **Sequential when needed** - Filters and scorers run sequentially for correctness
5. **Graceful degradation** - Errors in one component don't crash the pipeline
6. **Comprehensive logging** - Every stage logs its request_id for debugging
7. **Side effects are non-blocking** - User gets their feed without waiting for logging
8. **Post-selection stages** - Additional processing only on the final candidates (efficient)
