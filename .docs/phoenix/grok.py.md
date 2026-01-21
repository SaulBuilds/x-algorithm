# Understanding grok.py

**File Location**: `phoenix/grok.py`
**Purpose**: This file contains the core "brain" of the recommendation system - a Transformer neural network that processes information to understand what posts you might like.

---

## What This File Does

This file implements a **Transformer** - the same type of artificial intelligence architecture that powers ChatGPT and other modern AI systems. In the X algorithm, this Transformer looks at:
1. Who you are (your user profile)
2. What posts you've engaged with before (your history)
3. Posts that might be shown to you (candidates)

It then predicts how likely you are to engage with each candidate post.

---

## Line-by-Line Explanation

### Lines 1-13: License Header
```python
# Copyright 2026 X.AI Corp.
# Licensed under the Apache License, Version 2.0 (the "License");
```
This is legal text stating who owns the code (X.AI Corp) and how others can use it (Apache 2.0 license, which is a permissive open-source license).

---

### Lines 15-22: Importing Tools
```python
import logging
from dataclasses import dataclass
from typing import NamedTuple, Optional, Sequence, Union

import haiku as hk
import jax
import jax.numpy as jnp
```

**What this means:**

- `logging` - A tool for recording messages about what the program is doing (for debugging)
- `dataclass` - A Python feature that makes it easy to create simple classes that hold data
- `typing` - Tools for describing what types of data variables should contain
- `haiku` (imported as `hk`) - A library from DeepMind (Google's AI research lab) for building neural networks
- `jax` - A library from Google for doing math really fast on GPUs (specialized computer chips for AI)
- `jax.numpy` (imported as `jnp`) - JAX's version of NumPy, a library for working with arrays of numbers

---

### Lines 23-24: Creating a Logger
```python
logger = logging.getLogger(__name__)
```

This creates a "logger" that can record messages. `__name__` is a special Python variable that contains the name of this file ("grok").

---

### Lines 26-30: TrainingState
```python
class TrainingState(NamedTuple):
    """Container for the training state."""
    params: hk.Params
```

**What this is:**
A simple container that holds the model's "parameters" (also called weights). Parameters are numbers that the model learns during training. Think of them as the model's "knowledge" - they determine how the model makes decisions.

---

### Lines 32-36: FFN Size Calculation
```python
def ffn_size(emb_size, widening_factor):
    _ffn_size = int(widening_factor * emb_size) * 2 // 3
    _ffn_size = _ffn_size + (8 - _ffn_size) % 8
    logger.debug(f"emd_size: {emb_size} adjusted ffn_size: {_ffn_size}")
    return _ffn_size
```

**What this does:**
Calculates the size of a "feed-forward network" layer inside the Transformer.

**Line by line:**
1. `widening_factor * emb_size` - Makes the layer wider than the input. For example, if `emb_size=128` and `widening_factor=4`, this gives 512.
2. `* 2 // 3` - Takes two-thirds of that value (a design choice for efficiency)
3. `+ (8 - _ffn_size) % 8` - Rounds up to the nearest multiple of 8 (GPUs work faster with multiples of 8)
4. Logs the result for debugging
5. Returns the calculated size

---

### Lines 39-71: Recommendation System Attention Mask
```python
def make_recsys_attn_mask(
    seq_len: int,
    candidate_start_offset: int,
    dtype: jnp.dtype = jnp.float32,
) -> jax.Array:
```

**What this is:**
This function creates a special "attention mask" - a grid of 1s and 0s that tells the Transformer which parts of the input can "look at" which other parts.

**Why it matters:**
This is one of the most important parts of the algorithm. It ensures that when ranking posts:
- Each candidate post can look at your user profile and history
- Each candidate post can look at itself
- **Candidate posts CANNOT look at each other**

This last point is critical because it means each post is scored independently. If candidate A could see candidate B, the score for A might change based on what B is - that would make the system unpredictable.

**Line by line:**

```python
causal_mask = jnp.tril(jnp.ones((1, 1, seq_len, seq_len), dtype=dtype))
```
Creates a lower-triangular matrix of 1s. This is a "causal mask" - position 5 can see positions 1-5 but not 6+.

```python
attn_mask = causal_mask.at[:, :, candidate_start_offset:, candidate_start_offset:].set(0)
```
Sets the bottom-right corner (where candidates would look at other candidates) to 0.

```python
candidate_indices = jnp.arange(candidate_start_offset, seq_len)
attn_mask = attn_mask.at[:, :, candidate_indices, candidate_indices].set(1)
```
Adds back 1s on the diagonal so candidates can still "see themselves".

---

### Lines 74-86: Output Container Classes
```python
class MHAOutput(NamedTuple):
    embeddings: jax.Array

class DecoderOutput(NamedTuple):
    embeddings: jax.Array

class TransformerOutput(NamedTuple):
    embeddings: jax.Array
```

These are simple containers that hold the output of different parts of the Transformer. They all contain "embeddings" - arrays of numbers that represent information in a form the AI can work with.

---

### Lines 88-109: Transformer Configuration
```python
@dataclass
class TransformerConfig:
    emb_size: int
    key_size: int
    num_q_heads: int
    num_kv_heads: int
    num_layers: int
    widening_factor: float = 4.0
    attn_output_multiplier: float = 1.0
    name: Optional[str] = None
```

**What each setting means:**

- `emb_size` - The dimension of embeddings. If this is 128, then every post, user, or word is represented as 128 numbers.
- `key_size` - Size of the "key" vectors in attention (explained below)
- `num_q_heads` - Number of "query" attention heads. Multiple heads let the model look at different aspects simultaneously.
- `num_kv_heads` - Number of "key/value" heads. Can be smaller than query heads for efficiency.
- `num_layers` - How many Transformer layers to stack. More layers = more processing power but slower.
- `widening_factor` - How much to expand the feed-forward layers (default 4x)
- `attn_output_multiplier` - A scaling factor for attention (default 1.0)

---

### Lines 112-118: RMS Normalization Helper
```python
def hk_rms_norm(x: jax.Array, fixed_scale=False) -> jax.Array:
    """Applies a unique LayerNorm to x with default settings."""
    ln = RMSNorm(axis=-1, create_scale=not fixed_scale)
    return ln(x)
```

**What normalization does:**
Neural networks can produce numbers that are very large or very small. Normalization keeps them in a reasonable range. "RMS" stands for "Root Mean Square" - it divides by the square root of the average squared value.

---

### Lines 121-159: Linear Layer
```python
class Linear(hk.Linear):
```

**What a Linear layer is:**
The most basic building block of neural networks. It takes input numbers, multiplies each by a "weight", and adds them up.

Mathematically: `output = input × weights + bias`

**Critical Issue Found in Audit (Line 148):**
```python
w = hk.get_parameter("w", [input_size, output_size], jnp.float32, init=hk.initializers.Constant(0))
```
The weights are initialized to 0. This is a bug - if all weights start at 0, the layer outputs 0 and cannot learn. This should be random initialization.

---

### Lines 162-194: RMS Normalization
```python
class RMSNorm(hk.RMSNorm):
```

**What this does:**
Normalizes the input by dividing by the "root mean square" of the values.

**Formula:** `output = (input / sqrt(mean(input²) + epsilon)) × scale`

The `epsilon` (1e-5) prevents division by zero. The `scale` is a learned parameter.

**Critical Issue Found in Audit (Line 180):**
```python
init=hk.initializers.Constant(0)
```
Scale is initialized to 0. This means the output will always be 0. Should be initialized to 1.

---

### Lines 197-261: Rotary Position Embedding (RoPE)
```python
class RotaryEmbedding(hk.Module):
```

**What position embeddings do:**
In a sequence of words or posts, position matters. "The dog bit the man" means something different from "The man bit the dog." Position embeddings tell the model where each item is in the sequence.

**What "rotary" means:**
RoPE encodes position by rotating vectors in pairs of dimensions. It's a modern technique (from 2021) that works better than older approaches for long sequences.

**How it works:**
1. Creates frequency values that decrease with dimension
2. Multiplies by position to get phase angles
3. Applies rotation using sine and cosine

The `rotate_half` function (lines 197-202) takes the input, splits it in half, negates one half, and swaps them - this is part of the rotation operation.

---

### Lines 264-376: Multi-Head Attention
```python
class MultiHeadAttention(hk.Module):
```

**This is the core of the Transformer.** Attention is how the model decides what to "pay attention to" when processing each position.

**Key concepts:**

1. **Query, Key, Value**: Every position gets three representations:
   - Query: "What am I looking for?"
   - Key: "What do I contain?"
   - Value: "What information can I provide?"

2. **How attention works:**
   - Each query looks at all keys and computes a similarity score
   - These scores become weights (using softmax to sum to 1)
   - The output is a weighted sum of values

3. **Multi-head**: Instead of one attention operation, we do several in parallel ("heads"), each looking at different aspects. Then we combine them.

**Line 338-343: Attention calculation**
```python
attn_logits = jnp.einsum("...thHd,...Thd->...hHtT", query_heads, key_heads).astype(jnp.float32)
attn_logits *= self.attn_output_multiplier
max_attn_val = jnp.array(30.0, dtype=attn_logits.dtype)
attn_logits = max_attn_val * jnp.tanh(attn_logits / max_attn_val)
```

- `einsum` is a compact way to write matrix multiplication
- The result is clamped using tanh to prevent extreme values (numerical stability)

**Line 353-354: Applying the mask and softmax**
```python
attn_logits = jnp.where(mask, attn_logits, -1e30)
attn_weights = jax.nn.softmax(attn_logits)
```
Where the mask is 0, we set the attention to a huge negative number (-1e30). After softmax, this becomes essentially 0, meaning "don't attend to this position."

---

### Lines 378-411: MHA Block
```python
class MHABlock(hk.Module):
```

A wrapper around MultiHeadAttention that provides a cleaner interface. The `@hk.transparent` decorator means this module doesn't create its own namespace for parameters.

---

### Lines 414-440: Dense Block (Feed-Forward Network)
```python
class DenseBlock(hk.Module):
```

**What this does:**
After attention, each position goes through a "feed-forward network" - two linear layers with an activation function in between.

**Structure:**
1. Expand: 128 → 341 (using calculated ffn_size)
2. Apply GELU activation (smooth version of ReLU)
3. Compress: 341 → 128

The `h_v` and `h_w1` pattern is called "Gated Linear Units" (GLU) - an advanced technique that multiplies two parallel pathways together.

---

### Lines 443-497: Decoder Layer
```python
class DecoderLayer(hk.Module):
```

**What this is:**
One complete layer of the Transformer. The full Transformer stacks multiple of these.

**What happens in each layer:**
1. Layer normalize the input
2. Apply attention
3. Normalize the attention output
4. Add to the original input (residual connection)
5. Normalize again
6. Apply dense block (feed-forward network)
7. Normalize the output
8. Add to the input again

**Why residual connections (the `h += h_attn` parts)?**
They help information flow through deep networks and make training more stable.

---

### Lines 504-586: The Full Transformer
```python
class Transformer(hk.Module):
```

**What this does:**
Puts everything together. Takes embeddings (numerical representations), passes them through multiple decoder layers, and outputs transformed embeddings.

**Key section (lines 541-551):**
```python
if candidate_start_offset is not None:
    attn_mask = make_recsys_attn_mask(seq_len, candidate_start_offset, fprop_dtype)
    mask = mask * attn_mask
else:
    causal_mask = jnp.tril(jnp.ones((1, 1, seq_len, seq_len)))
    mask = mask * causal_mask
```

This is where the special recommendation system attention mask (that prevents candidates from seeing each other) gets applied.

**The main loop (lines 574-582):**
```python
for i in range(self.num_layers):
    decoder_output = block(h, mask, padding_mask, layer_index=i, name=f"decoder_layer_{i}")
    h = decoder_output.embeddings
```

This applies the decoder layer multiple times. Each iteration refines the embeddings further.

---

## Summary

This file implements the Transformer neural network that powers X's recommendation system. The key innovation is the special attention mask that ensures each candidate post is scored independently based on your profile and history, without being influenced by what other candidate posts are in the batch.

**Critical Issues:**
1. Linear layer weights initialized to 0 (line 148)
2. RMSNorm scale initialized to 0 (line 180)

Both of these bugs would cause the model to output zeros and fail to learn. These must be fixed for the system to work.
