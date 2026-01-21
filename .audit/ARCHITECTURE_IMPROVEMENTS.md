# Architecture Improvements

## 1. Pipeline Parallelization

### 1.1 Current State

```
Query Hydrators (parallel) → Sources (parallel) → Hydrators (parallel)
    → Filters (SEQUENTIAL) → Scorers (SEQUENTIAL) → Selection
```

### 1.2 Proposed: Parallel Filter Groups

**Current** (`candidate-pipeline/candidate_pipeline.rs:219-273`): All 12 filters run sequentially.

**Proposed**: Group independent filters for parallel execution.

```
Filter Group A (parallel):          Filter Group B (sequential):
├─ DropDuplicatesFilter            ├─ CoreDataHydrationFilter
├─ AgeFilter                       ├─ RepostDeduplicationFilter
├─ SelfTweetFilter                 └─ VFFilter (depends on hydration)
└─ PreviouslySeenPostsFilter

Filter Group C (parallel):
├─ MutedKeywordFilter
├─ AuthorSocialgraphFilter
└─ IneligibleSubscriptionFilter
```

**Implementation**:
```rust
async fn filter_parallel_groups(&self, candidates: Vec<C>) -> Vec<C> {
    // Group A: No dependencies
    let group_a = join_all([
        self.drop_duplicates.filter(&candidates),
        self.age_filter.filter(&candidates),
        self.self_tweet_filter.filter(&candidates),
    ]).await;

    let kept_a = intersect_filter_results(group_a);

    // Group B: Depends on hydration
    let kept_b = self.hydration_filter.filter(&kept_a).await;

    // Group C: Independent of B
    let group_c = join_all([
        self.muted_keyword.filter(&kept_b),
        self.author_social.filter(&kept_b),
    ]).await;

    intersect_filter_results(group_c)
}
```

**Expected Impact**: 30-40% reduction in filter stage latency.

---

## 2. Event Tracking Architecture

### 2.1 Current Gaps

- No per-stage latency metrics
- No per-filter exclusion reason tracking
- Side effects fire-and-forget without completion tracking
- No distributed tracing support

### 2.2 Proposed: Structured Pipeline Events

```rust
#[derive(Debug, Serialize)]
pub struct PipelineEvent {
    request_id: String,
    user_id: u64,
    stage: PipelineStage,
    event_type: EventType,
    timestamp_ms: u64,
    metadata: HashMap<String, Value>,
}

pub enum PipelineStage {
    QueryHydration,
    CandidateSourcing,
    Hydration,
    Filtering,
    Scoring,
    Selection,
    PostSelection,
    SideEffects,
}

pub enum EventType {
    StageStart,
    StageComplete,
    CandidateExcluded { reason: String, filter_name: String },
    ScoreComputed { candidate_id: u64, score: f64 },
    Error { message: String },
}
```

### 2.3 Proposed: Event Sink Architecture

```
Pipeline Stages → Event Buffer → Event Sink
                                    ↓
                    ┌───────────────┼───────────────┐
                    ↓               ↓               ↓
              Kafka Topic     Prometheus      Distributed
             (analytics)       (metrics)      Tracing (Jaeger)
```

**Implementation**:
```rust
pub trait EventSink: Send + Sync {
    fn emit(&self, event: PipelineEvent);
    fn flush(&self) -> impl Future<Output = Result<(), Error>>;
}

pub struct CompositeEventSink {
    sinks: Vec<Box<dyn EventSink>>,
}

impl EventSink for CompositeEventSink {
    fn emit(&self, event: PipelineEvent) {
        for sink in &self.sinks {
            sink.emit(event.clone());
        }
    }
}
```

---

## 3. Scoring Architecture Redesign

### 3.1 Current: Single-Signal Ranking

```python
# Only uses favorite_score
primary_scores = probs[:, :, 0]
ranked_indices = jnp.argsort(-primary_scores, axis=-1)
```

### 3.2 Proposed: Multi-Objective Scoring Framework

```python
class MultiObjectiveScorer:
    def __init__(self, config: ScoringConfig):
        self.positive_weights = config.positive_weights
        self.negative_weights = config.negative_weights
        self.diversity_penalty = config.diversity_penalty
        self.recency_decay = config.recency_decay

    def score(self, probs: jnp.ndarray, metadata: CandidateMetadata) -> jnp.ndarray:
        # Base engagement score
        engagement = sum(
            self.positive_weights[action] * probs[:, :, idx]
            for action, idx in POSITIVE_ACTIONS.items()
        )

        # Negative engagement penalty
        negative = sum(
            self.negative_weights[action] * probs[:, :, idx]
            for action, idx in NEGATIVE_ACTIONS.items()
        )

        # Author diversity penalty
        author_counts = count_by_author(metadata.author_ids)
        diversity_penalty = self.diversity_penalty * (author_counts - 1)

        # Recency decay
        age_hours = (now() - metadata.created_at) / 3600
        recency_factor = jnp.exp(-self.recency_decay * age_hours)

        return (engagement + negative - diversity_penalty) * recency_factor
```

### 3.3 Proposed: Frequency-Based Spam Penalty

```python
class FrequencyPenaltyScorer:
    def __init__(self, config: FrequencyConfig):
        self.hourly_threshold = config.hourly_post_threshold  # e.g., 20
        self.decay_rate = config.decay_rate

    def compute_penalty(self, author_stats: AuthorStats) -> float:
        posts_last_hour = author_stats.posts_in_window(hours=1)

        if posts_last_hour <= self.hourly_threshold:
            return 0.0

        # Exponential penalty beyond threshold
        excess = posts_last_hour - self.hourly_threshold
        return 1.0 - jnp.exp(-self.decay_rate * excess)

    def apply(self, scores: jnp.ndarray, author_stats: List[AuthorStats]) -> jnp.ndarray:
        penalties = jnp.array([self.compute_penalty(s) for s in author_stats])
        return scores * (1.0 - penalties)
```

---

## 4. Quality Tower Architecture

### 4.1 Current: Flat Embedding Concatenation

```python
embeddings = jnp.concatenate([
    user_embeddings,      # [B, 1, D]
    history_embeddings,   # [B, S, D]
    candidate_embeddings, # [B, C, D]
], axis=1)
```

### 4.2 Proposed: Separate Quality Assessment Tower

```
                    ┌─────────────────────────────┐
                    │     User Tower              │
                    │  (engagement history)       │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ↓              ↓              ↓
            ┌───────────┐  ┌───────────┐  ┌───────────┐
            │ Relevance │  │  Quality  │  │   Bot     │
            │   Tower   │  │   Tower   │  │ Detection │
            └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
                  │              │              │
                  └──────────────┼──────────────┘
                                 ↓
                    ┌─────────────────────────────┐
                    │     Fusion Layer            │
                    │  (learned weights)          │
                    └──────────────┬──────────────┘
                                   ↓
                            Final Score
```

**Implementation**:
```python
class QualityTower(hk.Module):
    def __call__(self, content_features: ContentFeatures) -> jnp.ndarray:
        # Quantitative signals
        quant = jnp.stack([
            content_features.toxicity_score,
            content_features.has_media.astype(jnp.float32),
            content_features.url_quality_tier / 3.0,  # Normalize
            jnp.log1p(content_features.engagement_count) / 10.0,
        ], axis=-1)

        quant_embedding = hk.Linear(self.hidden_dim)(quant)

        # Qualitative embedding (learned from content)
        qual_embedding = self.content_encoder(content_features.text_embedding)

        # Fusion
        combined = jnp.concatenate([quant_embedding, qual_embedding], axis=-1)
        quality_score = jax.nn.sigmoid(hk.Linear(1)(combined))

        return quality_score
```

---

## 5. Bot Detection Architecture

### 5.1 Proposed: Behavioral Anomaly Detection Layer

```python
class BotDetectionLayer(hk.Module):
    def __call__(self, user_features: UserFeatures, history: HistoryFeatures) -> jnp.ndarray:
        # Temporal regularity
        inter_action_times = history.timestamps[1:] - history.timestamps[:-1]
        time_std = jnp.std(inter_action_times)
        regularity_score = jax.nn.sigmoid(1.0 - time_std / 60.0)  # Bots have low std

        # Action diversity
        action_counts = jnp.sum(history.actions, axis=0)
        action_entropy = -jnp.sum(
            action_counts * jnp.log(action_counts + 1e-8) / jnp.sum(action_counts)
        )
        diversity_score = action_entropy / jnp.log(len(ACTIONS))  # Normalize

        # Author concentration
        unique_authors = jnp.unique(history.author_ids).shape[0]
        author_diversity = unique_authors / history.author_ids.shape[0]

        # Account age
        account_age_days = (now() - user_features.created_at) / 86400
        age_score = jax.nn.sigmoid(account_age_days / 30 - 1)  # Penalize < 30 days

        # Combine signals
        bot_probability = hk.Linear(1)(jnp.stack([
            regularity_score,
            1.0 - diversity_score,  # Low diversity = more bot-like
            1.0 - author_diversity,
            1.0 - age_score,
        ], axis=-1))

        return jax.nn.sigmoid(bot_probability)
```

### 5.2 Integration with Scoring

```python
def final_score(engagement_score, quality_score, bot_probability):
    # Trust-weighted score
    trust_factor = 1.0 - bot_probability
    return engagement_score * quality_score * trust_factor
```

---

## 6. Caching Architecture

### 6.1 Current: No Request-Scoped Caching

Each hydrator fetches independently, potentially duplicate calls.

### 6.2 Proposed: Hierarchical Cache

```
Request-Scoped Cache (in-memory, per-request)
    ↓ miss
Session Cache (Redis, 5-minute TTL)
    ↓ miss
User Cache (Redis, 1-hour TTL)
    ↓ miss
Service Call (Gizmoduck, etc.)
```

**Implementation**:
```rust
pub struct HierarchicalCache<K, V> {
    request_cache: DashMap<K, V>,
    session_cache: Arc<RedisCache<K, V>>,
    user_cache: Arc<RedisCache<K, V>>,
}

impl<K, V> HierarchicalCache<K, V> {
    pub async fn get_or_fetch<F, Fut>(&self, key: K, fetch: F) -> Result<V, Error>
    where
        F: FnOnce() -> Fut,
        Fut: Future<Output = Result<V, Error>>,
    {
        // Check request cache
        if let Some(v) = self.request_cache.get(&key) {
            return Ok(v.clone());
        }

        // Check session cache
        if let Some(v) = self.session_cache.get(&key).await? {
            self.request_cache.insert(key.clone(), v.clone());
            return Ok(v);
        }

        // Fetch and populate all levels
        let v = fetch().await?;
        self.request_cache.insert(key.clone(), v.clone());
        self.session_cache.set(&key, &v).await?;

        Ok(v)
    }
}
```

---

## 7. Summary of Architectural Changes

| Change | Impact | Effort | Priority |
|--------|--------|--------|----------|
| Parallel filter groups | 30-40% filter latency reduction | Medium | P1 |
| Structured event tracking | Full observability | Medium | P1 |
| Multi-objective scoring | Better engagement | Low | P0 |
| Quality tower | Content quality signals | High | P1 |
| Bot detection layer | Spam reduction | Medium | P1 |
| Frequency penalty | Slop mitigation | Low | P0 |
| Hierarchical caching | Reduced service calls | Medium | P2 |
