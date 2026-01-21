# Home Mixer Module

## What This Module Does

The home-mixer is the **main service** that builds your X feed. When you open X and see posts, this is the code that decided which posts to show you and in what order. It's called "home mixer" because it mixes together posts from different sources to create your home timeline.

This module uses the candidate-pipeline framework (the blueprints) to create a real, working recommendation system.

## How Home Mixer Fits in the System

```
┌─────────────────────────────────────────────────────────────────────┐
│                          X Mobile App                               │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 │ "Get my feed"
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         HOME MIXER                                   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              PhoenixCandidatePipeline                        │   │
│  │                                                              │   │
│  │  Query Hydrators → Sources → Hydrators → Filters →          │   │
│  │  → Scorers → Selector → Post-Selection → Side Effects        │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
        │              │              │               │
        ▼              ▼              ▼               ▼
   ┌────────┐    ┌────────┐    ┌────────┐     ┌────────────┐
   │Phoenix │    │Thunder │    │Gizmoduck│    │Visibility  │
   │(ML)    │    │(Posts) │    │(Users) │    │Filtering   │
   └────────┘    └────────┘    └────────┘     └────────────┘
```

## The Files in This Module

### Core Files

| File | Purpose |
|------|---------|
| `lib.rs` | Lists all submodules |
| `main.rs` | Starts the server |
| `server.rs` | Handles incoming requests |

### Candidate Pipeline

| File | Purpose |
|------|---------|
| `candidate_pipeline/phoenix_candidate_pipeline.rs` | Assembles the full pipeline |
| `candidate_pipeline/candidate.rs` | Defines what a post candidate looks like |
| `candidate_pipeline/query.rs` | Defines what a user request looks like |
| `candidate_pipeline/candidate_features.rs` | Features extracted from posts |
| `candidate_pipeline/query_features.rs` | Features extracted from users |

### Sources (Where Posts Come From)

| File | Purpose |
|------|---------|
| `sources/phoenix_source.rs` | Gets posts via Phoenix ML retrieval |
| `sources/thunder_source.rs` | Gets posts from Thunder database |

### Hydrators (Adding Data to Posts)

| File | Purpose |
|------|---------|
| `candidate_hydrators/core_data_candidate_hydrator.rs` | Adds basic post data |
| `candidate_hydrators/gizmoduck_hydrator.rs` | Adds author info |
| `candidate_hydrators/in_network_candidate_hydrator.rs` | Marks if author is followed |
| `candidate_hydrators/subscription_hydrator.rs` | Adds subscription status |
| `candidate_hydrators/vf_candidate_hydrator.rs` | Adds visibility filtering data |
| `candidate_hydrators/video_duration_candidate_hydrator.rs` | Adds video length |

### Query Hydrators (Adding Data About User)

| File | Purpose |
|------|---------|
| `query_hydrators/user_action_seq_query_hydrator.rs` | Gets user's recent actions |
| `query_hydrators/user_features_query_hydrator.rs` | Gets user features for ML |

### Filters (Removing Posts)

| File | Purpose |
|------|---------|
| `filters/age_filter.rs` | Removes old posts |
| `filters/author_socialgraph_filter.rs` | Removes blocked/muted authors |
| `filters/core_data_hydration_filter.rs` | Removes failed hydrations |
| `filters/dedup_conversation_filter.rs` | Removes duplicate conversations |
| `filters/drop_duplicates_filter.rs` | Removes duplicate posts |
| `filters/ineligible_subscription_filter.rs` | Removes subscription content without access |
| `filters/muted_keyword_filter.rs` | Removes posts with muted words |
| `filters/previously_seen_posts_filter.rs` | Removes already seen posts |
| `filters/previously_served_posts_filter.rs` | Removes already served posts |
| `filters/retweet_deduplication_filter.rs` | Removes duplicate retweets |
| `filters/self_tweet_filter.rs` | Removes your own posts |
| `filters/vf_filter.rs` | Applies visibility filtering rules |

### Scorers (Ranking Posts)

| File | Purpose |
|------|---------|
| `scorers/phoenix_scorer.rs` | Gets ML predictions from Phoenix |
| `scorers/weighted_scorer.rs` | Combines predictions into one score |
| `scorers/author_diversity_scorer.rs` | Penalizes same author appearing too often |
| `scorers/oon_scorer.rs` | Adjusts score for out-of-network posts |

### Selectors (Picking Top Posts)

| File | Purpose |
|------|---------|
| `selectors/top_k_score_selector.rs` | Picks the top N highest-scored posts |

### Side Effects (Background Tasks)

| File | Purpose |
|------|---------|
| `side_effects/cache_request_info_side_effect.rs` | Caches request info |

## The 19 Engagement Signals

The algorithm predicts 19 different ways you might engage with each post:

| Signal | What It Predicts |
|--------|------------------|
| `favorite_score` | Will you like this? |
| `reply_score` | Will you reply? |
| `retweet_score` | Will you repost? |
| `photo_expand_score` | Will you tap to see the full image? |
| `click_score` | Will you click the post? |
| `profile_click_score` | Will you click the author's profile? |
| `vqv_score` | Will you watch the video with quality? |
| `share_score` | Will you share it? |
| `share_via_dm_score` | Will you share via direct message? |
| `share_via_copy_link_score` | Will you copy the link? |
| `dwell_score` | Will you pause and read it? |
| `quote_score` | Will you quote tweet it? |
| `quoted_click_score` | Will you click on quoted content? |
| `follow_author_score` | Will you follow the author? |
| `not_interested_score` | Will you mark "not interested"? |
| `block_author_score` | Will you block the author? |
| `mute_author_score` | Will you mute the author? |
| `report_score` | Will you report the post? |
| `dwell_time` | How long will you look at it? |

## Pipeline Flow

```
1. REQUEST ARRIVES
   User: 12345, Country: US, Language: EN

2. QUERY HYDRATION
   + user_action_sequence (last 100 actions)
   + user_features (ML features for this user)

3. FETCH CANDIDATES
   Phoenix Source → ~800 candidates (ML retrieval)
   Thunder Source → ~200 candidates (recent posts)
   Total: ~1000 candidates

4. HYDRATE CANDIDATES
   + core_data (post text, timestamps)
   + author_info (name, avatar, follower count)
   + in_network flag (do you follow them?)
   + video_duration (if applicable)
   + subscription_info (if applicable)

5. FILTER CANDIDATES
   - duplicates
   - old posts (> 48 hours)
   - your own posts
   - blocked authors
   - muted keywords
   - already seen posts
   - already served posts
   After filters: ~500 candidates

6. SCORE CANDIDATES
   Phoenix Scorer → 19 engagement predictions each
   Weighted Scorer → combine into single score
   Author Diversity → penalize repeated authors
   OON Scorer → adjust out-of-network scores

7. SELECT TOP CANDIDATES
   Sort by score, take top 50

8. POST-SELECTION
   + visibility filtering check
   - remove any that fail visibility rules
   - deduplicate conversations

9. SIDE EFFECTS (background)
   - cache request info

10. RETURN FEED
    50 scored posts → Your timeline
```

## Key Terms

- **In-Network**: Posts from accounts you follow
- **Out-of-Network (OON)**: Posts from accounts you don't follow
- **Gizmoduck**: X's user profile service
- **Thunder**: X's post storage service
- **Strato**: X's key-value storage service
- **Visibility Filtering (VF)**: Content policy enforcement
- **TES**: Tweet Entity Service (post metadata)

## Reading Order

For best understanding, read the files in this order:

1. `server.rs` - Entry point for requests
2. `candidate_pipeline/candidate.rs` - What a post candidate looks like
3. `candidate_pipeline/phoenix_candidate_pipeline.rs` - How the pipeline is assembled
4. `sources/` - Where posts come from
5. `filters/` - What gets removed
6. `scorers/` - How posts are ranked
7. `selectors/` - How top posts are chosen
