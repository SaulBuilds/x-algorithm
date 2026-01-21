# Phoenix ML Code Audit

## 1. Model Efficiency Issues

### 1.1 Redundant Dtype Conversions

**Files**: `recsys_model.py:112-114, 174-176, 236-238`
```python
user_embedding = jnp.dot(user_embedding.astype(proj_mat_1.dtype), proj_mat_1).astype(
    user_embeddings.dtype
)
```

**Impact**: 3-4 extra memory allocations per block (bfloat16 → float32 → bfloat16).

### 1.2 Missing JIT Compilation

**File**: `runners.py:152, 373-374`
```python
self.forward = self.make_forward_fn()  # Not JIT-compiled
```

**Fix**:
```python
self.forward = jax.jit(self.make_forward_fn(), static_argnums=(0,))
```

### 1.3 Inefficient Embedding Lookups

**File**: `recsys_model.py:349-350`
```python
input_one_hot = jax.nn.one_hot(input, vocab_size)
output = jnp.dot(input_one_hot, embedding_table)
```

**Problem**: One-hot creates dense [B, S, vocab_size] tensor for categorical indices.
**Fix**: Use `jnp.take(embedding_table, indices, axis=0)` directly.

### 1.4 Attention Logit Clamping Per-Head

**File**: `grok.py:342-343`
```python
max_attn_val = jnp.array(30.0, dtype=attn_logits.dtype)  # Created every call
attn_logits = max_attn_val * jnp.tanh(attn_logits / max_attn_val)
```

**Fix**: Pre-compute `max_attn_val` as module constant.

### 1.5 No Gradient Checkpointing

**File**: `grok.py:574-582`
```python
for i in range(self.num_layers):
    decoder_output = block(...)
    h = decoder_output.embeddings  # All activations stored
```

**Impact**: Memory scales with depth. Add `jax.checkpoint()` for training.

---

## 2. CRITICAL: Zero Initialization Bug

### 2.1 Linear Layers

**File**: `grok.py:148-149`
```python
w = hk.get_parameter(
    "w", [input_size, output_size], jnp.float32, init=hk.initializers.Constant(0)
)
```

**Impact**: All linear layers output zeros. Model cannot learn.

### 2.2 RMSNorm Scale

**File**: `grok.py:176-180`
```python
scale = hk.get_parameter(
    "scale",
    param_shape,
    dtype=jnp.float32,
    init=hk.initializers.Constant(0),  # CRITICAL: Scale = 0
)
```

**Impact**: All normalized outputs are zeros.

**Fix for both**:
```python
init=hk.initializers.VarianceScaling(1.0, "fan_avg", "truncated_normal")
# or for scale:
init=hk.initializers.Constant(1.0)
```

---

## 3. CRITICAL: Unused Engagement Signals

### 3.1 Current State

**File**: `runners.py:202-221, 336-371`

The model predicts 19 engagement signals:
```python
ACTIONS = [
    "favorite_score",          # Index 0 - USED FOR RANKING
    "reply_score",             # Index 1 - NOT USED
    "repost_score",            # Index 2 - NOT USED
    "photo_expand_score",      # Index 3 - NOT USED
    "click_score",             # Index 4 - NOT USED
    "profile_click_score",     # Index 5 - NOT USED
    "vqv_score",               # Index 6 - NOT USED
    "share_score",             # Index 7 - NOT USED
    "share_via_dm_score",      # Index 8 - NOT USED
    "share_via_copy_link_score", # Index 9 - NOT USED
    "dwell_score",             # Index 10 - NOT USED
    "quote_score",             # Index 11 - NOT USED
    "quoted_click_score",      # Index 12 - NOT USED
    "follow_author_score",     # Index 13 - NOT USED
    "not_interested_score",    # Index 14 - NOT USED (negative)
    "block_author_score",      # Index 15 - NOT USED (negative)
    "mute_author_score",       # Index 16 - NOT USED (negative)
    "report_score",            # Index 17 - NOT USED (negative)
    "dwell_time",              # Index 18 - NOT USED (continuous)
]
```

**But ranking only uses**:
```python
primary_scores = probs[:, :, 0]  # Only favorite_score!
```

### 3.2 Proposed Multi-Action Scoring

```python
# Positive engagement weights
POSITIVE_WEIGHTS = {
    "favorite_score": 1.0,
    "reply_score": 2.0,        # Quality signal
    "repost_score": 1.5,
    "quote_score": 2.5,        # Highest effort engagement
    "follow_author_score": 3.0, # Strong signal
    "dwell_score": 0.5,
    "vqv_score": 1.0,          # Video completion
}

# Negative engagement weights
NEGATIVE_WEIGHTS = {
    "not_interested_score": -2.0,
    "block_author_score": -5.0,
    "mute_author_score": -3.0,
    "report_score": -10.0,
}

def weighted_score(probs):
    score = 0.0
    for action, weight in {**POSITIVE_WEIGHTS, **NEGATIVE_WEIGHTS}.items():
        idx = ACTIONS.index(action)
        score += weight * probs[:, :, idx]
    return score
```

---

## 4. Missing Quality Signals

### 4.1 Current Input Features

**File**: `recsys_model.py:62-77`
```python
class RecsysBatch(NamedTuple):
    user_hashes: ...              # User identity
    history_post_hashes: ...      # Posts viewed
    history_author_hashes: ...    # Authors viewed
    history_actions: ...          # Action vectors
    history_product_surface: ...  # Where seen
    candidate_post_hashes: ...    # Candidates
    candidate_author_hashes: ...  # Candidate authors
    candidate_product_surface: ... # Where to show
```

### 4.2 Missing Features (Critical for Quality)

```python
# SHOULD ADD:
class EnhancedRecsysBatch(NamedTuple):
    # Existing...

    # Temporal (for bot detection)
    history_timestamps: jax.typing.ArrayLike      # [B, S]
    user_created_at: jax.typing.ArrayLike         # [B]
    interaction_intervals: jax.typing.ArrayLike   # [B, S]

    # Content Quality
    content_toxicity_scores: jax.typing.ArrayLike # [B, C]
    content_language_ids: jax.typing.ArrayLike    # [B, C]
    has_media: jax.typing.ArrayLike               # [B, C]

    # Author Quality
    author_verified: jax.typing.ArrayLike         # [B, C]
    author_follower_count_log: jax.typing.ArrayLike # [B, C]
    author_account_age_days: jax.typing.ArrayLike # [B, C]
```

---

## 5. Security Issues

### 5.1 No Input Bounds Validation

**File**: `runners.py:421-423`
```python
user_hashes = rng.integers(1, num_user_embeddings, size=...)
```

No validation that hashes are within embedding table bounds.

**Attack**: Submit `hash = embedding_table_size + 1` → JAX wraps around silently.

### 5.2 Action Vector Injection

**File**: `recsys_model.py:314-321`
```python
actions_signed = (2 * actions - 1).astype(jnp.float32)
```

Assumes `actions ∈ [0, 1]`. No validation.

**Attack**: Submit `actions = [100, -100]` → extreme embeddings.

### 5.3 Product Surface Index OOB

**File**: `recsys_model.py:384-395`

No bounds check on `product_surface` indices against `vocab_size=16`.

---

## 6. Numerical Stability

### 6.1 RMS Norm Division Edge Case

**File**: `grok.py:190`
```python
normed_inputs = inputs * jax.lax.rsqrt(mean_squared + self.eps)
```

With all-zero input: `rsqrt(1e-5) = 316` → massive scaling.

### 6.2 Softmax Masking

**File**: `grok.py:353-354`
```python
attn_logits = jnp.where(mask, attn_logits, -1e30)
attn_weights = jax.nn.softmax(attn_logits)
```

Using `-1e30` can interact badly with other extreme values → NaN.

### 6.3 Integer Overflow Risk

**File**: `runners.py:421`
```python
.astype(np.int32)
```

If `num_user_embeddings > 2.1B`, int32 wraps.

---

## 7. Summary Table

| Category | File | Lines | Severity |
|----------|------|-------|----------|
| Zero Init (Linear) | grok.py | 148-149 | CRITICAL |
| Zero Init (RMSNorm) | grok.py | 176-180 | CRITICAL |
| Unused Signals | runners.py | 336-371 | CRITICAL |
| Missing JIT | runners.py | 152, 373 | HIGH |
| Hash OOB | recsys_model.py | 349 | HIGH |
| Action Validation | recsys_model.py | 314-321 | HIGH |
| Dtype Conversion | recsys_model.py | 112-114 | MEDIUM |
| One-hot Inefficiency | recsys_model.py | 349-350 | MEDIUM |
| Attention Clamping | grok.py | 342-343 | MEDIUM |
| No Checkpointing | grok.py | 574-582 | MEDIUM |
| Softmax Masking | grok.py | 353 | MEDIUM |
