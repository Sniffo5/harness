# Harness Architecture Document

> An AI agent harness for coding, review, autonomous agents, and more.
> Open-source Electron + React + TypeScript application.

---

## Table of Contents

1. [Core Principles](#1-core-principles)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Process System](#3-process-system)
4. [Deslopper System](#4-deslopper-system)
5. [Training / Fine-Tuning System](#5-training--fine-tuning-system)
6. [Profile System](#6-profile-system)
7. [Documentation Awareness System](#7-documentation-awareness-system)
8. [Provider Abstraction Layer](#8-provider-abstraction-layer)
9. [Git Integration](#9-git-integration)
10. [UI Architecture](#10-ui-architecture)
11. [Agent System (Sub-Agents)](#11-agent-system-sub-agents)
12. [Directory Structure](#12-directory-structure)
13. [Process Flows](#13-process-flows)
14. [What To Build First](#14-what-to-build-first)

---

## 1. Core Principles

- **Everything is a process.** Chat, review, code generation, autonomous agents — all run through a unified process runtime.
- **Deslopping is mandatory.** Every model output passes through the deslopper. No skips.
- **Training is prompt evolution.** Not real training — the system iteratively refines system prompts + few-shot examples based on user feedback.
- **Provider-agnostic.** User brings their own API key or local model. Abstraction layer between app and LLM.
- **Electron main process handles I/O and system access.** Renderer handles UI. IPC between them is typed.
- **Open-source means no secrets in the code.** Everything user-specific is in a gitignored config directory.

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Electron Shell                        │
│  ┌──────────────────────┐    ┌────────────────────────┐ │
│  │    Main Process       │    │   Renderer Process     │ │
│  │  (Node.js backend)    │◄──►│  (React + Tailwind)    │ │
│  │                       │ IPC │                        │ │
│  │  - Process Engine     │    │  - Chat Interface      │ │
│  │  - LLM Provider Layer │    │  - Process Manager UI  │ │
│  │  - Deslopper Engine   │    │  - Profile Editor      │ │
│  │  - Git Integration    │    │  - Training UI         │ │
│  │  - File System Access │    │  - Git Diff Viewer     │ │
│  │  - Documentation      │    │  - Settings            │ │
│  │    Fetch Service      │    │                        │ │
│  │  - Profile Store      │    │                        │ │
│  └──────────────────────┘    └────────────────────────┘ │
│                          │                               │
│                    ┌─────┴──────┐                        │
│                    │  Context   │                        │
│                    │  Isolator  │                        │
│                    │ (Sandbox)  │                        │
│                    └────────────┘                        │
└─────────────────────────────────────────────────────────┘
```

### 2.1 Main Process Responsibilities

| Module | Responsibility |
|--------|---------------|
| **Process Engine** | Orchestrates all agent runs. Handles lifecycle (init → prompt → call → deslop → evaluate → loop). |
| **LLM Provider Layer** | Abstracts OpenAI, Anthropic, Google, Ollama, vLLM, etc. |
| **Deslopper Engine** | Run deslopping rules against all outputs. Pluggable rule sets. |
| **Git Integration** | Diff parsing, commit generation, PR creation via hosted git APIs or local git. |
| **File System Access** | Read/write project files, manage workspace. |
| **Doc Service** | Fetch and cache library documentation (npm registry, typedoc, MDN, etc.). |
| **Profile Store** | Load/save profiles and trained prompt configurations. |
| **Context Isolator** | Sandbox for running agent-generated code safely (optional). |

### 2.2 Renderer Process Responsibilities

| Module | Responsibility |
|--------|---------------|
| **Chat UI** | Multi-threaded chat interface (one per process). |
| **Process Manager** | Start, stop, monitor, and replay processes. |
| **Profile Editor** | Create/edit/train profiles. |
| **Training UI** | Interactive feedback loop for prompt refinement. |
| **Git Panel** | Diff viewer, commit composer, PR preview. |
| **Settings** | Provider config, API keys, deslopper rules, theme. |
| **Doc Browser** | Browse cached documentation inline. |

### 2.3 IPC Contract

All IPC is fully typed with a shared schema:

```typescript
// shared/ipc.ts
export interface IpcContract {
  // LLM
  'llm:complete': { provider: string; model: string; messages: Message[]; profile: Profile }
  'llm:stream':   { provider: string; model: string; messages: Message[]; profile: Profile }
  'llm:abort':    { requestId: string }

  // Process
  'process:start':   { type: ProcessType; config: ProcessConfig }
  'process:stop':    { processId: string }
  'process:status':  { processId: string }  // -> ProcessStatus

  // Git
  'git:diff':        { repoPath: string; staged?: boolean }
  'git:commit':      { repoPath: string; message: string }
  'git:pr':          { repoPath: string; title: string; body: string; branch: string }

  // Profiles
  'profiles:list':   {}  // -> Profile[]
  'profiles:save':   { profile: Profile }
  'profiles:train':  { profileId: string; examples: FeedbackExample[] }

  // Docs
  'docs:lookup':     { library: string; version?: string; query: string }
  'docs:cache':      { library: string; version: string }
}
```

---

## 3. Process System

### 3.1 Process Types

```
Process
├── Chat               ← Interactive conversation with an agent
├── Code Generation    ← Generate code from a spec
├── Code Review        ← Review existing code or diff
├── Autonomous Task    ← Multi-step agent with tool use
├── UI Generation      ← Generate + visually evaluate UI
├── Commit Generation  ← Generate commit messages from diff
├── PR Generation      ← Generate PR descriptions from branch
└── Doc Query          ← Query library documentation
```

### 3.2 Process Lifecycle

```
                  ┌──────────┐
                  │  INIT    │
                  └────┬─────┘
                       │
                  ┌────▼─────┐
                  │ PREPARE  │ ← Load profile, gather context, fetch docs
                  └────┬─────┘
                       │
                  ┌────▼─────┐
                  │ PROMPT   │ ← Build the full prompt (system + context + user)
                  └────┬─────┘
                       │
                  ┌────▼─────┐
             ┌────► CALL LLM │ ← Send to provider, stream response
             │    └────┬─────┘
             │         │
             │    ┌────▼──────┐
             │    │ DESLOP    │ ← Run deslopper on raw output
             │    └────┬──────┘
             │         │
             │    ┌────▼────────┐
             │    │ EVALUATE    │ ← Check quality, apply rules
             │    └────┬────────┘
             │         │
             │    ┌────▼──────┐
             │    │ FEEDBACK? │ ← User can provide feedback here
             │    └────┬──────┘
             │    pass │   │ need improvement
             │         │   │
             │     ┌───▼┐  │ (loop: refine prompt with feedback)
             │     │DONE│  │
             │     └────┘  │
             └─────────────┘
```

### 3.3 Process Engine Core

```typescript
// main/process-engine.ts
class ProcessEngine {
  private processes: Map<string, ProcessInstance>;

  async start(config: ProcessConfig): Promise<string> {
    const process = this.createProcess(config.type, config);
    this.processes.set(process.id, process);

    while (process.status !== 'completed' && process.status !== 'failed') {
      const prompt = this.promptBuilder.build(process);
      const rawOutput = await this.provider.call(prompt, process.profile);
      // Run through context isolator if needed
      const sanitized = await this.sanitize(rawOutput, config.sandbox);
      const deslopped = await this.deslopper.run(sanitized, process.profile.deslopperRules);
      const evaluation = await this.evaluate(deslopped, process);

      if (evaluation.passed) {
        process.output = deslopped;
        process.status = 'completed';
      } else if (evaluation.shouldRetry) {
        process.history.push({ prompt, output: deslopped, evaluation });
        // Feedback loop: refine prompt with the evaluation
        process.status = 'iterating';
      } else {
        process.status = 'failed';
        process.error = evaluation.reason;
      }
    }

    return process.id;
  }
}
```

---

## 4. Deslopper System

### 4.1 What Is Slop?

Content that exhibits telltale AI patterns:
- Overuse of hedging ("I believe", "in my opinion", "it's important to note")
- Excessive bullet points in prose
- Generic platitudes ("leverage", "delve", "in today's digital landscape")
- Hallucinated confidence on things the model doesn't know
- Verbose restatement of the obvious
- "I'd be happy to help!" energy

### 4.2 Architecture

```
Deslopper
├── Parser           ← Tokenize and structure the output
├── Rule Engine      ← Run rules against parsed output
│   ├── Pattern Rules    ← Regex/pattern-based matching
│   ├── Style Rules      ← Tone, verbosity, hedging
│   ├── Signal Rules     ← Model-specific tell markers
│   └── Custom Rules     ← User-defined rules
├── Scorer           ← Produce a slop score (0-100)
├── Rewriter         ← Optionally rewrite to remove slop
└── Reporter         ← Tell the user what was removed/changed
```

### 4.3 Rule Definition Format

```typescript
// shared/deslopper/rule.ts
interface DeslopperRule {
  id: string;
  name: string;
  category: 'pattern' | 'style' | 'signal' | 'custom' | 'tone';
  severity: 'suggestion' | 'warning' | 'block';
  patterns: string[];          // regex patterns
  contextPatterns?: string[];  // only flag if these also match nearby
  remediation?: 'strip' | 'rewrite' | 'flag' | 'block';
  rewriteTemplate?: string;    // for 'rewrite' remediation
}

interface DeslopperConfig {
  rules: DeslopperRule[];
  thresholds: {
    flag: number;    // score at which to warn user
    block: number;   // score at which to demand rewrite
    autoRewrite: number; // score at which to auto-rewrite
  };
  modes: {
    passive: boolean;   // just flag, let user decide
    active: boolean;    // rewrite automatically
  };
}
```

### 4.4 Built-in Slop Patterns

| Pattern | Example | Remediation |
|---------|---------|-------------|
| Hedging | "I think", "perhaps", "it seems" | Strip or flag |
| Gratitude | "Great question!", "I'd be happy to help" | Strip |
| Pompous verbs | "leverage", "utilize", "delve" | Rewrite to simpler form |
| Redundant phrases | "In order to" → "To" | Rewrite |
| False confidence | "This is definitely the best approach" | Flag |
| Over-bulleting | 5+ consecutive bullet points in prose | Flag |
| Verbose signposting | "First, let me explain...", "Now that we've covered..." | Strip |
| Generic compliments | "That's an excellent idea", "Interesting perspective" | Flag |

### 4.5 Deslopper Engine Pipeline

```
Raw Output
    │
    ▼
[Tokenizer] ──► [Sentence Splitter] ──► [Pattern Matcher]
                                              │
    ┌─────────────────────────────────────────┘
    ▼
[Score Calculator] ──► [Threshold Check]
                              │
                    ┌─────────┴────────┐
                    │                  │
               Below flag         Above flag
                    │                  │
               [PASS]          ┌───────┴───────┐
                               │               │
                          Auto-rewrite?   Flag for user
                               │               │
                          [Rewrite]      [User Decision]
```

---

## 5. Training / Fine-Tuning System

### 5.1 What "Training" Really Means

This is not model fine-tuning. It's **prompt evolution** — the system iteratively refines the system prompt, few-shot examples, instructions, and deslopper rules based on user feedback.

### 5.2 Feedback Cycle

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│ User sees    │────►│ User provides│────►│ System       │
│ output       │     │ feedback     │     │ analyzes     │
└─────────────┘     └──────────────┘     │ patterns     │
                                          └──────┬───────┘
                                                 │
                                          ┌──────▼───────┐
                                          │ Refines      │
                                          │ Profile      │
                                          └──────┬───────┘
                                                 │
                                          ┌──────▼───────┐
                                          │ New outputs   │
                                          │ use updated   │
                                          │ profile       │
                                          └──────────────┘
```

### 5.3 Feedback Types

```typescript
// shared/training/types.ts
interface FeedbackExample {
  id: string;
  input: string;             // What the user asked
  output: string;            // What the model produced (pre-deslop)
  deslopped: string;         // What it looked like after deslopping
  userCorrection: string;    // The user's rewritten/edited version
  userNotes: string;         // Free-text notes from user
  ratings: {
    clarity: number;         // 1-5
    conciseness: number;     // 1-5
    accuracy: number;        // 1-5
    style: number;           // 1-5
    slopLevel: number;       // 1-5 (5 = too much slop)
  };
  timestamp: number;
}
```

### 5.4 Prompt Evolution Engine

```typescript
// main/training/evolver.ts
class PromptEvolver {
  /**
   * Take a set of feedback examples and produce refinements to:
   * 1. System prompt instructions
   * 2. Few-shot examples (add/remove/order)
   * 3. Deslopper rule weights
   * 4. Output format preferences
   */
  evolve(profile: Profile, examples: FeedbackExample[]): ProfileDelta {
    const analysis = this.analyzeFeedback(examples);

    return {
      systemPromptChanges: this.derivePromptChanges(analysis),
      fewShotExamples: this.curateExamples(analysis, profile),
      deslopperAdjustments: this.adjustRules(analysis),
      stylePreferences: this.extractPreferences(analysis),
    };
  }

  /**
   * Analysis identifies patterns:
   * - "User always shortens code comments" → add "minimal comments" to prompt
   * - "User removes bullet lists" → add "avoid bullet lists" + deslopper rule
   * - "User rewrites async patterns" → add few-shot with async patterns
   */
  private analyzeFeedback(examples: FeedbackExample[]): FeedbackAnalysis {
    // Compare output vs userCorrection to find consistent edits
    // Use diff algorithm to identify patterns of changes
    // Cluster similar edits into categories
    // Return structured analysis
  }
}
```

### 5.5 Profile Structure

```typescript
// shared/profile/types.ts
interface Profile {
  id: string;
  name: string;
  description: string;
  domain: 'code' | 'text' | 'ui' | 'review' | 'general';

  // Core prompt
  systemPrompt: string;
  userPromptTemplate?: string;

  // Few-shot examples
  fewShotExamples: FewShotExample[];

  // Deslopper configuration (profile-specific overrides)
  deslopperConfig: {
    enabled: boolean;
    rules: string[];              // Rule IDs to apply
    ruleOverrides: Record<string, Partial<DeslopperRule>>;
    thresholds?: Partial<DeslopperConfig['thresholds']>;
  };

  // Provider preferences
  preferredProvider: string;
  preferredModel: string;
  modelParameters: {
    temperature: number;
    maxTokens: number;
    topP: number;
  };

  // Training history
  trainingHistory: TrainingSession[];

  // Versioning
  version: number;
  parentId?: string;             // For forking profiles
  createdAt: number;
  updatedAt: number;
}
```

### 5.6 Built-in Profiles

| Profile | Domain | Focus |
|---------|--------|-------|
| **Sniffo Code** | code | Anti-slop code generation. Concise, pragmatic, no unnecessary comments, no ceremony. |
| **Sniffo UI** | ui | Generate clean React components. Prefers shadcn/ui patterns, tailwind, accessible. |
| **Sniffo Text** | text | Direct, factual, no filler. Technical writing style. |
| **Sniffo Review** | review | Thorough but concise code review. Calls out actual issues, skips nitpicks. |
| **Clean Code** | code | Classic clean code patterns. Slightly more verbose. |
| **Minimal** | general | Maximum conciseness. For quick answers. |

---

## 6. Profile System

### 6.1 Storage

```
%APP_DATA%/harness/
├── config.json              # App settings, provider keys
├── profiles/
│   ├── built-in/            # Read-only, shipped with app
│   │   ├── sniffo-code.json
│   │   ├── sniffo-ui.json
│   │   ├── sniffo-text.json
│   │   ├── sniffo-review.json
│   │   ├── clean-code.json
│   │   └── minimal.json
│   └── user/                # User-created and trained profiles
│       ├── my-profile.json
│       └── team-review.json
├── training-logs/           # Raw feedback data for analysis
│   └── 2026-05-07.jsonl
└── doc-cache/               # Cached documentation
    ├── react@19.0.0.json
    ├── prisma@7.0.0.json
    └── next@15.0.0.json
```

### 6.2 Profile Inheritance

```typescript
interface ProfileInheritance {
  extends?: string;       // Profile ID this is based on
  overrides: {
    systemPrompt?: string;        // Full override (no inheritance)
    systemPromptAppend?: string;  // Append to parent
    deslopperRules?: 'inherit' | 'replace' | 'extend';
    fewShotExamples?: 'inherit' | 'replace' | 'extend';
  };
}
```

### 6.3 Profile Manager

```typescript
// main/profiles/manager.ts
class ProfileManager {
  list(): Profile[];
  get(id: string): Profile;
  create(name: string, base?: string): Profile;
  save(profile: Profile): void;
  delete(id: string): void;
  fork(id: string, newName: string): Profile;
  resetToBuiltIn(id: string): Profile;
  export(id: string): string;   // JSON string for sharing
  import(json: string): Profile;
}
```

---

## 7. Documentation Awareness System

### 7.1 Problem

LLMs hallucinate API signatures from outdated training data. When you ask for Prisma 7 code, it might give you Prisma 5 patterns.

### 7.2 Solution

A documentation caching system that:
1. Detects libraries in the user's project (package.json, imports)
2. Looks up their installed version
3. Fetches and caches the relevant API docs
4. Injects relevant documentation into the system prompt
5. Cites the version it's using

### 7.3 Architecture

```
┌────────────────────────────────────┐
│           Doc Service              │
│                                    │
│  ┌──────────┐   ┌──────────────┐   │
│  │ Resolver │   │ Cache (SQLite)│   │
│  └────┬─────┘   └──────┬───────┘   │
│       │                │           │
│  ┌────▼─────┐          │           │
│  │ Fetcher  │          │           │
│  └────┬─────┘          │           │
│       │                │           │
│  ┌────▼─────┐          │           │
│  │ Indexer  │          │           │
│  └────┬─────┘          │           │
│       │                │           │
│  ┌────▼─────┐          │           │
│  │ Injector │          │           │
│  └──────────┘          │           │
└────────────────────────────────────┘
```

### 7.4 Doc Sources

| Source | How | Libraries |
|--------|-----|-----------|
| **npm registry** | Fetch package metadata → find documentation URL | Node.js packages |
| **unpkg / jsdelivr** | Fetch TypeScript definitions directly | NPM packages |
| **TypeDoc / API Extractor** | Generate structured API docs | TypeScript libraries |
| **MDN** | Web API documentation | DOM, Web APIs |
| **GitHub README** | Fallback | General |
| **Custom URL** | User-configured docs | Project-specific |

### 7.5 Flow

```
User: "Create a Prisma 7 schema for users and posts"

1. Doc service checks: does user's package.json have prisma?
2. Resolves installed version: prisma@7.1.0
3. Cache hit? No → Fetch docs
4. Fetcher: npm registry → package info → fetch TypeScript defs
5. Indexer: Parse TSDoc comments → extract API signatures + descriptions
6. Injector: Build context string:

```
Using prisma@7.1.0 documentation:
- PrismaClient: https://www.prisma.io/docs/orm/prisma-client
- Model operations: create(), findMany(), update(), delete()
- v7 breaking changes: [relevant v6→v7 changes]
```

7. Append to system prompt for the current process
8. LLM generates code using correct v7 API signatures
```

### 7.6 Version Awareness Strategy

```typescript
// main/docs/version-aware.ts
interface VersionAwareness {
  library: string;
  installed: string;            // From package.json
  latest: string;               // From npm registry
  hasBreakingChanges: boolean;   // Major version difference
  breakingChanges: string[];     // Key differences to highlight
  docInjection: string;         // The doc snippet to inject
}

class VersionAwarenessService {
  async getContext(projectPath: string): Promise<VersionAwareness[]> {
    const manifest = await this.readPackageJson(projectPath);
    const libraries = this.extractDependencies(manifest);

    const contexts = await Promise.all(
      libraries.map(lib => this.resolveLibrary(lib))
    );

    return contexts.filter(c => c !== null);
  }

  private async resolveLibrary(lib: Dependency): Promise<VersionAwareness | null> {
    const installed = lib.version;
    const latest = await this.fetchLatestVersion(lib.name);

    return {
      library: lib.name,
      installed,
      latest,
      hasBreakingChanges: semver.major(installed) < semver.major(latest),
      breakingChanges: await this.findBreakingChanges(lib.name, installed, latest),
      docInjection: await this.buildInjection(lib.name, installed),
    };
  }
}
```

---

## 8. Provider Abstraction Layer

### 8.1 Architecture

```
┌──────────────────────────────────────────────────┐
│               Provider Abstraction               │
│  ┌──────────────┐  ┌──────────────┐  ┌────────┐ │
│  │  LLMService  │  │  VisionService│  │ Embed- │ │
│  │              │  │              │  │ Service│ │
│  └──────┬───────┘  └──────┬───────┘  └───┬────┘ │
│         │                 │               │      │
└─────────┼─────────────────┼───────────────┼──────┘
          │                 │               │
    ┌─────▼─────────────────▼───────────────▼──┐
    │           Provider Adapters              │
    │  ┌────────┐ ┌────────┐ ┌──────┐ ┌─────┐ │
    │  │OpenAI  │ │Anthropic│ │Google│ │Other│ │
    │  └────────┘ └────────┘ └──────┘ └─────┘ │
    │  ┌────────┐ ┌────────┐ ┌──────────┐     │
    │  │ Ollama  │ │ vLLM   │ │ Together │    │
    │  └────────┘ └────────┘ └──────────┘     │
    └──────────────────────────────────────────┘
```

### 8.2 Provider Adapter Interface

```typescript
// main/providers/adapter.ts
interface ModelCapabilities {
  streaming: boolean;
  vision: boolean;
  maxTokens: number;
  contextWindow: number;
  toolUse: boolean;
  embedding: boolean;
  jsonMode: boolean;
}

interface ProviderAdapter {
  id: string;
  name: string;

  complete(req: CompletionRequest): Promise<CompletionResponse>;
  stream(req: CompletionRequest): AsyncIterable<StreamChunk>;
  abort(requestId: string): void;
  listModels(): Promise<ModelInfo[]>;

  // Vision support
  analyzeImage?(req: VisionRequest): Promise<VisionResponse>;

  // Embedding support
  embed?(req: EmbeddingRequest): Promise<EmbeddingResponse>;

  getCapabilities(model: string): ModelCapabilities;
}

interface CompletionRequest {
  model: string;
  messages: Message[];
  systemPrompt?: string;
  temperature?: number;
  maxTokens?: number;
  tools?: Tool[];
  profile?: Profile;
}

interface CompletionResponse {
  id: string;
  content: string;
  finishReason: 'stop' | 'length' | 'tool_calls' | 'error';
  usage: {
    inputTokens: number;
    outputTokens: number;
  };
  model: string;
  deslopped?: boolean;        // Was deslopping applied?
  deslopReport?: DeslopReport; // What was changed
}
```

### 8.3 Provider Configuration

```typescript
// shared/settings/provider-config.ts
interface ProviderConfig {
  id: string;
  name: string;
  enabled: boolean;
  apiKey?: string;              // Encrypted at rest
  apiKeyLocation?: 'env' | 'keychain' | 'file';
  baseUrl?: string;             // For self-hosted / proxies
  models: string[];             // Available models
  defaultModel: string;
  priority: number;             // Which provider to try first
}
```

---

## 9. Git Integration

### 9.1 Capabilities

- **Diff parsing:** Parse `git diff` output into structured hunks
- **Commit generation:** From diff → LLM generates commit message → deslop → user approves/edits
- **PR/MR generation:** From branch → diff → LLM generates title + body → deslop → user edits → push
- **Diff visualization:** In-app diff viewer with syntax highlighting
- **Staging management:** Stage/unstage files from within the app
- **Conflict resolution:** LLM-assisted merge conflict resolution

### 9.2 Architecture

```
┌──────────────────────────┐
│     GitService           │
│  ┌────────────────────┐  │
│  │ isogit (pure JS)  │  │  ← Works in Electron main process
│  │ OR nodegit       │  │  ← Native bindings (faster)
│  └────────────────────┘  │
│                          │
│  ┌────────────────────┐  │
│  │ Diff Parser         │  │
│  │ Commit Generator   │  │
│  │ PR Generator       │  │
│  │ Conflict Helper    │  │
│  └────────────────────┘  │
└──────────────────────────┘
```

### 9.3 Git Service API

```typescript
// main/git/service.ts
interface GitService {
  getDiff(repoPath: string, staged?: boolean, ref?: string): Promise<DiffResult>;
  stageFiles(repoPath: string, files: string[]): Promise<void>;
  unstageFiles(repoPath: string, files: string[]): Promise<void>;
  commit(repoPath: string, message: string): Promise<CommitResult>;
  getBranchInfo(repoPath: string): Promise<BranchInfo>;
  createPR(repoPath: string, remote: string, title: string, body: string, head: string, base: string): Promise<PRResult>;
  getCommitLog(repoPath: string, count?: number): Promise<Commit[]>;
  resolveConflict(repoPath: string, file: string): Promise<ConflictInfo>;
}

interface DiffResult {
  files: DiffFile[];
  stats: {
    additions: number;
    deletions: number;
    filesChanged: number;
  };
  summary: string;  // Textual summary for LLM
}

interface DiffFile {
  path: string;
  status: 'added' | 'modified' | 'deleted' | 'renamed';
  hunks: Hunk[];
  oldContent?: string;
  newContent?: string;
}
```

### 9.4 Commit Generation Flow

```
User makes changes in their IDE
         │
         ▼
Switch to Harness → sees dirty files
         │
         ▼
Reviews diff in Harness diff viewer
         │
         ▼
Clicks "Generate Commit Message"
         │
         ▼
GitService.getDiff() → structured diff
         │
         ▼
Process Engine: Commit Generation
  - Prepares prompt: "Generate a conventional commit message for this diff"
  - Profile: Sniffo Review (concise, semantic)
  - Doc injection: Conventional Commits spec
  - CALL LLM → response
  - DESLOP → remove verbose commit messages
  - EVALUATE → passes?
         │
         ▼
User sees generated message, can edit
         │
         ▼
User clicks "Commit" → git commit -m "..."
```

---

## 10. UI Architecture

### 10.1 Tech Stack

- **Framework:** React 19 + TypeScript 5.x
- **Styling:** Tailwind CSS v4
- **UI Library:** shadcn/ui (or custom component library)
- **State:** Zustand (lightweight, works well with Electron IPC)
- **Routing:** React Router (or wouter for simplicity)
- **Diff Viewer:** Monaco Editor (read-only mode) or react-diff-viewer
- **Rich Text:** CodeMirror or Monaco for editor panes
- **IPC Bridge:** Custom typed IPC hook wrapping contextBridge/preload

### 10.2 Component Tree (Simplified)

```
<App>
  <Layout>
    <Sidebar>
      <ProcessList />         ← Running/completed processes
      <ProfileList />         ← Profile selector
      <DocBrowser />          ← Documentation browser
    </Sidebar>
    <MainPanel>
      <Router>
        <ChatView />          ← Active process chat
        <ProcessView />       ← Full process dashboard
        <TrainingView />      ← Feedback/training panel
        <GitView>
          <DiffPanel />       ← Diff visualization
          <CommitPanel />     ← Commit composer
          <PrPanel />         ← PR generator
        </GitView>
        <SettingsPage>
          <ProviderSettings />
          <ProfileSettings />
          <DeslopperSettings />
        </SettingsPage>
      </Router>
    </MainPanel>
    <DeslopPanel>             ← Folding panel showing deslopper activity
      <DeslopReport />        ← What was flagged/rewritten
    </DeslopPanel>
    <StatusBar>               ← Process status, git branch, etc
    </StatusBar>
  </Layout>
</App>
```

### 10.3 Layout Concept

```
┌──────────────────────────────────────────────────────┐
│  [Process] [Profiles] [Docs] [Git] [Settings]        │  ← Tabs
├──────┬──────────────────────────────────────┬────────┤
│      │                                      │        │
│      │                                      │ Deslop │
│  S   │           Main Content               │ Panel  │
│  I   │                                      │ ────── │
│  D   │                                      │ "Delve"│
│  E   │                                      │ flagged│
│  B   │                                      │        │
│  A   │                                      │        │
│  R   │                                      │        │
│      │                                      │        │
└──────┴──────────────────────────────────────┴────────┘
└── Status Bar ── Running │ git/main │ 3 unsaved ──────┘
```

### 10.4 Preload / IPC Bridge

```typescript
// electron/preload.ts
import { contextBridge, ipcRenderer } from 'electron';
import { IpcContract } from '../shared/ipc';

const api = {
  invoke: <K extends keyof IpcContract>(
    channel: K,
    ...args: Parameters<IpcContract[K]>
  ): Promise<ReturnType<IpcContract[K]>> =>
    ipcRenderer.invoke(channel, ...args),

  on: <K extends keyof IpcContract>(
    channel: K,
    callback: (data: any) => void
  ): void => {
    ipcRenderer.on(channel, (_, data) => callback(data));
  },

  // Streaming support for LLM responses
  onStream: (requestId: string, callback: (chunk: string) => void) => {
    ipcRenderer.on(`stream:${requestId}`, (_, chunk) => callback(chunk));
  },
};

contextBridge.exposeInMainWorld('harness', api);
```

### 10.5 Zustand Store Pattern

```typescript
// renderer/stores/process-store.ts
import { create } from 'zustand';

interface ProcessStore {
  activeProcess: ProcessInstance | null;
  processes: Map<string, ProcessInstance>;
  history: ProcessHistory[];

  startProcess: (config: ProcessConfig) => Promise<string>;
  stopProcess: (id: string) => void;
  setActiveProcess: (id: string) => void;
  addStreamChunk: (processId: string, chunk: string) => void;
  provideFeedback: (processId: string, feedback: FeedbackExample) => void;
}

export const useProcessStore = create<ProcessStore>((set, get) => ({
  activeProcess: null,
  processes: new Map(),
  history: [],

  startProcess: async (config) => {
    const id = await window.harness.invoke('process:start', config);
    return id;
  },
  // ... implementations
}));
```

---

## 11. Agent System (Sub-Agents)

### 11.1 Concept

An **Agent** is an independently configured entity with its own profile, tool access, and instructions — spawned by the user or orchestrated by a parent agent (sub-agent mode).

```
User
 │
 ├── Agent A (direct, full access)
 ├── Agent B (code review, read-only)
 │
 └── Orchestrator Agent
       ├── Sub-Agent: Code Writer
       ├── Sub-Agent: Reviewer
       └── Sub-Agent: Test Writer
```

Sub-agents can be **toggled on/off**. When off, the orchestrator does the work itself. When on, it delegates to specialized sub-agents and aggregates results.

### 11.2 Agent Configuration

```typescript
// shared/agent/types.ts

interface Agent {
  id: string;
  name: string;
  description: string;

  // Identity
  profileId: string;               // Links to a Profile (system prompt + formatting)
  role: 'user' | 'assistant' | 'orchestrator' | 'sub-agent';

  // Prompt
  instructions: string;            // Additional instructions beyond the profile
  systemPromptOverride?: string;   // Optional: completely replace profile prompt
  systemPromptAppend?: string;     // Append to profile prompt

  // Sub-agent orchestration
  subAgents: AgentRef[];           // Children (spawned by this agent)
  allowSubAgents: boolean;        // Toggle sub-agent delegation on/off
  subAgentStrategy: 'auto' | 'manual' | 'parallel' | 'sequential';

  // Capabilities (granular toggle)
  capabilities: AgentCapabilities;

  // Tool access (what the agent can do)
  tools: ToolAssignment[];

  // Resource access (what the agent can see)
  resourceAccess: ResourceAccess;

  // Spawn configuration
  spawn: SpawnConfig;

  // Training
  training: AgentTraining;

  // Metadata
  tags: string[];
  version: number;
  createdAt: number;
  updatedAt: number;
  builtIn: boolean;                // Read-only preset
}

interface AgentRef {
  agentId: string;
  enabled: boolean;                // Toggle individual sub-agent
  delegationType: 'full' | 'review' | 'specific';
  delegationRules?: string;        // "Only delegate code generation, handle everything else yourself"
}

interface AgentCapabilities {
  canGenerateCode: boolean;
  canReviewCode: boolean;
  canReadFiles: boolean;
  canWriteFiles: boolean;
  canExecuteCommands: boolean;     // DANGEROUS — opt-in
  canAccessNetwork: boolean;
  canSpawnSubAgents: boolean;
  canUseBrowser: boolean;
  canQueryDocs: boolean;
  canAnalyzeImages: boolean;
}

interface ToolAssignment {
  toolId: string;
  enabled: boolean;
  config?: Record<string, unknown>; // Tool-specific settings
}

interface ResourceAccess {
  allowedDirectories: string[];    // Glob patterns, empty = no filesystem
  allowedExtensions: string[];     // File type filters
  maxFileSize: number;             // Bytes
  readOnly: boolean;               // Write prohibited
  allowedHosts: string[];          // Network access filters
}

interface SpawnConfig {
  maxConcurrentSubAgents: number;  // Limit parallelism
  timeoutMs: number;               // Kill after timeout
  retryOnFailure: number;          // Auto-retry count
  maxIterations: number;           // For autonomous loops
}

interface AgentTraining {
  enabled: boolean;
  feedbackExamples: FeedbackExample[];
  evolvedPrompt: string;           // Refined by training cycles
}
```

### 11.3 Built-in Agent Presets

| Preset | Role | Sub-agents | Capabilities | Best For |
|--------|------|------------|-------------|----------|
| **Orchestrator** | orchestrator | Code Writer, Reviewer, Test Writer, Doc Lookup | All | Full autonomous task |
| **Code Writer** | sub-agent | None | generateCode=true, readFiles=true, queryDocs=true | Single-file code gen |
| **Reviewer** | sub-agent | None | reviewCode=true, readFiles=true | Code review (read-only) |
| **Test Writer** | sub-agent | None | generateCode=true, readFiles=true, writeFiles=limited | Writing tests |
| **Doc Lookup** | sub-agent | None | queryDocs=true, readFiles=true | Fetching documentation |
| **Assistant** | assistant | Toggle orchestrator | Varies | Interactive help |
| **Chat** | user | None | queryDocs=true | Direct conversation |
| **Autonomous** | orchestrator | All sub-agents, auto-parallel | All | Unsolicited tasks |

### 11.4 Sub-Agent Orchestration Flows

#### 11.4.1 Sequential (default)

```
User asks: "Add Prisma schema for users, write the API, add tests"

Orchestrator receives task
  └─► Sub-agent 1 (Doc Lookup): "Get Prisma 7 schema syntax"
  └─► Sub-agent 2 (Code Writer): "Write schema.prisma"
  └─► [User reviews schema]
  └─► Sub-agent 3 (Code Writer): "Write API endpoints"
  └─► Sub-agent 4 (Reviewer): "Review API code"
  └─► [If review passes] Sub-agent 5 (Test Writer): "Write tests"
  └─► [If review fails] Loop back
  └─► Present final result to user
```

#### 11.4.2 Parallel

```
User asks: "Generate a React dashboard with all CRUD operations"

Orchestrator splits task
  ├─► Sub-agent Code Writer (left panel): "Write user CRUD components"
  ├─► Sub-agent Code Writer (right panel): "Write order CRUD components"
  ├─► Sub-agent Test Writer: "Write tests for both"
  └─► [All finish] → Reviewer merges and reviews
```

#### 11.4.3 Manual Delegation

```
User: "Review this PR"

Orchestrator: "I'll review it myself by default. Use /review to delegate to Reviewer agent."
User: "/review"
  └─► Orchestrator spawns Reviewer sub-agent with the diff
  └─► Reviewer returns review
  └─► Deslop → Present
```

### 11.5 Agent Manager

```typescript
// main/agents/manager.ts

class AgentManager {
  private agents: Map<string, Agent>;

  // CRUD
  list(): Agent[];
  get(id: string): Agent;
  create(config: Partial<Agent>): Agent;
  update(id: string, changes: Partial<Agent>): Agent;
  delete(id: string): void;

  // Presets
  loadPreset(name: string): Agent;
  saveAsPreset(agent: Agent, name: string): Agent;

  // Lifecycle
  spawn(agentId: string, task: string): Promise<AgentProcess>;
  spawnSubAgent(parentId: string, childId: string, task: string): Promise<AgentProcess>;
  stop(processId: string): void;

  // Training
  train(agentId: string, examples: FeedbackExample[]): Promise<Agent>;
  applyTraining(agentId: string, evolvedPrompt: string): void;

  // Import/Export
  export(agentId: string): string;
  import(json: string): Agent;
}
```

### 11.6 Sub-Agent Runtime

When an orchestrator spawns a sub-agent, a new mini-process is created:

```typescript
// main/agents/runtime.ts

class SubAgentRuntime {
  async delegate(
    parentProcess: ProcessInstance,
    childAgent: Agent,
    task: string,
    context: TaskContext
  ): Promise<TaskResult> {
    // 1. Build child prompt from agent's profile + instructions + task
    const childProcess = new ProcessInstance({
      agent: childAgent,
      parent: parentProcess.id,
      type: 'autonomous',
      context: {
        ...context,
        allowedDirectories: childAgent.resourceAccess.allowedDirectories,
        tools: childAgent.tools.filter(t => t.enabled),
      },
    });

    // 2. Execute (same process lifecycle: prompt → call → deslop → evaluate)
    const result = await this.processEngine.start(childProcess);

    // 3. Apply child's deslopper (each agent runs its own)
    // 4. Return result to parent
    return result;
  }

  async delegateParallel(
    parent: ProcessInstance,
    children: { agent: Agent; task: string }[]
  ): Promise<TaskResult[]> {
    // Respect maxConcurrentSubAgents
    return Promise.all(
      children.map(c => this.delegate(parent, c.agent, c.task, parent.context))
    );
  }
}
```

### 11.7 Context Propagation

Parent → Child context is explicitly controlled:

```typescript
interface ContextPropagation {
  inheritProfile: boolean;       // Use parent's profile if child has none?
  inheritTools: boolean;         // Pass parent's tools down?
  inheritResources: boolean;     // Pass file access down?
  inheritHistory: boolean;       // Pass conversation history?
  maxHistoryTokens: number;      // Truncate history for child
  additionalContext: string;     // Extra context to inject
}
```

### 11.8 UI: Agent Config View

```
┌──────────────────────────────────────────┐
│  Agent Configuration                     │
│                                          │
│  Name: [Code Assistant             ]     │
│  Profile: [Sniffo Code          ▼]      │
│  Instructions: [Write concise React...]  │
│                                          │
│  ┌─ Capabilities ──────────────────┐     │
│  │ ☑ Generate Code  ☑ Read Files   │     │
│  │ ☐ Write Files    ☐ Exec Cmds    │     │
│  │ ☑ Query Docs     ☐ Sub-agents   │     │
│  └──────────────────────────────────┘     │
│                                          │
│  ┌─ Sub-Agents (togglable) ─────────┐   │
│  │ ☑ Reviewer         [Configure]    │   │
│  │ ☐ Code Writer      [Configure]    │   │
│  │ ☑ Doc Lookup       [Configure]    │   │
│  │ ☐ Test Writer      [Configure]    │   │
│  │ ┌─ Strat: [Sequential      ▼] ┐   │   │
│  │ └──────────────────────────────┘   │   │
│  └──────────────────────────────────┘     │
│                                          │
│  ┌─ Resources ──────────────────────┐     │
│  │ Read Dirs: ./src/*               │     │
│  │ Allowed Exts: .ts, .tsx, .css    │     │
│  │ Read-only: ☑                     │     │
│  └──────────────────────────────────┘     │
│                                          │
│  [Train Agent...] [Save as Preset]        │
└──────────────────────────────────────────┘
```

### 11.9 Training at Agent Level

Each agent has its own training history. When you train an agent:

```
Agent: "Code Writer v2"
  ├── Profile: Sniffo Code
  ├── Own instructions: "Always use arrow functions, no default exports"
  └── Training:
        ├── Session 1: Got feedback on 3 code generations
        │     → Evolved instruction: "Prefer async/await over .then()"
        ├── Session 2: Got feedback on 2 reviews
        │     → Evolved instruction: "Call out missing error handling specifically"
        └── Session 3: User added 2 few-shot examples
```

Training refines just that agent's instructions, not the base profile. Multiple agents can share a profile but have diverging trained behaviors.

### 11.10 Standard Agent Presets

**Presets are read-only shipped configs** that users can fork and modify.

| Preset Name | Profile | Sub-agents | Best For |
|------------|---------|------------|----------|
| **Solid Code Gen** | Sniffo Code | None (does it all) | Quick code generation |
| **Pair Programmer** | Sniffo Code | Reviewer (review=true) | Code + immediate review |
| **TDD Flow** | Sniffo Code | Test Writer (parallel), then Code Writer, then Reviewer | Test-first development |
| **Architecture Review** | Sniffo Review | None | Deep PR/design review |
| **Full Stack Builder** | Orchestrator | All sub-agents, sequential | Building features end-to-end |
| **Doc Explorer** | Minimal | Doc Lookup (auto) | Quick API lookups |
| **Debug Detective** | Sniffo Code | Reviewer, Doc Lookup | Debugging with doc context |
| **Refactor Pro** | Sniffo Code | Reviewer, Test Writer | Safe refactoring |
| **UI Crafter** | Sniffo UI | None (+ vision eval on generated UI) | Generating UI components |
| **Merge Helper** | Sniffo Review | None | PR review + merge suggestions |

---

## 12. Directory Structure