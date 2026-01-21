# Understanding run_ranker.py

**File Location**: `phoenix/run_ranker.py`
**Purpose**: A demonstration script that shows how to use the ranking model to predict engagement probabilities for posts.

---

## What This File Does

This is an example script that:
1. Sets up the ranking model with specific configuration
2. Creates fake example data
3. Runs the model to get predictions
4. Displays the results in a readable format

This is useful for understanding how the system works and for testing.

---

## Line-by-Line Explanation

### Lines 24-54: Configuration
```python
def main():
    emb_size = 128          # Each entity is represented by 128 numbers
    num_actions = len(ACTIONS)  # 19 different engagement types
    history_seq_len = 32    # Consider last 32 posts the user engaged with
    candidate_seq_len = 8   # Rank 8 candidate posts at once
```

**Key configuration values explained:**

- `emb_size = 128`: The "vocabulary" of the model. Every user, post, and author is converted to 128 numbers. Larger = more expressive but slower.

- `history_seq_len = 32`: How much history to consider. 32 recent interactions is enough to understand short-term interests.

- `candidate_seq_len = 8`: How many posts to rank simultaneously. In production this might be 32 or more.

**Hash configuration (lines 32-36):**
```python
hash_config = HashConfig(
    num_user_hashes=2,
    num_item_hashes=2,
    num_author_hashes=2,
)
```
Each entity uses 2 hash functions. This reduces memory while maintaining uniqueness.

**Transformer configuration (lines 45-53):**
```python
model=TransformerConfig(
    emb_size=emb_size,           # 128 dimensions
    widening_factor=2,           # FFN is 2x wider than embedding
    key_size=64,                 # Attention key size
    num_q_heads=2,               # 2 query attention heads
    num_kv_heads=2,              # 2 key/value heads
    num_layers=2,                # Only 2 layers (very small for demo)
    attn_output_multiplier=0.125, # Scale attention scores down
)
```

This is a **very small** model for demonstration. Production models would have:
- `num_layers=12` or more
- `num_q_heads=48` or more
- `emb_size=512` or more

---

### Lines 56-67: Creating the Runner
```python
inference_runner = RecsysInferenceRunner(
    runner=ModelRunner(
        model=recsys_model,
        bs_per_device=0.125,  # Small batch for demo
    ),
    name="recsys_local",
)

print("Initializing model...")
inference_runner.initialize()
```

This:
1. Creates the model runner with our configuration
2. Initializes it (creates random parameters, compiles the model)

---

### Lines 69-85: Creating Example Data
```python
batch_size = 1
example_batch, example_embeddings = create_example_batch(
    batch_size=batch_size,
    emb_size=emb_size,
    history_len=history_seq_len,
    num_candidates=candidate_seq_len,
    num_actions=num_actions,
    ...
)
```

Creates fake data for one user with:
- Random history of posts they've engaged with
- Random candidate posts to rank
- Random embeddings (since we don't have real trained embeddings)

---

### Lines 89-92: Checking History
```python
valid_history_count = int((example_batch.history_post_hashes[:, :, 0] != 0).sum())
print(f"\nUser has viewed {valid_history_count} posts in their history")
```

Counts how many history positions are real (not padding). This shows the model has something to work with.

---

### Lines 94-95: Running the Model
```python
ranking_output = inference_runner.rank(example_batch, example_embeddings)
```

**This is the key line!** It:
1. Takes the user's profile, history, and candidate posts
2. Runs them through the Transformer
3. Returns predictions for all 19 engagement types for each candidate

---

### Lines 97-112: Displaying Results
```python
scores = np.array(ranking_output.scores[0])  # [8 candidates, 19 actions]
ranked_indices = np.array(ranking_output.ranked_indices[0])  # [8] - order by score

for rank, idx in enumerate(ranked_indices):
    print(f"\nRank {rank + 1}: ")
    for action_idx, action_name in enumerate(action_names):
        prob = float(scores[idx, action_idx])
        bar = "█" * int(prob * 20)  # Visual bar
        print(f"    {action_name}: {bar} {prob:.3f}")
```

For each candidate (in ranked order):
- Shows all 19 engagement probabilities
- Uses a visual bar chart to make it easy to see

---

## Example Output

When you run this script, you'll see something like:

```
RANKING RESULTS (ordered by predicted 'Favorite Score' probability)
----------------------------------------------------------------------

Rank 1:
  Predicted engagement probabilities:
    Favorite Score          : █████████████░░░░░░░░ 0.642
    Reply Score             : ████░░░░░░░░░░░░░░░░░ 0.213
    Repost Score            : ██████░░░░░░░░░░░░░░░ 0.298
    ...
    Not Interested Score    : ██░░░░░░░░░░░░░░░░░░░ 0.089
    Block Author Score      : ░░░░░░░░░░░░░░░░░░░░░ 0.012
    ...

Rank 2:
  ...
```

---

## Running This Script

```bash
cd phoenix/
uv run run_ranker.py
```

**What to expect:**
- The first run will compile the model (takes a few seconds)
- You'll see predictions for 8 fake posts
- The predictions will be random (since the model isn't trained)

---

## Summary

This script demonstrates the full ranking pipeline:
1. Configure the model
2. Initialize with random parameters
3. Create example inputs
4. Run inference to get predictions
5. Display results

It's a great starting point for understanding how the recommendation system works.
