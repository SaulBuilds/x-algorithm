# X Algorithm Documentation

This folder contains detailed explanations of every file in the X recommendation algorithm codebase. Each explanation is written for people with little to no computing experience who want to understand how the algorithm works.

---

## How to Read This Documentation

Start with the **Overview** below to understand the big picture, then dive into individual files based on what you want to learn.

**Recommended reading order:**
1. This README (overview)
2. Phoenix ML files (the "brain")
3. Candidate Pipeline (the framework)
4. Home Mixer (orchestration)
5. Thunder (data storage)

---

## Overview: How X's Recommendation System Works

When you open X and see your "For You" feed, here's what happens in about 200 milliseconds:

### Step 1: Request
Your phone sends a request to X's servers: "Show me some posts"

### Step 2: Retrieval (Phoenix Retrieval Model)
The system needs to find posts you might like from hundreds of millions of possibilities.

**How it works:**
- Your profile and recent activity are converted into 128 numbers (a "user embedding")
- Every post has been pre-converted into 128 numbers (a "post embedding")
- The system finds posts whose numbers are most similar to yours
- Result: ~1,000 candidate posts

### Step 3: Ranking (Phoenix Ranking Model)
Those 1,000 candidates need to be ranked by relevance.

**How it works:**
- The Transformer neural network processes your user info, history, and candidates
- For each candidate, it predicts 19 probabilities:
  - Will you like it?
  - Will you reply?
  - Will you repost?
  - Will you block the author?
  - etc.
- Posts are ranked by these predictions

### Step 4: Filtering (Home Mixer Filters)
Remove posts you shouldn't see:
- Posts you've already seen
- Posts from authors you've blocked/muted
- Posts with keywords you've muted
- Posts that violate policies

### Step 5: Selection
Pick the top 10-50 posts to show you

### Step 6: Response
Send the posts to your phone

---

## The Codebase Structure

```
x-algorithm/
├── phoenix/          # The ML "brain" (Python)
│   ├── grok.py       # Transformer neural network
│   ├── recsys_model.py         # Ranking model
│   ├── recsys_retrieval_model.py  # Retrieval model
│   └── runners.py    # Running the models
│
├── candidate-pipeline/  # Reusable pipeline framework (Rust)
│   ├── candidate_pipeline.rs  # Main execution engine
│   ├── source.rs     # Where candidates come from
│   ├── filter.rs     # What gets removed
│   ├── scorer.rs     # How things get ranked
│   └── selector.rs   # Picking the winners
│
├── home-mixer/       # Orchestration layer (Rust)
│   ├── server.rs     # API endpoint
│   ├── sources/      # Candidate sources
│   ├── filters/      # Various filters
│   ├── scorers/      # ML integration
│   └── selectors/    # Final selection
│
└── thunder/          # In-memory post store (Rust)
    ├── post_store.rs # Where recent posts live
    └── kafka/        # Real-time post updates
```

---

## File Documentation

### Phoenix (Python ML Models)

| File | Description | Documentation |
|------|-------------|---------------|
| grok.py | Transformer neural network - the core AI | [grok.py.md](./phoenix/grok.py.md) |
| recsys_model.py | Ranking model - predicts engagement | [recsys_model.py.md](./phoenix/recsys_model.py.md) |
| recsys_retrieval_model.py | Retrieval model - finds candidates | [recsys_retrieval_model.py.md](./phoenix/recsys_retrieval_model.py.md) |
| runners.py | Model execution code | [runners.py.md](./phoenix/runners.py.md) |
| run_ranker.py | Demo: ranking posts | [run_ranker.py.md](./phoenix/run_ranker.py.md) |
| run_retrieval.py | Demo: retrieving posts | [run_retrieval.py.md](./phoenix/run_retrieval.py.md) |

### Candidate Pipeline (Rust Framework)

The pipeline framework that all recommendation logic uses.

| File | Description | Documentation |
|------|-------------|---------------|
| Overview | Module introduction | [README.md](./candidate-pipeline/README.md) |
| lib.rs | Module entry point | [lib.rs.md](./candidate-pipeline/lib.rs.md) |
| candidate_pipeline.rs | Main execution engine | [candidate_pipeline.rs.md](./candidate-pipeline/candidate_pipeline.rs.md) |
| source.rs | Where candidates come from | [source.rs.md](./candidate-pipeline/source.rs.md) |
| hydrator.rs | Adding data to candidates | [hydrator.rs.md](./candidate-pipeline/hydrator.rs.md) |
| query_hydrator.rs | Adding data about user | [query_hydrator.rs.md](./candidate-pipeline/query_hydrator.rs.md) |
| filter.rs | Removing candidates | [filter.rs.md](./candidate-pipeline/filter.rs.md) |
| scorer.rs | Scoring candidates | [scorer.rs.md](./candidate-pipeline/scorer.rs.md) |
| selector.rs | Selecting winners | [selector.rs.md](./candidate-pipeline/selector.rs.md) |
| side_effect.rs | Background tasks | [side_effect.rs.md](./candidate-pipeline/side_effect.rs.md) |

### Home Mixer (Rust Orchestration)

The main service that builds your feed.

| File | Description | Documentation |
|------|-------------|---------------|
| Overview | Module introduction | [README.md](./home-mixer/README.md) |
| main.rs | Server entry point | [main.rs.md](./home-mixer/main.rs.md) |
| server.rs | gRPC API server | [server.rs.md](./home-mixer/server.rs.md) |
| candidate.rs | Post candidate structure | [candidate.rs.md](./home-mixer/candidate_pipeline/candidate.rs.md) |
| phoenix_candidate_pipeline.rs | Pipeline assembly | [phoenix_candidate_pipeline.rs.md](./home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs.md) |
| phoenix_scorer.rs | ML prediction scorer | [phoenix_scorer.rs.md](./home-mixer/scorers/phoenix_scorer.rs.md) |
| weighted_scorer.rs | Combining predictions | [weighted_scorer.rs.md](./home-mixer/scorers/weighted_scorer.rs.md) |

### Thunder (Rust Post Store)

In-memory storage for recent posts.

| File | Description | Documentation |
|------|-------------|---------------|
| Overview | Module introduction | [README.md](./thunder/README.md) |
| thunder_service.rs | gRPC API | [thunder_service.rs.md](./thunder/thunder_service.rs.md) |

---

## Key Concepts Explained

### Embeddings
Numbers that represent things. Instead of storing "User John who likes sports," we store `[0.23, -0.45, 0.89, ...]` - 128 numbers that capture John's characteristics. Similar users have similar numbers.

### Transformer
A type of neural network that's really good at understanding sequences. It can look at your history of interactions and understand patterns in what you engage with.

### Attention
The Transformer's way of deciding what's important. When scoring a post, it "pays attention" to relevant parts of your history. If you've liked many sports posts, it will pay attention to those when scoring a sports post.

### Hash Embeddings
A memory-saving trick. Instead of storing billions of unique embeddings, we use hash functions to map users/posts to a smaller set of embeddings. Two different users might share some embedding components.

### Two-Tower Architecture
Retrieval uses two "towers":
1. User Tower: Complex processing of user + history
2. Candidate Tower: Simple processing of post + author

Both output embeddings of the same size, so we can compare them using dot product.

### Candidate Isolation
When ranking multiple posts, each post is scored independently - it can't "see" the other candidates. This ensures consistent scores that can be cached.

---

## Critical Issues Found

Our audit identified several serious problems:

### 1. Zero-Initialized Weights (CRITICAL)
The neural network's weights start at zero, meaning it outputs zeros and cannot learn.

**Location:** `phoenix/grok.py` lines 148, 180
**Impact:** Model doesn't work
**Status:** Documented in audit

### 2. Unused Predictions (CRITICAL)
The model predicts 19 engagement types but only uses 1 (likes) for ranking.

**Location:** `phoenix/runners.py` line 345
**Impact:** Wasted computation, suboptimal ranking
**Status:** Fix proposed in Sprint 1

### 3. No Bot Detection
Zero mechanisms to detect or penalize bot accounts.

**Impact:** Spam and manipulation
**Status:** Solution proposed in Sprint 2

### 4. Empty Kafka Topics
Critical configuration is empty strings.

**Location:** `thunder/kafka_utils.rs` lines 15-19
**Impact:** Service cannot start
**Status:** Documented in Sprint 0

---

## Running the Code

### Prerequisites
- Python 3.10+
- uv (Python package manager)
- Rust toolchain

### Phoenix (Python)
```bash
cd phoenix/
uv run run_ranker.py      # Demo ranking
uv run run_retrieval.py   # Demo retrieval
uv run pytest             # Run tests
```

### Rust Components
```bash
cargo build --release
cargo test
```

---

## Questions This Documentation Answers

- How does X decide what posts to show me?
- What is a Transformer and how does it work?
- How can millions of posts be searched quickly?
- What data does X use about me?
- How is engagement predicted?
- Why do I see certain posts and not others?
- What are the security vulnerabilities?
- How could the algorithm be improved?

---

## Contributing

If you find errors or want to improve these explanations, contributions are welcome. The goal is to make this algorithm understandable to everyone, regardless of technical background.
