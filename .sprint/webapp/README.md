# X Algorithm Explorer - Web App Planning

**Project Name**: X Algorithm Explorer
**Purpose**: An interactive educational web application that helps users understand how X's recommendation algorithm works, including the vulnerabilities and improvements identified in our audit.

---

## Vision

Create a web application where users can:
1. **Learn** how the X algorithm decides what posts to show
2. **Explore** the codebase interactively with visual explanations
3. **Understand** the vulnerabilities and how they could affect their feed
4. **Connect** their X account (via Privy) to see personalized insights
5. **Experiment** with simulated feed ranking

---

## Tech Stack

| Component | Technology | Reason |
|-----------|------------|--------|
| Framework | Next.js 14+ | Server components, API routes, great DX |
| Language | TypeScript | Type safety, better tooling |
| Styling | Tailwind CSS | Rapid development, consistent design |
| Animations | Framer Motion | Smooth, performant animations |
| Auth | Privy | Easy X (Twitter) OAuth, wallet support |
| State | Zustand | Simple, performant state management |
| Diagrams | React Flow | Interactive flowcharts |
| Code Display | Shiki | Beautiful syntax highlighting |
| Database | PostgreSQL (optional) | User progress, saved states |
| Deployment | Vercel | Seamless Next.js deployment |

---

## Core Features

### 1. Interactive Algorithm Walkthrough
Step-by-step visualization of how a post goes from creation to appearing in your feed.

### 2. Code Explorer
Browse the codebase with inline explanations, highlighting, and visual aids.

### 3. Vulnerability Dashboard
Interactive display of all audit findings with severity, impact, and proposed fixes.

### 4. Personal Feed Simulator
After connecting X account, show a simulation of how the algorithm would rank sample posts for you.

### 5. Concept Library
Glossary of terms (embeddings, transformers, attention) with interactive examples.

---

## Planning Documents

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | System architecture and component design |
| [FEATURES.md](./FEATURES.md) | Detailed feature specifications |
| [PAGES.md](./PAGES.md) | Page-by-page design and content |
| [ANIMATIONS.md](./ANIMATIONS.md) | Animation specifications |
| [PRIVY_INTEGRATION.md](./PRIVY_INTEGRATION.md) | X/Twitter authentication setup |
| [DATA_FLOW.md](./DATA_FLOW.md) | How data moves through the app |

---

## Timeline

### Phase 1: Foundation (Week 1-2)
- Set up Next.js project
- Configure Tailwind and Framer Motion
- Implement basic page structure
- Create navigation and layout

### Phase 2: Content (Week 3-4)
- Build algorithm walkthrough pages
- Create code explorer component
- Implement syntax highlighting
- Add basic animations

### Phase 3: Interactive Features (Week 5-6)
- Build vulnerability dashboard
- Create interactive diagrams
- Implement concept library
- Add progress tracking

### Phase 4: Privy Integration (Week 7)
- Set up Privy authentication
- Connect X OAuth flow
- Build personalized features
- Create feed simulator

### Phase 5: Polish (Week 8)
- Performance optimization
- Accessibility audit
- Mobile responsiveness
- Final testing and deployment

---

## Getting Started

Once the project is created:

```bash
cd webapp
npm install
npm run dev
```

Visit `http://localhost:3000` to see the app.

---

## Key Design Principles

1. **Accessibility First**: Everyone should be able to understand, regardless of technical background
2. **Progressive Disclosure**: Start simple, let users dive deeper if they want
3. **Visual Learning**: Prioritize diagrams, animations, and interactive elements
4. **Honest Transparency**: Don't hide the problems - the vulnerabilities are part of the story
5. **Engagement**: Make learning fun with interactive elements and progress tracking
