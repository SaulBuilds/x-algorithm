# Animation Specifications

## Philosophy

Animations should:
1. **Educate** - Help users understand complex concepts
2. **Guide** - Draw attention to important elements
3. **Delight** - Make the experience enjoyable
4. **Perform** - Never block interaction or feel slow

---

## Animation Library: Framer Motion

```bash
npm install framer-motion
```

### Basic Setup

```tsx
// components/animations/FadeIn.tsx
'use client';

import { motion } from 'framer-motion';
import { ReactNode } from 'react';

interface FadeInProps {
  children: ReactNode;
  delay?: number;
  duration?: number;
}

export function FadeIn({ children, delay = 0, duration = 0.5 }: FadeInProps) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ delay, duration, ease: 'easeOut' }}
    >
      {children}
    </motion.div>
  );
}
```

---

## Animation 1: Algorithm Pipeline Flow

### Description
Animated diagram showing a post flowing through the algorithm stages.

### Implementation

```tsx
// components/animations/PipelineFlow.tsx
'use client';

import { motion, AnimatePresence } from 'framer-motion';
import { useState, useEffect } from 'react';

const stages = [
  { id: 'request', label: 'Request', color: '#3B82F6' },
  { id: 'retrieval', label: 'Retrieval', color: '#8B5CF6' },
  { id: 'ranking', label: 'Ranking', color: '#EC4899' },
  { id: 'filtering', label: 'Filtering', color: '#F59E0B' },
  { id: 'selection', label: 'Selection', color: '#10B981' },
  { id: 'response', label: 'Your Feed', color: '#06B6D4' },
];

export function PipelineFlow() {
  const [activeStage, setActiveStage] = useState(0);
  const [isPlaying, setIsPlaying] = useState(true);

  useEffect(() => {
    if (!isPlaying) return;

    const interval = setInterval(() => {
      setActiveStage((prev) => (prev + 1) % stages.length);
    }, 2000);

    return () => clearInterval(interval);
  }, [isPlaying]);

  return (
    <div className="pipeline-container">
      {/* Stage nodes */}
      <div className="stages flex justify-between">
        {stages.map((stage, index) => (
          <motion.div
            key={stage.id}
            className="stage-node"
            animate={{
              scale: activeStage === index ? 1.2 : 1,
              backgroundColor: activeStage >= index ? stage.color : '#374151',
            }}
            transition={{ duration: 0.3 }}
          >
            <span className="stage-label">{stage.label}</span>
          </motion.div>
        ))}
      </div>

      {/* Animated packet */}
      <motion.div
        className="packet"
        animate={{
          x: `${activeStage * 100}%`,
        }}
        transition={{
          duration: 1.5,
          ease: 'easeInOut',
        }}
      >
        <PostIcon />
      </motion.div>

      {/* Controls */}
      <button onClick={() => setIsPlaying(!isPlaying)}>
        {isPlaying ? 'Pause' : 'Play'}
      </button>
    </div>
  );
}
```

---

## Animation 2: Embedding Visualization

### Description
Show how a user/post gets converted into numbers (embeddings).

### Implementation

```tsx
// components/animations/EmbeddingVisualizer.tsx
'use client';

import { motion } from 'framer-motion';
import { useState } from 'react';

interface Props {
  label: string;
  dimensions?: number;
}

export function EmbeddingVisualizer({ label, dimensions = 8 }: Props) {
  const [isAnimating, setIsAnimating] = useState(false);
  const [values, setValues] = useState<number[]>([]);

  const animate = () => {
    setIsAnimating(true);
    setValues([]);

    // Animate values appearing one by one
    for (let i = 0; i < dimensions; i++) {
      setTimeout(() => {
        setValues((prev) => [...prev, Math.random() * 2 - 1]);
      }, i * 200);
    }

    setTimeout(() => setIsAnimating(false), dimensions * 200 + 500);
  };

  return (
    <div className="embedding-viz">
      {/* Input */}
      <motion.div
        className="input-box"
        animate={isAnimating ? { scale: [1, 1.1, 1] } : {}}
      >
        {label}
      </motion.div>

      {/* Arrow */}
      <motion.div
        className="arrow"
        animate={isAnimating ? { opacity: 1, x: 0 } : { opacity: 0.5 }}
      >
        →
      </motion.div>

      {/* Embedding values */}
      <div className="embedding-values flex gap-1">
        {Array.from({ length: dimensions }).map((_, i) => (
          <motion.div
            key={i}
            className="value-box"
            initial={{ opacity: 0, scale: 0 }}
            animate={{
              opacity: values[i] !== undefined ? 1 : 0.2,
              scale: values[i] !== undefined ? 1 : 0.5,
              backgroundColor: values[i] !== undefined
                ? values[i] > 0
                  ? `rgba(34, 197, 94, ${Math.abs(values[i])})`
                  : `rgba(239, 68, 68, ${Math.abs(values[i])})`
                : '#374151',
            }}
            transition={{ duration: 0.3 }}
          >
            {values[i]?.toFixed(2) || '?'}
          </motion.div>
        ))}
      </div>

      <button onClick={animate} disabled={isAnimating}>
        {isAnimating ? 'Converting...' : 'Convert to Embedding'}
      </button>

      <p className="text-sm text-gray-400 mt-2">
        Each number represents a different aspect of "{label}"
      </p>
    </div>
  );
}
```

---

## Animation 3: Attention Heatmap

### Description
Visualize how the Transformer pays attention to different parts of the sequence.

### Implementation

```tsx
// components/animations/AttentionHeatmap.tsx
'use client';

import { motion } from 'framer-motion';
import { useState } from 'react';

interface Props {
  tokens: string[];
}

export function AttentionHeatmap({ tokens }: Props) {
  const [selectedToken, setSelectedToken] = useState<number | null>(null);

  // Generate mock attention weights
  const getAttention = (from: number, to: number) => {
    if (from < to) return 0; // Causal mask
    return Math.random() * 0.5 + (from === to ? 0.5 : 0);
  };

  return (
    <div className="attention-heatmap">
      <div className="grid" style={{ gridTemplateColumns: `repeat(${tokens.length + 1}, 1fr)` }}>
        {/* Header row */}
        <div className="cell header"></div>
        {tokens.map((token, i) => (
          <div key={`h-${i}`} className="cell header">
            {token}
          </div>
        ))}

        {/* Data rows */}
        {tokens.map((rowToken, rowIdx) => (
          <>
            <div key={`r-${rowIdx}`} className="cell header">
              {rowToken}
            </div>
            {tokens.map((_, colIdx) => {
              const attention = getAttention(rowIdx, colIdx);
              const isHighlighted = selectedToken === rowIdx || selectedToken === colIdx;

              return (
                <motion.div
                  key={`${rowIdx}-${colIdx}`}
                  className="cell"
                  style={{
                    backgroundColor: `rgba(59, 130, 246, ${attention})`,
                  }}
                  animate={{
                    scale: isHighlighted ? 1.1 : 1,
                    borderColor: isHighlighted ? '#fff' : 'transparent',
                  }}
                  onHoverStart={() => setSelectedToken(rowIdx)}
                  onHoverEnd={() => setSelectedToken(null)}
                >
                  {attention.toFixed(2)}
                </motion.div>
              );
            })}
          </>
        ))}
      </div>

      <p className="text-sm text-gray-400 mt-4">
        Hover over a row to see what that token "pays attention to"
      </p>
    </div>
  );
}
```

---

## Animation 4: Ranking Visualization

### Description
Show posts being ranked by score, with animated reordering.

### Implementation

```tsx
// components/animations/RankingVisualizer.tsx
'use client';

import { motion, AnimatePresence, Reorder } from 'framer-motion';
import { useState } from 'react';

interface Post {
  id: string;
  title: string;
  score: number;
}

export function RankingVisualizer({ initialPosts }: { initialPosts: Post[] }) {
  const [posts, setPosts] = useState(initialPosts);
  const [isRanking, setIsRanking] = useState(false);

  const rankPosts = async () => {
    setIsRanking(true);

    // Simulate scoring
    await new Promise((r) => setTimeout(r, 500));

    // Assign random scores
    const scored = posts.map((p) => ({
      ...p,
      score: Math.random(),
    }));

    // Animate sorting
    const sorted = [...scored].sort((a, b) => b.score - a.score);

    setPosts(sorted);
    setIsRanking(false);
  };

  return (
    <div className="ranking-viz">
      <Reorder.Group axis="y" values={posts} onReorder={setPosts}>
        {posts.map((post, index) => (
          <Reorder.Item key={post.id} value={post}>
            <motion.div
              className="post-card"
              layout
              animate={{
                scale: isRanking ? [1, 1.02, 1] : 1,
                backgroundColor: isRanking ? '#1E3A5F' : '#1F2937',
              }}
              transition={{ duration: 0.5 }}
            >
              <span className="rank">#{index + 1}</span>
              <span className="title">{post.title}</span>
              <motion.span
                className="score"
                animate={{ opacity: post.score ? 1 : 0.3 }}
              >
                {post.score ? `Score: ${post.score.toFixed(3)}` : 'Scoring...'}
              </motion.span>
            </motion.div>
          </Reorder.Item>
        ))}
      </Reorder.Group>

      <button onClick={rankPosts} disabled={isRanking}>
        {isRanking ? 'Ranking...' : 'Rank Posts'}
      </button>
    </div>
  );
}
```

---

## Animation 5: Vulnerability Severity Pulse

### Description
Pulse animation based on vulnerability severity.

### Implementation

```tsx
// components/animations/SeverityPulse.tsx
'use client';

import { motion } from 'framer-motion';

type Severity = 'critical' | 'high' | 'medium' | 'low';

const severityConfig = {
  critical: { color: '#EF4444', pulseSpeed: 0.5 },
  high: { color: '#F97316', pulseSpeed: 1 },
  medium: { color: '#EAB308', pulseSpeed: 1.5 },
  low: { color: '#3B82F6', pulseSpeed: 2 },
};

export function SeverityPulse({ severity }: { severity: Severity }) {
  const config = severityConfig[severity];

  return (
    <motion.div
      className="severity-indicator"
      animate={{
        scale: [1, 1.2, 1],
        boxShadow: [
          `0 0 0 0 ${config.color}40`,
          `0 0 0 10px ${config.color}00`,
          `0 0 0 0 ${config.color}40`,
        ],
      }}
      transition={{
        duration: config.pulseSpeed,
        repeat: Infinity,
        ease: 'easeInOut',
      }}
      style={{ backgroundColor: config.color }}
    />
  );
}
```

---

## Animation Presets

```tsx
// lib/animations.ts
export const fadeInUp = {
  initial: { opacity: 0, y: 20 },
  animate: { opacity: 1, y: 0 },
  exit: { opacity: 0, y: -20 },
  transition: { duration: 0.4, ease: 'easeOut' },
};

export const scaleIn = {
  initial: { opacity: 0, scale: 0.9 },
  animate: { opacity: 1, scale: 1 },
  exit: { opacity: 0, scale: 0.9 },
  transition: { duration: 0.3 },
};

export const slideInLeft = {
  initial: { opacity: 0, x: -50 },
  animate: { opacity: 1, x: 0 },
  exit: { opacity: 0, x: 50 },
  transition: { duration: 0.4, ease: 'easeOut' },
};

export const staggerChildren = {
  animate: {
    transition: {
      staggerChildren: 0.1,
    },
  },
};
```

---

## Performance Guidelines

1. **Use `layout` prop sparingly** - Only for actual layout changes
2. **Prefer CSS transforms** - `scale`, `rotate`, `translate` over `width`, `height`
3. **Use `will-change` carefully** - Only on elements that actually animate
4. **Reduce motion for accessibility** - Respect `prefers-reduced-motion`
5. **Lazy load complex animations** - Don't load unused animation code

```tsx
// Respect reduced motion preference
import { useReducedMotion } from 'framer-motion';

export function AnimatedComponent() {
  const shouldReduceMotion = useReducedMotion();

  return (
    <motion.div
      animate={{ scale: shouldReduceMotion ? 1 : 1.1 }}
      transition={{ duration: shouldReduceMotion ? 0 : 0.3 }}
    >
      Content
    </motion.div>
  );
}
```
