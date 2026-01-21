# phoenix_scorer.rs - Getting ML Predictions

## File Location
`home-mixer/scorers/phoenix_scorer.rs`

## Purpose
This file calls the Phoenix ML service to get predictions for each post. Phoenix predicts 19 different ways you might engage with a post - like, reply, repost, etc. These predictions are the foundation of how posts get ranked.

## Line-by-Line Explanation

### Imports (Lines 1-10)

```rust
use crate::candidate_pipeline::candidate::{PhoenixScores, PostCandidate};
```
**Line 1**: Import the candidate types (where scores get stored).

```rust
use crate::candidate_pipeline::query::ScoredPostsQuery;
```
**Line 2**: Import the query type (user info).

```rust
use crate::clients::phoenix_prediction_client::PhoenixPredictionClient;
```
**Line 3**: Import the client for calling Phoenix.

```rust
use std::collections::HashMap;
```
**Line 5**: HashMap for mapping tweet IDs to predictions.

```rust
use std::time::{SystemTime, UNIX_EPOCH};
```
**Line 7**: For getting current timestamp.

```rust
use xai_recsys_proto::{ActionName, ContinuousActionName};
```
**Line 10**: The action names for the 19 predictions.

### The Scorer Structure (Lines 12-14)

```rust
pub struct PhoenixScorer {
    pub phoenix_client: Arc<dyn PhoenixPredictionClient + Send + Sync>,
}
```
The scorer holds a reference to the Phoenix client.
- `Arc<dyn ...>` = Thread-safe pointer to any type implementing the client trait

### The score Method (Lines 16-76)

```rust
async fn score(
    &self,
    query: &ScoredPostsQuery,
    candidates: &[PostCandidate],
) -> Result<Vec<PostCandidate>, String> {
```
**Lines 19-23**: Method signature - takes query and candidates, returns scored candidates.

```rust
    let user_id = query.user_id as u64;
    let prediction_request_id = request_util::generate_request_id();
    let last_scored_at_ms = Self::current_timestamp_millis();
```
**Lines 24-26**: Prepare request metadata.
- `user_id` = Who is making the request
- `prediction_request_id` = Unique ID for this prediction batch
- `last_scored_at_ms` = Current timestamp

```rust
    if let Some(sequence) = &query.user_action_sequence {
```
**Line 28**: Check if we have the user's action history.
- Without history, we can't make personalized predictions

```rust
        let tweet_infos: Vec<xai_recsys_proto::TweetInfo> = candidates
            .iter()
            .map(|c| {
                let tweet_id = c.retweeted_tweet_id.unwrap_or(c.tweet_id as u64);
                let author_id = c.retweeted_user_id.unwrap_or(c.author_id);
                xai_recsys_proto::TweetInfo {
                    tweet_id,
                    author_id,
                    ..Default::default()
                }
            })
            .collect();
```
**Lines 29-40**: Prepare info about each post for Phoenix.
- For retweets, use the original tweet ID
- Each TweetInfo contains tweet_id and author_id

```rust
        let result = self
            .phoenix_client
            .predict(user_id, sequence.clone(), tweet_infos)
            .await;
```
**Lines 42-45**: **CALL PHOENIX** - Send the request.
- `user_id` = Who is asking
- `sequence` = User's recent actions
- `tweet_infos` = Posts to score

```rust
        if let Ok(response) = result {
            let predictions_map = self.build_predictions_map(&response);
```
**Lines 47-48**: If successful, build a map of predictions.

```rust
            let scored_candidates = candidates
                .iter()
                .map(|c| {
                    let lookup_tweet_id = c.retweeted_tweet_id.unwrap_or(c.tweet_id as u64);

                    let phoenix_scores = predictions_map
                        .get(&lookup_tweet_id)
                        .map(|preds| self.extract_phoenix_scores(preds))
                        .unwrap_or_default();

                    PostCandidate {
                        phoenix_scores,
                        prediction_request_id: Some(prediction_request_id),
                        last_scored_at_ms,
                        ..Default::default()
                    }
                })
                .collect();

            return Ok(scored_candidates);
```
**Lines 50-70**: Map predictions back to candidates.
- Look up each candidate's predictions
- Extract the 19 scores
- Return new candidates with scores attached

```rust
    // Return candidates unchanged if no scoring could be done
    Ok(candidates.to_vec())
```
**Lines 74-75**: If scoring failed, return candidates without scores.

### The update Method (Lines 78-82)

```rust
fn update(&self, candidate: &mut PostCandidate, scored: PostCandidate) {
    candidate.phoenix_scores = scored.phoenix_scores;
    candidate.prediction_request_id = scored.prediction_request_id;
    candidate.last_scored_at_ms = scored.last_scored_at_ms;
}
```
Copy the scored fields back to the original candidate.

### Helper Methods (Lines 85-158)

#### build_predictions_map (Lines 87-127)
```rust
fn build_predictions_map(
    &self,
    response: &xai_recsys_proto::PredictNextActionsResponse,
) -> HashMap<u64, ActionPredictions> {
```
Converts the Phoenix response into a map from tweet_id to predictions.

```rust
    let action_probs: HashMap<usize, f64> = distribution
        .top_log_probs
        .iter()
        .enumerate()
        .map(|(idx, log_prob)| (idx, (*log_prob as f64).exp()))
        .collect();
```
**Lines 103-108**: Convert log probabilities to regular probabilities.
- `log_prob.exp()` = Convert from log space (e^x)
- Neural networks often output log probabilities for numerical stability

#### extract_phoenix_scores (Lines 129-151)
```rust
fn extract_phoenix_scores(&self, p: &ActionPredictions) -> PhoenixScores {
    PhoenixScores {
        favorite_score: p.get(ActionName::ServerTweetFav),
        reply_score: p.get(ActionName::ServerTweetReply),
        retweet_score: p.get(ActionName::ServerTweetRetweet),
        // ... all 19 scores
    }
}
```
Maps the raw predictions to named fields.

### The 19 Actions

```rust
favorite_score: p.get(ActionName::ServerTweetFav),
```
**Line 131**: P(user will like this post)

```rust
reply_score: p.get(ActionName::ServerTweetReply),
```
**Line 132**: P(user will reply)

```rust
retweet_score: p.get(ActionName::ServerTweetRetweet),
```
**Line 133**: P(user will repost)

```rust
photo_expand_score: p.get(ActionName::ClientTweetPhotoExpand),
```
**Line 134**: P(user will tap to expand photo)

```rust
click_score: p.get(ActionName::ClientTweetClick),
```
**Line 135**: P(user will click the post)

```rust
profile_click_score: p.get(ActionName::ClientTweetClickProfile),
```
**Line 136**: P(user will click author's profile)

```rust
vqv_score: p.get(ActionName::ClientTweetVideoQualityView),
```
**Line 137**: P(user will watch video with quality)

```rust
share_score: p.get(ActionName::ClientTweetShare),
```
**Line 138**: P(user will share)

```rust
share_via_dm_score: p.get(ActionName::ClientTweetClickSendViaDirectMessage),
```
**Line 139**: P(user will share via DM)

```rust
share_via_copy_link_score: p.get(ActionName::ClientTweetShareViaCopyLink),
```
**Line 140**: P(user will copy link)

```rust
dwell_score: p.get(ActionName::ClientTweetRecapDwelled),
```
**Line 141**: P(user will pause and read)

```rust
quote_score: p.get(ActionName::ServerTweetQuote),
```
**Line 142**: P(user will quote tweet)

```rust
quoted_click_score: p.get(ActionName::ClientQuotedTweetClick),
```
**Line 143**: P(user will click quoted content)

```rust
follow_author_score: p.get(ActionName::ClientTweetFollowAuthor),
```
**Line 144**: P(user will follow author)

```rust
not_interested_score: p.get(ActionName::ClientTweetNotInterestedIn),
```
**Line 145**: P(user will mark "not interested")

```rust
block_author_score: p.get(ActionName::ClientTweetBlockAuthor),
```
**Line 146**: P(user will block author)

```rust
mute_author_score: p.get(ActionName::ClientTweetMuteAuthor),
```
**Line 147**: P(user will mute author)

```rust
report_score: p.get(ActionName::ClientTweetReport),
```
**Line 148**: P(user will report)

```rust
dwell_time: p.get_continuous(ContinuousActionName::DwellTime),
```
**Line 149**: Predicted dwell time (how long user will look)

## The Prediction Flow

```
┌────────────────────────────────────────────────────────────────────┐
│                        PhoenixScorer                                │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                     Prepare Request                                 │
│                                                                     │
│  User ID: 12345                                                    │
│  User History: [liked A, replied B, scrolled past C, ...]          │
│  Candidates: [Post 100, Post 200, Post 300, ...]                   │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                      Phoenix ML Service                             │
│                                                                     │
│  Input: User + History + Posts                                     │
│                                                                     │
│  Neural Network processes all inputs                               │
│                                                                     │
│  Output: 19 predictions per post                                   │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                     Process Response                                │
│                                                                     │
│  Post 100:                                                         │
│  ├─ P(like) = 0.72                                                │
│  ├─ P(reply) = 0.23                                               │
│  ├─ P(repost) = 0.45                                              │
│  ├─ P(block) = 0.001                                              │
│  └─ ... (19 total)                                                │
│                                                                     │
│  Post 200:                                                         │
│  ├─ P(like) = 0.89                                                │
│  └─ ...                                                           │
└────────────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Phoenix is the ML brain** - It predicts how you'll engage with each post
2. **19 engagement types** - Comprehensive prediction of all interactions
3. **Personalized to you** - Uses your action history
4. **Batch predictions** - All candidates scored in one call
5. **Log probabilities** - Converted from log space for numerical stability
6. **Graceful degradation** - Returns unscored if Phoenix fails
7. **Negative signals** - Predicts block/mute/report too (used later to penalize)
