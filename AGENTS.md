# Zener Web - Agent Coding Guidelines

## Project Overview
- **Stack**: Next.js 14+ (App Router), React 18+, TypeScript, shadcn/ui, Tailwind CSS
- **Purpose**: AI Remote Assistance Platform - UI Navigator for Google AI Hackathon
- **AI Backend**: Google ADK + Gemini Computer Use API

## Build & Development Commands

```bash
# Install dependencies
npm install

# Development server
npm run dev

# Build for production
npm run build

# Start production server
npm start

# Lint code
npm run lint

# TypeScript type checking
npm run typecheck
```

### Running Tests

```bash
# Run all tests
npm test

# Run single test file
npm test -- path/to/testfile.test.ts

# Run single test (regex pattern)
npm test -- --testNamePattern="test name"

# Run in watch mode
npm test -- --watch

# Run with coverage
npm test -- --coverage
```

## Code Style Guidelines

### General Principles
- Use functional components with TypeScript
- Prefer React Server Components (RSC) where possible
- Keep components small and focused (single responsibility)
- Extract reusable logic into custom hooks
- Use early returns for cleaner conditionals

### TypeScript
- Always define explicit return types for functions
- Use `interface` for object shapes, `type` for unions/aliases
- Avoid `any` - use `unknown` when type is truly unknown
- Enable strict mode in tsconfig.json

### Naming Conventions
- **Components**: PascalCase (e.g., `ScreenMirror.tsx`)
- **Hooks**: camelCase with `use` prefix (e.g., `useSession.ts`)
- **Utilities**: camelCase (e.g., `formatDate.ts`)
- **Constants**: SCREAMING_SNAKE_CASE (e.g., `MAX_RETRY_COUNT`)
- **Files**: kebab-case for utilities, PascalCase for components

### Imports
```typescript
// Group imports in order:
// 1. React/Next imports
// 2. External libraries
// 3. Internal components/hooks
// 4. Utils/types
// 5. Assets/styles

import { useState, useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { motion } from 'framer-motion';
import { Button } from '@/components/ui/button';
import { useAuth } from '@/hooks/useAuth';
import { formatTimestamp } from '@/lib/utils';
import type { Session } from '@/types';
```

### React Patterns
- Use functional components with hooks
- Destructure props with TypeScript
- Memoize expensive computations with `useMemo`
- Use `useCallback` for callbacks passed to children
- Prefer composition over inheritance

### Tailwind CSS
- Use semantic class ordering (layout → spacing → visual → states)
- Use `cn()` utility for conditional classes (from shadcn)
- Prefer Tailwind tokens over arbitrary values
- Keep responsive classes grouped together

### shadcn/ui Usage
- Components live in `components/ui/`
- Customize via `components.json` and `tailwind.config.ts`
- Use `npx shadcn@latest add <component>` to add new components

## Error Handling
- Use TypeScript for type-safe error handling
- Create custom error types for domain-specific errors
- Handle async errors with try/catch in components
- Show user-friendly error messages in UI

## Git Conventions
- Use meaningful commit messages
- Create feature branches: `feature/description`
- Bug fixes: `fix/description`
- Avoid committing secrets (use .env.local)

## WebSocket Protocol (Session Communication)
```typescript
// Client -> Server
type ClientMessage =
  | { type: "start_session"; task: string; token: string }
  | { type: "confirm"; confirmed: boolean }
  | { type: "abort" };

// Server -> Client
type AgentMessage =
  | { type: "session_ready"; sessionId: string; agentUrl: string }
  | { type: "screenshot"; data: string; timestamp: number }
  | { type: "thinking"; text: string }
  | { type: "action_start"; name: string; args: Record<string, unknown> }
  | { type: "action_done"; name: string; result: string }
  | { type: "confirm_required"; action: { name: string; args: unknown } }
  | { type: "task_complete"; summary: string }
  | { type: "error"; message: string };
```

## Firebase/Firestore Structure
```typescript
// users/{uid}
interface User {
  uid: string;
  email: string;
  createdAt: Timestamp;
  plan: "free" | "pro";
  usageMinutes: number;
}

// sessions/{sessionId}
interface Session {
  uid: string;
  status: "waiting" | "active" | "complete" | "error";
  task: string;
  createdAt: Timestamp;
  lastActive: Timestamp;
  actionCount: number;
  agentToken: string;
}
```

## Agent Architecture
- **Local Agent**: Python with mss, PyAutoGUI (screen capture + execution)
- **Cloud Agent**: Google ADK on Cloud Run with Gemini Computer Use
- **Safety Gate**: `require_confirmation` flow for risky actions

## Key Files Structure
```
src/
├── app/                    # Next.js App Router pages
├── components/
│   ├── ui/                # shadcn/ui components
│   └── *                  # Feature components
├── hooks/                 # Custom React hooks
├── lib/                   # Utilities
├── types/                 # TypeScript types
└── styles/               # Global styles
```
