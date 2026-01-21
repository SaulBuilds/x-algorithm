# Backlog

Items for future sprints, prioritized by impact.

---

## Sprint 3: Quality Signals (Planned)

### Quality Tower Integration
- Add separate quality assessment tower to Phoenix model
- Integrate toxicity classifier
- Add informativeness scoring
- Author reputation features

### Content Analysis
- Language detection and matching
- URL domain quality scoring
- Media quality assessment
- Conversation context quality

---

## Sprint 4: Advanced Features (Planned)

### Hierarchical Caching
- Request-scoped cache
- Session cache (Redis, 5-min TTL)
- User cache (Redis, 1-hour TTL)

### Parallel Filter Execution
- Group independent filters
- Run groups with `join_all()`
- Expected 30-40% latency reduction

### Event Tracking
- Structured pipeline events
- Per-stage latency metrics
- Filter exclusion reason tracking
- Distributed tracing support

---

## Future Ideas

### Diversity Mechanisms
- Topic diversity in feed
- Source diversity (different author types)
- Format diversity (text, image, video)
- Time diversity (not all recent posts)

### Personalization Improvements
- Interest decay modeling
- Context-aware ranking (time of day, device)
- Mood detection from recent engagement
- Social graph influence

### Content Recycling Detection
- Near-duplicate detection
- Quote chain identification
- Link spam patterns
- Hashtag abuse detection

### Advanced Bot Detection
- Network analysis (coordinated behavior)
- Language pattern analysis
- Engagement timing networks
- Cross-account correlation

### Explainability
- Feature importance logging
- "Why am I seeing this?" API
- Ranking decision audit trail
- Bias detection metrics

---

## Technical Debt

### P2 Priority
- [ ] Fix unnecessary Vec clones in pipeline
- [ ] Add retry logic to all external calls
- [ ] Implement circuit breakers
- [ ] Pre-allocate vectors with capacity hints

### P3 Priority
- [ ] Replace one-hot with `jnp.take()`
- [ ] Add gradient checkpointing
- [ ] Pre-compute attention constants
- [ ] Optimize dtype conversions

### Documentation
- [ ] API documentation
- [ ] Architecture diagrams
- [ ] Runbook for common issues
- [ ] Performance tuning guide

---

## Research Topics

### ML Improvements
- Multi-task learning with task-specific heads
- Contrastive learning for embeddings
- Online learning for weight updates
- Reinforcement learning from user feedback

### System Architecture
- Serverless scoring functions
- Edge caching strategies
- Real-time feature computation
- Stream processing for signals

### User Experience
- Feed diversity optimization
- Serendipity in recommendations
- Filter bubble mitigation
- Healthy engagement patterns
