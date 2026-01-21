# Sprint Planning

This folder contains implementation plans for features and improvements identified in the [audit](./../.audit/AUDIT_SUMMARY.md).

## Sprint Overview

| Sprint | Focus | Status |
|--------|-------|--------|
| Sprint 0 | Critical Fixes | PLANNED |
| Sprint 1 | Multi-Action Scoring | PLANNED |
| Sprint 2 | Bot Detection | PLANNED |
| Sprint 3 | Quality Signals | PLANNED |
| Sprint 4 | Advanced Features | PLANNED |

## Documents

| Document | Description |
|----------|-------------|
| [SPRINT_0_CRITICAL_FIXES.md](./SPRINT_0_CRITICAL_FIXES.md) | Emergency fixes for production blockers |
| [SPRINT_1_SCORING.md](./SPRINT_1_SCORING.md) | Multi-action weighted scoring implementation |
| [SPRINT_2_BOT_DETECTION.md](./SPRINT_2_BOT_DETECTION.md) | Bot detection and spam mitigation |
| [SPRINT_3_QUALITY.md](./SPRINT_3_QUALITY.md) | Quality signal integration |
| [BACKLOG.md](./BACKLOG.md) | Future improvements and ideas |

## Priority Matrix

### P0 - Critical (Sprint 0)
These block production deployment:
1. Fix zero initialization in Phoenix model
2. Configure Kafka topic constants
3. Add input validation

### P1 - High (Sprint 1-2)
These significantly impact user experience:
1. Enable multi-action scoring (18 unused signals)
2. Implement frequency-based spam penalty
3. Add bot detection layer

### P2 - Medium (Sprint 3)
These improve quality and reliability:
1. Add quality tower to Phoenix
2. Implement hierarchical caching
3. Parallel filter execution

### P3 - Lower (Backlog)
These are optimizations:
1. Gradient checkpointing
2. JIT compilation improvements
3. Advanced content analysis
