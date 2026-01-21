# Scoring & Bot Mitigation Strategies

## 1. Current Scoring Analysis

### 1.1 How Scoring Works Today

**File**: `phoenix/runners.py:336-371`

```python
def hk_rank_candidates(batch, recsys_embeddings) -> RankingOutput:
    output = hk_forward(batch, recsys_embeddings)
    logits = output.logits  # [B, num_candidates, 19 actions]
    probs = jax.nn.sigmoid(logits)
    primary_scores = probs[:, :, 0]  # ONLY favorite_score used
    ranked_indices = jnp.argsort(-primary_scores, axis=-1)
```

**Problem**: The model predicts 19 engagement signals but only uses `favorite_score` for ranking.

### 1.2 Available But Unused Signals

| Index | Signal | Type | Usage |
|-------|--------|------|-------|
| 0 | favorite_score | Positive | USED |
| 1 | reply_score | Positive | NOT USED |
| 2 | repost_score | Positive | NOT USED |
| 3 | photo_expand_score | Positive | NOT USED |
| 4 | click_score | Positive | NOT USED |
| 5 | profile_click_score | Positive | NOT USED |
| 6 | vqv_score | Positive (Video) | NOT USED |
| 7 | share_score | Positive | NOT USED |
| 8 | share_via_dm_score | Positive | NOT USED |
| 9 | share_via_copy_link_score | Positive | NOT USED |
| 10 | dwell_score | Positive | NOT USED |
| 11 | quote_score | Positive | NOT USED |
| 12 | quoted_click_score | Positive | NOT USED |
| 13 | follow_author_score | Positive | NOT USED |
| 14 | not_interested_score | **Negative** | NOT USED |
| 15 | block_author_score | **Negative** | NOT USED |
| 16 | mute_author_score | **Negative** | NOT USED |
| 17 | report_score | **Negative** | NOT USED |
| 18 | dwell_time | Continuous | NOT USED |

---

## 2. Proposed Multi-Action Scoring

### 2.1 Weighted Engagement Score

```python
# Proposed weights (tunable)
SCORING_WEIGHTS = {
    # High-effort positive engagement
    "quote_score": 3.0,           # Requires thought
    "reply_score": 2.5,           # Requires effort
    "follow_author_score": 2.5,   # Strong intent signal

    # Medium-effort positive engagement
    "repost_score": 2.0,
    "share_score": 1.8,
    "share_via_dm_score": 1.5,    # Personal recommendation

    # Low-effort positive engagement
    "favorite_score": 1.0,        # Baseline
    "click_score": 0.8,
    "photo_expand_score": 0.7,
    "profile_click_score": 0.6,

    # Time-based engagement
    "dwell_score": 0.5,           # Actually read it
    "vqv_score": 1.2,             # Video completion

    # Negative signals (penalties)
    "not_interested_score": -3.0,
    "mute_author_score": -5.0,
    "block_author_score": -8.0,
    "report_score": -15.0,        # Strongest negative
}

def compute_weighted_score(probs: jnp.ndarray) -> jnp.ndarray:
    """
    Args:
        probs: [B, C, 19] engagement probabilities
    Returns:
        scores: [B, C] weighted engagement scores
    """
    score = jnp.zeros(probs.shape[:2])

    for action, weight in SCORING_WEIGHTS.items():
        idx = ACTIONS.index(action)
        score += weight * probs[:, :, idx]

    return score
```

### 2.2 Engagement Quality Ratio

```python
def engagement_quality_ratio(probs: jnp.ndarray) -> jnp.ndarray:
    """
    Ratio of high-effort to low-effort engagement.
    High ratio = quality content that drives conversation.
    """
    high_effort = (
        probs[:, :, ACTIONS.index("quote_score")] +
        probs[:, :, ACTIONS.index("reply_score")] +
        probs[:, :, ACTIONS.index("follow_author_score")]
    )

    low_effort = (
        probs[:, :, ACTIONS.index("favorite_score")] +
        probs[:, :, ACTIONS.index("repost_score")] +
        probs[:, :, ACTIONS.index("click_score")]
    )

    return high_effort / (low_effort + 1e-6)
```

---

## 3. Frequency-Based Slop Mitigation

### 3.1 Author Posting Velocity Penalty

**Problem**: High-frequency posters (brands, bots, engagement farmers) flood feeds.

```python
class PostingVelocityPenalty:
    def __init__(self, config: VelocityConfig):
        self.hourly_soft_limit = config.hourly_soft_limit  # e.g., 5
        self.hourly_hard_limit = config.hourly_hard_limit  # e.g., 20
        self.daily_soft_limit = config.daily_soft_limit    # e.g., 30
        self.daily_hard_limit = config.daily_hard_limit    # e.g., 100

    def compute(self, author_stats: AuthorStats) -> float:
        """
        Returns penalty in [0, 1] where 1 = maximum penalty.
        """
        hourly_posts = author_stats.posts_last_hour
        daily_posts = author_stats.posts_last_day

        # Hourly penalty (sharper curve)
        if hourly_posts <= self.hourly_soft_limit:
            hourly_penalty = 0.0
        elif hourly_posts >= self.hourly_hard_limit:
            hourly_penalty = 0.8
        else:
            progress = (hourly_posts - self.hourly_soft_limit) / (
                self.hourly_hard_limit - self.hourly_soft_limit
            )
            hourly_penalty = 0.8 * (1 - jnp.exp(-3 * progress))

        # Daily penalty (gentler curve)
        if daily_posts <= self.daily_soft_limit:
            daily_penalty = 0.0
        elif daily_posts >= self.daily_hard_limit:
            daily_penalty = 0.5
        else:
            progress = (daily_posts - self.daily_soft_limit) / (
                self.daily_hard_limit - self.daily_soft_limit
            )
            daily_penalty = 0.5 * progress

        return max(hourly_penalty, daily_penalty)

    def apply(self, scores: jnp.ndarray, author_stats_list: List[AuthorStats]) -> jnp.ndarray:
        penalties = jnp.array([self.compute(s) for s in author_stats_list])
        return scores * (1.0 - penalties[:, None])
```

### 3.2 Content Recycling Detection

```python
class ContentRecyclingDetector:
    def __init__(self, similarity_threshold: float = 0.9):
        self.similarity_threshold = similarity_threshold

    def detect_recycled(
        self,
        candidate_hashes: jnp.ndarray,  # [C, num_hashes]
        recent_post_hashes: jnp.ndarray  # [N, num_hashes] from last 24h
    ) -> jnp.ndarray:
        """
        Returns recycling score in [0, 1] for each candidate.
        """
        # Jaccard similarity between candidate and recent posts
        similarities = []
        for c_hash in candidate_hashes:
            max_sim = 0.0
            for r_hash in recent_post_hashes:
                intersection = jnp.sum(c_hash == r_hash)
                union = len(c_hash)  # Assuming fixed hash length
                sim = intersection / union
                max_sim = max(max_sim, sim)
            similarities.append(max_sim)

        return jnp.array(similarities)

    def apply_penalty(self, scores: jnp.ndarray, recycling_scores: jnp.ndarray) -> jnp.ndarray:
        # Penalize highly similar content
        penalty = jnp.where(
            recycling_scores > self.similarity_threshold,
            0.5,  # 50% penalty for recycled content
            0.0
        )
        return scores * (1.0 - penalty)
```

---

## 4. Bot Detection Strategies

### 4.1 Behavioral Signals

| Signal | Bot Indicator | Human Indicator |
|--------|---------------|-----------------|
| Inter-action time std | < 1 second | > 10 seconds |
| Action entropy | < 1.0 (one action dominates) | > 2.0 (varied) |
| Author diversity | < 5% unique authors | > 30% unique |
| Product surface variety | Single surface | Multiple surfaces |
| Account age | < 7 days | > 30 days |
| Follower/following ratio | Extreme (< 0.01 or > 100) | 0.5 - 5.0 |

### 4.2 Temporal Anomaly Detection

```python
class TemporalAnomalyDetector:
    def __init__(self):
        self.metronomic_threshold = 2.0  # seconds std
        self.burst_threshold = 10  # actions in 60 seconds

    def analyze(self, timestamps: jnp.ndarray) -> Dict[str, float]:
        """
        Analyze engagement timestamps for bot-like patterns.
        """
        if len(timestamps) < 2:
            return {"bot_probability": 0.0}

        # Inter-action times
        deltas = timestamps[1:] - timestamps[:-1]
        mean_delta = jnp.mean(deltas)
        std_delta = jnp.std(deltas)

        # Metronomic detection (very regular intervals)
        cv = std_delta / (mean_delta + 1e-6)  # Coefficient of variation
        metronomic_score = jax.nn.sigmoid(1.0 - cv * 10)  # Low CV = bot-like

        # Burst detection (many actions in short window)
        burst_windows = []
        for i in range(len(timestamps) - self.burst_threshold):
            window = timestamps[i:i + self.burst_threshold]
            if window[-1] - window[0] < 60:  # 60 second window
                burst_windows.append(1)
        burst_score = len(burst_windows) / max(1, len(timestamps) - self.burst_threshold)

        # No sleep cycle (24h activity)
        hours = (timestamps % 86400) // 3600
        unique_hours = len(jnp.unique(hours))
        always_on_score = unique_hours / 24.0  # Humans have gaps

        bot_probability = (metronomic_score + burst_score + always_on_score) / 3

        return {
            "bot_probability": float(bot_probability),
            "metronomic_score": float(metronomic_score),
            "burst_score": float(burst_score),
            "always_on_score": float(always_on_score),
        }
```

### 4.3 Engagement Pattern Analysis

```python
class EngagementPatternAnalyzer:
    def analyze(self, history_actions: jnp.ndarray) -> Dict[str, float]:
        """
        Analyze engagement patterns for authenticity.

        Args:
            history_actions: [S, num_actions] binary action matrix
        """
        # Action distribution
        action_sums = jnp.sum(history_actions, axis=0)
        action_probs = action_sums / (jnp.sum(action_sums) + 1e-6)

        # Entropy (varied engagement = more human)
        entropy = -jnp.sum(action_probs * jnp.log(action_probs + 1e-8))
        max_entropy = jnp.log(history_actions.shape[1])
        normalized_entropy = entropy / max_entropy

        # Single-action dominance (bots often only retweet)
        max_action_ratio = jnp.max(action_probs)

        # Negative engagement ratio (humans mute/block sometimes)
        negative_actions = action_sums[14:18]  # not_interested through report
        negative_ratio = jnp.sum(negative_actions) / (jnp.sum(action_sums) + 1e-6)

        # Dwell time presence (bots don't read)
        dwell_ratio = action_sums[10] / (jnp.sum(action_sums) + 1e-6)

        return {
            "entropy": float(normalized_entropy),
            "max_action_ratio": float(max_action_ratio),
            "negative_ratio": float(negative_ratio),
            "dwell_ratio": float(dwell_ratio),
            "authenticity_score": float(
                (normalized_entropy * 0.3) +
                ((1 - max_action_ratio) * 0.3) +
                (negative_ratio * 0.1) +  # Some negativity is human
                (dwell_ratio * 0.3)
            ),
        }
```

---

## 5. Quality Differentiation

### 5.1 Linear/Quantitative Signals

```python
@dataclass
class QuantitativeQualitySignals:
    # Engagement metrics
    like_count: int
    reply_count: int
    repost_count: int
    quote_count: int

    # Engagement velocity
    likes_per_hour: float
    engagement_acceleration: float  # Second derivative

    # Author metrics
    author_follower_count: int
    author_following_count: int
    author_post_count: int
    author_account_age_days: int

    # Content metrics
    post_length_chars: int
    has_media: bool
    has_url: bool
    url_domain_trust_score: float  # Pre-computed whitelist score

def compute_quantitative_score(signals: QuantitativeQualitySignals) -> float:
    """
    Pure quantitative scoring based on measurable metrics.
    """
    # Engagement quality (not just volume)
    reply_to_like = signals.reply_count / (signals.like_count + 1)
    quote_to_repost = signals.quote_count / (signals.repost_count + 1)
    engagement_quality = (reply_to_like + quote_to_repost) / 2

    # Author credibility
    follower_ratio = signals.author_follower_count / (signals.author_following_count + 1)
    author_maturity = min(1.0, signals.author_account_age_days / 365)
    author_score = (jnp.log1p(signals.author_follower_count) / 20) * author_maturity

    # Content richness
    content_score = (
        (signals.has_media * 0.3) +
        (min(1.0, signals.post_length_chars / 280) * 0.2) +
        ((1 - signals.has_url) * 0.1 + signals.url_domain_trust_score * signals.has_url * 0.2)
    )

    return (engagement_quality * 0.4) + (author_score * 0.3) + (content_score * 0.3)
```

### 5.2 Qualitative Signals

```python
@dataclass
class QualitativeQualitySignals:
    # Content analysis (from ML models)
    toxicity_score: float        # 0-1, from content classifier
    sentiment_score: float       # -1 to 1
    informativeness_score: float # 0-1, how much new info

    # Context quality
    is_reply_to_verified: bool
    is_in_quality_conversation: bool
    conversation_depth: int

    # Author quality
    author_verified: bool
    author_expertise_match: float  # How relevant to user's interests

    # Engagement quality
    quality_engager_ratio: float  # % of engagers who are quality accounts
    reply_sentiment_distribution: List[float]  # Sentiment of replies

def compute_qualitative_score(signals: QualitativeQualitySignals) -> float:
    """
    Qualitative scoring based on content and context analysis.
    """
    # Content quality
    content_quality = (
        (1 - signals.toxicity_score) * 0.3 +
        ((signals.sentiment_score + 1) / 2) * 0.1 +  # Normalize to 0-1
        signals.informativeness_score * 0.2
    )

    # Context quality
    context_quality = (
        (signals.is_reply_to_verified * 0.1) +
        (signals.is_in_quality_conversation * 0.2) +
        (min(1.0, signals.conversation_depth / 5) * 0.1)
    )

    # Author quality
    author_quality = (
        (signals.author_verified * 0.3) +
        (signals.author_expertise_match * 0.2)
    )

    # Engagement quality
    engagement_quality = signals.quality_engager_ratio * 0.3

    return (content_quality * 0.3 + context_quality * 0.2 +
            author_quality * 0.25 + engagement_quality * 0.25)
```

### 5.3 Combined Scoring

```python
def compute_final_score(
    engagement_probs: jnp.ndarray,
    quant_signals: QuantitativeQualitySignals,
    qual_signals: QualitativeQualitySignals,
    author_stats: AuthorStats,
    bot_detection: Dict[str, float],
) -> float:
    """
    Combine all signals into final ranking score.
    """
    # Base engagement score
    engagement_score = compute_weighted_score(engagement_probs)

    # Quality scores
    quant_score = compute_quantitative_score(quant_signals)
    qual_score = compute_qualitative_score(qual_signals)
    quality_score = (quant_score * 0.4 + qual_score * 0.6)

    # Penalties
    velocity_penalty = PostingVelocityPenalty().compute(author_stats)
    bot_penalty = bot_detection["bot_probability"]

    # Trust factor
    trust_factor = (1 - velocity_penalty) * (1 - bot_penalty)

    # Final score
    final_score = engagement_score * quality_score * trust_factor

    return final_score
```

---

## 6. Implementation Roadmap

### Phase 1: Quick Wins (1-2 weeks)

1. **Enable multi-action scoring** (change ~20 lines in `runners.py`)
2. **Add posting velocity penalty** (new scorer in Rust pipeline)
3. **Log all 19 predicted signals** for analysis

### Phase 2: Bot Detection (2-4 weeks)

1. **Add timestamp data** to `RecsysBatch`
2. **Implement temporal anomaly detector**
3. **Implement engagement pattern analyzer**
4. **Create bot probability scorer**

### Phase 3: Quality Signals (4-6 weeks)

1. **Add quantitative signals** to candidate metadata
2. **Integrate toxicity classifier** (external service)
3. **Implement quality tower** in Phoenix model
4. **Train combined scoring model**

### Phase 4: Advanced (6-8 weeks)

1. **Content recycling detection**
2. **Conversation quality analysis**
3. **Dynamic weight learning** (online optimization)
4. **A/B testing framework** for scoring changes
