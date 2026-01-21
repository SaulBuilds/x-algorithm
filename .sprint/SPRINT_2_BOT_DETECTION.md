# Sprint 2: Bot Detection & Spam Mitigation

**Priority**: P1 - High Impact
**Status**: PLANNED
**Dependencies**: Sprint 0 (Critical Fixes)

## Overview

Implement behavioral analysis to detect bot accounts and reduce spam/slop in feeds. Currently the system has zero bot detection capabilities.

---

## Objectives

1. Add temporal data to model inputs
2. Implement behavioral anomaly detection
3. Create engagement pattern analysis
4. Integrate bot probability into scoring

---

## Implementation Tasks

### Task 1: Add Temporal Data to RecsysBatch

**File**: `phoenix/recsys_model.py`

**Changes to RecsysBatch**:

```python
class RecsysBatch(NamedTuple):
    # Existing fields...
    user_hashes: jax.typing.ArrayLike
    history_post_hashes: jax.typing.ArrayLike
    history_author_hashes: jax.typing.ArrayLike
    history_actions: jax.typing.ArrayLike
    history_product_surface: jax.typing.ArrayLike
    candidate_post_hashes: jax.typing.ArrayLike
    candidate_author_hashes: jax.typing.ArrayLike
    candidate_product_surface: jax.typing.ArrayLike

    # NEW: Temporal fields
    history_timestamps: jax.typing.ArrayLike  # [B, S] Unix timestamps
    user_created_at: jax.typing.ArrayLike     # [B] Account creation timestamp
    candidate_created_at: jax.typing.ArrayLike # [B, C] Post creation timestamps

    # NEW: Author behavior fields
    author_posts_last_hour: jax.typing.ArrayLike   # [B, C]
    author_posts_last_day: jax.typing.ArrayLike    # [B, C]
    author_account_age_days: jax.typing.ArrayLike  # [B, C]
    author_follower_count: jax.typing.ArrayLike    # [B, C]
    author_following_count: jax.typing.ArrayLike   # [B, C]
```

---

### Task 2: Temporal Anomaly Detector

**File**: `phoenix/bot_detection.py` (NEW)

```python
import jax.numpy as jnp
from typing import Dict


class TemporalAnomalyDetector:
    """Detect bot-like temporal patterns in engagement history."""

    def __init__(
        self,
        metronomic_threshold: float = 2.0,  # seconds std
        burst_window: int = 60,              # seconds
        burst_threshold: int = 10,           # actions in window
    ):
        self.metronomic_threshold = metronomic_threshold
        self.burst_window = burst_window
        self.burst_threshold = burst_threshold

    def analyze(
        self,
        timestamps: jnp.ndarray,  # [B, S]
        padding_mask: jnp.ndarray,  # [B, S] True where valid
    ) -> Dict[str, jnp.ndarray]:
        """
        Analyze engagement timestamps for bot-like patterns.

        Returns dict of scores in [0, 1] where 1 = most bot-like.
        """
        B, S = timestamps.shape

        # Compute inter-action times
        deltas = timestamps[:, 1:] - timestamps[:, :-1]
        delta_mask = padding_mask[:, 1:] & padding_mask[:, :-1]

        # Metronomic detection (very regular intervals)
        mean_delta = jnp.sum(deltas * delta_mask, axis=1) / jnp.sum(delta_mask, axis=1).clip(1)
        var_delta = jnp.sum(((deltas - mean_delta[:, None]) ** 2) * delta_mask, axis=1) / jnp.sum(delta_mask, axis=1).clip(1)
        std_delta = jnp.sqrt(var_delta)

        # Coefficient of variation (CV) - low CV = metronomic = bot-like
        cv = std_delta / mean_delta.clip(1e-6)
        metronomic_score = jax.nn.sigmoid(1.0 - cv * 5)  # Scale factor tunable

        # Burst detection (many actions in short window)
        def count_bursts(ts, mask):
            count = 0
            for i in range(len(ts) - self.burst_threshold):
                window = ts[i:i + self.burst_threshold]
                window_mask = mask[i:i + self.burst_threshold]
                if jnp.all(window_mask) and (window[-1] - window[0] < self.burst_window):
                    count += 1
            return count

        # Vectorized burst counting (simplified)
        window_spans = timestamps[:, self.burst_threshold - 1:] - timestamps[:, :-self.burst_threshold + 1]
        burst_indicators = (window_spans < self.burst_window).astype(jnp.float32)
        burst_score = jnp.mean(burst_indicators, axis=1)

        # Sleep cycle analysis (humans have inactive periods)
        hours = ((timestamps % 86400) // 3600).astype(jnp.int32)
        unique_hours_per_batch = jnp.array([
            len(jnp.unique(hours[b][padding_mask[b]])) for b in range(B)
        ])
        always_on_score = unique_hours_per_batch / 24.0

        # Combined bot probability
        bot_probability = (
            metronomic_score * 0.4 +
            burst_score * 0.3 +
            always_on_score * 0.3
        )

        return {
            "bot_probability": bot_probability,
            "metronomic_score": metronomic_score,
            "burst_score": burst_score,
            "always_on_score": always_on_score,
        }
```

---

### Task 3: Engagement Pattern Analyzer

**File**: `phoenix/bot_detection.py` (continued)

```python
class EngagementPatternAnalyzer:
    """Analyze engagement patterns for authenticity signals."""

    def analyze(
        self,
        history_actions: jnp.ndarray,  # [B, S, num_actions]
        history_author_hashes: jnp.ndarray,  # [B, S, num_hashes]
        padding_mask: jnp.ndarray,  # [B, S]
    ) -> Dict[str, jnp.ndarray]:
        """
        Analyze engagement patterns.

        Returns dict of signals where higher = more human-like.
        """
        B, S, num_actions = history_actions.shape

        # Mask invalid positions
        masked_actions = history_actions * padding_mask[:, :, None]

        # Action distribution entropy
        action_sums = jnp.sum(masked_actions, axis=1)  # [B, num_actions]
        action_total = jnp.sum(action_sums, axis=1, keepdims=True).clip(1)
        action_probs = action_sums / action_total

        # Shannon entropy (higher = more varied = more human)
        entropy = -jnp.sum(
            action_probs * jnp.log(action_probs + 1e-8),
            axis=1
        )
        max_entropy = jnp.log(num_actions)
        normalized_entropy = entropy / max_entropy

        # Single-action dominance (bots often only retweet)
        max_action_ratio = jnp.max(action_probs, axis=1)

        # Author diversity (bots engage with same few accounts)
        def author_diversity_batch(author_hashes, mask):
            diversities = []
            for b in range(author_hashes.shape[0]):
                valid_authors = author_hashes[b][mask[b]]
                if len(valid_authors) == 0:
                    diversities.append(1.0)
                else:
                    unique = jnp.unique(valid_authors[:, 0])  # First hash
                    diversities.append(len(unique) / len(valid_authors))
            return jnp.array(diversities)

        author_diversity = author_diversity_batch(history_author_hashes, padding_mask)

        # Negative engagement presence (humans sometimes mute/block)
        negative_indices = [14, 15, 16, 17]  # not_interested through report
        negative_sum = jnp.sum(action_sums[:, negative_indices], axis=1)
        negative_ratio = negative_sum / action_total.squeeze()

        # Dwell presence (bots don't read)
        dwell_ratio = action_sums[:, 10] / action_total.squeeze()  # dwell_score

        # Authenticity score (higher = more human)
        authenticity_score = (
            normalized_entropy * 0.25 +
            (1 - max_action_ratio) * 0.25 +
            author_diversity * 0.2 +
            negative_ratio.clip(0, 0.1) * 1.0 +  # Some negativity is human
            dwell_ratio * 0.2
        )

        return {
            "authenticity_score": authenticity_score,
            "entropy": normalized_entropy,
            "max_action_ratio": max_action_ratio,
            "author_diversity": author_diversity,
            "negative_ratio": negative_ratio,
            "dwell_ratio": dwell_ratio,
        }
```

---

### Task 4: Bot Detection Scorer (Rust)

**File**: `home-mixer/scorers/bot_detection_scorer.rs` (NEW)

```rust
use async_trait::async_trait;

pub struct BotDetectionScorer {
    temporal_detector: TemporalAnomalyDetector,
    pattern_analyzer: EngagementPatternAnalyzer,
}

impl BotDetectionScorer {
    pub fn new() -> Self {
        Self {
            temporal_detector: TemporalAnomalyDetector::new(2.0, 60, 10),
            pattern_analyzer: EngagementPatternAnalyzer::new(),
        }
    }

    fn compute_trust_factor(&self, signals: &BotSignals) -> f64 {
        // Combine signals into trust factor
        let temporal_trust = 1.0 - signals.temporal_bot_probability;
        let pattern_trust = signals.authenticity_score;
        let account_age_trust = (signals.account_age_days as f64 / 365.0).min(1.0);

        // Weighted combination
        let trust = (
            temporal_trust * 0.4 +
            pattern_trust * 0.4 +
            account_age_trust * 0.2
        );

        // Ensure minimum trust floor (don't completely suppress)
        trust.max(0.1)
    }
}

#[async_trait]
impl Scorer<PhoenixQuery, PhoenixCandidate> for BotDetectionScorer {
    async fn score(
        &self,
        query: &PhoenixQuery,
        candidates: &mut [PhoenixCandidate],
    ) -> Result<(), String> {
        // Analyze viewer's engagement patterns
        let viewer_signals = self.analyze_viewer(query)?;

        for candidate in candidates.iter_mut() {
            // Analyze author's posting patterns
            let author_signals = BotSignals {
                temporal_bot_probability: candidate.author_temporal_bot_score,
                authenticity_score: candidate.author_authenticity_score,
                account_age_days: candidate.author_account_age_days,
            };

            let author_trust = self.compute_trust_factor(&author_signals);

            // Apply trust factor to score
            candidate.bot_trust_factor = author_trust;
            candidate.score *= author_trust;
        }

        Ok(())
    }
}
```

---

### Task 5: Account Age Penalty

**File**: `home-mixer/scorers/account_age_scorer.rs` (NEW)

```rust
pub struct AccountAgeScorer {
    new_account_threshold_days: u32,
    penalty_factor: f64,
}

impl AccountAgeScorer {
    pub fn new() -> Self {
        Self {
            new_account_threshold_days: 30,
            penalty_factor: 0.5,  // 50% penalty for very new accounts
        }
    }

    fn compute_penalty(&self, account_age_days: u32) -> f64 {
        if account_age_days >= self.new_account_threshold_days {
            0.0
        } else {
            // Linear ramp from full penalty to no penalty
            let progress = account_age_days as f64 / self.new_account_threshold_days as f64;
            self.penalty_factor * (1.0 - progress)
        }
    }
}

#[async_trait]
impl Scorer<PhoenixQuery, PhoenixCandidate> for AccountAgeScorer {
    async fn score(
        &self,
        _query: &PhoenixQuery,
        candidates: &mut [PhoenixCandidate],
    ) -> Result<(), String> {
        for candidate in candidates.iter_mut() {
            let penalty = self.compute_penalty(candidate.author_account_age_days);
            candidate.account_age_penalty = penalty;
            candidate.score *= 1.0 - penalty;
        }
        Ok(())
    }
}
```

---

### Task 6: Update Query Hydration for Temporal Data

**File**: `home-mixer/query_hydrators/user_temporal_hydrator.rs` (NEW)

```rust
pub struct UserTemporalHydrator {
    engagement_service: Arc<EngagementServiceClient>,
}

#[async_trait]
impl QueryHydrator<PhoenixQuery> for UserTemporalHydrator {
    async fn hydrate(&self, query: &mut PhoenixQuery) -> Result<(), String> {
        // Fetch user's engagement history with timestamps
        let history = self.engagement_service
            .get_user_engagement_history(
                query.viewer_id,
                128,  // history length
                true, // include timestamps
            )
            .await?;

        query.history_timestamps = history.timestamps;
        query.user_created_at = history.user_created_at;

        // Pre-compute temporal signals
        let temporal_signals = TemporalAnomalyDetector::analyze(&history.timestamps);
        query.viewer_temporal_signals = temporal_signals;

        Ok(())
    }
}
```

---

## Integration Architecture

```
User Request
    ↓
┌─────────────────────────────────────────────────────────┐
│ Query Hydration                                         │
│  ├─ UserTemporalHydrator (NEW)                         │
│  │   └─ Fetches engagement timestamps                  │
│  └─ UserFeaturesHydrator                               │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ Candidate Hydration                                     │
│  ├─ AuthorStatsHydrator (NEW)                          │
│  │   └─ Fetches author posting velocity, age           │
│  └─ CoreDataHydrator                                   │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ Scoring                                                 │
│  ├─ PhoenixScorer                                      │
│  ├─ WeightedScorer (Sprint 1)                          │
│  ├─ VelocityPenaltyScorer (Sprint 1)                   │
│  ├─ BotDetectionScorer (NEW)                           │
│  │   └─ Temporal + Pattern analysis                    │
│  ├─ AccountAgeScorer (NEW)                             │
│  └─ AuthorDiversityScorer                              │
└─────────────────────────────────────────────────────────┘
    ↓
Final Score = base_score × velocity_factor × bot_trust × age_factor
```

---

## Metrics to Track

| Metric | Description | Target |
|--------|-------------|--------|
| `bot_detected_rate` | % of candidates flagged as bot | Baseline |
| `new_account_penalty_rate` | % penalized for account age | Baseline |
| `spam_report_rate` | User spam reports | -40% |
| `author_concentration` | Posts from top 1% authors | -30% |
| `engagement_authenticity` | Replies vs likes ratio | +20% |

---

## Rollout Plan

1. **Week 1**: Deploy detectors in shadow mode (log only)
2. **Week 2**: Analyze detection rates, tune thresholds
3. **Week 3**: Enable with low penalty (10% max)
4. **Week 4**: Increase penalty based on effectiveness
5. **Week 5**: Full rollout with A/B test
