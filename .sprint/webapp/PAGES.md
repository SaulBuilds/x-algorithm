# Page Specifications

## Page 1: Landing Page (`/`)

### Purpose
Introduce the project and invite users to explore.

### Layout
```
┌─────────────────────────────────────────────────────────┐
│                      [Header]                            │
│   Logo   |   Learn   Explore   Audit   Simulate   Login │
├─────────────────────────────────────────────────────────┤
│                                                         │
│              [Hero Section]                             │
│                                                         │
│        Understand How X's Algorithm                     │
│           Decides What You See                          │
│                                                         │
│     Explore the open-source recommendation              │
│     algorithm powering your For You feed                │
│                                                         │
│        [Start Learning]    [View Code]                  │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│              [Feature Cards]                            │
│                                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │  Learn  │  │ Explore │  │  Audit  │  │Simulate │    │
│  │         │  │  Code   │  │ Vulns   │  │  Feed   │    │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│              [Algorithm Preview]                        │
│                                                         │
│      [Animated pipeline diagram]                        │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│              [Stats Section]                            │
│                                                         │
│   7,700 lines    47 issues    19 engagement            │
│   of code        found        signals                   │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                      [Footer]                            │
│   GitHub   |   Audit Report   |   About                 │
└─────────────────────────────────────────────────────────┘
```

### Content
- Headline: "Understand How X's Algorithm Decides What You See"
- Subheadline: "Explore the open-source recommendation algorithm powering your For You feed"
- CTA buttons: "Start Learning", "View Code"
- Feature cards linking to main sections
- Animated pipeline preview
- Key statistics

---

## Page 2: Learning Hub (`/learn`)

### Purpose
Central hub for all educational content.

### Layout
```
┌─────────────────────────────────────────────────────────┐
│                      [Header]                            │
├───────────┬─────────────────────────────────────────────┤
│           │                                             │
│ [Sidebar] │           [Main Content]                    │
│           │                                             │
│ Overview  │    How the Algorithm Works                  │
│ Retrieval │                                             │
│ Ranking   │    [Progress: 25% complete]                 │
│ Filtering │                                             │
│ Concepts  │    ┌─────────────────────────────┐          │
│           │    │                             │          │
│           │    │   [Algorithm Walkthrough]   │          │
│           │    │                             │          │
│           │    │   Click to begin            │          │
│           │    │                             │          │
│           │    └─────────────────────────────┘          │
│           │                                             │
│           │    [Section Cards]                          │
│           │                                             │
│           │    ┌────────────┐  ┌────────────┐          │
│           │    │ Retrieval  │  │  Ranking   │          │
│           │    │ How posts  │  │ How posts  │          │
│           │    │ are found  │  │ are scored │          │
│           │    └────────────┘  └────────────┘          │
│           │                                             │
└───────────┴─────────────────────────────────────────────┘
```

### Sections
1. **Overview** - 5-minute introduction
2. **Retrieval** - How candidates are found
3. **Ranking** - How posts are scored
4. **Filtering** - What gets removed
5. **Concepts** - Glossary of terms

---

## Page 3: Algorithm Overview (`/learn/overview`)

### Purpose
High-level explanation of the full pipeline.

### Content
1. **The Request** - What happens when you open X
2. **Finding Candidates** - Retrieval explained simply
3. **Scoring Posts** - Ranking explained simply
4. **Removing Posts** - Filtering explained simply
5. **Your Feed** - How it all comes together

### Interactive Elements
- Animated pipeline diagram
- Click-through steps
- "Try it yourself" mini-simulation

---

## Page 4: Retrieval Deep-Dive (`/learn/retrieval`)

### Purpose
Detailed explanation of the retrieval system.

### Content
1. **The Problem** - Too many posts to consider
2. **Two Towers** - User tower and candidate tower
3. **Embeddings** - Converting to numbers
4. **Similarity** - Finding matches
5. **Top-K** - Selecting candidates

### Interactive Elements
- Embedding visualizer
- Similarity calculator
- Code snippets from `recsys_retrieval_model.py`

---

## Page 5: Ranking Deep-Dive (`/learn/ranking`)

### Purpose
Detailed explanation of the ranking model.

### Content
1. **The Input** - User, history, candidates
2. **The Transformer** - How attention works
3. **Candidate Isolation** - Why posts are scored independently
4. **19 Predictions** - All the engagement types
5. **The Problem** - Only 1/19 is used!

### Interactive Elements
- Attention heatmap
- Score breakdown visualizer
- Before/after multi-action scoring

---

## Page 6: Code Explorer (`/explore`)

### Purpose
Browse and understand the codebase.

### Layout
```
┌─────────────────────────────────────────────────────────┐
│                      [Header]                            │
├───────────┬─────────────────────────────────────────────┤
│           │                                             │
│ [File     │           [Code Viewer]                     │
│  Tree]    │                                             │
│           │    phoenix/grok.py                          │
│ phoenix/  │    ────────────────────                     │
│  ├ grok   │                                             │
│  ├ recsys │    1  │ # Transformer implementation       │
│  └ runner │    2  │ import jax                         │
│           │    3  │ import haiku as hk                 │
│ home-     │    4  │                                    │
│ mixer/    │    5  │ class Transformer:                 │
│  ├ filter │    6  │   """A transformer stack."""       │
│  └ scorer │    7  │                                    │
│           │    ─────────────────────────                │
│ thunder/  │                                             │
│           │    [Line Explanation Panel]                 │
│           │                                             │
│           │    Line 5: This defines the Transformer    │
│           │    class, which is the core AI model...    │
│           │                                             │
└───────────┴─────────────────────────────────────────────┘
```

### Features
- File tree navigation
- Syntax-highlighted code
- Line-by-line explanations
- Search across all files
- Link to documentation

---

## Page 7: Audit Dashboard (`/audit`)

### Purpose
Display all security and quality findings.

### Layout
```
┌─────────────────────────────────────────────────────────┐
│                      [Header]                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│              [Summary Stats]                            │
│                                                         │
│   ●7 Critical   ●19 High   ●22 Medium   ●6 Low         │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [Filters]                                              │
│  Severity: [All] [Critical] [High] [Medium] [Low]       │
│  Category: [All] [Security] [Performance] [ML]          │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [Vulnerability Cards]                                  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ ● CRITICAL: Zero-Initialized Weights            │   │
│  │                                                  │   │
│  │ The neural network weights start at zero,       │   │
│  │ meaning the model outputs zeros and cannot      │   │
│  │ learn anything.                                 │   │
│  │                                                  │   │
│  │ File: phoenix/grok.py:148                       │   │
│  │                                                  │   │
│  │ [View Code] [See Fix]                           │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ ● CRITICAL: 18 Unused Engagement Signals        │   │
│  │ ...                                             │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Features
- Summary statistics
- Severity filters
- Category filters
- Detailed vulnerability cards
- Links to code
- Proposed fixes

---

## Page 8: Feed Simulator (`/simulate`)

### Purpose
Let users see how the algorithm would rank posts for them.

### Layout (Unauthenticated)
```
┌─────────────────────────────────────────────────────────┐
│                      [Header]                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│              [Hero]                                     │
│                                                         │
│        See How the Algorithm Ranks Posts for You        │
│                                                         │
│        Connect your X account to simulate               │
│        personalized feed ranking                        │
│                                                         │
│              [Connect with X]                           │
│                                                         │
│              [Privacy Notice]                           │
│        We only access your public profile.              │
│        No tweets, followers, or DMs.                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Layout (Authenticated)
```
┌─────────────────────────────────────────────────────────┐
│                      [Header]                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [Profile Card]         [Simulation Area]               │
│                                                         │
│  ┌──────────┐           ┌───────────────────────────┐  │
│  │ [Avatar] │           │                           │  │
│  │ @user    │           │   [Sample Posts]          │  │
│  │ Name     │           │                           │  │
│  └──────────┘           │   ┌───────────────────┐   │  │
│                         │   │ Post 1            │   │  │
│  [Simulation            │   │ Score: 0.823      │   │  │
│   Controls]             │   │ [Score Breakdown] │   │  │
│                         │   └───────────────────┘   │  │
│  [x] Use all 19         │                           │  │
│      signals            │   ┌───────────────────┐   │  │
│                         │   │ Post 2            │   │  │
│  [x] Penalize           │   │ Score: 0.756      │   │  │
│      high-frequency     │   │ [Score Breakdown] │   │  │
│      posters            │   └───────────────────┘   │  │
│                         │                           │  │
│  [Run Simulation]       │   ...                     │  │
│                         │                           │  │
│                         └───────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Features
- X account connection
- Sample post selection
- Scoring visualization
- Score breakdown
- "What if" toggles
- Ranking animation

---

## Page 9: Concept Library (`/learn/concepts`)

### Purpose
Glossary of technical terms.

### Layout
```
┌─────────────────────────────────────────────────────────┐
│                      [Header]                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [Search]                                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Search concepts...                              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  [Categories]                                           │
│  [All] [AI Basics] [Algorithm] [Technical] [System]     │
│                                                         │
│  [Concept Cards]                                        │
│                                                         │
│  ┌─────────────────────┐  ┌─────────────────────┐      │
│  │ Embeddings          │  │ Transformer         │      │
│  │                     │  │                     │      │
│  │ Numbers that        │  │ A type of neural    │      │
│  │ represent things    │  │ network that's      │      │
│  │                     │  │ good at sequences   │      │
│  │ [Learn More]        │  │ [Learn More]        │      │
│  └─────────────────────┘  └─────────────────────┘      │
│                                                         │
│  ┌─────────────────────┐  ┌─────────────────────┐      │
│  │ Attention           │  │ Hash Embeddings     │      │
│  │ ...                 │  │ ...                 │      │
│  └─────────────────────┘  └─────────────────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Features
- Search
- Category filters
- Quick definitions
- Detailed explanations
- Interactive examples
- Related concepts

---

## Navigation Structure

```
/                           Landing page
├── /learn                  Learning hub
│   ├── /overview           Algorithm overview
│   ├── /retrieval          Retrieval deep-dive
│   ├── /ranking            Ranking deep-dive
│   ├── /filtering          Filtering explained
│   └── /concepts           Concept library
│       └── /[concept]      Individual concept
├── /explore                Code explorer
│   ├── /phoenix            Phoenix module
│   │   └── /[file]         Individual file
│   ├── /home-mixer         Home mixer module
│   ├── /thunder            Thunder module
│   └── /search             Code search
├── /audit                  Audit dashboard
│   ├── /critical           Critical issues
│   ├── /security           Security issues
│   └── /improvements       Proposed fixes
└── /simulate               Feed simulator
    └── /results            Simulation results
```
