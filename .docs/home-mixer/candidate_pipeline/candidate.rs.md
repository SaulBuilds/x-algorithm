# candidate.rs - What a Post Looks Like

## File Location
`home-mixer/candidate_pipeline/candidate.rs`

## Purpose
This file defines `PostCandidate` - the structure that represents a post as it flows through the pipeline. Every post starts with basic info and gets enriched with more data as it passes through hydrators. This is the central data structure of the recommendation system.

## Line-by-Line Explanation

### Imports (Lines 1-3)

```rust
use std::collections::HashMap;
```
**Line 1**: Import HashMap for key-value storage.
- Used to map user IDs to screen names

```rust
use xai_home_mixer_proto as pb;
```
**Line 2**: Import protocol buffer definitions.

```rust
use xai_visibility_filtering::models as vf;
```
**Line 3**: Import visibility filtering models.
- Used for content policy filtering reasons

### The PostCandidate Structure (Lines 5-27)

```rust
#[derive(Clone, Debug, Default)]
pub struct PostCandidate {
```
**Lines 5-6**: Define the PostCandidate structure.
- `#[derive(Clone, Debug, Default)]` = Automatically implement:
  - `Clone` = Can be copied
  - `Debug` = Can be printed for debugging
  - `Default` = Can be created with default values
- `pub struct` = Public structure

#### Basic Post Information

```rust
    pub tweet_id: i64,
```
**Line 7**: The unique identifier for this post.
- `i64` = 64-bit signed integer
- Every post on X has a unique ID

```rust
    pub author_id: u64,
```
**Line 8**: The user ID of who wrote this post.
- `u64` = 64-bit unsigned integer

```rust
    pub tweet_text: String,
```
**Line 9**: The actual text content of the post.

#### Reply and Repost Information

```rust
    pub in_reply_to_tweet_id: Option<u64>,
```
**Line 10**: If this is a reply, the ID of the post it's replying to.
- `Option<u64>` = Either contains a value (`Some(123)`) or nothing (`None`)
- `None` if this is not a reply

```rust
    pub retweeted_tweet_id: Option<u64>,
```
**Line 11**: If this is a repost, the ID of the original post.

```rust
    pub retweeted_user_id: Option<u64>,
```
**Line 12**: If this is a repost, the user ID of the original author.

#### ML Scores (Added by Phoenix Scorer)

```rust
    pub phoenix_scores: PhoenixScores,
```
**Line 13**: Container for all 19 engagement predictions.
- See PhoenixScores structure below

```rust
    pub prediction_request_id: Option<u64>,
```
**Line 14**: ID of the prediction request (for debugging/tracing).

```rust
    pub last_scored_at_ms: Option<u64>,
```
**Line 15**: Timestamp when this post was last scored.
- In milliseconds since Unix epoch

```rust
    pub weighted_score: Option<f64>,
```
**Line 16**: Combined score from weighted_scorer.
- Result of combining multiple engagement predictions

```rust
    pub score: Option<f64>,
```
**Line 17**: Final score after all scoring adjustments.
- This is what determines position in your feed

#### Network and Serving Information

```rust
    pub served_type: Option<pb::ServedType>,
```
**Line 18**: How this post was served (e.g., algorithm, chronological).

```rust
    pub in_network: Option<bool>,
```
**Line 19**: Is this from someone the user follows?
- `true` = In-network (followed account)
- `false` = Out-of-network (not followed)

```rust
    pub ancestors: Vec<u64>,
```
**Line 20**: Parent posts if this is part of a conversation thread.
- List of tweet IDs leading to this post

#### Additional Metadata (Added by Hydrators)

```rust
    pub video_duration_ms: Option<i32>,
```
**Line 21**: If this post has a video, how long is it (in milliseconds)?

```rust
    pub author_followers_count: Option<i32>,
```
**Line 22**: How many followers does the author have?

```rust
    pub author_screen_name: Option<String>,
```
**Line 23**: The author's @username.

```rust
    pub retweeted_screen_name: Option<String>,
```
**Line 24**: If reposted, the original author's @username.

```rust
    pub visibility_reason: Option<vf::FilteredReason>,
```
**Line 25**: If visibility filtering applies, why?
- Used for content policy explanations

```rust
    pub subscription_author_id: Option<u64>,
```
**Line 26**: If this is subscription-only content, the creator's ID.

```rust
}
```
**Line 27**: End of PostCandidate structure.

### The PhoenixScores Structure (Lines 29-51)

```rust
#[derive(Clone, Debug, Default)]
pub struct PhoenixScores {
```
**Lines 29-30**: Container for all ML predictions.

#### Engagement Predictions

```rust
    pub favorite_score: Option<f64>,
```
**Line 31**: Probability the user will like this post.

```rust
    pub reply_score: Option<f64>,
```
**Line 32**: Probability the user will reply.

```rust
    pub retweet_score: Option<f64>,
```
**Line 33**: Probability the user will repost.

```rust
    pub photo_expand_score: Option<f64>,
```
**Line 34**: Probability the user will tap to expand a photo.

```rust
    pub click_score: Option<f64>,
```
**Line 35**: Probability the user will click the post.

```rust
    pub profile_click_score: Option<f64>,
```
**Line 36**: Probability the user will click the author's profile.

```rust
    pub vqv_score: Option<f64>,
```
**Line 37**: Video Quality Views - probability of quality video viewing.

```rust
    pub share_score: Option<f64>,
```
**Line 38**: Probability the user will share.

```rust
    pub share_via_dm_score: Option<f64>,
```
**Line 39**: Probability of sharing via direct message.

```rust
    pub share_via_copy_link_score: Option<f64>,
```
**Line 40**: Probability of copying the link.

```rust
    pub dwell_score: Option<f64>,
```
**Line 41**: Probability the user will pause and read.

```rust
    pub quote_score: Option<f64>,
```
**Line 42**: Probability the user will quote tweet.

```rust
    pub quoted_click_score: Option<f64>,
```
**Line 43**: Probability of clicking on quoted content.

```rust
    pub follow_author_score: Option<f64>,
```
**Line 44**: Probability of following the author.

#### Negative Engagement Predictions

```rust
    pub not_interested_score: Option<f64>,
```
**Line 45**: Probability the user will mark "not interested".

```rust
    pub block_author_score: Option<f64>,
```
**Line 46**: Probability the user will block the author.

```rust
    pub mute_author_score: Option<f64>,
```
**Line 47**: Probability the user will mute the author.

```rust
    pub report_score: Option<f64>,
```
**Line 48**: Probability the user will report the post.

#### Continuous Predictions

```rust
    // Continuous actions
    pub dwell_time: Option<f64>,
```
**Lines 49-50**: Predicted time the user will spend looking at this post.
- Unlike other scores (0-1 probability), this is actual time

```rust
}
```
**Line 51**: End of PhoenixScores structure.

### Helper Trait and Implementation (Lines 53-70)

```rust
pub trait CandidateHelpers {
    fn get_screen_names(&self) -> HashMap<u64, String>;
}
```
**Lines 53-55**: Define a helper trait.
- `get_screen_names()` = Returns a map of user IDs to usernames

```rust
impl CandidateHelpers for PostCandidate {
    fn get_screen_names(&self) -> HashMap<u64, String> {
        let mut screen_names = HashMap::<u64, String>::new();
```
**Lines 57-59**: Implement the helper for PostCandidate.
- Create an empty HashMap

```rust
        if let Some(author_screen_name) = self.author_screen_name.clone() {
            screen_names.insert(self.author_id, author_screen_name);
        }
```
**Lines 60-62**: Add author's screen name if available.
- `if let Some(...)` = Pattern matching to extract value if present
- `screen_names.insert(...)` = Add to the map

```rust
        if let (Some(retweeted_screen_name), Some(retweeted_user_id)) =
            (self.retweeted_screen_name.clone(), self.retweeted_user_id)
        {
            screen_names.insert(retweeted_user_id, retweeted_screen_name);
        }
```
**Lines 63-67**: Add reposted author's screen name if this is a repost.
- `if let (Some(...), Some(...))` = Only if both values are present

```rust
        screen_names
    }
}
```
**Lines 68-70**: Return the map.

## How a Post Gets Enriched

```
INITIAL STATE (from Source)
┌─────────────────────────────────┐
│ tweet_id: 12345                 │
│ author_id: 67890                │
│ (everything else: None/empty)  │
└─────────────────────────────────┘
            │
            ▼
AFTER CoreDataHydrator
┌─────────────────────────────────┐
│ tweet_id: 12345                 │
│ author_id: 67890                │
│ tweet_text: "Hello world!"      │
│ in_reply_to_tweet_id: None      │
│ retweeted_tweet_id: None        │
└─────────────────────────────────┘
            │
            ▼
AFTER GizmoduckHydrator
┌─────────────────────────────────┐
│ + author_screen_name: "@user"   │
│ + author_followers_count: 1000  │
└─────────────────────────────────┘
            │
            ▼
AFTER InNetworkHydrator
┌─────────────────────────────────┐
│ + in_network: true              │
└─────────────────────────────────┘
            │
            ▼
AFTER PhoenixScorer
┌─────────────────────────────────┐
│ + phoenix_scores:               │
│   - favorite_score: 0.72        │
│   - reply_score: 0.23           │
│   - retweet_score: 0.45         │
│   - ... (all 19 scores)         │
└─────────────────────────────────┘
            │
            ▼
AFTER WeightedScorer
┌─────────────────────────────────┐
│ + weighted_score: 0.67          │
└─────────────────────────────────┘
            │
            ▼
FINAL STATE
┌─────────────────────────────────┐
│ + score: 0.65                   │
│ (after diversity adjustments)   │
└─────────────────────────────────┘
```

## Why All Fields Are Optional

Most fields are `Option<T>` because:

1. **Gradual enrichment**: Posts start minimal and get data added over time
2. **Failure tolerance**: If a hydrator fails, the post still has its other data
3. **Source variation**: Different sources provide different initial data
4. **Efficiency**: Not all fields are needed for all posts

## Key Takeaways

1. **Central data structure** - Every post in the pipeline is a PostCandidate
2. **19 engagement signals** - The ML model predicts 19 different user behaviors
3. **Gradual enrichment** - Data is added as the post moves through the pipeline
4. **Options everywhere** - Fields might not be populated, which is okay
5. **Scoring hierarchy** - phoenix_scores → weighted_score → final score
6. **Helper methods** - CandidateHelpers provides utility functions
