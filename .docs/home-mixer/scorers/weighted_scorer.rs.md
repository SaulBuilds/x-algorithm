# weighted_scorer.rs - Combining Predictions Into One Score

## File Location
`home-mixer/scorers/weighted_scorer.rs`

## Purpose
This file takes the 19 engagement predictions from Phoenix and combines them into a single score. Each prediction gets multiplied by a weight (how important that action is), and then they're all added together. The result is one number that represents how "good" a post is for you.

## Line-by-Line Explanation

### Imports (Lines 1-6)

```rust
use crate::candidate_pipeline::candidate::{PhoenixScores, PostCandidate};
```
**Line 1**: Import the candidate types.

```rust
use crate::params as p;
```
**Line 3**: Import parameters - the weights for each action.
- `p::FAVORITE_WEIGHT`, `p::REPLY_WEIGHT`, etc.

```rust
use crate::util::score_normalizer::normalize_score;
```
**Line 4**: Import score normalization function.

### The Scorer Structure (Line 8)

```rust
pub struct WeightedScorer;
```
A simple structure with no fields - all the logic is in methods.

### The score Method (Lines 10-32)

```rust
async fn score(
    &self,
    _query: &ScoredPostsQuery,
    candidates: &[PostCandidate],
) -> Result<Vec<PostCandidate>, String> {
```
**Lines 13-17**: Method signature.
- `_query` = Prefixed with underscore because it's not used

```rust
    let scored = candidates
        .iter()
        .map(|c| {
            let weighted_score = Self::compute_weighted_score(c);
            let normalized_weighted_score = normalize_score(c, weighted_score);

            PostCandidate {
                weighted_score: Some(normalized_weighted_score),
                ..Default::default()
            }
        })
        .collect();

    Ok(scored)
```
**Lines 18-31**: For each candidate:
1. Compute the weighted score
2. Normalize it (scale to a standard range)
3. Return a new candidate with the score

### The update Method (Lines 34-36)

```rust
fn update(&self, candidate: &mut PostCandidate, scored: PostCandidate) {
    candidate.weighted_score = scored.weighted_score;
}
```
Copy the weighted_score to the original candidate.

### Helper Methods

#### apply (Lines 40-42)
```rust
fn apply(score: Option<f64>, weight: f64) -> f64 {
    score.unwrap_or(0.0) * weight
}
```
Multiply a score by its weight, treating None as 0.

#### compute_weighted_score (Lines 44-70)
```rust
fn compute_weighted_score(candidate: &PostCandidate) -> f64 {
    let s: &PhoenixScores = &candidate.phoenix_scores;

    let vqv_weight = Self::vqv_weight_eligibility(candidate);

    let combined_score = Self::apply(s.favorite_score, p::FAVORITE_WEIGHT)
        + Self::apply(s.reply_score, p::REPLY_WEIGHT)
        + Self::apply(s.retweet_score, p::RETWEET_WEIGHT)
        + Self::apply(s.photo_expand_score, p::PHOTO_EXPAND_WEIGHT)
        + Self::apply(s.click_score, p::CLICK_WEIGHT)
        + Self::apply(s.profile_click_score, p::PROFILE_CLICK_WEIGHT)
        + Self::apply(s.vqv_score, vqv_weight)
        + Self::apply(s.share_score, p::SHARE_WEIGHT)
        + Self::apply(s.share_via_dm_score, p::SHARE_VIA_DM_WEIGHT)
        + Self::apply(s.share_via_copy_link_score, p::SHARE_VIA_COPY_LINK_WEIGHT)
        + Self::apply(s.dwell_score, p::DWELL_WEIGHT)
        + Self::apply(s.quote_score, p::QUOTE_WEIGHT)
        + Self::apply(s.quoted_click_score, p::QUOTED_CLICK_WEIGHT)
        + Self::apply(s.dwell_time, p::CONT_DWELL_TIME_WEIGHT)
        + Self::apply(s.follow_author_score, p::FOLLOW_AUTHOR_WEIGHT)
        + Self::apply(s.not_interested_score, p::NOT_INTERESTED_WEIGHT)
        + Self::apply(s.block_author_score, p::BLOCK_AUTHOR_WEIGHT)
        + Self::apply(s.mute_author_score, p::MUTE_AUTHOR_WEIGHT)
        + Self::apply(s.report_score, p::REPORT_WEIGHT);

    Self::offset_score(combined_score)
}
```

This is the core formula. Each prediction is multiplied by its weight and summed:

```
Score = (P_like × W_like) + (P_reply × W_reply) + ... + (P_report × W_report)
```

#### vqv_weight_eligibility (Lines 72-81)
```rust
fn vqv_weight_eligibility(candidate: &PostCandidate) -> f64 {
    if candidate
        .video_duration_ms
        .is_some_and(|ms| ms > p::MIN_VIDEO_DURATION_MS)
    {
        p::VQV_WEIGHT
    } else {
        0.0
    }
}
```
Only apply video quality view weight if the video is long enough.
- Short videos/non-videos get 0 weight for VQV

#### offset_score (Lines 83-91)
```rust
fn offset_score(combined_score: f64) -> f64 {
    if p::WEIGHTS_SUM == 0.0 {
        combined_score.max(0.0)
    } else if combined_score < 0.0 {
        (combined_score + p::NEGATIVE_WEIGHTS_SUM) / p::WEIGHTS_SUM * p::NEGATIVE_SCORES_OFFSET
    } else {
        combined_score + p::NEGATIVE_SCORES_OFFSET
    }
}
```
Adjust the score to ensure it's in a usable range:
- If negative, scale it up
- Add an offset to separate from zero

## How Weighting Works

```
Post Predictions                    Weights                 Contribution
──────────────                    ─────────                 ────────────
P(like) = 0.72            ×       W_like = 1.0       =      0.720
P(reply) = 0.23           ×       W_reply = 0.5      =      0.115
P(repost) = 0.45          ×       W_repost = 2.0     =      0.900
P(photo_expand) = 0.30    ×       W_expand = 0.3     =      0.090
P(click) = 0.60           ×       W_click = 0.5      =      0.300
P(profile_click) = 0.15   ×       W_profile = 0.3    =      0.045
P(vqv) = 0.40             ×       W_vqv = 1.0        =      0.400  (if video)
P(share) = 0.10           ×       W_share = 1.0      =      0.100
P(dwell) = 0.80           ×       W_dwell = 0.5      =      0.400
...
P(not_interested) = 0.05  ×       W_not_int = -1.0   =     -0.050
P(block) = 0.01           ×       W_block = -5.0     =     -0.050
P(mute) = 0.02            ×       W_mute = -3.0      =     -0.060
P(report) = 0.001         ×       W_report = -10.0   =     -0.010
                                                          ─────────
                                          Raw Score:        2.900
                                          After offset:     3.100
```

## Why Weights Matter

The weights encode X's priorities:

### Positive Engagement (Higher Weight = More Important)
- **Repost (2.0)**: Creating new content is highly valued
- **Quote (1.5)**: Adding commentary is valuable
- **Like (1.0)**: Standard positive signal
- **Share (1.0)**: Spreading content is good

### Lighter Positive Signals (Lower Weight)
- **Click (0.5)**: Shows interest, but less commitment
- **Reply (0.5)**: Engagement, but could be negative replies
- **Dwell (0.5)**: Reading is good, but passive
- **Profile click (0.3)**: Mild curiosity

### Negative Signals (Negative Weight)
- **Report (-10.0)**: Strong signal of bad content
- **Block (-5.0)**: User wants to avoid this author
- **Mute (-3.0)**: User wants less of this
- **Not interested (-1.0)**: Mild negative signal

## The Formula in Plain Language

```
Score = (Good predictions × Positive weights) + (Bad predictions × Negative weights)
```

A post with:
- High like/repost/share predictions = High score
- High block/mute/report predictions = Lower score
- Both = Depends on magnitude

## Example Calculations

### Post A: Tech news article
```
P(like)=0.60, P(reply)=0.10, P(repost)=0.30
P(block)=0.01, P(report)=0.001

Score = (0.60 × 1.0) + (0.10 × 0.5) + (0.30 × 2.0)
      + (0.01 × -5.0) + (0.001 × -10.0)
      = 0.60 + 0.05 + 0.60 - 0.05 - 0.01
      = 1.19
```

### Post B: Controversial political take
```
P(like)=0.40, P(reply)=0.50, P(repost)=0.20
P(block)=0.15, P(report)=0.08

Score = (0.40 × 1.0) + (0.50 × 0.5) + (0.20 × 2.0)
      + (0.15 × -5.0) + (0.08 × -10.0)
      = 0.40 + 0.25 + 0.40 - 0.75 - 0.80
      = -0.50
```
Negative score! This post would rank very low despite high engagement predictions.

### Post C: Cute animal video
```
P(like)=0.85, P(repost)=0.60, P(vqv)=0.70
P(block)=0.001, P(report)=0.0005

Score = (0.85 × 1.0) + (0.60 × 2.0) + (0.70 × 1.0)
      + (0.001 × -5.0) + (0.0005 × -10.0)
      = 0.85 + 1.20 + 0.70 - 0.005 - 0.005
      = 2.74
```
High score! Would rank near the top.

## Key Takeaways

1. **Weighted sum** - All 19 predictions combined into one score
2. **Weights encode priorities** - X decides what engagement matters most
3. **Negative signals matter** - Block/mute/report predictions lower scores
4. **Video special case** - VQV weight only applies to long videos
5. **Normalization** - Scores are scaled to a standard range
6. **Simple but powerful** - Linear combination is interpretable
7. **Tunable** - Changing weights changes feed behavior
