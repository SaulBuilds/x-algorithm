# Feature Specifications

## Feature 1: Interactive Algorithm Walkthrough

### Description
A step-by-step animated journey through how a post travels from creation to your feed.

### User Story
As a user, I want to see exactly what happens when I open X, so I understand why I see certain posts.

### Components
- `StepWalkthrough` - Main container
- `FlowDiagram` - Visual pipeline
- `StepCard` - Individual step details
- `AnimatedArrow` - Connections between steps

### Steps

**Step 1: The Request**
- Visual: Phone icon sending request
- Description: "When you open X, your phone sends a request to X's servers"
- Animation: Request packet traveling to server

**Step 2: Who Are You?**
- Visual: User profile card
- Description: "The system looks up your profile and recent activity"
- Animation: User data loading

**Step 3: Retrieval**
- Visual: Ocean of posts with spotlight
- Description: "From 100 million posts, find ~1,000 that might interest you"
- Animation: Posts being filtered down

**Step 4: Ranking**
- Visual: Posts being scored
- Description: "The AI predicts how likely you are to engage with each post"
- Animation: Scores appearing on posts

**Step 5: Filtering**
- Visual: Posts being removed
- Description: "Remove posts you've seen, from blocked accounts, etc."
- Animation: Posts fading out

**Step 6: Selection**
- Visual: Top posts highlighted
- Description: "Pick the best 10-50 posts to show you"
- Animation: Posts rising to top

**Step 7: Your Feed**
- Visual: Phone showing feed
- Description: "The posts arrive on your phone in about 200ms"
- Animation: Feed appearing

### Interaction
- Click any step to see detailed explanation
- Hover to pause animation
- Toggle "technical mode" for code snippets

---

## Feature 2: Code Explorer

### Description
Browse the entire codebase with inline explanations, syntax highlighting, and visual aids.

### User Story
As a user, I want to read the actual code with explanations, so I can verify what I'm learning.

### Components
- `FileTree` - Navigate files
- `CodeViewer` - Display code
- `LineExplainer` - Per-line explanations
- `SearchBar` - Find code
- `Breadcrumbs` - Current location

### Features

**File Navigation**
- Tree view of all files
- Icons for Python/Rust
- Collapse/expand folders
- Search by filename

**Code Display**
- Syntax highlighting (Shiki)
- Line numbers
- Copy button
- Link to GitHub

**Explanations**
- Hover line for tooltip
- Click line for detailed explanation
- Highlight related lines
- Link to relevant concepts

**Search**
- Full-text search
- Filter by language
- Filter by module
- Show context around matches

### URL Structure
```
/explore                     # Explorer home
/explore/phoenix             # Phoenix module
/explore/phoenix/grok.py     # Specific file
/explore/search?q=attention  # Search results
```

---

## Feature 3: Vulnerability Dashboard

### Description
Interactive display of all audit findings with severity, impact, and fixes.

### User Story
As a user, I want to see what's wrong with the algorithm, so I understand its limitations.

### Components
- `AuditSummary` - Overview statistics
- `VulnerabilityList` - Filterable list
- `VulnerabilityCard` - Individual finding
- `ImpactChart` - Visual severity
- `FixProposal` - Suggested fix

### Categories

**Critical (Red)**
- Zero-initialized weights
- Empty Kafka topics
- Unused engagement signals

**High (Orange)**
- No input validation
- Missing retry logic
- Resource exhaustion risks

**Medium (Yellow)**
- Performance issues
- Lock contention
- Missing caching

**Low (Blue)**
- Code quality
- Documentation gaps

### Filters
- By severity
- By category (Security, Performance, ML)
- By file/module
- By status (Open, In Progress, Fixed)

### Details View
- Problem description (plain language)
- Technical explanation (for developers)
- Code snippet showing issue
- Proposed fix with code
- Impact on users

---

## Feature 4: Feed Simulator

### Description
After connecting X account, simulate how the algorithm would rank sample posts.

### User Story
As a user, I want to see how the algorithm would rank posts for me specifically.

### Components
- `SimulatorSetup` - Configure simulation
- `PostCard` - Display sample posts
- `ScoreBreakdown` - Show scoring details
- `RankingVisualizer` - Animated ranking
- `CompareView` - Before/after changes

### Flow

**1. Authentication**
- Connect X account via Privy
- Request minimal permissions (profile only)
- Show what data will be used

**2. Profile Analysis**
- Display user's public profile
- Show recent engagement patterns (mocked)
- Explain how this affects recommendations

**3. Sample Posts**
- Present 8 sample posts (curated or random)
- Each post has different characteristics:
  - From followed account
  - From unfollowed account
  - High engagement
  - Low engagement
  - With media
  - Text only
  - Recent
  - Older

**4. Simulation**
- Animated ranking process
- Show each scoring step
- Display final rankings

**5. Score Breakdown**
- For each post, show:
  - P(like): 0.72
  - P(reply): 0.23
  - P(repost): 0.45
  - P(block): 0.01
  - etc.
- Highlight which signals dominated

**6. What-If**
- Toggle: "What if we used all 19 signals?"
- Toggle: "What if we penalized high-frequency posters?"
- Show how rankings would change

### Privacy
- No data stored beyond session
- User can see exactly what's accessed
- Clear explanation of simulation vs. real algorithm

---

## Feature 5: Concept Library

### Description
Glossary of technical terms with interactive examples.

### User Story
As a user, I want to look up unfamiliar terms, so I can understand the explanations.

### Components
- `ConceptSearch` - Find concepts
- `ConceptCard` - Display definition
- `InteractiveExample` - Try it yourself
- `RelatedConcepts` - Learn more

### Concepts to Include

**Core AI Concepts**
- Neural Network
- Transformer
- Attention Mechanism
- Embeddings
- Training vs. Inference

**Algorithm-Specific**
- Hash Embeddings
- Two-Tower Architecture
- Candidate Isolation
- Multi-Action Prediction

**Technical Terms**
- RMS Normalization
- Softmax
- Dot Product Similarity
- Top-K Selection

**System Concepts**
- gRPC
- Kafka
- Caching
- Rate Limiting

### Interactive Examples

**Embeddings Demo**
- Input: Type a word
- Output: Show similar words based on embedding distance

**Attention Demo**
- Input: A sentence
- Output: Visualize what attends to what

**Ranking Demo**
- Input: 3 posts
- Output: Animated ranking with scores

---

## Feature 6: Progress Tracking

### Description
Track learning progress through the content.

### User Story
As a user, I want to track what I've learned, so I can continue where I left off.

### Components
- `ProgressBar` - Overall completion
- `SectionChecklist` - Per-section progress
- `AchievementBadge` - Milestones
- `ResumePrompt` - Continue learning

### Sections Tracked
- [ ] Algorithm Overview
- [ ] Retrieval Deep-Dive
- [ ] Ranking Deep-Dive
- [ ] Filtering Explained
- [ ] Vulnerability Overview
- [ ] Security Issues
- [ ] Performance Issues
- [ ] Code Exploration (per module)
- [ ] Feed Simulation

### Achievements
- "First Steps" - Complete overview
- "Deep Diver" - Complete all deep-dives
- "Bug Hunter" - View all vulnerabilities
- "Code Reader" - Explore 10+ files
- "Experimenter" - Run feed simulation

### Storage
- Local storage for anonymous users
- Database for authenticated users
- Sync across devices when logged in
