# Security & Compliance Audit

## 1. Critical Security Vulnerabilities

### 1.1 Input Validation Gaps (Rust)

**File**: `home-mixer/server.rs:33-35`

**Current validation**:
```rust
// Only checks viewer_id != 0
```

**Missing validations**:
- No upper bounds on `seen_ids` and `served_ids` (DoS via 1M items)
- No validation of `bloom_filter_entries` structure integrity
- No rate limiting per user

**Recommendation**:
```rust
const MAX_SEEN_IDS: usize = 10_000;
const MAX_SERVED_IDS: usize = 1_000;

if query.seen_ids.len() > MAX_SEEN_IDS {
    return Err(Status::invalid_argument("seen_ids exceeds limit"));
}
```

### 1.2 Input Validation Gaps (Python)

**File**: `phoenix/recsys_model.py`

No validation on:
- Hash values (could be negative or exceed table size)
- Action vectors (assumed binary but no check)
- Product surface indices

**Attack vectors**:
1. **Model Poisoning**: Submit extreme hash values to retrieve wrong embeddings
2. **Gradient Explosion**: Submit action vectors > 1.0 to blow up embeddings
3. **Index OOB**: Submit surface indices > vocab_size

---

## 2. Resource Exhaustion Vulnerabilities

### 2.1 Unbounded PostStore Growth

**File**: `thunder/posts/post_store.rs:41-48`
```rust
original_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
```

No per-user limits on timeline size. A single prolific author can consume unlimited memory.

**Recommendation**:
```rust
const MAX_POSTS_PER_USER: usize = 10_000;

if user_posts.len() >= MAX_POSTS_PER_USER {
    user_posts.pop_front();  // Remove oldest
}
```

### 2.2 Silent Input Truncation

**File**: `thunder/thunder_service.rs:250-272`
```rust
let following_user_ids: Vec<u64> = following_user_ids
    .into_iter()
    .take(MAX_INPUT_LIST_SIZE)
    .collect();
```

Input lists truncated silently without client notification.

**Risk**: Client receives partial results without knowing.

### 2.3 No Timeouts on Filter Operations

**File**: `candidate-pipeline/candidate_pipeline.rs:61`

Filter operations have no timeout. Malicious/buggy filter could hang indefinitely.

---

## 3. Injection Risks

### 3.1 Muted Keyword Processing

**File**: `home-mixer/filters/muted_keyword_filter.rs:38-40`
```rust
let tokenized = muted_keywords.iter().map(|k| self.tokenizer.tokenize(k));
```

User muted keywords tokenized without sanitization.

**Risks**:
- ReDoS (Regex Denial of Service) if tokenizer uses regex
- No limits on keyword length or count

**Recommendation**:
```rust
const MAX_KEYWORD_LENGTH: usize = 100;
const MAX_KEYWORDS: usize = 1000;

muted_keywords.iter()
    .take(MAX_KEYWORDS)
    .filter(|k| k.len() <= MAX_KEYWORD_LENGTH)
    .map(|k| self.tokenizer.tokenize(k))
```

### 3.2 Unvalidated String Inputs

**File**: `home-mixer/candidate_pipeline/query.rs:24-50`

`country_code` and `language_code` from client have no format validation. Could be unbounded length strings.

---

## 4. Error Recovery Issues

### 4.1 Panics on Recoverable Errors

| File | Line | Impact |
|------|------|--------|
| `thunder/kafka/tweet_events_listener.rs` | 101-104 | Kafka failure kills thread |
| `thunder/kafka/tweet_events_listener_v2.rs` | 101-105 | Processing thread exit kills feeder |
| `thunder/posts/post_store.rs` | 475 | Task pool exhaustion panics |
| `thunder/kafka/tweet_events_listener.rs` | 210 | Malformed message panics consumer |

**Risk**: Single component failure cascades to full service outage.

### 4.2 Missing Circuit Breakers

**Files**: `home-mixer/sources/phoenix_source.rs`, `thunder_source.rs`

No circuit breaker pattern. Downstream service degradation causes request pile-up.

---

## 5. Data Privacy Concerns

### 5.1 Logging Practices

Review needed for:
- User IDs in log messages
- Engagement history logging
- Request tracing data retention

### 5.2 Cache Side Effects

**File**: `candidate-pipeline/candidate_pipeline.rs:321`

Side effects write to Strato cache without explicit TTL or deletion policy documentation.

---

## 6. Compliance Considerations

### 6.1 GDPR/CCPA

| Requirement | Status | Notes |
|-------------|--------|-------|
| Right to deletion | UNKNOWN | No documented deletion flow for user data in PostStore |
| Data minimization | PARTIAL | History length capped but no clear justification |
| Consent tracking | NOT FOUND | No consent signals in model inputs |

### 6.2 Algorithmic Transparency

| Requirement | Status | Notes |
|-------------|--------|-------|
| Explainability | POOR | Black-box transformer, no feature importance |
| Audit trail | PARTIAL | Metrics exist but no per-decision logging |
| Bias testing | NOT FOUND | No fairness metrics in codebase |

---

## 7. Recommendations

### P0 - Critical (Fix Immediately)

1. **Add input bounds validation** on all API inputs
2. **Implement per-user resource limits** in PostStore
3. **Replace panics with graceful degradation**
4. **Add timeout wrappers** on all external calls

### P1 - High (Fix Before Production)

1. **Implement rate limiting** per user/IP
2. **Add circuit breakers** for downstream services
3. **Validate and sanitize** muted keywords
4. **Document data retention policies**

### P2 - Medium (Compliance)

1. **Implement user data deletion** flow
2. **Add consent signals** to model inputs
3. **Create explainability layer** for ranking decisions
4. **Establish bias testing** framework

---

## 8. Security Checklist

| Category | Check | Status |
|----------|-------|--------|
| Input Validation | All inputs have bounds checks | FAIL |
| Input Validation | String inputs have length limits | FAIL |
| Resource Limits | Per-user memory limits | FAIL |
| Resource Limits | Request timeout on all operations | FAIL |
| Error Handling | No panics on user input | FAIL |
| Error Handling | Circuit breakers on external calls | FAIL |
| Injection | Keyword sanitization | FAIL |
| Injection | Hash value validation | FAIL |
| DoS Protection | Rate limiting | NOT FOUND |
| DoS Protection | Input size limits enforced | PARTIAL |
| Privacy | PII logging review | NEEDS REVIEW |
| Privacy | Data retention documented | NEEDS REVIEW |
