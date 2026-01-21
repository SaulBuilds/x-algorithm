# server.rs - Request Handler

## File Location
`home-mixer/server.rs`

## Purpose
This file handles incoming requests for scored posts. When the X app asks "what posts should I show this user?", this is the code that receives that request and returns the answer.

## Line-by-Line Explanation

### Imports (Lines 1-10)

```rust
use crate::candidate_pipeline::candidate::CandidateHelpers;
```
**Line 1**: Import helper functions for working with candidates.

```rust
use crate::candidate_pipeline::phoenix_candidate_pipeline::PhoenixCandidatePipeline;
```
**Line 2**: Import the main pipeline that processes posts.

```rust
use crate::candidate_pipeline::query::ScoredPostsQuery;
```
**Line 3**: Import the query structure (describes what the user wants).

```rust
use log::info;
```
**Line 4**: Import logging function.

```rust
use std::sync::Arc;
```
**Line 5**: Import Arc for thread-safe shared ownership.

```rust
use std::time::Instant;
```
**Line 6**: Import Instant for measuring how long things take.

```rust
use tonic::{Request, Response, Status};
```
**Line 7**: Import gRPC types.
- `Request` = An incoming request
- `Response` = An outgoing response
- `Status` = Error status codes (like "invalid argument")

```rust
use xai_candidate_pipeline::candidate_pipeline::CandidatePipeline;
```
**Line 8**: Import the CandidatePipeline trait.

```rust
use xai_home_mixer_proto as pb;
```
**Line 9**: Import protocol buffer definitions.

```rust
use xai_home_mixer_proto::{ScoredPost, ScoredPostsResponse};
```
**Line 10**: Import specific message types.

### The HomeMixerServer Structure (Lines 12-22)

```rust
pub struct HomeMixerServer {
    phx_candidate_pipeline: Arc<PhoenixCandidatePipeline>,
}
```
**Lines 12-14**: Define the server structure.
- `pub struct` = A public structure
- `phx_candidate_pipeline` = The pipeline that does all the work
- `Arc<...>` = Thread-safe shared pointer (multiple requests can use the same pipeline)

```rust
impl HomeMixerServer {
    pub async fn new() -> Self {
        HomeMixerServer {
            phx_candidate_pipeline: Arc::new(PhoenixCandidatePipeline::prod().await),
        }
    }
}
```
**Lines 16-22**: Create a new server instance.
- `pub async fn new()` = Public async constructor
- `PhoenixCandidatePipeline::prod()` = Create a production pipeline (connects to real services)
- `Arc::new(...)` = Wrap in thread-safe pointer

### The gRPC Service Implementation (Lines 24-83)

```rust
#[tonic::async_trait]
impl pb::scored_posts_service_server::ScoredPostsService for HomeMixerServer {
```
**Lines 24-25**: Implement the gRPC service interface.
- `#[tonic::async_trait]` = Enable async methods in trait implementation
- `impl ... for HomeMixerServer` = HomeMixerServer provides this service

```rust
    #[xai_stats_macro::receive_stats]
    async fn get_scored_posts(
        &self,
        request: Request<pb::ScoredPostsQuery>,
    ) -> Result<Response<ScoredPostsResponse>, Status> {
```
**Lines 26-30**: The main method that handles requests.
- `#[xai_stats_macro::receive_stats]` = Collect statistics about this endpoint
- `async fn get_scored_posts` = The method name (defined in protobuf)
- `request: Request<pb::ScoredPostsQuery>` = The incoming request
- `-> Result<Response<...>, Status>` = Returns either posts or an error

```rust
        let proto_query = request.into_inner();
```
**Line 31**: Extract the query from the request wrapper.
- `into_inner()` = Get the actual data from the Request envelope

```rust
        if proto_query.viewer_id == 0 {
            return Err(Status::invalid_argument("viewer_id must be specified"));
        }
```
**Lines 33-35**: Validate the request.
- If viewer_id is 0 (not set), return an error immediately
- `Status::invalid_argument` = Standard gRPC error code

```rust
        let start = Instant::now();
```
**Line 37**: Start timing the request.
- Used to measure how long the pipeline takes

```rust
        let query = ScoredPostsQuery::new(
            proto_query.viewer_id,
            proto_query.client_app_id,
            proto_query.country_code,
            proto_query.language_code,
            proto_query.seen_ids,
            proto_query.served_ids,
            proto_query.in_network_only,
            proto_query.is_bottom_request,
            proto_query.bloom_filter_entries,
        );
```
**Lines 38-48**: Create an internal query from the protobuf query.
- `viewer_id` = The user requesting posts
- `client_app_id` = Which X app they're using (iOS, Android, web)
- `country_code` = User's country
- `language_code` = User's language
- `seen_ids` = Posts they've already seen
- `served_ids` = Posts already served to them
- `in_network_only` = Only show posts from followed accounts?
- `is_bottom_request` = Loading more (scrolled to bottom)?
- `bloom_filter_entries` = Efficient filter for seen content

```rust
        info!("Scored Posts request - request_id {}", query.request_id);
```
**Line 49**: Log that we received a request.

```rust
        let pipeline_result = self.phx_candidate_pipeline.execute(query).await;
```
**Line 50**: **RUN THE PIPELINE** - This is where all the magic happens.
- Calls the execute method from CandidatePipeline
- Returns scored, filtered, selected posts

```rust
        let scored_posts: Vec<ScoredPost> = pipeline_result
            .selected_candidates
            .into_iter()
            .map(|candidate| {
```
**Lines 52-55**: Convert internal candidates to protobuf format.
- `selected_candidates` = The final posts chosen by the pipeline
- `into_iter()` = Iterate over them, consuming the vector
- `map(|candidate| {...})` = Transform each candidate

```rust
                let screen_names = candidate.get_screen_names();
                ScoredPost {
                    tweet_id: candidate.tweet_id as u64,
                    author_id: candidate.author_id,
                    retweeted_tweet_id: candidate.retweeted_tweet_id.unwrap_or(0),
                    retweeted_user_id: candidate.retweeted_user_id.unwrap_or(0),
                    in_reply_to_tweet_id: candidate.in_reply_to_tweet_id.unwrap_or(0),
                    score: candidate.score.unwrap_or(0.0) as f32,
                    in_network: candidate.in_network.unwrap_or(false),
                    served_type: candidate.served_type.map(|t| t as i32).unwrap_or_default(),
                    last_scored_timestamp_ms: candidate.last_scored_at_ms.unwrap_or(0),
                    prediction_request_id: candidate.prediction_request_id.unwrap_or(0),
                    ancestors: candidate.ancestors,
                    screen_names,
                    visibility_reason: candidate.visibility_reason.map(|r| r.into()),
                }
            })
            .collect();
```
**Lines 56-73**: Build the ScoredPost protobuf message.
- Copy all relevant fields from our internal format to the protobuf format
- `unwrap_or(0)` / `unwrap_or(false)` = Use default if not set
- `.collect()` = Gather all transformed items into a vector

Fields included:
- `tweet_id` = The post's unique ID
- `author_id` = Who wrote it
- `retweeted_tweet_id` = If it's a repost, the original post ID
- `score` = The relevance score
- `in_network` = Is this from a followed account?
- `served_type` = How was this post served?
- `ancestors` = Parent posts in conversation
- `screen_names` = Usernames involved
- `visibility_reason` = Why this might be filtered

```rust
        info!(
            "Scored Posts response - request_id {} - {} posts ({} ms)",
            pipeline_result.query.request_id,
            scored_posts.len(),
            start.elapsed().as_millis()
        );
```
**Lines 75-80**: Log the response with timing.
- How many posts were returned
- How long it took in milliseconds

```rust
        Ok(Response::new(ScoredPostsResponse { scored_posts }))
    }
}
```
**Lines 81-83**: Return the successful response.
- `Ok(...)` = Success
- `Response::new(...)` = Wrap in gRPC Response
- `ScoredPostsResponse { scored_posts }` = The protobuf response with posts

## Request-Response Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                       INCOMING REQUEST                               │
│                                                                      │
│  ScoredPostsQuery {                                                 │
│    viewer_id: 12345,                                                │
│    client_app_id: "twitter-ios",                                    │
│    country_code: "US",                                              │
│    language_code: "en",                                             │
│    seen_ids: [100, 200, 300],                                       │
│    served_ids: [100, 200],                                          │
│    in_network_only: false,                                          │
│    is_bottom_request: false,                                        │
│    bloom_filter_entries: [...],                                      │
│  }                                                                   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    VALIDATION                                        │
│                                                                      │
│  viewer_id != 0 ? ✓                                                 │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              PIPELINE EXECUTION (~150ms)                             │
│                                                                      │
│  phx_candidate_pipeline.execute(query)                              │
│                                                                      │
│  1. Query Hydration (user info)                                     │
│  2. Fetch Candidates (posts)                                        │
│  3. Hydrate Candidates (details)                                    │
│  4. Filter (remove unwanted)                                        │
│  5. Score (rank by relevance)                                       │
│  6. Select (pick top N)                                             │
│  7. Post-Selection (final checks)                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      OUTGOING RESPONSE                               │
│                                                                      │
│  ScoredPostsResponse {                                              │
│    scored_posts: [                                                  │
│      { tweet_id: 456, author_id: 789, score: 0.92, ... },          │
│      { tweet_id: 457, author_id: 790, score: 0.87, ... },          │
│      { tweet_id: 458, author_id: 791, score: 0.83, ... },          │
│      ... (50 posts)                                                 │
│    ]                                                                 │
│  }                                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Single entry point** - All feed requests come through get_scored_posts
2. **Input validation** - Checks viewer_id before processing
3. **Pipeline delegation** - The actual work happens in the pipeline
4. **Format conversion** - Converts internal format to protobuf for network
5. **Performance logging** - Tracks timing for each request
6. **Error handling** - Returns proper gRPC errors on failure
