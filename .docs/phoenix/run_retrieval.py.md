# Understanding run_retrieval.py

**File Location**: `phoenix/run_retrieval.py`
**Purpose**: A demonstration script that shows how to use the retrieval model to find relevant posts from a large corpus.

---

## What This File Does

This script demonstrates the two-tower retrieval system:
1. Creates a "corpus" of 1,000 fake posts
2. Creates 2 example users with engagement histories
3. Retrieves the top 10 most relevant posts for each user
4. Displays the results

---

## Line-by-Line Explanation

### Lines 31-61: Configuration
```python
def main():
    emb_size = 128
    num_actions = len(ACTIONS)
    history_seq_len = 32
    candidate_seq_len = 8

    hash_config = HashConfig(...)

    retrieval_model_config = PhoenixRetrievalModelConfig(
        emb_size=emb_size,
        history_seq_len=history_seq_len,
        candidate_seq_len=candidate_seq_len,
        hash_config=hash_config,
        product_surface_vocab_size=16,
        model=TransformerConfig(...),
    )
```

Same configuration as the ranking model. The retrieval model uses the same architecture for encoding users.

---

### Lines 63-74: Creating the Runner
```python
inference_runner = RecsysRetrievalInferenceRunner(
    runner=RetrievalModelRunner(
        model=retrieval_model_config,
        bs_per_device=0.125,
    ),
    name="retrieval_local",
)

print("Initializing retrieval model...")
inference_runner.initialize()
```

Creates and initializes the retrieval model.

---

### Lines 76-92: Creating User Data
```python
batch_size = 2  # Two users for demo
example_batch, example_embeddings = create_example_batch(
    batch_size=batch_size,
    emb_size=emb_size,
    history_len=history_seq_len,
    num_candidates=candidate_seq_len,
    num_actions=num_actions,
    ...
)
```

Creates fake data for **2 users**. This demonstrates that retrieval works for multiple users at once.

---

### Lines 98-113: Creating the Corpus
```python
corpus_size = 1000  # 1000 fake posts
corpus_embeddings, corpus_post_ids = create_example_corpus(
    corpus_size=corpus_size,
    emb_size=emb_size,
    seed=456,
)
print(f"Corpus embeddings shape: {corpus_embeddings.shape}")  # (1000, 128)

inference_runner.set_corpus(corpus_embeddings, corpus_post_ids)
```

**What is a corpus?**
In retrieval, the "corpus" is the collection of all posts that could potentially be shown. In production, this might be tens of millions of posts.

**What `set_corpus` does:**
Stores the pre-computed embeddings for all posts. During retrieval, the user's embedding is compared against all these embeddings.

**In production:**
The corpus embeddings would be:
1. Pre-computed offline using the candidate tower
2. Stored in an efficient index (like FAISS)
3. Updated periodically as new posts are created

---

### Lines 115-125: Running Retrieval
```python
top_k = 10
retrieval_output = inference_runner.retrieve(
    example_batch,
    example_embeddings,
    top_k=top_k,
)
```

**This is the key operation!**

For each user:
1. Encode their profile and history into a single vector
2. Compare that vector against all 1,000 posts
3. Return the 10 most similar posts

---

### Lines 127-140: Displaying Results
```python
top_k_indices = np.array(retrieval_output.top_k_indices)  # [2 users, 10 posts]
top_k_scores = np.array(retrieval_output.top_k_scores)    # [2 users, 10 scores]

for user_idx in range(batch_size):
    print(f"\n  User {user_idx + 1}:")
    for rank in range(top_k):
        post_id = top_k_indices[user_idx, rank]
        score = top_k_scores[user_idx, rank]
        print(f"    Rank {rank + 1}: Post {post_id}, Score {score:.4f}")
```

Shows which posts were retrieved for each user and their similarity scores.

---

## Example Output

```
RETRIEVAL SYSTEM DEMO
======================================================================

Users have viewed 48 posts total in their history

----------------------------------------------------------------------
STEP 1: Creating Candidate Corpus
----------------------------------------------------------------------
Corpus size: 1000 posts
Corpus embeddings shape: (1000, 128)

----------------------------------------------------------------------
STEP 2: Retrieving Top-K Candidates
----------------------------------------------------------------------

Retrieved top 10 candidates for each of 2 users:

  User 1:
    Rank   Post ID      Score
    ------------------------------
    1      742          █████████████████████ 0.3421
    2      891          ████████████████████░ 0.3156
    3      123          ███████████████████░░ 0.2987
    ...

  User 2:
    Rank   Post ID      Score
    ------------------------------
    1      456          █████████████████████ 0.3589
    2      789          ████████████████████░ 0.3234
    ...
```

Notice that:
- Different users get different posts (personalization works!)
- Scores indicate similarity (higher = more relevant)
- In this demo, scores are random since the model isn't trained

---

## Running This Script

```bash
cd phoenix/
uv run run_retrieval.py
```

---

## How This Differs from Ranking

| Aspect | Retrieval | Ranking |
|--------|-----------|---------|
| Purpose | Find candidates | Score candidates |
| Input | All posts in corpus | ~32 candidate posts |
| Output | Top-K post IDs | Engagement probabilities |
| Speed | Fast (dot products) | Slower (full Transformer) |
| Candidates | Millions possible | ~32 at a time |

---

## The Two-Stage Pipeline

In production, X's recommendation system works in two stages:

**Stage 1: Retrieval (this script)**
- Input: User + their history
- Process: Compare user embedding to corpus
- Output: ~1,000 candidate posts
- Time: ~10ms for millions of posts (with indexing)

**Stage 2: Ranking (run_ranker.py)**
- Input: User + 32 candidates at a time
- Process: Full Transformer inference
- Output: Engagement probabilities
- Time: ~50ms per batch

This two-stage approach is how X can:
- Consider millions of posts for relevance
- Still provide personalized feeds in real-time
- Scale to hundreds of millions of users

---

## Summary

This script demonstrates:
1. Creating a corpus of candidate posts
2. Encoding users into embedding space
3. Finding the most relevant posts via similarity
4. Personalizing results for different users

The two-tower architecture enables efficient retrieval at scale by pre-computing candidate embeddings and using simple dot products for similarity.
