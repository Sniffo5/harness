# Harness — Implementation Plan

> Phased, dependency-aware build plan from zero to shippable.
> Each phase produces something testable. No building without validation.

---

## Table of Contents

1. [Phases Overview](#1-phases-overview)
2. [Phase 0: Project Scaffold](#2-phase-0-project-scaffold)
3. [Phase 1: Core Types + Profiles](#3-phase-1-core-types--profiles)
4. [Phase 2: Provider Layer](#4-phase-2-provider-layer)
5. [Phase 3: Deslopper](#5-phase-3-deslopper)
6. [Phase 4: Cache System](#6-phase-4-cache-system)
7. [Phase 5: Process Engine + IPC](#7-phase-5-process-engine--ipc)
8. [Phase 6: UI Shell + Chat](#8-phase-6-ui-shell--chat)
9. [Phase 7: Git Integration](#9-phase-7-git-integration)
10. [Phase 8: Doc Awareness](#10-phase-8-doc-awareness)
11. [Phase 9: Agent System](#11-phase-9-agent-system)
12. [Phase 10: Training UI](#12-phase-10-training-ui)
13. [Phase 11: Polish & Ship](#13-phase-11-polish--ship)
14. [Infrastructure](#14-infrastructure)

---

## 2. Phase 0: Project Scaffold

**Goal:** Working Electron + React + TypeScript app that opens a window.

### Tasks

- [ ] Initialize npm project
- [ ] Set up Vite + React + TypeScript
- [ ] Configure Electron (main + preload + renderer)
- [ ] Set up Tailwind CSS
- [ ] Add electron-builder config
- [ ] Verify: `npm run dev` opens an Electron window
- [ ] Git init with proper .gitignore

### Dependencies

```json
{
  "devDependencies": {
    "electron": "^35.0.0",
    "electron-builder": "^25.0.0",
    "vite": "^6.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "typescript": "^5.7.0",
    "tailwindcss": "^4.0.0"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  }
}
```

### Test: App window opens with "Harness" title.

---

## 3. Phase 1: Core Types + Profiles

**Goal:** Shared type definitions + profile system with CRUD.

### Tasks

- [ ] Define all types in `src/shared/types.ts`
- [ ] Define profile types in `src/shared/profile-types.ts`
- [ ] Define IPC contract in `src/shared/ipc.ts`
- [ ] Build profile manager (list, get, create, save, delete, fork)
- [ ] Write built-in profile JSON files (Sniffo Code, Sniffo UI, etc.)
- [ ] Test: Profile CRUD works as pure Node.js module (no Electron needed yet)

### Key Decisions

- Profiles are JSON files on disk, loaded into memory
- Built-in profiles are read-only (shipped in `src/presets/`)
- User profiles go in `user-config/profiles/`
- Profile versioning: each save increments `version`, old versions are archived

### Test: `npm run test:profiles` — create, save, load, fork, reset profile.

---

## 4. Phase 2: Provider Layer

**Goal:** Abstract LLM interface with one functional adapter (OpenAI).

### Tasks

- [ ] Define `ProviderAdapter` interface in `src/main/providers/adapter.ts`
- [ ] Define `CompletionRequest` / `CompletionResponse` / `ModelCapabilities`
- [ ] Build OpenAI adapter (`src/main/providers/openai.ts`)
  - [ ] `complete()` — non-streaming
  - [ ] `stream()` — async iterable
  - [ ] `abort()` — cancel in-flight requests
  - [ ] `listModels()` — available models
- [ ] Build provider registry (`src/main/providers/registry.ts`)
- [ ] Add stub adapters for Anthropic and Ollama (throws "not implemented")
- [ ] Store API keys encrypted via `safeStorage` (Electron's built-in OS keychain bridge)
- [ ] Test: Send a real prompt to OpenAI and get a response back

### Key Decisions

- API keys stored via Electron `safeStorage` → OS keychain (macOS Keychain, Windows Credential Manager)
- Provider config in `user-config/config.json` (key names only, not values)
- Abort uses `AbortController` internally
- Streaming delivers chunks via IPC events (`stream:{requestId}`)

### Test: `npm run test:provider` — send prompt, get response (with real API key).

---

## 5. Phase 3: Deslopper

**Goal:** Deslopper engine that can parse, score, and rewrite output.

### Tasks

- [ ] Define deslopper types (`src/shared/deslopper-types.ts`)
- [ ] Build tokenizer + sentence splitter
- [ ] Write built-in pattern rules:
  - [ ] Hedging words ("I think", "perhaps")
  - [ ] Fake gratitude ("Great question!")
  - [ ] Pompous verbs ("leverage", "delve")
  - [ ] Redundant phrases ("in order to")
  - [ ] Verbose signposting ("First, let me...")
  - [ ] Bullet density check
- [ ] Build rule engine (pattern matching per rule)
- [ ] Build scorer (weighted score aggregation, 0–100)
- [ ] Build threshold checker (pass / flag / block)
- [ ] Build rewriter (strip → rewrite for automatic remediation)
- [ ] Build deslop report generator (what was flagged, line numbers)
- [ ] Test: Feed sample AI slop, verify score matches expectations

### Key Decisions

- Rules are defined as JSON (not code) so users can edit them
- Each rule has severity + remediation type
- Rewriter uses string replacement + templating, not an LLM call
- Deslopper runs synchronously in main process (no IPC needed, it's fast)

### Test: `npm run test:deslopper` — 10 test cases with known slop, verify scores + rewrites.

---

## 6. Phase 4: Cache System

**Goal:** Database setup + memory cache + all table schemas.

### Tasks

- [ ] Add better-sqlite3 + sqlite-vec dependencies
- [ ] Build CacheService (`src/main/cache/service.ts`)
  - [ ] Database initialization (tables, indexes, WAL mode)
  - [ ] L1 memory cache class
  - [ ] Get/set/invalidate/stats methods
  - [ ] Eviction policy (LRU + TTL)
- [ ] Create all SQL tables:
  - [ ] `cache_entries` — generic key-value
  - [ ] `doc_cache` + `doc_fts` — documentation + full-text search
  - [ ] `doc_embeddings` (vec0) + `doc_embedding_meta` — vector search
  - [ ] `llm_response_cache` — exact-match LLM response
  - [ ] `llm_embedding_cache` — embedding text→vector
  - [ ] `git_diff_cache` — structured diffs
- [ ] Build embedding interface (call to OpenAI/Ollama for vectorization)
- [ ] Build memory cache for hot items (profile, recent LLM responses)
- [ ] Test: Cache a value, retrieve it, verify TTL eviction

### Key Decisions

- One `cache.db` file for everything
- sqlite-vec loaded as extension via better-sqlite3
- Embedding model configurable (default: text-embedding-3-small)
- Cache is disposable — user can delete it safely
- WAL mode + NORMAL synchronous for write performance

### Test: `npm run test:cache` — store, retrieve, invalidate, evict, verify FTS5 + vec0 search.

---

## 7. Phase 5: Process Engine + IPC

**Goal:** Process lifecycle + typed IPC bridge between main and renderer.

### Tasks

- [ ] Define process types (`Chat`, `CodeGen`, `Review`, etc.)
- [ ] Build ProcessEngine class:
  - [ ] `start(config)` → process ID
  - [ ] `stop(processId)` → abort + cleanup
  - [ ] Lifecycle: INIT → PREPARE → PROMPT → CALL → DESLOP → EVALUATE → FEEDBACK
  - [ ] Prompt builder (system prompt + profile + context + doc injection)
  - [ ] Evaluation step (pass/fail/retry logic)
  - [ ] Retry loop with prompt refinement
- [ ] Register all IPC handlers in `src/main/ipc-handlers.ts`
  - [ ] `llm:complete`, `llm:stream`, `llm:abort`
  - [ ] `process:start`, `process:stop`, `process:status`
  - [ ] `profiles:list`, `profiles:save`, `profiles:train`
  - [ ] `docs:lookup`, `docs:cache`
  - [ ] `git:diff`, `git:commit`, `git:pr`
- [ ] Build preload script with `contextBridge`
- [ ] Test: Send IPC message from renderer, get response from main

### Key Decisions

- Process engine owns the lifecycle loop
- Prompt builder responsible for assembling the final prompt (profile + docs + context)
- Evaluation step runs deslopper scorer — if score > threshold, auto-retry with "less slop, more direct" appended
- Streaming LLM responses delivered as chunks over IPC events
- All IPC channels are typed from a single contract interface

### Test: `test/manual:process` — start a code gen process, watch it flow through lifecycle, see deslopped output.

---

## 8. Phase 6: UI Shell + Chat

**Goal:** Working UI with sidebar, chat interface, and deslop panel.

### Tasks

- [ ] Install UI dependencies:
  ```json
  {
    "zustand": "^5.0.0",
    "lucide-react": "^0.400.0",
    "react-router-dom": "^7.0.0",
    "codemirror": "^6.0.0",
    "@codemirror/lang-javascript": "^6.0.0"
  }
  ```
- [ ] Build layout components:
  - [ ] `Sidebar` — process list, agent list, profile selector
  - [ ] `TabBar` — Chat, Git, Agents, Profiles, Docs, Settings
  - [ ] `StatusBar` — active process, git branch, provider info
  - [ ] `DeslopPanel` — collapsible panel showing deslopper activity
- [ ] Build ChatView:
  - [ ] Message list with streaming updates
  - [ ] Message bubble component (user / assistant, with deslop indicator)
  - [ ] Input area with submit + profile selector
  - [ ] Streaming output that progressively renders
- [ ] Build Zustand stores:
  - [ ] `process-store` — active processes, history
  - [ ] `settings-store` — provider config, preferences
- [ ] Wire IPC hooks:
  - [ ] `useProcess` — start, stop, stream process
  - [ ] `useIpc` — generic invoke wrapper
- [ ] Test: Type a message, see LLM response stream in, deslop panel updates

### Key Decisions

- Zustand over Redux (lighter, no boilerplate, works great with IPC)
- lucide-react for icons (tree-shakeable, good variety)
- CodeMirror 6 for code display + diff highlighting (Monaco is 20MB+)
- Deslop panel shows: what was flagged, before/after, score
- Chat supports markdown rendering (remark + rehype)

### Test: Open app, type "Write a React component" → see streaming response → see deslop report.

---

## 9. Phase 7: Git Integration

**Goal:** Diff viewing, commit generation, PR generation.

### Tasks

- [ ] Add dependency: `isomorphic-git` (pure JS git, works in main process)
- [ ] Build `GitService`:
  - [ ] `getDiff()` — run git diff, parse into structured hunks
  - [ ] `stageFiles()`, `unstageFiles()`
  - [ ] `commit()` — execute git commit
  - [ ] `getBranchInfo()` — current branch, ahead/behind
  - [ ] `getCommitLog()` — recent commits
- [ ] Build diff parser (`src/main/git/diff-parser.ts`):
  - [ ] Parse unified diff → file-level hunks
  - [ ] Extract additions/deletions per file
  - [ ] Generate text summary for LLM consumption
- [ ] Build commit generator:
  - [ ] Prepare prompt: "Generate conventional commit for: {diff_summary}"
  - [ ] Run through process engine (deslopper strips verbose commits)
  - [ ] Present to user for editing → direct commit
- [ ] Build PR generator:
  - [ ] Get branch info + commit log + diff from base
  - [ ] Generate title + body → deslop → present → push to GitHub/GitLab API
- [ ] Build GitPanel (renderer):
  - [ ] File list with changed files + status icons
  - [ ] Diff view with syntax highlighting
  - [ ] "Generate commit" button → preview → edit → commit
  - [ ] "Generate PR" button → preview → push

### Key Decisions

- isomorphic-git is slower than native git but avoids native dependencies
- For performance-critical ops (large diffs), shell out to `git` CLI as fallback
- Diff parsing runs in main process, result sent over IPC
- Commit/PR generation uses the process engine = deslopper runs on commit messages (no "This commit aims to..." slop)

### Test: Open a real git repo, see dirty files, generate a commit message, commit it.

---

## 10. Phase 8: Doc Awareness

**Goal:** Fetch, cache, embed, and inject library documentation into prompts.

### Tasks

- [ ] Build `DocResolver`:
  - [ ] Read package.json from project
  - [ ] Resolve installed versions vs latest
  - [ ] Detect breaking changes (major version gaps)
- [ ] Build `DocFetcher`:
  - [ ] npm registry API → package info + documentation URL
  - [ ] unpkg/TypeScript definition fetcher
  - [ ] Fallback: GitHub README
  - [ ] User-configurable doc sources
- [ ] Build `DocIndexer`:
  - [ ] Parse fetched docs into structured chunks
  - [ ] Extract API signatures + descriptions
  - [ ] Store in `doc_cache` + `doc_fts` tables
  - [ ] Queue for embedding → store in `doc_embeddings`
- [ ] Build `DocInjector`:
  - [ ] On process start: look at project dependencies
  - [ ] Query doc_cache for each dependency
  - [ ] Build context snippet: version + API signatures + breaking changes
  - [ ] Append to system prompt
- [ ] Build `DocBrowser` (renderer):
  - [ ] List cached libraries + versions
  - [ ] Search via FTS5 (keyword) or vec0 (semantic)
  - [ ] Browse docs inline

### Key Decisions

- Docs are cached per-library per-version. Prisma 6 docs != Prisma 7 docs.
- Embedding is async — user gets doc injection without embeddings immediately, embeddings fill in over time
- Breaking changes get a prominent warning block in the prompt
- User can manually trigger doc refresh per library

### Test: Open a project with prisma@7 → see "Using prisma@7.1.0" injected → generated code uses v7 API patterns.

---

## 11. Phase 9: Agent System

**Goal:** Agent config, sub-agent management, and runtime delegation.

### Tasks

- [ ] Define agent types (`src/shared/agent-types.ts`)
- [ ] Build AgentManager (`src/main/agents/manager.ts`):
  - [ ] CRUD operations
  - [ ] Preset loading (from `src/presets/agents.json`)
  - [ ] Save/load user agents
- [ ] Build SubAgentRuntime (`src/main/agents/runtime.ts`):
  - [ ] `delegate()` — spawn child process with filtered context
  - [ ] `delegateParallel()` — multiple children concurrently
  - [ ] Context propagation (what parent passes to child)
  - [ ] Sub-agent timeout + retry
  - [ ] Result merging back to parent
- [ ] Write built-in agent presets:
  - [ ] Orchestrator, Code Writer, Reviewer, Test Writer, Doc Lookup
  - [ ] Full Stack Builder, Pair Programmer, TDD Flow, etc.
- [ ] Build AgentConfigView (renderer):
  - [ ] Profile selector
  - [ ] Capability checkboxes
  - [ ] Sub-agent toggles (enable/disable per child)
  - [ ] Resource access editor
  - [ ] Training button
- [ ] Test: Configure an orchestrator agent, give it a complex task, watch sub-agents fire

### Key Decisions

- Each agent has its own deslopper rules (inherits from profile, can override)
- Sub-agents get truncated context + filtered tools from parent
- Parallel delegation respects `maxConcurrentSubAgents`
- Agent config is stored as JSON alongside profiles
- Agents can be marked `builtIn` (read-only presets) — user must fork to edit

### Test: `test/e2e:agents` — Orchestrator with 3 sub-agents generates a feature end-to-end.

---

## 12. Phase 10: Training UI

**Goal:** Interactive feedback loop for prompt evolution.

### Tasks

- [ ] Build `FeedbackAnalyzer` (`src/main/training/analyzer.ts`):
  - [ ] Diff output vs user correction
  - [ ] Cluster edits by type
  - [ ] Generate structured analysis
- [ ] Build `PromptEvolver` (`src/main/profiles/evolver.ts`):
  - [ ] Translate analysis into prompt changes
  - [ ] Update system prompt, few-shots, deslopper weights
  - [ ] Version the profile
- [ ] Build training logger:
  - [ ] Append feedback examples to training log
  - [ ] Store in JSONL format for easy export
- [ ] Build TrainingView (renderer):
  - [ ] Show original output + deslopped output + user correction side-by-side
  - [ ] Rating sliders (clarity, conciseness, accuracy, style, slop)
  - [ ] Free-text notes field
  - [ ] "Apply corrections to profile" button
- [ ] Build profile diff viewer:
  - [ ] Show before/after of evolved prompt
  - [ ] Show what changed
  - [ ] Allow rollback to previous version
- [ ] Test: Run code gen 5 times, provide feedback each time → verify prompts evolve

### Key Decisions

- Training operates on the **agent level**, not profile level (multiple agents can share a profile without interfering)
- Evolution is deterministic — no LLM involved in the evolver, just pattern matching + diff analysis
- User can always roll back to any previous version
- Training logs are JSONL — easy to export, analyze, or use for actual fine-tuning later

### Test: `test/manual:training` — 3 cycles of code gen + feedback → verify evolved prompts produce less slop.

---

## 13. Phase 11: Polish & Ship

**Goal:** Production-ready app with packaging, tests, and documentation.

### Tasks

- [ ] End-to-end tests for all flows:
  - [ ] Chat → deslop → present
  - [ ] Code gen → review → loop → commit
  - [ ] Agent orchestration with sub-agents
  - [ ] Doc awareness + version injection
  - [ ] Profile training → evolution
- [ ] Error handling pass:
  - [ ] Network errors (API unavailable)
  - [ ] Rate limiting (429 handling)
  - [ ] Invalid API keys
  - [ ] OpenRouter-style fallback
  - [ ] Process crashes → recover state
- [ ] Packaging:
  - [ ] electron-builder config for Windows, macOS, Linux
  - [ ] Auto-update (electron-updater)
  - [ ] Code signing (notarization for macOS + Windows)
- [ ] Documentation:
  - [ ] README with quick start
  - [ ] User guide (how to configure providers, agents, profiles)
  - [ ] Developer guide (how to contribute)
  - [ ] API documentation
- [ ] CI/CD:
  - [ ] GitHub Actions: lint + test on PR
  - [ ] GitHub Actions: build + release on tag
  - [ ] Prettier + ESLint config
- [ ] Performance:
  - [ ] Profile startup time
  - [ ] Cache warm-up on first launch
  - [ ] Lazy load providers
  - [ ] Virtual list for long chats

---

## 14. Infrastructure Decisions

### Testing Strategy

```
test/
├── unit/                  # Pure logic tests (no Electron)
│   ├── deslopper.test.ts
│   ├── profiles.test.ts
│   ├── cache.test.ts
│   ├── diff-parser.test.ts
│   └── prompt-builder.test.ts
├── integration/           # Tests with Electron runtime
│   ├── process-engine.test.ts
│   ├── provider.test.ts
│   └── ipc.test.ts
└── e2e/                   # Full app tests with Playwright
    ├── chat.spec.ts
    └── git.spec.ts
```

- Vitest for unit tests (fast, native TypeScript, compatible with Vite)
- Playwright for e2e (spectron is deprecated, Playwright Electron works)
- No Jest — Vitest does everything Jest does with better DX

### Directory Structure Recap

```
harness/
├── src/
│   ├── main/              # Electron main process
│   │   ├── deslopper/
│   │   ├── providers/
│   │   ├── agents/
│   │   ├── git/
│   │   ├── docs/
│   │   ├── profiles/
│   │   ├── training/
│   │   ├── cache/
│   │   ├── process-engine.ts
│   │   └── ipc-handlers.ts
│   ├── renderer/          # React frontend
│   │   ├── components/
│   │   ├── stores/
│   │   └── hooks/
│   └── shared/            # Types shared between both
│       ├── ipc.ts
│       └── types.ts
├── electron/
│   ├── main.ts
│   └── preload.ts
├── test/
├── presets/               # Shipped JSON presets
└── user-config/           # Created at runtime, gitignored
```

### Dependency Priority

| Phase | Dependencies to add |
|-------|--------------------|
| 0 | electron, vite, react, tailwindcss, typescript |
| 1 | (none — pure types) |
| 2 | openai, @anthropic-ai/sdk (optional), @google/generative-ai (optional) |
| 3 | (none — pure Node.js) |
| 4 | better-sqlite3, sqlite-vec |
| 5 | (none — uses shared types) |
| 6 | zustand, lucide-react, react-router, codemirror, remark |
| 7 | isomorphic-git |
| 8 | (none — uses native fetch) |
| 9 | (none — uses existing systems) |
| 10 | (none — uses existing systems) |
| 11 | electron-builder, electron-updater |


## Summary: Build Phases

```
Phase 0:  Scaffold           ─── "it opens a window"            [1–2 days]
Phase 1:  Core types         ─── "shared types work"            [2–3 days]
Phase 2:  Provider layer     ─── "it calls OpenAI"              [3–4 days]
Phase 3:  Deslopper          ─── "it catches slop"              [3–4 days]
Phase 4:  Cache system       ─── "it caches things"             [2–3 days]
Phase 5:  Process engine     ─── "processes run and stream"     [4–5 days]
Phase 6:  UI shell + chat    ─── "I can talk to an LLM"         [4–5 days]
Phase 7:  Git integration    ─── "I can commit from the app"    [3–4 days]
Phase 8:  Doc awareness      ─── "it knows Prisma 7 exists"     [4–5 days]
Phase 9:  Agent system       ─── "sub-agents work"              [4–5 days]
Phase 10: Training UI        ─── "I can train my agent"         [3–4 days]
Phase 11: Polish & ship      ─── "I can ship it"                 [5–7 days]
                              ─────────────────────────────────
                              Total: ~35–50 days of focused work
```
