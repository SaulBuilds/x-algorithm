# phoenix_candidate_pipeline.rs - The Complete Pipeline Assembly

## File Location
`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`

## Purpose
This is where the entire recommendation pipeline gets assembled. It takes all the individual components (sources, hydrators, filters, scorers) and wires them together into a working system. Think of it as the recipe that combines all the ingredients.

## High-Level Structure

The file does three things:
1. Imports all the components
2. Defines the pipeline structure
3. Implements the CandidatePipeline trait

## Line-by-Line Explanation

### Imports (Lines 1-58)

The first 58 lines import all the components. Let me highlight the key categories:

#### Hydrators (Lines 1-6)
```rust
use crate::candidate_hydrators::core_data_candidate_hydrator::CoreDataCandidateHydrator;
use crate::candidate_hydrators::gizmoduck_hydrator::GizmoduckCandidateHydrator;
use crate::candidate_hydrators::in_network_candidate_hydrator::InNetworkCandidateHydrator;
use crate::candidate_hydrators::subscription_hydrator::SubscriptionHydrator;
use crate::candidate_hydrators::vf_candidate_hydrator::VFCandidateHydrator;
use crate::candidate_hydrators::video_duration_candidate_hydrator::VideoDurationCandidateHydrator;
```
These hydrators add data to posts:
- `CoreDataCandidateHydrator` = Basic post info (text, timestamps)
- `GizmoduckCandidateHydrator` = Author info (name, followers)
- `InNetworkCandidateHydrator` = Whether author is followed
- `SubscriptionHydrator` = Subscription status
- `VFCandidateHydrator` = Visibility filtering data
- `VideoDurationCandidateHydrator` = Video length

#### Clients (Lines 9-21)
```rust
use crate::clients::phoenix_prediction_client::...;
use crate::clients::phoenix_retrieval_client::...;
use crate::clients::thunder_client::ThunderClient;
use crate::clients::gizmoduck_client::...;
// etc.
```
These are connections to other services:
- `PhoenixPredictionClient` = Calls Phoenix for ML predictions
- `PhoenixRetrievalClient` = Calls Phoenix for candidate retrieval
- `ThunderClient` = Calls Thunder for post data
- `GizmoduckClient` = Calls Gizmoduck for user data
- `StratoClient` = Calls Strato for key-value storage
- `TESClient` = Calls Tweet Entity Service
- `SocialGraphClient` = Calls Social Graph for follow relationships

#### Filters (Lines 22-33)
```rust
use crate::filters::age_filter::AgeFilter;
use crate::filters::author_socialgraph_filter::AuthorSocialgraphFilter;
use crate::filters::drop_duplicates_filter::DropDuplicatesFilter;
use crate::filters::muted_keyword_filter::MutedKeywordFilter;
// etc.
```
Filters that remove posts:
- `AgeFilter` = Removes old posts
- `AuthorSocialgraphFilter` = Removes blocked/muted authors
- `DropDuplicatesFilter` = Removes duplicate posts
- `MutedKeywordFilter` = Removes posts with muted words
- And many more...

#### Scorers (Lines 37-40)
```rust
use crate::scorers::author_diversity_scorer::AuthorDiversityScorer;
use crate::scorers::oon_scorer::OONScorer;
use crate::scorers::phoenix_scorer::PhoenixScorer;
use crate::scorers::weighted_scorer::WeightedScorer;
```
Scorers that rank posts:
- `PhoenixScorer` = Gets ML predictions from Phoenix
- `WeightedScorer` = Combines predictions into one score
- `AuthorDiversityScorer` = Penalizes repeated authors
- `OONScorer` = Adjusts out-of-network scores

### The Pipeline Structure (Lines 60-70)

```rust
pub struct PhoenixCandidatePipeline {
    query_hydrators: Vec<Box<dyn QueryHydrator<ScoredPostsQuery>>>,
    sources: Vec<Box<dyn Source<ScoredPostsQuery, PostCandidate>>>,
    hydrators: Vec<Box<dyn Hydrator<ScoredPostsQuery, PostCandidate>>>,
    filters: Vec<Box<dyn Filter<ScoredPostsQuery, PostCandidate>>>,
    scorers: Vec<Box<dyn Scorer<ScoredPostsQuery, PostCandidate>>>,
    selector: TopKScoreSelector,
    post_selection_hydrators: Vec<Box<dyn Hydrator<ScoredPostsQuery, PostCandidate>>>,
    post_selection_filters: Vec<Box<dyn Filter<ScoredPostsQuery, PostCandidate>>>,
    side_effects: Arc<Vec<Box<dyn SideEffect<ScoredPostsQuery, PostCandidate>>>>,
}
```

This structure holds all the pipeline components:
- `query_hydrators` = List of query hydrators (gather user info)
- `sources` = List of sources (fetch posts)
- `hydrators` = List of hydrators (add data to posts)
- `filters` = List of filters (remove posts)
- `scorers` = List of scorers (rank posts)
- `selector` = The selector (pick top posts)
- `post_selection_hydrators` = Hydrators that run after selection
- `post_selection_filters` = Filters that run after selection
- `side_effects` = Background tasks

### Pipeline Assembly (Lines 72-160)

The `build_with_clients` method assembles everything:

#### Query Hydrators (Lines 83-89)
```rust
let query_hydrators: Vec<Box<dyn QueryHydrator<ScoredPostsQuery>>> = vec![
    Box::new(UserActionSeqQueryHydrator::new(uas_fetcher)),
    Box::new(UserFeaturesQueryHydrator {
        strato_client: strato_client.clone(),
    }),
];
```
Two query hydrators:
1. `UserActionSeqQueryHydrator` = Gets user's recent actions
2. `UserFeaturesQueryHydrator` = Gets ML features for the user

#### Sources (Lines 91-97)
```rust
let phoenix_source = Box::new(PhoenixSource {
    phoenix_retrieval_client,
});
let thunder_source = Box::new(ThunderSource { thunder_client });
let sources: Vec<Box<dyn Source<ScoredPostsQuery, PostCandidate>>> =
    vec![phoenix_source, thunder_source];
```
Two sources of posts:
1. `PhoenixSource` = ML-based retrieval (finds relevant posts)
2. `ThunderSource` = Database retrieval (recent posts from follows)

#### Hydrators (Lines 99-106)
```rust
let hydrators: Vec<Box<dyn Hydrator<ScoredPostsQuery, PostCandidate>>> = vec![
    Box::new(InNetworkCandidateHydrator),
    Box::new(CoreDataCandidateHydrator::new(tes_client.clone()).await),
    Box::new(VideoDurationCandidateHydrator::new(tes_client.clone()).await),
    Box::new(SubscriptionHydrator::new(tes_client.clone()).await),
    Box::new(GizmoduckCandidateHydrator::new(gizmoduck_client).await),
];
```
Five hydrators add data in parallel:
1. `InNetworkCandidateHydrator` = Mark if author is followed
2. `CoreDataCandidateHydrator` = Get basic post data
3. `VideoDurationCandidateHydrator` = Get video length
4. `SubscriptionHydrator` = Get subscription info
5. `GizmoduckCandidateHydrator` = Get author info

#### Filters (Lines 108-120)
```rust
let filters: Vec<Box<dyn Filter<ScoredPostsQuery, PostCandidate>>> = vec![
    Box::new(DropDuplicatesFilter),
    Box::new(CoreDataHydrationFilter),
    Box::new(AgeFilter::new(Duration::from_secs(params::MAX_POST_AGE))),
    Box::new(SelfTweetFilter),
    Box::new(RetweetDeduplicationFilter),
    Box::new(IneligibleSubscriptionFilter),
    Box::new(PreviouslySeenPostsFilter),
    Box::new(PreviouslyServedPostsFilter),
    Box::new(MutedKeywordFilter::new()),
    Box::new(AuthorSocialgraphFilter),
];
```
Ten filters run sequentially:
1. `DropDuplicatesFilter` = Remove duplicate post IDs
2. `CoreDataHydrationFilter` = Remove posts that failed hydration
3. `AgeFilter` = Remove old posts
4. `SelfTweetFilter` = Remove user's own posts
5. `RetweetDeduplicationFilter` = Remove duplicate retweets
6. `IneligibleSubscriptionFilter` = Remove subscription content without access
7. `PreviouslySeenPostsFilter` = Remove already-seen posts
8. `PreviouslyServedPostsFilter` = Remove already-served posts
9. `MutedKeywordFilter` = Remove posts with muted words
10. `AuthorSocialgraphFilter` = Remove blocked/muted authors

#### Scorers (Lines 122-132)
```rust
let phoenix_scorer = Box::new(PhoenixScorer { phoenix_client });
let weighted_scorer = Box::new(WeightedScorer);
let author_diversity_scorer = Box::new(AuthorDiversityScorer::default());
let oon_scorer = Box::new(OONScorer);
let scorers: Vec<Box<dyn Scorer<ScoredPostsQuery, PostCandidate>>> = vec![
    phoenix_scorer,
    weighted_scorer,
    author_diversity_scorer,
    oon_scorer,
];
```
Four scorers run sequentially:
1. `PhoenixScorer` = Get 19 engagement predictions
2. `WeightedScorer` = Combine into single weighted score
3. `AuthorDiversityScorer` = Penalize same author appearing repeatedly
4. `OONScorer` = Adjust scores for out-of-network posts

#### Post-Selection Components (Lines 137-147)
```rust
let post_selection_hydrators: Vec<Box<dyn Hydrator<ScoredPostsQuery, PostCandidate>>> =
    vec![Box::new(VFCandidateHydrator::new(vf_client.clone()).await)];

let post_selection_filters: Vec<Box<dyn Filter<ScoredPostsQuery, PostCandidate>>> =
    vec![Box::new(VFFilter), Box::new(DedupConversationFilter)];
```
After selection:
- `VFCandidateHydrator` = Check visibility filtering (expensive, so only on final posts)
- `VFFilter` = Remove posts that fail visibility filtering
- `DedupConversationFilter` = Remove duplicate conversation threads

### Production Configuration (Lines 162-212)

The `prod()` method creates a production-ready pipeline:
```rust
pub async fn prod() -> PhoenixCandidatePipeline {
    let uas_fetcher = Arc::new(UserActionSequenceFetcher::new()...);
    let phoenix_client = Arc::new(ProdPhoenixPredictionClient::new()...);
    let phoenix_retrieval_client = Arc::new(ProdPhoenixRetrievalClient::new()...);
    let thunder_client = Arc::new(ThunderClient::new()...);
    let strato_client = Arc::new(ProdStratoClient::new()...);
    let tes_client = Arc::new(ProdTESClient::new()...);
    let gizmoduck_client = Arc::new(ProdGizmoduckClient::new()...);
    let vf_client = Arc::new(ProdVisibilityFilteringClient::new()...);

    PhoenixCandidatePipeline::build_with_clients(
        uas_fetcher, phoenix_client, phoenix_retrieval_client,
        thunder_client, strato_client, tes_client,
        gizmoduck_client, vf_client,
    ).await
}
```

This initializes connections to all the production services.

### Trait Implementation (Lines 215-255)

The trait implementation just returns the stored components:
```rust
impl CandidatePipeline<ScoredPostsQuery, PostCandidate> for PhoenixCandidatePipeline {
    fn query_hydrators(&self) -> &[...] { &self.query_hydrators }
    fn sources(&self) -> &[...] { &self.sources }
    fn hydrators(&self) -> &[...] { &self.hydrators }
    fn filters(&self) -> &[...] { &self.filters }
    fn scorers(&self) -> &[...] { &self.scorers }
    fn selector(&self) -> &dyn Selector<...> { &self.selector }
    fn post_selection_hydrators(&self) -> &[...] { &self.post_selection_hydrators }
    fn post_selection_filters(&self) -> &[...] { &self.post_selection_filters }
    fn side_effects(&self) -> Arc<Vec<...>> { Arc::clone(&self.side_effects) }
    fn result_size(&self) -> usize { params::RESULT_SIZE }
}
```

## Complete Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PhoenixCandidatePipeline                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  QUERY HYDRATORS (parallel)                                        │
│  ├─ UserActionSeqQueryHydrator → Recent 100 actions                │
│  └─ UserFeaturesQueryHydrator → ML features                        │
│                                                                     │
│                              ↓                                      │
│                                                                     │
│  SOURCES (parallel)                                                │
│  ├─ PhoenixSource → ~800 ML-retrieved candidates                   │
│  └─ ThunderSource → ~200 chronological candidates                  │
│                                                                     │
│                              ↓                                      │
│                                                                     │
│  HYDRATORS (parallel)                                              │
│  ├─ InNetworkCandidateHydrator                                     │
│  ├─ CoreDataCandidateHydrator                                      │
│  ├─ VideoDurationCandidateHydrator                                 │
│  ├─ SubscriptionHydrator                                           │
│  └─ GizmoduckCandidateHydrator                                     │
│                                                                     │
│                              ↓                                      │
│                                                                     │
│  FILTERS (sequential)                                              │
│  1. DropDuplicatesFilter                                           │
│  2. CoreDataHydrationFilter                                        │
│  3. AgeFilter                                                      │
│  4. SelfTweetFilter                                                │
│  5. RetweetDeduplicationFilter                                     │
│  6. IneligibleSubscriptionFilter                                   │
│  7. PreviouslySeenPostsFilter                                      │
│  8. PreviouslyServedPostsFilter                                    │
│  9. MutedKeywordFilter                                             │
│  10. AuthorSocialgraphFilter                                       │
│                                                                     │
│                              ↓                                      │
│                                                                     │
│  SCORERS (sequential)                                              │
│  1. PhoenixScorer → 19 engagement predictions                      │
│  2. WeightedScorer → Combined score                                │
│  3. AuthorDiversityScorer → Diversity penalty                      │
│  4. OONScorer → Out-of-network adjustment                          │
│                                                                     │
│                              ↓                                      │
│                                                                     │
│  SELECTOR                                                          │
│  └─ TopKScoreSelector → Top 50 by score                           │
│                                                                     │
│                              ↓                                      │
│                                                                     │
│  POST-SELECTION HYDRATORS                                          │
│  └─ VFCandidateHydrator → Visibility filtering data               │
│                                                                     │
│                              ↓                                      │
│                                                                     │
│  POST-SELECTION FILTERS                                            │
│  ├─ VFFilter → Remove policy violations                           │
│  └─ DedupConversationFilter → Dedupe conversations                │
│                                                                     │
│                              ↓                                      │
│                                                                     │
│  SIDE EFFECTS (background)                                         │
│  └─ CacheRequestInfoSideEffect → Cache for future requests        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
                        YOUR FEED
                    (~50 scored posts)
```

## Key Takeaways

1. **Assembly point** - This file wires everything together
2. **Component lists** - Each category has its own list of implementations
3. **Order matters** - Filters and scorers run in defined order
4. **Parallel vs sequential** - Hydrators parallel, filters/scorers sequential
5. **Production clients** - `prod()` connects to real services
6. **Post-selection** - Some expensive operations only run on final posts
7. **Configurable** - Could easily swap out components
