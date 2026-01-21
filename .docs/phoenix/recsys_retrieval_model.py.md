# Understanding recsys_retrieval_model.py

**File Location**: `phoenix/recsys_retrieval_model.py`
**Purpose**: This file implements the "retrieval" stage - finding candidate posts from millions of possibilities before the ranking model scores them.

---

## What This File Does

Imagine you have 100 million posts that could be shown to a user. Running the full ranking model on all 100 million would be too slow. Instead:

1. **Retrieval** (this file): Quickly find ~1,000 potentially relevant posts
2. **Ranking** (recsys_model.py): Carefully score those ~1,000 to pick the best 10-50

This file implements a "two-tower" architecture:
- **User Tower**: Creates a numerical representation of you and your interests
- **Candidate Tower**: Creates a numerical representation of each post

Posts are retrieved by finding candidates whose representations are most similar to your user representation.

---

## Line-by-Line Explanation

### Lines 34-35: Constants
```python
EPS = 1e-12
INF = 1e12
```

Small and large numbers used to prevent mathematical errors:
- `EPS` (epsilon): Added to prevent division by zero
- `INF` (infinity): Used to represent impossibly bad scores

---

### Lines 38-44: Retrieval Output
```python
class RetrievalOutput(NamedTuple):
    user_representation: jax.Array    # [B, D] - Your interests as numbers
    top_k_indices: jax.Array          # [B, K] - Which posts to retrieve
    top_k_scores: jax.Array           # [B, K] - How relevant each post is
```

This is what the retrieval model returns:
- A numerical vector representing your interests
- The indices of the K most relevant posts
- How relevant each retrieved post is

---

### Lines 46-99: Candidate Tower
```python
@dataclass
class CandidateTower(hk.Module):
```

**What this does:**
Takes a post's embedding (and its author's embedding) and converts them into a normalized vector that can be compared to user vectors.

**Step by step (lines 57-99):**

1. **Reshape** (lines 68-73):
```python
post_author_embedding = jnp.reshape(post_author_embedding, (B, C, -1))
```
Flatten the multiple hash embeddings into one long vector.

2. **First projection** (lines 77-82):
```python
proj_1 = hk.get_parameter("candidate_tower_projection_1", ...)
hidden = jnp.dot(post_author_embedding, proj_1)
```
Project through a learned matrix.

3. **Activation** (line 92):
```python
hidden = jax.nn.silu(hidden)
```
Apply SiLU activation (Sigmoid Linear Unit) - a smooth non-linearity.

4. **Second projection** (lines 84-93):
```python
proj_2 = hk.get_parameter("candidate_tower_projection_2", ...)
candidate_embeddings = jnp.dot(hidden, proj_2)
```

5. **Normalize** (lines 95-97):
```python
candidate_norm = jnp.sqrt(jnp.maximum(candidate_norm_sq, EPS))
candidate_representation = candidate_embeddings / candidate_norm
```
Divide by the vector's length to get unit length. This is critical because similarity will be computed using dot product, and normalizing ensures all candidates are on the same scale.

---

### Lines 102-141: Retrieval Model Configuration
```python
@dataclass
class PhoenixRetrievalModelConfig:
    model: TransformerConfig
    emb_size: int
    history_seq_len: int = 128
    candidate_seq_len: int = 32
    hash_config: HashConfig = None
    product_surface_vocab_size: int = 16
```

Similar to the ranking model config, but for retrieval.

---

### Lines 144-159: The Retrieval Model Class
```python
@dataclass
class PhoenixRetrievalModel(hk.Module):
    """A two-tower retrieval model using the Phoenix transformer for user encoding."""
```

The model has two "towers":
1. **User Tower**: Uses the full Transformer to understand your interests from your history
2. **Candidate Tower**: A simpler neural network that just processes post+author embeddings

---

### Lines 206-276: Building User Representation
```python
def build_user_representation(self, batch, recsys_embeddings):
```

**What this does:**
Creates a single vector that captures your interests based on your profile and engagement history.

**Step by step:**

1. **Get product surface embeddings** (lines 227-232):
Where did you see each historical post?

2. **Get action embeddings** (line 234):
What did you do with each post?

3. **Combine user features** (lines 236-242):
```python
user_embeddings, user_padding_mask = block_user_reduce(...)
```

4. **Combine history features** (lines 244-253):
```python
history_embeddings, history_padding_mask = block_history_reduce(...)
```

5. **Concatenate** (lines 255-256):
```python
embeddings = jnp.concatenate([user_embeddings, history_embeddings], axis=1)
```
Note: Unlike the ranking model, we don't include candidates here. The user tower only processes user+history.

6. **Run Transformer** (lines 258-262):
```python
model_output = self.model(embeddings, padding_mask, candidate_start_offset=None)
```
`candidate_start_offset=None` means we use standard causal attention (no special candidate masking).

7. **Average pooling** (lines 266-270):
```python
mask_float = padding_mask.astype(jnp.float32)[:, :, None]
user_embeddings_masked = user_outputs * mask_float
user_embedding_sum = jnp.sum(user_embeddings_masked, axis=1)
user_representation = user_embedding_sum / jnp.maximum(mask_sum, 1.0)
```
Average all non-padding positions to get a single vector.

8. **Normalize** (lines 272-274):
```python
user_norm = jnp.sqrt(jnp.maximum(user_norm_sq, EPS))
user_representation = user_representation / user_norm
```
Make the vector unit length for dot-product similarity.

---

### Lines 278-312: Building Candidate Representations
```python
def build_candidate_representation(self, batch, recsys_embeddings):
```

**What this does:**
Processes candidate posts through the candidate tower.

Much simpler than user representation:
1. Concatenate post + author embeddings
2. Pass through candidate tower (MLP)
3. Output is already normalized (done inside CandidateTower)

---

### Lines 314-344: Main Forward Pass
```python
def __call__(self, batch, recsys_embeddings, corpus_embeddings, top_k, corpus_mask=None):
```

**What this does:**
Given a user and a corpus of posts, find the top-k most relevant.

**Steps:**

1. **Encode user** (line 334):
```python
user_representation, _ = self.build_user_representation(batch, recsys_embeddings)
```

2. **Retrieve** (lines 336-338):
```python
top_k_indices, top_k_scores = self._retrieve_top_k(
    user_representation, corpus_embeddings, top_k, corpus_mask
)
```

---

### Lines 346-372: Retrieval Logic
```python
def _retrieve_top_k(self, user_representation, corpus_embeddings, top_k, corpus_mask=None):
```

**This is where retrieval actually happens.**

**Line 365:**
```python
scores = jnp.matmul(user_representation, corpus_embeddings.T)
```
Compute dot product between user vector and all candidate vectors. Since both are unit length, this gives cosine similarity.

If user_representation is `[B, D]` and corpus_embeddings is `[N, D]`, the result is `[B, N]` - a score for every user-candidate pair.

**Lines 367-368:**
```python
if corpus_mask is not None:
    scores = jnp.where(corpus_mask[None, :], scores, -INF)
```
If some corpus entries are invalid, set their scores to negative infinity.

**Line 370:**
```python
top_k_scores, top_k_indices = jax.lax.top_k(scores, top_k)
```
Get the K highest scores and their indices. This is the retrieval!

---

## How Retrieval Works in Production

In production, the system doesn't compute similarity against all 100 million posts every time. Instead:

1. **Offline**: Pre-compute embeddings for all posts using the candidate tower
2. **Index**: Store these embeddings in a specialized index (like FAISS or ScaNN)
3. **Online**: When a user arrives:
   - Compute their user representation using the user tower
   - Query the index for nearest neighbors
   - Return top-K candidates for ranking

The index uses approximate nearest neighbor (ANN) algorithms to find similar vectors in milliseconds even with billions of items.

---

## Summary

This file implements efficient retrieval using a two-tower architecture:

**User Tower** (complex):
- Processes user profile + engagement history
- Uses full Transformer
- Creates a single "interest vector"

**Candidate Tower** (simple):
- Processes post + author embeddings
- Uses a 2-layer neural network
- Creates a "content vector"

**Retrieval**:
- Compare user vector to candidate vectors using dot product
- Return the K most similar candidates

**Why two towers?**
- User representation changes constantly (new engagements)
- Candidate representations can be pre-computed and indexed
- This separation enables efficient retrieval from massive corpuses

**Key Properties:**
- Normalized vectors ensure fair comparison
- Dot product = cosine similarity (since vectors are unit length)
- Can be accelerated with approximate nearest neighbor indexes
