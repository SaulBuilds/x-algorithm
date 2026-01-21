# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **X For You Feed Algorithm** - an open-source recommendation system powering X's "For You" feed. It combines in-network and out-of-network content, ranks posts using Grok-based transformer models, and serves personalized feeds via gRPC APIs.

**Core Philosophy:** No hand-engineered features. The system relies entirely on transformer models to learn relevance from user engagement sequences.

## Build & Run Commands

### Python (Phoenix ML Models)
```bash
cd phoenix/
uv run run_ranker.py         # Run ranking inference example
uv run run_retrieval.py      # Run retrieval inference example
uv run pytest                # Run all tests
uv run pytest test_recsys_model.py test_recsys_retrieval_model.py  # Run specific tests
```

### Rust (Home-Mixer, Thunder, Candidate-Pipeline)
```bash
cargo build --release
cargo test

# Running services
# Thunder (in-network post store):
thunder --grpc_port 8090 --metrics_port 9090 --post_retention_seconds 604800

# Home-mixer (orchestration):
home-mixer --grpc_port 8091 --metrics_port 9091 --reload_interval_minutes 10 --chunk_size 1000
```

## Architecture

### System Flow
```
REQUEST → HOME-MIXER (Orchestration) → RANKED FEED RESPONSE
              ↓
         ├─ Query Hydration (user context)
         ├─ Candidate Sources:
         │  ├─ THUNDER (in-network from followed accounts)
         │  └─ PHOENIX (out-of-network ML retrieval)
         ├─ Hydration (enrich with metadata)
         ├─ Filtering (remove ineligible posts)
         ├─ Scoring (Phoenix ML model predictions)
         ├─ Selection (top-K by score)
         └─ Post-Selection Filtering (final validation)
```

### Four Main Modules

1. **HOME-MIXER** (`/home-mixer`) - Rust/gRPC orchestration layer
   - Assembles the For You feed by orchestrating the candidate pipeline
   - Implements `ScoredPostsService` via `HomeMixerServer`
   - Key subdirectories: `sources/`, `candidate_hydrators/`, `filters/`, `scorers/`, `selectors/`

2. **CANDIDATE-PIPELINE** (`/candidate-pipeline`) - Reusable Rust framework
   - Generic pipeline framework with traits: `Source`, `Hydrator`, `Filter`, `Scorer`, `Selector`, `SideEffect`, `QueryHydrator`
   - Query hydrators and sources run **in parallel**; filters and scorers run **sequentially**

3. **THUNDER** (`/thunder`) - Rust in-memory post store
   - Sub-millisecond lookups for recent posts from followed accounts
   - Consumes Kafka events for post create/delete
   - Auto-trims old posts based on retention settings

4. **PHOENIX** (`/phoenix`) - Python/JAX ML models
   - **Ranking Model:** Transformer with candidate isolation attention masking (candidates cannot attend to each other)
   - **Retrieval Model:** Two-tower architecture (user tower + candidate tower)
   - Built on Grok-1 transformer implementation (`grok.py`)
   - Predicts 15+ engagement probabilities: favorite, reply, repost, click, dwell, video_view, not_interested, block_author, etc.

### Key Design Patterns

- **Candidate Isolation:** ML scores don't depend on batch composition (cacheable)
- **Arc-wrapped shared state** for thread safety
- **Async traits** with `#[tonic::async_trait]` throughout
- **gRPC communication** between services (Gzip + Zstd compression)
- **Graceful error handling:** Pipeline continues on component failure

### Scoring System

Phoenix outputs multi-action probabilities combined via weighted scoring:
```
weighted_score = Σ(weight_i × P(action_i))
```
Includes positive weights (favorite, reply, repost) and negative weights (block, mute, not_interested).

## Extending the System

To add new pipeline components in `home-mixer/`:

- **Filter:** Implement `Filter<Q, C>` trait in `filters/`
- **Hydrator:** Implement `Hydrator<Q, C>` trait in `candidate_hydrators/`
- **Scorer:** Implement `Scorer<Q, C>` trait in `scorers/`
- **Source:** Implement `Source<Q, C>` trait in `sources/`

Register new components in `phoenix_candidate_pipeline.rs`.

## Excluded from Open Source

The following are not included in the open-source release:
- `home-mixer/clients/` - Service client implementations
- `home-mixer/params/` - Configuration (weights, thresholds)
- `home-mixer/util/` - Utility functions
