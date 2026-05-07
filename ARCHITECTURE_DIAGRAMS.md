# Harness — Architecture with Mermaid Diagrams

> An AI agent harness for coding, review, autonomous agents, and more.
> Open-source Electron + React + TypeScript application.

> **Note:** These diagrams use [Mermaid](https://mermaid.js.org/). They render automatically on GitHub, GitLab, Notion, and with any Markdown viewer that has a Mermaid plugin.

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [Process Lifecycle](#2-process-lifecycle)
3. [Sub-Agent Orchestration Flows](#3-sub-agent-orchestration-flows)
4. [Deslopper Pipeline](#4-deslopper-pipeline)
5. [Training / Prompt Evolution](#5-training--prompt-evolution)
6. [Doc Awareness System](#6-doc-awareness-system)
7. [Provider Abstraction](#7-provider-abstraction)
8. [Git Integration Flow](#8-git-integration-flow)
9. [Agent Configuration Model](#9-agent-configuration-model)
10. [UI Layout](#10-ui-layout)
11. [Directory Structure](#11-directory-structure)
12. [Build Order](#12-build-order)

---

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Electron["Electron Shell"]
        direction TB
        
        subgraph MainProcess["Main Process (Node.js)"]
            PE[Process Engine]
            PL[Provider Layer<br/>OpenAI • Anthropic • Google • Ollama]
            DE[Deslopper Engine]
            GI[Git Integration]
            FS[File System Access]
            DS[Doc Service<br/>Fetch + Cache]
            PS[Profile Store]
            CI[Context Isolator<br/>Sandbox]
            AM[Agent Manager]
        end
        
        subgraph Renderer["Renderer (React + Tailwind)"]
            CIU[Chat Interface]
            PM[Process Manager]
            PEU[Profile Editor]
            TU[Training UI]
            GP[Git Panel<br/>Diff • Commit • PR]
            ST[Settings]
            DB[Doc Browser]
            AV[Agent Config View]
        end
        
        subgraph IPC["Typed IPC Bridge"]
            IPC_BUS[contextBridge + ipcRenderer]
        end
        
        MainProcess <--> IPC_BUS
        IPC_BUS <--> Renderer
    end
    
    subgraph External["External Systems"]
        LLM[LLM Providers]
        GIT[Git Hosts<br/>GitHub • GitLab]
        NPM[NPM Registry]
        
        LLM <--> PL
        GIT <--> GI
        NPM <--> DS
    end
    
    PE --> PL
    PE --> DE
    PE --> GI
    PE --> DS
    AM --> PE
    PL --> CI
```

---

## 2. Process Lifecycle

Every agent run — chat, code gen, review, autonomous task — follows this lifecycle:

```mermaid
stateDiagram-v2
    [*] --> INIT
    INIT --> PREPARE: Load profile ❯ gather context ❯ fetch docs
    
    PREPARE --> PROMPT_BUILD: Build full prompt\n(system + context + user)
    
    PROMPT_BUILD --> CALL_LLM: Send to provider
    
    CALL_LLM --> DESLOP: Raw output received
    DESLOP --> EVALUATE: Deslopped output
    
    EVALUATE --> PASSED: Score under threshold
    EVALUATE --> FAILED: Score too high ❯ needs improvement
    
    PASSED --> COMPLETED
    FAILED --> FEEDBACK_LOOP
    
    COMPLETED --> [*]
    
    state FEEDBACK_LOOP {
        [*] --> USER_FEEDBACK
        USER_FEEDBACK --> REFINE_PROMPT: Analyze feedback ❯ evolve instructions
        REFINE_PROMPT --> PROMPT_BUILD: Retry with refined prompt
    }
```

### Retry & Iteration Logic

```mermaid
flowchart LR
    A[User Input] --> B[Process Engine]
    B --> C{Build Prompt}
    C --> D[Call LLM]
    D --> E[Raw Output]
    E --> F[Deslopper]
    F --> G{Score OK?}
    
    G -->|Yes ✓| H[Present to User]
    
    G -->|No ✗| I{Retry left?}
    I -->|Yes| J[Refine Prompt<br/>with feedback]
    J --> C
    
    I -->|No| K[Flag to User<br/>with deslop report]
    K --> H
    
    H --> L{User satisfied?}
    L -->|Yes| M[Done ✓]
    L -->|No ➜ refine| N[User gives<br/>structured feedback]
    N --> O[Save to<br/>Training Log]
    O --> P[Evolve Agent Profile]
    P --> C
```

---

## 3. Sub-Agent Orchestration Flows

### 3.1 Sequential Delegation

```mermaid
sequenceDiagram
    participant U as User
    participant O as Orchestrator
    participant DL as Doc Lookup
    participant CW as Code Writer
    participant RV as Reviewer
    participant TW as Test Writer
    
    U->>O: "Add Prisma schema for users,<br/>write the API, add tests"
    
    O->>DL: "Get Prisma 7 schema syntax"
    DL-->>O: Schema documentation
    
    O->>CW: "Write schema.prisma"
    CW-->>O: Schema file
    
    O-->>U: Review schema? ✋
    U->>O: Looks good, continue
    
    O->>CW: "Write API endpoints"
    CW-->>O: API code
    
    O->>RV: "Review API code"
    RV-->>O: Review comments
    
    alt Review Passes
        O->>TW: "Write tests"
        TW-->>O: Test files
        O-->>U: ✅ Full result
    else Review Fails
        O->>CW: Fix issues A, B, C
        CW-->>O: Fixed code
        O->>RV: Re-review
        RV-->>O: Approved
        O->>TW: Write tests
        O-->>U: ✅ Full result
    end
```

### 3.2 Parallel Delegation

```mermaid
sequenceDiagram
    participant U as User
    participant O as Orchestrator
    participant CW1 as Code Writer (Users)
    participant CW2 as Code Writer (Orders)
    participant TW as Test Writer
    participant RV as Reviewer
    
    U->>O: "Generate a React dashboard<br/>with full CRUD"
    
    par Parallel
        O->>CW1: "Write user CRUD components"
        O->>CW2: "Write order CRUD components"
        O->>TW: "Write tests for both"
    end
    
    CW1-->>O: User components
    CW2-->>O: Order components
    TW-->>O: Test files
    
    O->>RV: "Merge & review all"
    RV-->>O: Review + suggestions
    
    O-->>U: ✅ Dashboard complete
```

### 3.3 Sub-Agent Toggling

```mermaid
flowchart LR
    subgraph Active["Sub-agents: ON"]
        direction TB
        A1[Orchestrator] --> A2[Code Writer]
        A1 --> A3[Reviewer]
        A1 --> A4[Test Writer]
        A1 --> A5[Doc Lookup ✓]
        A1 --> A6[Browser ✓]
    end
    
    subgraph Disabled["Sub-agents: OFF"]
        direction TB
        B1[Orchestrator does all work<br/>using its own profile] 
    end
    
    User -->|Toggle| Active
    User -->|Toggle| Disabled
```

### 3.4 Manual Delegation

```mermaid
sequenceDiagram
    participant U as User
    participant O as Orchestrator
    participant RV as Reviewer
    
    U->>O: "Review this PR"
    O-->>U: "I'll do it. Use /review for deeper review"
    U->>O: "/review"
    O->>RV: Delegate diff review
    RV-->>O: In-depth review
    O-->>U: Presented ✓
```

---

## 4. Deslopper Pipeline

```mermaid
flowchart TB
    RAW[Raw LLM Output] --> TOK[Tokenizer]
    TOK --> SS[Sentence Splitter]
    SS --> PM[Pattern Matcher]
    
    subgraph Rules["Rule Engine"]
        direction TB
        PR[Pattern Rules<br/>• Hedge words<br/>• Fake gratitude<br/>• Pompous verbs]
        SR[Style Rules<br/>• Verbosity<br/>• Tone markers<br/>• Bullet density]
        SIG[Signal Rules<br/>• Model-specific tells<br/>• Confidence markers]
        CR[Custom Rules<br/>• User-defined]
    end
    
    PM --> Rules
    
    Rules --> SC[Score Calculator<br/>0 - 100]
    SC --> TC{Threshold Check}
    
    TC -->|"< 30"| PASS[PASS ✓<br/>Output delivered]
    TC -->|"30-60"| FLAG[FLAG 🔶<br/>Show warnings to user]
    TC -->|"> 60"| BLOCK[BLOCK 🔴<br/>Auto-rewrite or block]
    
    BLOCK --> RW{Strategy?}
    RW -->|Auto-rewrite| RW2[Deslopper Rewriter<br/>strips & rewrites]
    RW -->|Flag & ask| QU[User decides]
    
    RW2 --> RP[Re-scored Output]
    QU --> RP
    
    FLAG --> REP[Deslop Report<br/>• What was flagged<br/>• Line numbers<br/>• Suggested fixes]
    REP --> OUT[Deliver + Report]
    
    PASS --> OUT
    
    subgraph Builtin_Slop["Built-in Slop Patterns"]
        H1[• "I think" / "perhaps" / "it seems"]
        H2[• "Great question!" / "I'd be happy to help"]
        H3[• "leverage" / "utilize" / "delve"]
        H4[• "In order to" → "To"]
        H5[• "This is definitely the best approach"]
        H6[• 5+ consecutive bullet points]
        H7[• "First, let me explain..." / "Now that we've covered"]
        H8[• "Interesting perspective"]
    end
```

---

## 5. Training / Prompt Evolution

```mermaid
flowchart LR
    subgraph Cycle["Training Cycle"]
        direction TB
        
        GEN[Agent produces output] --> USER{User satisfied?}
        USER -->|Yes ✓| DONE[Done]
        
        USER -->|No ✗| FB[User provides feedback<br/>• Edit the output<br/>• Rate clarity/style/accuracy/slop<br/>• Add notes]
        
        FB --> AN[Feedback Analyzer]
        
        subgraph AN["Analysis Engine"]
            direction TB
            DIFF[Diff output vs correction] --> PATT[Pattern Detection]
            PATT --> CLUST[Cluster changes by type<br/>• Always shortens comments<br/>• Removes bullet lists<br/>• Rewrites async patterns]
        end
        
        AN --> EVOLVE[Prompt Evolver]
        
        EVOLVE --> REFINEMENTS[Refinements]
        
        REFINEMENTS -->|System Prompt| SP["Append: 'Keep comments minimal'"]
        REFINEMENTS -->|Few-Shot| FS["Add user's preferred examples"]
        REFINEMENTS -->|Deslopper| DR["Adjust rule weights"]
        REFINEMENTS -->|Format| FP["Prefer concise output structure"]
        
        SP --> PROFILE[Updated Agent Profile]
        FS --> PROFILE
        DR --> PROFILE
        FP --> PROFILE
        
        PROFILE --> GEN
    end
```

### Feedback Data Model

```mermaid
classDiagram
    class FeedbackExample {
        +string id
        +string input
        +string output
        +string deslopped
        +string userCorrection
        +string userNotes
        +Ratings ratings
        +number timestamp
    }
    
    class Ratings {
        +number clarity
        +number conciseness
        +number accuracy
        +number style
        +number slopLevel
    }
    
    class ProfileDelta {
        +string systemPromptChanges
        +FewShotExample[] fewShotExamples
        +DeslopperAdjustment deslopperAdjustments
        +StylePreferences stylePreferences
    }
    
    FeedbackExample --> Ratings
    FeedbackExample ..> ProfileDelta: evolves to
```

---

## 6. Doc Awareness System

```mermaid
flowchart TB
    subgraph Project["User's Project"]
        PJ[package.json]
        IMPORTS[import statements]
    end
    
    subgraph DocService["Doc Service"]
        RES[Resolver<br/>Detects libraries + versions]
        CACHE[SQLite Cache<br/>• API signatures<br/>• Breaking changes<br/>• TSDoc comments]
        FETCH[Fetcher<br/>• npm registry<br/>• unpkg/jsdelivr<br/>• TypeDoc<br/>• GitHub README]
        IDX[Indexer<br/>• Parse TSDoc<br/>• Extract signatures<br/>• Tag versions]
        INJ[Injector<br/>• Build context string<br/>• Note breaking changes<br/>• Cite version]
    end
    
    subgraph LLM_Process["LLM Process"]
        PROMPT[System Prompt<br/>+ Doc Context]
        LLM[LLM Provider]
        OUT[Version-aware Output]
    end
    
    Project --> RES
    RES -->|Check cache| CACHE
    CACHE -->|Cache miss| FETCH
    FETCH --> IDX
    IDX --> CACHE
    CACHE --> INJ
    INJ --> PROMPT
    PROMPT --> LLM
    LLM --> OUT
    
    subgraph DocSources["Documentation Sources"]
        NPM[NPM Registry<br/>package metadata]
        UNPKG[unpkg / jsdelivr<br/>TypeScript definitions]
        TD[TypeDoc / API Extractor<br/>Structured API docs]
        MDN[MDN Web Docs<br/>DOM / Web APIs]
        GH[GitHub README<br/>Fallback]
        CUSTOM[Custom URL<br/>Project-specific]
    end
    
    FETCH --> DocSources
```

### Version Awareness Detail

```mermaid
flowchart LR
    EX[package.json<br/>prisma: ^7.0.0] --> CMP{Compare versions}
    REG[npm registry<br/>prisma@7.1.0 latest] --> CMP
    
    CMP -->|Same major| OK[OK ✓<br/>Use installed version docs]
    CMP -->|Major bump| ALERT[🔴 BREAKING<br/>List v6→v7 changes]
    
    ALERT --> BLOCK[Build alert block:<br/>• prisma@7.0.0 installed<br/>• v6 syntax no longer valid<br/>• Key changes: ...<br/>• Examples use v7 API]
    
    OK --> INJECT[Inject docs into prompt]
    BLOCK --> INJECT
    
    INJECT --> LLM[LLM generates<br/>version-correct code]
```

---

## 7. Provider Abstraction

```mermaid
classDiagram
    class ProviderAdapter {
        <<interface>>
        +string id
        +string name
        +complete(req) CompletionResponse
        +stream(req) AsyncIterable
        +abort(requestId) void
        +listModels() ModelInfo[]
        +getCapabilities(model) ModelCapabilities
    }
    
    class ModelCapabilities {
        +boolean streaming
        +boolean vision
        +int maxTokens
        +int contextWindow
        +boolean toolUse
        +boolean embedding
        +boolean jsonMode
    }
    
    class CompletionRequest {
        +string model
        +Message[] messages
        +string systemPrompt
        +float temperature
        +int maxTokens
        +Tool[] tools
        +Profile profile
    }
    
    class CompletionResponse {
        +string id
        +string content
        +FinishReason finishReason
        +Usage usage
        +bool deslopped
        +DeslopReport deslopReport
    }
    
    ProviderAdapter --> CompletionRequest : receives
    ProviderAdapter --> CompletionResponse : returns
    ProviderAdapter --> ModelCapabilities : describes
    
    class OpenAIAdapter {
        +complete(req) 
        +stream(req)
    }
    
    class AnthropicAdapter {
        +complete(req)
        +stream(req)
    }
    
    class GoogleAdapter {
        +complete(req)
        +stream(req)
    }
    
    class OllamaAdapter {
        +complete(req)
        +stream(req)
        +baseUrl: string
    }
    
    ProviderAdapter <|-- OpenAIAdapter
    ProviderAdapter <|-- AnthropicAdapter
    ProviderAdapter <|-- GoogleAdapter
    ProviderAdapter <|-- OllamaAdapter
```

### Provider Selection Flow

```mermaid
flowchart TB
    PS[User configures providers<br/>with API keys + priority] --> PL
    
    subgraph PL["Provider Layer"]
        direction TB
        
        REQ[Completion Request] --> PRIO{Check priority}
        PRIO --> P1[Provider #1<br/>OpenAI GPT-4o]
        P1 --> AV1{Available?}
        AV1 -->|Yes| CALL[Call LLM]
        AV1 -->|No| P2[Provider #2<br/>Claude Opus]
        P2 --> AV2{Available?}
        AV2 -->|Yes| CALL
        AV2 -->|No| P3[Provider #3<br/>Local Ollama]
        P3 --> CALL
        
        CALL --> RES[Response]
    end
    
    RES --> OUT[Deslopped Output]
```

---

## 8. Git Integration Flow

```mermaid
flowchart TB
    subgraph IDE["User's IDE"]
        CHANGE[User makes changes]
        SAVE[Saves files]
    end
    
    subgraph Harness["Harness"]
        DIRTY[Detects dirty repo]
        DIFF[GitService.getDiff()]
        DVIEW[Render diff in UI]
        
        subgraph Commit["Commit Generation"]
            CDIFF[Structured diff] --> CPROMPT[Build commit prompt<br/>+ Conventional Commits spec]
            CPROMPT --> CLLM[LLM generates<br/>commit message]
            CLLM --> CDES[Deslopper<br/>(no verbose commit messages)]
            CDES --> CEDIT[User can edit]
            CEDIT --> CEXEC[git commit]
        end
        
        subgraph PR["PR Generation"]
            BINFO[Branch info<br/>+ commit history] --> BPROMPT[Build PR prompt]
            BPROMPT --> BLLM[LLM generates<br/>title + description]
            BLLM --> BDES[Deslopper]
            BDES --> BEDIT[User edits]
            BEDIT --> BEXEC[Create PR<br/>via GitHub/GitLab API]
        end
    end
    
    CHANGE -->|File watcher| DIRTY
    DIFF --> DVIEW
    DVIEW -->|"Generate Commit"| Commit
    DVIEW -->|"Generate PR"| PR
```

### Git Diff → Commit → PR Pipeline

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness
    participant LLM as LLM Provider
    participant GH as Git Host (GitHub)
    
    Note over U,GH: User has uncommitted changes
    
    U->>H: Open Harness
    H->>H: Detect dirty repo
    H-->>U: "3 files modified"
    
    U->>H: View Diff
    H->>H: Parse git diff → structured hunks
    H-->>U: Syntax-highlighted diff
    
    U->>H: Generate Commit Message
    H->>LLM: "Conventional commit for:<br/>{diff_summary}"
    LLM-->>H: "feat: add user subscription API"
    H->>H: Deslopper runs
    H-->>U: "feat: add user subscription API"
    
    U->>H: Looks good, commit
    H->>H: git commit -m "..."
    H-->>U: ✅ Committed
    
    U->>H: Push & Generate PR
    H->>LLM: "PR for branch feat/subscriptions<br/>{diff + commit log}"
    LLM-->>H: PR body
    H->>H: Deslopper
    H-->>U: PR preview
    U->>H: Edit title, approve
    H->>GH: POST /repos/../pulls
    GH-->>H: PR URL
    H-->>U: ✅ PR created
```

---

## 9. Agent Configuration Model

```mermaid
classDiagram
    class Agent {
        +string id
        +string name
        +string description
        +string profileId
        +Role role
        +string instructions
        +string systemPromptOverride
        +string systemPromptAppend
        +AgentRef[] subAgents
        +bool allowSubAgents
        +SubAgentStrategy subAgentStrategy
        +AgentCapabilities capabilities
        +ToolAssignment[] tools
        +ResourceAccess resourceAccess
        +SpawnConfig spawn
        +AgentTraining training
        +string[] tags
        +int version
        +DateTime createdAt
        +DateTime updatedAt
        +bool builtIn
    }
    
    class AgentCapabilities {
        +bool canGenerateCode
        +bool canReviewCode
        +bool canReadFiles
        +bool canWriteFiles
        +bool canExecuteCommands
        +bool canAccessNetwork
        +bool canSpawnSubAgents
        +bool canUseBrowser
        +bool canQueryDocs
        +bool canAnalyzeImages
    }
    
    class ResourceAccess {
        +string[] allowedDirectories
        +string[] allowedExtensions
        +int maxFileSize
        +bool readOnly
        +string[] allowedHosts
    }
    
    class SpawnConfig {
        +int maxConcurrentSubAgents
        +int timeoutMs
        +int retryOnFailure
        +int maxIterations
    }
    
    class AgentTraining {
        +bool enabled
        +FeedbackExample[] feedbackExamples
        +string evolvedPrompt
    }
    
    class AgentRef {
        +string agentId
        +bool enabled
        +DelegationType delegationType
        +string delegationRules
    }
    
    Agent --> AgentCapabilities
    Agent --> ResourceAccess
    Agent --> SpawnConfig
    Agent --> AgentTraining
    Agent --> AgentRef : subAgents
```

### Sub-Agent Process Flow (Internal)

```mermaid
sequenceDiagram
    participant PM as ProcessManager
    participant SM as SubAgentRuntime
    participant PE as ProcessEngine
    participant DL as Deslopper
    
    PM->>SM: delegate(parentId, childAgent, task, context)
    
    SM->>SM: Build child context<br/>• Apply resource filters<br/>• Limit tools to allowed set<br/>• Truncate history
    
    SM->>PE: startProcess(childProcess)
    
    PE->>PE: INIT → PREPARE → PROMPT
    
    Note over PE,DL: Standard process lifecycle
    
    PE->>PE: CALL LLM
    PE->>DL: Raw output
    DL->>DL: Deslop (child's own rules)
    DL-->>PE: Cleaned output
    
    PE-->>SM: TaskResult
    
    SM-->>PM: Result (with deslop report)
    
    PM->>PM: Merge into parent context
    PM-->>User: Partial/Complete result
```

---

## 10. UI Layout

```mermaid
flowchart TB
    subgraph App["Harness Window"]
        direction TB
        
        TABS[Tab Bar: Chat • Git • Agents • Profiles • Docs • Settings]
        
        subgraph Main["Main Content Area"]
            direction LR
            
            SIDEBAR[Sidebar<br/>• Process list<br/>• Agent list<br/>• Profile selector]
            CONTENT[Content Panel<br/>• Chat view<br/>• Diff viewer<br/>• Agent config<br/>• Training UI]
            DESLOP[Deslop Panel<br/>• Active flags<br/>• Rewrite log<br/>• Score history]
        end
        
        STATUSBAR[Status Bar<br/>• Active process • git/main • 3 unsaved files • Provider: GPT-4o]
        
        TABS --> Main
        Main --> STATUSBAR
    end
```

### Component Tree

```mermaid
flowchart TB
    APP[&lt;App /&gt;] --> LAY[&lt;Layout /&gt;]
    
    LAY --> TABS[&lt;TabBar /&gt;]
    LAY --> MAIN[&lt;MainPanel /&gt;]
    LAY --> STAT[&lt;StatusBar /&gt;]
    
    MAIN --> SIDEBAR[&lt;Sidebar /&gt;]
    MAIN --> CONTENT[&lt;ContentRouter /&gt;]
    MAIN --> DPANEL[&lt;DeslopPanel /&gt;]
    
    SIDEBAR --> PL[&lt;ProcessList /&gt;]
    SIDEBAR --> AL[&lt;AgentList /&gt;]
    SIDEBAR --> PS[&lt;ProfileSelector /&gt;]
    
    CONTENT --> CV[&lt;ChatView /&gt;]
    CONTENT --> GV[&lt;GitView /&gt;]
    CONTENT --> TV[&lt;TrainingView /&gt;]
    CONTENT --> ACV[&lt;AgentConfigView /&gt;]
    CONTENT --> SP[&lt;SettingsPage /&gt;]
    
    GV --> DP[&lt;DiffPanel /&gt;]
    GV --> CP[&lt;CommitPanel /&gt;]
    GV --> PRP[&lt;PRPanel /&gt;]
    
    DPANEL --> DR[&lt;DeslopReport /&gt;]
```

---

## 11. Directory Structure

```
harness/
├── package.json
├── tsconfig.json
├── electron-builder.yml          # Electron packaging config
├── tailwind.config.ts
├── vite.config.ts
├── .gitignore                   # node_modules, dist, user config, doc-cache
│
├── src/
│   ├── main/                    # Electron main process
│   │   ├── index.ts             # App entry, window creation
│   │   ├── ipc-handlers.ts      # IPC handler registration
│   │   ├── process-engine.ts    # Process orchestrator
│   │   ├── deslopper/
│   │   │   ├── engine.ts        # Deslopper core
│   │   │   ├── rules/           # Rule definitions
│   │   │   │   ├── pattern-rules.ts
│   │   │   │   ├── style-rules.ts
│   │   │   │   └── custom-rules.ts
│   │   │   ├── scorer.ts        # Slop scoring
│   │   │   └── rewriter.ts      # Auto-rewrite engine
│   │   ├── providers/
│   │   │   ├── adapter.ts       # ProviderAdapter interface
│   │   │   ├── openai.ts        # OpenAI adapter
│   │   │   ├── anthropic.ts     # Anthropic adapter
│   │   │   ├── google.ts        # Google AI adapter
│   │   │   ├── ollama.ts        # Ollama adapter
│   │   │   └── registry.ts      # Provider registry
│   │   ├── agents/
│   │   │   ├── manager.ts       # Agent CRUD
│   │   │   ├── runtime.ts       # Sub-agent spawn/delegate
│   │   │   └── presets.json     # Built-in presets
│   │   ├── git/
│   │   │   ├── service.ts       # Git operations
│   │   │   ├── diff-parser.ts   # Structured diff parsing
│   │   │   ├── commit-gen.ts    # Commit message generation
│   │   │   └── pr-gen.ts        # PR description generation
│   │   ├── docs/
│   │   │   ├── service.ts       # Documentation fetch & cache
│   │   │   ├── resolver.ts      # Library/version detection
│   │   │   ├── fetcher.ts       # Fetch from sources
│   │   │   ├── indexer.ts       # Parse & index docs
│   │   │   └── injector.ts      # Build context snippets
│   │   ├── profiles/
│   │   │   ├── manager.ts       # Profile CRUD
│   │   │   └── evolver.ts       # Prompt evolution engine
│   │   └── training/
│   │       ├── analyzer.ts      # Feedback pattern analysis
│   │       └── logger.ts        # Training log persistence
│   │
│   ├── renderer/                # React frontend
│   │   ├── index.html
│   │   ├── main.tsx             # React entry
│   │   ├── App.tsx              # Root component
│   │   ├── components/
│   │   │   ├── layout/
│   │   │   │   ├── Sidebar.tsx
│   │   │   │   ├── TabBar.tsx
│   │   │   │   ├── StatusBar.tsx
│   │   │   │   └── DeslopPanel.tsx
│   │   │   ├── chat/
│   │   │   │   ├── ChatView.tsx
│   │   │   │   ├── MessageBubble.tsx
│   │   │   │   └── StreamingOutput.tsx
│   │   │   ├── git/
│   │   │   │   ├── DiffPanel.tsx
│   │   │   │   ├── CommitPanel.tsx
│   │   │   │   └── PRPanel.tsx
│   │   │   ├── agents/
│   │   │   │   ├── AgentConfigView.tsx
│   │   │   │   └── AgentList.tsx
│   │   │   ├── profiles/
│   │   │   │   ├── ProfileEditor.tsx
│   │   │   │   └── ProfileSelector.tsx
│   │   │   ├── training/
│   │   │   │   ├── TrainingView.tsx
│   │   │   │   └── FeedbackForm.tsx
│   │   │   └── settings/
│   │   │       ├── ProviderSettings.tsx
│   │   │       └── DeslopperSettings.tsx
│   │   ├── stores/
│   │   │   ├── process-store.ts  # Zustand
│   │   │   ├── agent-store.ts
│   │   │   ├── git-store.ts
│   │   │   └── settings-store.ts
│   │   └── hooks/
│   │       ├── useProcess.ts
│   │       └── useIpc.ts        # IPC bridge hooks
│   │
│   ├── shared/                  # Shared between main & renderer
│   │   ├── ipc.ts               # IPC contract types
│   │   ├── types.ts             # Core types
│   │   ├── deslopper-types.ts   # Deslopper types
│   │   ├── agent-types.ts       # Agent types
│   │   ├── profile-types.ts     # Profile types
│   │   └── training-types.ts    # Training types
│   │
│   └── presets/                 # Shipped built-in data
│       ├── agents.json          # Standard agent presets
│       ├── profiles.json        # Built-in profiles
│       └── deslopper-rules.json # Default rule set
│
├── electron/
│   ├── main.ts                  # Electron entry
│   └── preload.ts               # contextBridge
│
└── user-config/                 # Gitignored, created at runtime
    ├── config.json              # App settings, provider keys
    ├── profiles/                # User profiles
    ├── training-logs/           # Raw feedback data
    └── doc-cache/               # Cached documentation
```

---

## 12. Build Order

```mermaid
gantt
    title Suggested Build Sequence
    dateFormat  YYYY-MM-DD
    axisFormat  %b
    
    section Phase 1: Core
    Profile system (types + manager + presets)    :p1a, 2026-05-10, 3d
    Provider adapter interface + one adapter      :p1b, after p1a, 3d
    Deslopper engine (parser + rules + scorer)    :p1c, after p1b, 4d
    
    section Phase 2: Process Loop
    Process engine (lifecycle + prompt builder)   :p2a, after p1c, 4d
    Chat UI (messaging + streaming)               :p2b, after p2a, 4d
    IPC bridge (typed handlers + preload)         :p2c, after p2b, 2d
    
    section Phase 3: Features
    Git integration (diff + commit)              :p3a, after p2c, 3d
    Doc awareness service                        :p3b, after p2c, 4d
    Agent system (manager + runtime + presets)    :p3c, after p2c, 4d
    
    section Phase 4: Polish
    Training UI + feedback loop                   :p4a, after p3c, 4d
    PR generation + diff viewer                   :p4b, after p3a, 3d
    Settings page + provider config UI            :p4c, after p2c, 2d
    
    section Phase 5: Stretch
    Vision UI evaluation                          :p5a, after p4a, 3d
    Sub-agent parallel execution                  :p5b, after p3c, 3d
    Sandbox / context isolator                    :p5c, after p4a, 3d
```

---

> Written for `D:\projects\harness`. The Mermaid diagrams render in GitHub, GitLab, Notion, Obsidian, VS Code (with plugin), and most Markdown editors.
