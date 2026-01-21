# Web App Architecture

## Project Structure

```
webapp/
├── app/                          # Next.js App Router
│   ├── layout.tsx               # Root layout with providers
│   ├── page.tsx                 # Landing page
│   ├── globals.css              # Global styles
│   │
│   ├── learn/                   # Educational content
│   │   ├── page.tsx            # Learning hub
│   │   ├── overview/           # Algorithm overview
│   │   ├── retrieval/          # Retrieval deep-dive
│   │   ├── ranking/            # Ranking deep-dive
│   │   ├── filtering/          # Filtering explained
│   │   └── concepts/           # Concept library
│   │
│   ├── explore/                 # Code exploration
│   │   ├── page.tsx            # Code explorer hub
│   │   ├── [module]/           # Dynamic module routes
│   │   │   └── [file]/         # Dynamic file routes
│   │   └── search/             # Code search
│   │
│   ├── audit/                   # Vulnerability dashboard
│   │   ├── page.tsx            # Audit overview
│   │   ├── critical/           # Critical issues
│   │   ├── security/           # Security vulnerabilities
│   │   └── improvements/       # Proposed improvements
│   │
│   ├── simulate/                # Feed simulator
│   │   ├── page.tsx            # Simulator interface
│   │   └── results/            # Simulation results
│   │
│   └── api/                     # API routes
│       ├── auth/               # Privy auth endpoints
│       ├── simulate/           # Simulation endpoints
│       └── progress/           # User progress tracking
│
├── components/                   # React components
│   ├── ui/                      # Base UI components
│   │   ├── Button.tsx
│   │   ├── Card.tsx
│   │   ├── Modal.tsx
│   │   └── ...
│   │
│   ├── layout/                  # Layout components
│   │   ├── Header.tsx
│   │   ├── Sidebar.tsx
│   │   ├── Footer.tsx
│   │   └── Navigation.tsx
│   │
│   ├── learn/                   # Learning components
│   │   ├── StepWalkthrough.tsx
│   │   ├── ConceptCard.tsx
│   │   ├── InteractiveDiagram.tsx
│   │   └── ProgressTracker.tsx
│   │
│   ├── code/                    # Code display components
│   │   ├── CodeViewer.tsx
│   │   ├── LineExplainer.tsx
│   │   ├── FileTree.tsx
│   │   └── SearchResults.tsx
│   │
│   ├── audit/                   # Audit components
│   │   ├── VulnerabilityCard.tsx
│   │   ├── SeverityBadge.tsx
│   │   ├── ImpactChart.tsx
│   │   └── FixProposal.tsx
│   │
│   ├── simulate/                # Simulator components
│   │   ├── PostCard.tsx
│   │   ├── ScoreBreakdown.tsx
│   │   ├── RankingVisualizer.tsx
│   │   └── HistoryInput.tsx
│   │
│   └── animations/              # Animation components
│       ├── FlowDiagram.tsx
│       ├── EmbeddingVisualizer.tsx
│       ├── AttentionHeatmap.tsx
│       └── TransformerAnimation.tsx
│
├── lib/                          # Utilities and helpers
│   ├── privy.ts                 # Privy configuration
│   ├── code-data.ts             # Code content loader
│   ├── audit-data.ts            # Audit findings data
│   ├── animations.ts            # Animation presets
│   └── utils.ts                 # General utilities
│
├── hooks/                        # Custom React hooks
│   ├── useProgress.ts           # User progress tracking
│   ├── useCodeExplorer.ts       # Code exploration state
│   ├── useSimulator.ts          # Simulation logic
│   └── useAuth.ts               # Authentication state
│
├── stores/                       # Zustand stores
│   ├── userStore.ts             # User state
│   ├── progressStore.ts         # Learning progress
│   └── simulatorStore.ts        # Simulation state
│
├── types/                        # TypeScript types
│   ├── code.ts                  # Code-related types
│   ├── audit.ts                 # Audit-related types
│   └── user.ts                  # User-related types
│
├── content/                      # Static content
│   ├── explanations/            # File explanations (from .docs)
│   ├── concepts/                # Concept definitions
│   └── audit/                   # Audit findings
│
└── public/                       # Static assets
    ├── images/
    ├── icons/
    └── diagrams/
```

---

## Component Architecture

### Provider Hierarchy

```tsx
// app/layout.tsx
<html>
  <body>
    <PrivyProvider>           {/* Authentication */}
      <ThemeProvider>         {/* Dark/Light mode */}
        <ProgressProvider>    {/* Learning progress */}
          <ToastProvider>     {/* Notifications */}
            {children}
          </ToastProvider>
        </ProgressProvider>
      </ThemeProvider>
    </PrivyProvider>
  </body>
</html>
```

---

## Data Flow

### 1. Static Content (No Auth Required)

```
/content/explanations/*.md
         ↓
    MDX Processor
         ↓
    React Components
         ↓
    Rendered Page
```

### 2. User Progress (Auth Required)

```
User Action (complete section)
         ↓
    Zustand Store
         ↓
    API Route (/api/progress)
         ↓
    Database (PostgreSQL)
         ↓
    Sync on reload
```

### 3. Feed Simulation (Auth Required)

```
User X Profile (via Privy)
         ↓
    Simulation API
         ↓
    Mock Ranking Engine
         ↓
    Results Display
```

---

## Key Components

### 1. CodeViewer

Displays code with syntax highlighting and inline explanations.

```tsx
interface CodeViewerProps {
  filePath: string;
  language: 'python' | 'rust';
  highlightLines?: number[];
  explanations?: Record<number, string>;
  onLineClick?: (line: number) => void;
}
```

### 2. StepWalkthrough

Animated step-by-step explanation.

```tsx
interface StepWalkthroughProps {
  steps: Step[];
  currentStep: number;
  onStepChange: (step: number) => void;
  animation?: 'fade' | 'slide' | 'scale';
}

interface Step {
  title: string;
  description: string;
  visual: ReactNode;
  code?: CodeSnippet;
}
```

### 3. VulnerabilityCard

Displays audit findings.

```tsx
interface VulnerabilityCardProps {
  severity: 'critical' | 'high' | 'medium' | 'low';
  title: string;
  description: string;
  file: string;
  line: number;
  impact: string;
  fix?: string;
  expanded?: boolean;
}
```

### 4. FlowDiagram

Interactive algorithm flow visualization.

```tsx
interface FlowDiagramProps {
  nodes: FlowNode[];
  edges: FlowEdge[];
  highlightedNode?: string;
  onNodeClick?: (nodeId: string) => void;
  animated?: boolean;
}
```

---

## State Management

### User Store (Zustand)

```typescript
interface UserState {
  // Auth
  isAuthenticated: boolean;
  user: User | null;
  xProfile: XProfile | null;

  // Actions
  setUser: (user: User) => void;
  setXProfile: (profile: XProfile) => void;
  logout: () => void;
}
```

### Progress Store (Zustand)

```typescript
interface ProgressState {
  // Completed sections
  completedSections: string[];
  currentSection: string | null;

  // Actions
  completeSection: (sectionId: string) => void;
  setCurrentSection: (sectionId: string) => void;
  getProgress: () => number; // 0-100
}
```

---

## API Routes

### POST /api/auth/privy
Handle Privy authentication callback.

### GET /api/progress
Fetch user's learning progress.

### POST /api/progress
Update user's learning progress.

### POST /api/simulate
Run feed simulation with user's profile.

### GET /api/code/[...path]
Fetch code file with explanations.

---

## Performance Considerations

1. **Code Splitting**: Each learn/ section is a separate chunk
2. **Image Optimization**: Use Next.js Image component
3. **Lazy Loading**: Load code explanations on demand
4. **Caching**: Cache API responses where appropriate
5. **Static Generation**: Pre-render content pages at build time

---

## Security Considerations

1. **Auth**: All user-specific features require Privy authentication
2. **API**: Rate limiting on simulation endpoints
3. **Data**: No sensitive user data stored; X profile is session-only
4. **XSS**: All user content sanitized
5. **CSRF**: Next.js built-in protection
