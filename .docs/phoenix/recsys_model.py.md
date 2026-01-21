# Understanding recsys_model.py

**File Location**: `phoenix/recsys_model.py`
**Purpose**: This file defines how the recommendation system processes users, their history, and candidate posts to predict engagement.

---

## What This File Does

This file builds on the Transformer from `grok.py` to create a complete recommendation model. It:
1. Takes raw data (user IDs, post IDs, author IDs, actions)
2. Converts them into "embeddings" (numerical representations)
3. Combines them in meaningful ways
4. Passes them through the Transformer
5. Outputs predictions for how likely you are to take different actions

---

## Line-by-Line Explanation

### Lines 1-29: Imports
```python
import haiku as hk
import jax
import jax.numpy as jnp

from grok import (
    TransformerConfig,
    Transformer,
    layer_norm,
)
```

This file imports the Transformer we explained in the previous file, along with JAX for numerical computation.

---

### Lines 32-38: Hash Configuration
```python
@dataclass
class HashConfig:
    """Configuration for hash-based embeddings."""
    num_user_hashes: int = 2
    num_item_hashes: int = 2
    num_author_hashes: int = 2
```

**What hashing means here:**

Instead of storing a unique embedding for every single user (which would require billions of embeddings), the system uses "hashing." A user ID like `12345678` gets converted into 2 different hash values, and each hash looks up an embedding from a smaller table.

For example:
- User ID `12345678` → Hash 1 = `892`, Hash 2 = `1547`
- Look up embedding at index 892 and embedding at index 1547
- Combine these two embeddings to represent this user

This is called "hash embeddings" or "hash trick" and dramatically reduces memory usage.

**Why multiple hashes?**
Using 2 hashes reduces "collisions" (different users getting the same representation). If two users share hash 1, they probably don't share hash 2.

---

### Lines 41-53: RecsysEmbeddings Container
```python
@dataclass
class RecsysEmbeddings:
    """Container for pre-looked-up embeddings from the embedding tables."""
    user_embeddings: jax.typing.ArrayLike
    history_post_embeddings: jax.typing.ArrayLike
    candidate_post_embeddings: jax.typing.ArrayLike
    history_author_embeddings: jax.typing.ArrayLike
    candidate_author_embeddings: jax.typing.ArrayLike
```

This holds all the numerical representations (embeddings) that have been looked up from tables. The embedding lookup happens before this model runs - this model receives the results.

**Shape notation:**
- `[B, ...]` means Batch size (how many users we're processing at once)
- `[S, ...]` means Sequence length (history)
- `[C, ...]` means Candidate count
- `[D]` means Dimension (size of each embedding, e.g., 128 numbers)

---

### Lines 56-59: Model Output
```python
class RecsysModelOutput(NamedTuple):
    """Output of the recommendation model."""
    logits: jax.Array
```

The model outputs "logits" - raw scores before converting to probabilities. The shape is `[B, C, num_actions]` meaning: for each user in the batch, for each candidate, we get a score for each type of action (like, reply, repost, etc.).

---

### Lines 62-76: Input Batch
```python
class RecsysBatch(NamedTuple):
    """Input batch for the recommendation model."""
    user_hashes: jax.typing.ArrayLike              # [B, num_user_hashes]
    history_post_hashes: jax.typing.ArrayLike      # [B, S, num_item_hashes]
    history_author_hashes: jax.typing.ArrayLike    # [B, S, num_author_hashes]
    history_actions: jax.typing.ArrayLike          # [B, S, num_actions]
    history_product_surface: jax.typing.ArrayLike  # [B, S]
    candidate_post_hashes: jax.typing.ArrayLike    # [B, C, num_item_hashes]
    candidate_author_hashes: jax.typing.ArrayLike  # [B, C, num_author_hashes]
    candidate_product_surface: jax.typing.ArrayLike # [B, C]
```

**What each field means:**

- `user_hashes`: The hash values representing who you are
- `history_post_hashes`: Hash values for posts you've seen/engaged with
- `history_author_hashes`: Hash values for authors of those posts
- `history_actions`: What you did with each history post (liked, replied, etc.) - encoded as 0s and 1s
- `history_product_surface`: Where you saw each post (For You feed, Search, etc.)
- `candidate_post_hashes`: Hash values for posts we want to rank
- `candidate_author_hashes`: Hash values for authors of candidate posts
- `candidate_product_surface`: Where candidates would be shown

---

### Lines 79-119: User Embedding Combination
```python
def block_user_reduce(
    user_hashes: jnp.ndarray,
    user_embeddings: jnp.ndarray,
    num_user_hashes: int,
    emb_size: int,
    embed_init_scale: float = 1.0,
) -> Tuple[jax.Array, jax.Array]:
```

**What this does:**
Takes the multiple hash embeddings for a user and combines them into a single user representation.

**Step by step:**

1. **Reshape** (line 102):
```python
user_embedding = user_embeddings.reshape((B, 1, num_user_hashes * D))
```
If we have 2 hashes and each embedding is 128 numbers, this creates a vector of 256 numbers.

2. **Project** (lines 104-114):
```python
proj_mat_1 = hk.get_parameter(...)
user_embedding = jnp.dot(user_embedding, proj_mat_1)
```
Multiply by a learned matrix to compress 256 numbers back to 128 numbers. This projection learns the best way to combine the hash embeddings.

3. **Create padding mask** (line 117):
```python
user_padding_mask = (user_hashes[:, 0] != 0)
```
Check if the first hash is not zero. Hash value 0 means "no user" (padding).

---

### Lines 122-182: History Embedding Combination
```python
def block_history_reduce(...)
```

**What this does:**
For each post in your history, combines:
- Post embedding (what the post is about)
- Author embedding (who wrote it)
- Action embedding (what you did - liked, replied, etc.)
- Product surface embedding (where you saw it)

**Step by step:**

1. **Reshape** (lines 151-154):
```python
history_post_embeddings_reshaped = history_post_embeddings.reshape((B, S, num_item_hashes * D))
history_author_embeddings_reshaped = history_author_embeddings.reshape((B, S, num_author_hashes * D))
```

2. **Concatenate** (lines 156-164):
```python
post_author_embedding = jnp.concatenate([
    history_post_embeddings_reshaped,      # What post was it?
    history_author_embeddings_reshaped,    # Who wrote it?
    history_actions_embeddings,             # What did you do?
    history_product_surface_embeddings,    # Where did you see it?
], axis=-1)
```

3. **Project** (lines 166-176):
Compress all this information back to the embedding dimension.

4. **Create padding mask** (line 180):
```python
history_padding_mask = (history_post_hashes[:, :, 0] != 0)
```
Which history positions are real (not padding)?

---

### Lines 185-242: Candidate Embedding Combination
```python
def block_candidate_reduce(...)
```

Similar to history, but for candidate posts. Note that candidates don't have "actions" (you haven't interacted with them yet).

Combines:
- Post embedding
- Author embedding
- Product surface embedding (where it would be shown)

---

### Lines 245-281: Phoenix Model Configuration
```python
@dataclass
class PhoenixModelConfig:
    model: TransformerConfig          # Transformer settings
    emb_size: int                      # Embedding dimension
    num_actions: int                   # How many action types (19)
    history_seq_len: int = 128         # Max history length
    candidate_seq_len: int = 32        # Max candidates to rank
    hash_config: HashConfig = None
    product_surface_vocab_size: int = 16  # 16 different surfaces
```

**Key numbers:**
- `history_seq_len = 128`: The model considers your last 128 interactions
- `candidate_seq_len = 32`: We rank up to 32 candidates at once
- `product_surface_vocab_size = 16`: 16 different places content can appear (For You, Following, Search, etc.)

---

### Lines 284-321: Action Embeddings
```python
def _get_action_embeddings(self, actions: jax.Array) -> jax.Array:
```

**What this does:**
Converts the binary action vector (0s and 1s) into a rich embedding.

**Key line (314):**
```python
actions_signed = (2 * actions - 1).astype(jnp.float32)
```
Converts 0→-1 and 1→+1. This centering helps the model learn better.

**Security Issue Identified:**
No validation that actions are actually 0 or 1. If someone sends 100 or -100, the math would break.

---

### Lines 323-351: Single-Hot to Embeddings
```python
def _single_hot_to_embeddings(self, input, vocab_size, emb_size, name):
```

**What this does:**
Converts a category index (like "product surface = 3") into an embedding.

**How it works (lines 349-350):**
```python
input_one_hot = jax.nn.one_hot(input, vocab_size)
output = jnp.dot(input_one_hot, embedding_table)
```

1. Convert index to one-hot: `3` → `[0, 0, 0, 1, 0, 0, ...]`
2. Multiply by embedding table to get that row

**Efficiency Note:**
This one-hot approach is inefficient. It creates a large sparse matrix. A direct lookup (`embedding_table[input]`) would be faster.

---

### Lines 353-363: Unembedding Matrix
```python
def _get_unembedding(self) -> jax.Array:
```

**What this is:**
The opposite of embedding. Takes the Transformer's output and converts it to action scores.

Shape: `[emb_size, num_actions]` → `[128, 19]`

---

### Lines 365-437: Building Model Inputs
```python
def build_inputs(self, batch: RecsysBatch, recsys_embeddings: RecsysEmbeddings):
```

**What this does:**
Assembles all the pieces into a single sequence that the Transformer can process.

**The sequence structure:**
```
[USER] [HISTORY_1] [HISTORY_2] ... [HISTORY_128] [CANDIDATE_1] ... [CANDIDATE_32]
```

Total length: 1 + 128 + 32 = 161 positions

**Lines 428-433:**
```python
embeddings = jnp.concatenate(
    [user_embeddings, history_embeddings, candidate_embeddings], axis=1
)
padding_mask = jnp.concatenate(
    [user_padding_mask, history_padding_mask, candidate_padding_mask], axis=1
)
```

**Line 435:**
```python
candidate_start_offset = user_padding_mask.shape[1] + history_padding_mask.shape[1]
```
Records where candidates start (position 129). This is passed to the Transformer for the special attention mask.

---

### Lines 439-474: Forward Pass
```python
def __call__(self, batch: RecsysBatch, recsys_embeddings: RecsysEmbeddings):
```

**This is the main function that runs the model.**

**Step by step:**

1. **Build inputs** (lines 453-455):
```python
embeddings, padding_mask, candidate_start_offset = self.build_inputs(batch, recsys_embeddings)
```

2. **Run Transformer** (lines 458-462):
```python
model_output = self.model(
    embeddings,
    padding_mask,
    candidate_start_offset=candidate_start_offset,
)
```
The Transformer processes all positions, with the special attention mask ensuring candidates are scored independently.

3. **Normalize output** (line 466):
```python
out_embeddings = layer_norm(out_embeddings)
```

4. **Extract candidate embeddings** (line 468):
```python
candidate_embeddings = out_embeddings[:, candidate_start_offset:, :]
```
We only care about the candidate positions, not user/history.

5. **Convert to scores** (lines 470-472):
```python
unembeddings = self._get_unembedding()
logits = jnp.dot(candidate_embeddings, unembeddings)
```
Multiply by unembedding matrix to get `[B, 32, 19]` - a score for each candidate for each action type.

---

## Summary

This file builds the recommendation model by:

1. **Embedding**: Converting IDs (users, posts, authors) into numerical vectors
2. **Combining**: Merging multiple hash embeddings and different types of information
3. **Sequencing**: Arranging everything into a sequence: [USER, HISTORY..., CANDIDATES...]
4. **Processing**: Running through the Transformer with special attention mask
5. **Predicting**: Converting outputs to scores for each action type

**Key Design Decisions:**
- Uses hash embeddings to handle billions of users/posts with limited memory
- Separates embeddings (looked up elsewhere) from features (in this model)
- Outputs scores for 19 different action types, enabling nuanced ranking

**Security Issues:**
- No validation on action values (should be 0-1)
- No validation on hash values (should be within table bounds)
