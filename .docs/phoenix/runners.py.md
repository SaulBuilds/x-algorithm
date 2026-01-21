# Understanding runners.py

**File Location**: `phoenix/runners.py`
**Purpose**: This file provides the interface for actually running the models - initializing them, loading parameters, and making predictions.

---

## What This File Does

The previous files define **what** the models compute. This file handles **how** to actually use them:
1. Initialize the model with random parameters
2. Load trained parameters (in production, these come from training)
3. Create the functions that run inference (making predictions)
4. Provide convenient APIs for ranking and retrieval

---

## Line-by-Line Explanation

### Lines 41-77: Creating Dummy Batches
```python
def create_dummy_batch_from_config(
    hash_config, history_len, num_candidates, num_actions, batch_size=1
) -> RecsysBatch:
```

**What this does:**
Creates a fake batch filled with zeros for model initialization. Neural network libraries like JAX need to know the exact shapes of inputs to compile the model. This function creates correctly-shaped placeholder data.

**Why zeros?**
Hash value 0 is reserved for "padding" (no data). Creating dummy data with zeros ensures the model handles padding correctly during initialization.

---

### Lines 80-117: Creating Dummy Embeddings
```python
def create_dummy_embeddings_from_config(...) -> RecsysEmbeddings:
```

Same idea - creates fake embeddings of the right shape for initialization.

---

### Lines 120-152: Base Model Runner
```python
@dataclass
class BaseModelRunner(ABC):
    bs_per_device: float = 2.0
    rng_seed: int = 42
```

**What this is:**
An abstract base class (template) that both the ranking and retrieval runners inherit from.

**Key settings:**
- `bs_per_device`: Batch size per GPU (how many users to process at once per GPU)
- `rng_seed`: Random seed for reproducibility (42 is a common default, a reference to The Hitchhiker's Guide to the Galaxy)

**The `initialize` method (lines 143-152):**
```python
def initialize(self):
    self.model.initialize()
    self.model.fprop_dtype = jnp.bfloat16
    num_local_gpus = len(jax.local_devices())
    self.batch_size = max(1, int(self.bs_per_device * num_local_gpus))
    self.forward = self.make_forward_fn()
```

This:
1. Initializes the model configuration
2. Sets the data type to `bfloat16` (a compact number format that uses less memory)
3. Counts available GPUs and calculates total batch size
4. Creates the forward function

---

### Lines 202-222: The Actions List
```python
ACTIONS: List[str] = [
    "favorite_score",          # 0: Did they like it?
    "reply_score",             # 1: Did they reply?
    "repost_score",            # 2: Did they retweet?
    "photo_expand_score",      # 3: Did they expand photos?
    "click_score",             # 4: Did they click?
    "profile_click_score",     # 5: Did they click the author's profile?
    "vqv_score",               # 6: Video Quality View (watched enough of video)
    "share_score",             # 7: Did they share?
    "share_via_dm_score",      # 8: Shared via direct message?
    "share_via_copy_link_score", # 9: Copied the link?
    "dwell_score",             # 10: Did they spend time looking at it?
    "quote_score",             # 11: Did they quote tweet?
    "quoted_click_score",      # 12: Clicked on a quote tweet?
    "follow_author_score",     # 13: Followed the author?
    "not_interested_score",    # 14: Clicked "not interested"?
    "block_author_score",      # 15: Blocked the author?
    "mute_author_score",       # 16: Muted the author?
    "report_score",            # 17: Reported the post?
    "dwell_time",              # 18: How long did they look at it?
]
```

**This is crucial!** The model predicts 19 different types of engagement:

**Positive signals (good):**
- Like, reply, repost, share, quote, follow, dwell

**Negative signals (bad):**
- Not interested, block, mute, report

**Neutral/informational:**
- Click, profile click, photo expand, video view, dwell time

The model learns to predict the probability of each action for each candidate post.

---

### Lines 225-254: Ranking Output
```python
class RankingOutput(NamedTuple):
    scores: jax.Array
    ranked_indices: jax.Array
    p_favorite_score: jax.Array
    p_reply_score: jax.Array
    # ... one field for each action
```

This contains all the model's predictions:
- `scores`: All 19 probabilities for all candidates
- `ranked_indices`: Candidates sorted by score
- `p_*_score`: Individual probability for each action type

---

### Lines 257-298: Model Runner
```python
class ModelRunner(BaseModelRunner):
```

**What this does:**
Handles initialization and running of the ranking model.

**The `make_forward_fn` method (lines 276-281):**
```python
def make_forward_fn(self):
    def forward(batch, recsys_embeddings):
        out = self.model.make()(batch, recsys_embeddings)
        return out
    return hk.transform(forward)
```

`hk.transform` converts a function that uses Haiku modules into a "pure" function that explicitly takes and returns parameters. This is required by JAX.

**The `init` method (lines 283-289):**
```python
def init(self, rng, data, embeddings) -> TrainingState:
    params = self.forward.init(init_rng, data, embeddings)
    return TrainingState(params=params)
```

Initializes the model's parameters by running the forward function once with random numbers.

---

### Lines 301-386: Inference Runner
```python
class RecsysInferenceRunner(BaseInferenceRunner):
```

**This is what actually gets used to make predictions.**

**The `initialize` method (lines 315-374):**

```python
def initialize(self):
    # Create dummy data
    dummy_batch = self.create_dummy_batch(batch_size=1)
    dummy_embeddings = self.create_dummy_embeddings(batch_size=1)

    # Initialize the underlying model
    runner.initialize()

    # Get initial parameters
    state = runner.load_or_init(dummy_batch, dummy_embeddings)
    self.params = state.params
```

Then it defines the ranking function:

```python
def hk_rank_candidates(batch, recsys_embeddings) -> RankingOutput:
    output = hk_forward(batch, recsys_embeddings)
    logits = output.logits
    probs = jax.nn.sigmoid(logits)
    primary_scores = probs[:, :, 0]  # <-- CRITICAL LINE
    ranked_indices = jnp.argsort(-primary_scores, axis=-1)
    ...
```

**CRITICAL ISSUE (line 345):**
```python
primary_scores = probs[:, :, 0]
```

This line says: "Use only the first action (favorite_score) for ranking."

**This means 18 of 19 predictions are computed but never used!**

The model predicts reply probability, block probability, etc., but the ranking only considers likes. This is a major missed opportunity identified in the audit.

**Lines 373-374:**
```python
rank_ = hk.without_apply_rng(hk.transform(hk_rank_candidates))
self.rank_candidates = rank_.apply
```

Creates the final ranking function that can be called with just parameters and data.

---

### Lines 388-487: Creating Example Data
```python
def create_example_batch(
    batch_size, emb_size, history_len, num_candidates, num_actions, ...
):
```

**What this does:**
Creates random example data for testing. This is helpful for:
- Unit tests
- Benchmarking
- Debugging

**Key parts:**

**Lines 421-431: Creating history with padding**
```python
history_post_hashes = rng.integers(1, num_post_embeddings, size=...)
for b in range(batch_size):
    valid_len = rng.integers(history_len // 2, history_len + 1)
    history_post_hashes[b, valid_len:, :] = 0  # Padding
```

This simulates users with different history lengths by zeroing out some positions.

**Lines 440-442: Action generation**
```python
history_actions = (rng.random(...) > 0.7).astype(np.float32)
```

Creates random binary actions - about 30% of actions are 1 (engaged).

---

### Lines 490-703: Retrieval Runner
```python
class RetrievalModelRunner(BaseModelRunner):
```

Similar to the ranking runner but for retrieval.

**Key method - `set_corpus` (lines 668-680):**
```python
def set_corpus(self, corpus_embeddings, corpus_post_ids):
    self.corpus_embeddings = corpus_embeddings
    self.corpus_post_ids = corpus_post_ids
```

This is how you provide the pre-computed post embeddings that retrieval will search through.

**Key method - `retrieve` (lines 682-703):**
```python
def retrieve(self, batch, recsys_embeddings, top_k=100, corpus_embeddings=None):
    return self.retrieve_fn(self.params, batch, recsys_embeddings, corpus_embeddings, top_k)
```

Returns the top-k most relevant posts for the given users.

---

### Lines 706-729: Creating Example Corpus
```python
def create_example_corpus(corpus_size, emb_size, seed=123):
```

**What this does:**
Creates a fake corpus of post embeddings for testing retrieval.

**Lines 723-725: Normalization**
```python
norms = np.linalg.norm(corpus_embeddings, axis=-1, keepdims=True)
corpus_embeddings = corpus_embeddings / np.maximum(norms, 1e-12)
```

Normalizes each embedding to unit length (required for dot-product similarity).

---

## Summary

This file is the "glue" that makes the models usable:

**ModelRunner**: Handles model initialization and parameter management
**RecsysInferenceRunner**: Provides the `rank()` function for scoring candidates
**RetrievalModelRunner**: Handles retrieval model setup
**RecsysRetrievalInferenceRunner**: Provides `encode_user()`, `encode_candidates()`, and `retrieve()` functions

**Critical Issue:**
The ranking function only uses `favorite_score` (index 0) despite computing all 19 action probabilities. The other 18 predictions are wasted.

**Proposed Fix:**
Implement weighted multi-action scoring:
```python
score = (
    1.0 * probs[:, :, 0] +   # favorite
    2.5 * probs[:, :, 1] +   # reply (high value)
    2.0 * probs[:, :, 2] +   # repost
    3.0 * probs[:, :, 11] +  # quote (highest effort)
    -3.0 * probs[:, :, 14] + # not_interested (penalty)
    -8.0 * probs[:, :, 15] + # block (big penalty)
    ...
)
```

This would dramatically improve feed quality by rewarding quality engagement and penalizing content that might be blocked or reported.
