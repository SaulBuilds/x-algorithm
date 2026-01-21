# Sprint 1: Multi-Action Scoring

**Priority**: P1 - High Impact
**Status**: PLANNED
**Dependencies**: Sprint 0 (Critical Fixes)

## Overview

Enable the 18 unused engagement signals in ranking to dramatically improve feed quality. Currently only `favorite_score` is used despite the model predicting 19 different engagement types.

---

## Objective

Transform ranking from single-signal to multi-objective scoring:

```
BEFORE: score = P(favorite)
AFTER:  score = Σ(weight_i × P(action_i)) × quality_factor × trust_factor
```

---

## Implementation Tasks

### Task 1: Weighted Engagement Scoring

**File**: `phoenix/runners.py`

**Changes**:

```python
# Add near ACTIONS definition (line ~202)
SCORING_WEIGHTS = {
    # High-effort positive (encourage quality engagement)
    "quote_score": 3.0,
    "reply_score": 2.5,
    "follow_author_score": 2.5,

    # Medium-effort positive
    "repost_score": 2.0,
    "share_score": 1.8,
    "share_via_dm_score": 1.5,

    # Low-effort positive
    "favorite_score": 1.0,
    "click_score": 0.8,
    "photo_expand_score": 0.7,
    "profile_click_score": 0.6,

    # Time-based
    "dwell_score": 0.5,
    "vqv_score": 1.2,

    # Negative (penalties)
    "not_interested_score": -3.0,
    "mute_author_score": -5.0,
    "block_author_score": -8.0,
    "report_score": -15.0,
}


def compute_weighted_engagement_score(probs: jnp.ndarray) -> jnp.ndarray:
    """
    Compute weighted engagement score from all action probabilities.

    Args:
        probs: [B, C, num_actions] engagement probabilities

    Returns:
        scores: [B, C] weighted scores
    """
    scores = jnp.zeros(probs.shape[:2])

    for action, weight in SCORING_WEIGHTS.items():
        if action in ACTIONS:
            idx = ACTIONS.index(action)
            scores = scores + weight * probs[:, :, idx]

    return scores
```

**Modify `hk_rank_candidates` (line ~336)**:

```python
def hk_rank_candidates(batch, recsys_embeddings) -> RankingOutput:
    output = hk_forward(batch, recsys_embeddings)
    logits = output.logits
    probs = jax.nn.sigmoid(logits)

    # NEW: Use weighted multi-action scoring
    primary_scores = compute_weighted_engagement_score(probs)

    # Continue with ranking...
    ranked_indices = jnp.argsort(-primary_scores, axis=-1)
    # ...
```

---

### Task 2: Add Rust-Side Weighted Scorer

**File**: `home-mixer/scorers/weighted_scorer.rs`

Create new scorer that combines Phoenix predictions with configured weights:

```rust
use std::collections::HashMap;

pub struct WeightedScorer {
    weights: HashMap<String, f64>,
}

impl WeightedScorer {
    pub fn new() -> Self {
        let mut weights = HashMap::new();

        // Positive engagement
        weights.insert("favorite_score".into(), 1.0);
        weights.insert("reply_score".into(), 2.5);
        weights.insert("repost_score".into(), 2.0);
        weights.insert("quote_score".into(), 3.0);
        weights.insert("follow_author_score".into(), 2.5);
        weights.insert("dwell_score".into(), 0.5);
        weights.insert("vqv_score".into(), 1.2);
        weights.insert("share_score".into(), 1.8);

        // Negative engagement (penalties)
        weights.insert("not_interested_score".into(), -3.0);
        weights.insert("mute_author_score".into(), -5.0);
        weights.insert("block_author_score".into(), -8.0);
        weights.insert("report_score".into(), -15.0);

        Self { weights }
    }

    pub fn score(&self, action_probs: &HashMap<String, f64>) -> f64 {
        let mut score = 0.0;

        for (action, weight) in &self.weights {
            if let Some(prob) = action_probs.get(action) {
                score += weight * prob;
            }
        }

        // Normalize to prevent extreme scores
        score.max(-10.0).min(10.0)
    }
}

#[async_trait]
impl Scorer<PhoenixQuery, PhoenixCandidate> for WeightedScorer {
    async fn score(
        &self,
        _query: &PhoenixQuery,
        candidates: &mut [PhoenixCandidate],
    ) -> Result<(), String> {
        for candidate in candidates.iter_mut() {
            let action_probs = &candidate.action_probabilities;
            candidate.weighted_score = self.score(action_probs);
        }
        Ok(())
    }
}
```

---

### Task 3: Author Posting Velocity Penalty

**File**: `home-mixer/scorers/velocity_penalty_scorer.rs` (NEW)

```rust
pub struct VelocityPenaltyScorer {
    hourly_soft_limit: u32,
    hourly_hard_limit: u32,
    daily_soft_limit: u32,
    daily_hard_limit: u32,
}

impl VelocityPenaltyScorer {
    pub fn new() -> Self {
        Self {
            hourly_soft_limit: 5,
            hourly_hard_limit: 20,
            daily_soft_limit: 30,
            daily_hard_limit: 100,
        }
    }

    fn compute_penalty(&self, posts_last_hour: u32, posts_last_day: u32) -> f64 {
        // Hourly penalty
        let hourly_penalty = if posts_last_hour <= self.hourly_soft_limit {
            0.0
        } else if posts_last_hour >= self.hourly_hard_limit {
            0.8
        } else {
            let progress = (posts_last_hour - self.hourly_soft_limit) as f64
                / (self.hourly_hard_limit - self.hourly_soft_limit) as f64;
            0.8 * (1.0 - (-3.0 * progress).exp())
        };

        // Daily penalty
        let daily_penalty = if posts_last_day <= self.daily_soft_limit {
            0.0
        } else if posts_last_day >= self.daily_hard_limit {
            0.5
        } else {
            let progress = (posts_last_day - self.daily_soft_limit) as f64
                / (self.daily_hard_limit - self.daily_soft_limit) as f64;
            0.5 * progress
        };

        hourly_penalty.max(daily_penalty)
    }
}

#[async_trait]
impl Scorer<PhoenixQuery, PhoenixCandidate> for VelocityPenaltyScorer {
    async fn score(
        &self,
        _query: &PhoenixQuery,
        candidates: &mut [PhoenixCandidate],
    ) -> Result<(), String> {
        for candidate in candidates.iter_mut() {
            let penalty = self.compute_penalty(
                candidate.author_posts_last_hour,
                candidate.author_posts_last_day,
            );
            candidate.velocity_penalty = penalty;
            candidate.score *= 1.0 - penalty;
        }
        Ok(())
    }
}
```

---

### Task 4: Add Author Stats to Candidate Hydration

**File**: `home-mixer/candidate_hydrators/author_stats_hydrator.rs` (NEW)

```rust
pub struct AuthorStatsHydrator {
    author_stats_client: Arc<AuthorStatsClient>,
}

#[async_trait]
impl Hydrator<PhoenixQuery, PhoenixCandidate> for AuthorStatsHydrator {
    async fn hydrate(
        &self,
        _query: &PhoenixQuery,
        candidates: &mut [PhoenixCandidate],
    ) -> Result<(), String> {
        let author_ids: Vec<u64> = candidates.iter().map(|c| c.author_id).collect();

        let stats = self.author_stats_client.batch_get(&author_ids).await?;

        for (candidate, stat) in candidates.iter_mut().zip(stats) {
            candidate.author_posts_last_hour = stat.posts_last_hour;
            candidate.author_posts_last_day = stat.posts_last_day;
            candidate.author_account_age_days = stat.account_age_days;
        }

        Ok(())
    }
}
```

---

### Task 5: Update Pipeline Registration

**File**: `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`

Add new scorers to the pipeline:

```rust
impl PhoenixCandidatePipeline {
    pub fn new(/* ... */) -> Self {
        // Existing scorers...
        let phoenix_scorer = Arc::new(PhoenixScorer::new(phoenix_client));

        // NEW: Add weighted and velocity scorers
        let weighted_scorer = Arc::new(WeightedScorer::new());
        let velocity_penalty_scorer = Arc::new(VelocityPenaltyScorer::new());

        Self {
            // ...
            scorers: vec![
                Box::new(phoenix_scorer),
                Box::new(weighted_scorer),
                Box::new(velocity_penalty_scorer),
                Box::new(author_diversity_scorer),
            ],
            // ...
        }
    }
}
```

---

## Metrics to Track

| Metric | Description | Target |
|--------|-------------|--------|
| `reply_rate` | Replies per impression | +15% |
| `quote_rate` | Quotes per impression | +20% |
| `not_interested_rate` | "Not interested" clicks | -25% |
| `mute_rate` | Author mutes from feed | -30% |
| `dwell_time_avg` | Average time on post | +10% |
| `high_frequency_author_exposure` | Posts from >20/hour authors | -50% |

---

## A/B Test Plan

### Control Group (50%)
- Current single-signal scoring (favorite_score only)

### Treatment Group (50%)
- Multi-action weighted scoring
- Velocity penalty enabled

### Success Criteria
- +10% in quality engagement (replies + quotes)
- -20% in negative engagement (not_interested + mute)
- No regression in total engagement
- No increase in p99 latency

---

## Rollout Plan

1. **Week 1**: Deploy to internal dogfood (1%)
2. **Week 2**: Expand to 5% with A/B test
3. **Week 3**: Analyze results, tune weights
4. **Week 4**: Expand to 25%
5. **Week 5**: Full rollout if metrics positive
