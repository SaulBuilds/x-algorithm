# Candidate Pipeline Module

## What This Module Does

The candidate-pipeline module is a **framework** - a set of building blocks that other parts of the system use to process posts. Think of it like a factory assembly line template that can be customized for different products.

When X needs to build your feed, it doesn't process posts randomly. It follows a structured pipeline (a series of steps) that:

1. Gets information about you (the user)
2. Finds posts that might interest you
3. Adds more details to those posts
4. Removes posts that shouldn't appear
5. Scores the remaining posts
6. Picks the best ones

This module provides the **blueprint** for that assembly line. The actual workers on the assembly line (specific filters, scorers, etc.) are defined in the home-mixer module.

## The Files in This Module

| File | Purpose |
|------|---------|
| `lib.rs` | Entry point - lists all the other files |
| `candidate_pipeline.rs` | The main assembly line controller |
| `source.rs` | Where posts come from |
| `hydrator.rs` | Adds details to posts |
| `query_hydrator.rs` | Adds details about the user |
| `filter.rs` | Removes unwanted posts |
| `scorer.rs` | Gives posts a score |
| `selector.rs` | Picks the best posts |
| `side_effect.rs` | Extra tasks that don't affect results |

## How They Work Together

```
User Request
     ↓
┌─────────────────────────────────────────────────────────────┐
│                    CANDIDATE PIPELINE                        │
│                                                              │
│  1. Query Hydrators  →  Learn about the user                │
│           ↓                                                  │
│  2. Sources          →  Find candidate posts                │
│           ↓                                                  │
│  3. Hydrators        →  Add details to posts                │
│           ↓                                                  │
│  4. Filters          →  Remove unwanted posts               │
│           ↓                                                  │
│  5. Scorers          →  Give each post a score              │
│           ↓                                                  │
│  6. Selector         →  Pick top posts                      │
│           ↓                                                  │
│  7. Post-Selection   →  Final details & filtering           │
│           ↓                                                  │
│  8. Side Effects     →  Logging, analytics (background)     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
     ↓
Your Feed
```

## Key Concepts

### Traits (Blueprints)

In Rust, a "trait" is like a job description. It says "anything that wants to be a Filter must be able to do these things." The actual filters are defined elsewhere - this module just defines what a filter should be capable of.

### Generics (Q and C)

You'll see `<Q, C>` throughout this code. These are placeholders:
- **Q** = Query (information about the user and their request)
- **C** = Candidate (a post that might appear in the feed)

Using placeholders makes the code flexible - the same pipeline code can work with different types of queries and candidates.

### Async/Await

Operations marked with `async` can wait for other things to happen (like network requests) without blocking everything else. When you see `await`, it means "wait here for this to finish."

## Reading Order

For best understanding, read the files in this order:

1. `lib.rs` - See what's available
2. `source.rs` - Where posts come from
3. `hydrator.rs` - How details are added
4. `filter.rs` - How posts are removed
5. `scorer.rs` - How posts are scored
6. `selector.rs` - How winners are picked
7. `query_hydrator.rs` - How user info is gathered
8. `side_effect.rs` - Background tasks
9. `candidate_pipeline.rs` - How it all fits together
