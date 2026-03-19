---
date: 2026-03-18 01:30:00 PST
ver: 1.0.0
author: Sliither (for Ice-ninja)
model: claude-opus-4-6
tags: [parallelism-detection, intelligence-scoring, model-routing, dynamic-escalation, intent-inference, cross-framework, unified-orchestrator, project-proposals]
---
# Addendum B: Under-Served Concepts, Similar Projects, and Unified Deliverables

Produced via V(8,3,1) deliberative refinement, DISCOVER mode, Expert Council. This addendum covers seven concepts that received insufficient depth in the prior reports, grounds each in named projects and research, and proposes three novel project deliverables that unify everything.

## Deliberative Refinement: Gap Identification

**Council composition:** 8 agents (Systems Architect, ML Researcher, Cost Analyst, Practitioner, Framework Expert, Graph Theorist, UX Researcher, Advocatus Diaboli)

**R1 identified 7 under-served concepts:**

1. **Automatic parallelism detection** -- mentioned as a DAG but no algorithm for how an LLM identifies which coding tasks are independent
2. **Intelligence scoring methodology** -- a 1-10 table exists but no rubric for how to score or what features indicate complexity
3. **Dynamic model escalation** -- "Sonnet fails, upgrade to Opus" stated but no retry/escalation logic defined
4. **User intent inference mechanism** -- UIR block defined but no technique for HOW the Curator infers intent
5. **Mid-session model switching** -- running multiple model tiers within a single conversation (not just per-subagent)
6. **Cross-framework agent coordination** -- orchestrating Claude Code + OpenHands + Hermes in one workflow
7. **Unified deliverable proposals** -- no concrete project specs that combine all concepts

**R2 refined:** Added that OpenClaw's delegation patterns and Capy's two-agent architecture (Captain/Build) deserve explicit treatment. Token Arbitrage (cloud provider cost-optimized routing, Sep 2025) is a named industry development that validates intelligence scoring.

**R3 converged.** All 7 gaps confirmed. Addendum follows.

---

## 1. Automatic Parallelism Detection for Coding Tasks

### The Problem

When a user says "build a JWT auth system with rate limiting and tests," how does the orchestrator automatically determine that the rate limiter and the auth middleware can be built in parallel (no shared files), but the test suite must wait for both?

### The Algorithm

This is a well-studied problem in computer science: **topological sort on a DAG with antichain extraction.**

**Step 1: Task Decomposition (LLM-driven).** The orchestrator (Opus) decomposes the objective into atomic subtasks, each with:
- A unique ID
- Input dependencies (which other task IDs must complete first)
- Output artifacts (files created/modified)
- Estimated file touchpoints

**Step 2: Dependency Graph Construction.** Build a DAG where nodes are tasks and edges represent "must complete before" relationships. Dependencies arise from two sources:
- **Explicit data flow:** Task B needs Task A's output (e.g., B imports a type A defines)
- **File contention:** Task B and Task C both modify the same file (serialize them)

**Step 3: Antichain Extraction.** An **antichain** is a set of nodes where no node is an ancestor or descendant of another. The **critical antichain** (maximum-size antichain) gives the maximum parallelism width. Topological sort orders all tasks respecting dependencies; tasks at the same topological level can execute simultaneously.

```
Example:
  T1: Create token utility         -> outputs: src/auth/tokens.ts
  T2: Create refresh endpoint      -> depends on T1, outputs: src/auth/refresh.ts
  T3: Add middleware integration   -> depends on T2, outputs: src/routes/hooks.ts
  T4: Add rate limiting            -> independent, outputs: src/auth/ratelimit.ts
  T5: E2E test suite               -> depends on T3 AND T4

  DAG:
    T1 -> T2 -> T3 -> T5
                       ^
    T4 ----------------+

  Topological levels:
    Level 0: [T1, T4]    <- PARALLEL (antichain, width=2)
    Level 1: [T2]        <- sequential (depends on T1)
    Level 2: [T3]        <- sequential (depends on T2)
    Level 3: [T5]        <- sequential (depends on T3, T4)

  Wave schedule:
    Wave 1: T1 (Haiku) + T4 (Sonnet)  -- parallel
    Wave 2: T2 (Sonnet)               -- after T1
    Wave 3: T3 (Sonnet)               -- after T2
    Wave 4: T5 (Haiku)                -- after T3 and T4
```

**Step 4: File Conflict Detection.** Even if two tasks have no explicit data dependency, if they both touch the same file, they must be serialized. Swarms' `swarm_validate_decomposition` already does this: it detects file conflicts and instruction conflicts in the decomposition output.

### Who Implements This Today?

| Project | Implementation | Maturity |
|---------|---------------|----------|
| **Swarms** (`swarm_decompose` + `swarm_validate_decomposition`) | LLM-driven decomposition with CellTreeSchema validation and file conflict detection | Production |
| **OpenHands Refactor SDK** | Dependency analysis based on directory boundaries and inter-component dependencies | Private beta |
| **LangGraph** | Directed graph with conditional edges; developer manually defines the DAG | Production (manual) |
| **Claude Code Agent Teams** | The team lead decomposes naturally but has no formal DAG; tasks are a flat list with informal dependencies | Experimental |
| **Capy** | Two-agent architecture (Captain plans, Build executes). Captain decomposes into parallel tasks automatically | Production |

### Gap

No system automatically infers file-level dependencies from code analysis AND constructs the DAG AND extracts antichains for wave scheduling in one integrated pass. OpenHands' Refactor SDK comes closest but is still in private beta. The missing piece is an **AST-aware dependency analyzer** that reads import/export relationships and constructs the DAG without relying solely on the LLM's judgment.

## 2. Intelligence Scoring: A Concrete Methodology

### The Problem

The prior report proposed a 1-10 intelligence score for tasks but provided no rubric. How do you actually score task complexity?

### Research Grounding

The field of **difficulty-aware agent orchestration** has produced concrete methods:

**AdaptiveLLM** (Cheng et al., Jun 2025): Uses Chain-of-Thought (CoT) length as a difficulty proxy. Tasks that require longer reasoning chains are harder. CoT length clusters coding tasks into difficulty tiers for cost-optimal model selection.

**DAAO (Difficulty-Aware Agent Orchestration)** (Su et al., Sep 2025): Uses a **variational autoencoder (VAE)** to estimate query difficulty from the input embedding alone, BEFORE any LLM processes it. The difficulty score dynamically controls both the depth of the reasoning pipeline AND which model handles each stage.

**RouteLLM** (Ong et al., ICLR 2025): Trains a lightweight router model on human preference data. The router predicts whether a query needs the strong model or the weak model, achieving 2x cost savings without significant quality loss. Key insight: routing from preference data generalizes across model pairs without retraining.

**FrugalGPT** (Chen et al., 2023): Cascading approach. Send the query to the cheapest model first. If the response doesn't pass a confidence threshold, escalate to the next tier. This is the "dynamic model escalation" pattern.

**AutoMix** (Aggarwal et al., 2024): The smaller model generates a response AND self-verifies its confidence. If confidence is below threshold, the query routes to the larger model. Single-query latency (no cascade needed).

**GreenServ** (2026): Multi-armed bandit (MAB) approach that extracts lightweight contextual features (task type, semantic cluster, text complexity) and learns routing policies online. 22% accuracy increase, 31% energy reduction vs. random routing.

**SCORE** (Harvard, ICLR 2025 Workshop): Formulates routing as a **constrained optimization problem** under incomplete information, with explicit cost and latency budgets.

### The Rubric

Synthesizing the research, here is a concrete intelligence scoring rubric for coding tasks:

| Score | Feature Indicators | Model Tier | Proxy Method |
|-------|-------------------|------------|-------------|
| 1-2 | File reads, grep, format checks, rename. No reasoning required. | Haiku | Rule-based: if task is read-only, score = 1-2 |
| 3-4 | Single-file edits with clear pattern (add a field, fix a typo, write a simple test). Short CoT expected. | Haiku/Sonnet | CoT length proxy: if estimated CoT < 500 tokens, score = 3-4 |
| 5-6 | Multi-file edits, standard implementation (CRUD endpoint, middleware, component). Moderate CoT. | Sonnet | Dependency count: if task touches 2-5 files and has 1-2 dependencies, score = 5-6 |
| 7-8 | Architecture decisions, complex debugging (requires understanding multiple subsystems). Long CoT expected. | Sonnet/Opus | Semantic complexity: if task involves cross-cutting concerns or requires understanding 5+ files, score = 7-8 |
| 9-10 | Novel algorithm design, security analysis, system-wide refactoring, resolving conflicting requirements. Extended thinking required. | Opus + extended thinking | If task has no clear pattern-match to prior work AND affects >10 files, score = 9-10 |

### Implementation

The cheapest implementation: have the orchestrator (running Opus) assign scores during task decomposition using a structured output format. Include the rubric in the orchestrator's system prompt. More sophisticated: train a lightweight classifier (distilled from Opus judgments) to score tasks from their descriptions alone, similar to RouteLLM's preference-trained router.

## 3. Dynamic Model Escalation

### The Pattern

```
attempt(task, model=Haiku)
  |
  if FAIL or LOW_CONFIDENCE:
    |
    attempt(task, model=Sonnet, context=previous_attempt_errors)
      |
      if FAIL or LOW_CONFIDENCE:
        |
        attempt(task, model=Opus, context=all_previous_attempts)
          |
          if FAIL: flag for human review
```

### Named Implementations

**FrugalGPT cascade:** Sequential escalation through model tiers. Each tier receives the previous tier's failure context.

**AutoMix self-verification:** Single-shot attempt with self-confidence check. If the model rates its own output below a threshold, escalate.

**Claude Code effort parameter:** Opus 4.5+ supports an effort parameter (low/medium/high) that controls reasoning depth. Set effort=medium first; if the task fails, retry with effort=high. This is model-INTERNAL escalation rather than cross-model.

**Anthropic's prompt caching:** When escalating from Sonnet to Opus, the prompt (project context, task description) is already cached. The escalation cost is primarily the delta in output token pricing, not a full re-read. This makes escalation cheaper than it appears.

### The Missing Piece

No framework currently implements automatic escalation as a first-class workflow primitive. You can build it with LangGraph conditional edges or a custom Claude Code hook, but there's no `escalation_policy: cascade` config option anywhere. This is a straightforward feature to build on top of any orchestration framework.

## 4. User Intent Inference

### The Problem

The Context Priming addendum defined a User Intent Register (UIR) but didn't specify HOW the Curator model infers intent from raw user commands.

### The Technique

**Level 1: Command Summarization.** Straightforward NLP: transform "can you fix the auth middleware so it doesn't break when the token expires" into "Fix token expiration handling in auth middleware."

**Level 2: Goal Inference.** Connect the command to the broader objective: "The user is building a robust JWT authentication system. This command addresses the token refresh flow, which is part of the session management feature they've been working on since turn 12."

**Level 3: Preference Extraction.** Observe patterns across multiple commands: "The user consistently prefers functional patterns over class-based (3 instances). They prioritize user-facing error messages over developer logs (2 instances). They dislike localStorage for auth state (1 explicit statement)."

**Level 4: Frustration Detection.** If the user repeats a request (variant phrasing, same intent), flag it. "User has asked for rate limiting 3 times. Attempts 1 and 2 were not addressed because the agent was focused on the middleware task. PRIORITY: address rate limiting in next task assignment."

### Implementation

The Curator (Haiku) receives every user message and runs a structured extraction prompt:

```
Given this user message in the context of a coding session:
"{user_message}"

Extract:
1. COMMAND: What specific action is the user requesting?
2. GOAL: What broader objective does this serve?
3. PREFERENCES: Any coding style, pattern, or approach preferences expressed?
4. FRUSTRATION: Is this a repeat of a prior unaddressed request? (yes/no + evidence)
5. AMBIGUITY: Does this contradict any prior instruction? (yes/no + details)

Respond in JSON.
```

This runs on Haiku at ~$0.001 per extraction. Over a 100-turn session, that's $0.10 total for a complete running intent register.

### Prior Art

**OpenAI's state-based memory** (2026 cookbook): Implements "distill memories during a run (tool call -> session notes)" with precedence rules (latest user input -> session overrides -> global defaults). This is the closest existing implementation of Level 1-2 intent inference.

**Hermes Agent's persistent memory:** Auto-generates skills from successful problem-solving, which implicitly captures user preferences (Level 3). But it's post-hoc, not real-time.

**Cursor/Windsurf rules files:** Static declarations of user preferences. No inference. The user must explicitly write "I prefer functional patterns."

## 5. Mid-Session Model Switching

### The Concept

Running multiple model tiers within a single Claude Code conversation, switching based on the current task's needs. Not per-subagent (that's already supported), but the primary session itself adapting its model in real-time.

### Current State in Claude Code

Claude Code supports `/model` to switch models mid-session. The built-in behavior already does partial switching: when you set Haiku as default, Claude Code automatically uses Sonnet for Plan mode and Haiku for execution. The Explore subagent runs on Haiku by default regardless of the main model.

**The gap:** This switching is static (configured once per session) or manual (`/model opus` when you hit a hard problem). There's no automatic switching where the session detects task complexity and adjusts its own model.

### How to Build It

A Claude Code PreToolUse hook could intercept task execution and route to different models:

```javascript
// hooks/model-router.js
export default async function preToolUse(context) {
  const taskComplexity = estimateComplexity(context.currentTask);

  if (taskComplexity <= 4 && context.model !== 'haiku') {
    return { suggestModel: 'haiku' };
  }
  if (taskComplexity >= 8 && context.model !== 'opus') {
    return { suggestModel: 'opus' };
  }
}
```

This doesn't exist as a hook capability today (hooks can't change the model mid-stream), but the Claude Agent SDK's programmatic interface supports per-call model selection. In a custom harness, this is buildable now.

## 6. Cross-Framework Agent Coordination

### The Problem

No system orchestrates Claude Code agents alongside OpenHands agents alongside Hermes agents in a single workflow. Each framework is a silo. But the best tool for each task isn't always from the same framework:

- Claude Code is best for interactive, context-rich coding with Opus
- OpenHands is best for model-agnostic batch refactoring at scale
- Hermes Agent is best for persistent memory, cross-platform messaging, and auto-skill generation
- Swarms is best for complex DAG/MoE orchestration topologies

### How to Bridge Them

**Layer 1: Shared filesystem.** All frameworks can read/write files. The simplest coordination: write task results to a shared directory. Agent A (Claude Code) writes `src/auth/tokens.ts`. Agent B (OpenHands) reads it and writes tests. No protocol needed.

**Layer 2: MCP as universal protocol.** MCP servers are supported by Claude Code, OpenHands (first-class MCP integration), and Hermes Agent (MCP support). A custom MCP server acting as the "task board" could coordinate across all three:

```
MCP Task Board Server
  |-- listTasks()       -> returns current DAG with status
  |-- claimTask(id)     -> locks a task for a specific agent
  |-- completeTask(id)  -> marks done, triggers dependents
  |-- getContext(id)     -> returns the relevant Context Priming blocks
  |-- sendMessage(to, msg) -> inter-agent messaging
```

Any agent that speaks MCP can participate. This is the "Agent Interlink Protocol" (AIP) concept that was ratified in February 2026 for cross-model agent communication.

**Layer 3: Git as coordination substrate.** All agents understand git. Branches, worktrees, PRs, and merge conflicts provide a natural coordination layer. Claude Code Agent Teams already use git-based file locking. Extend this: each agent framework gets its own worktree, pushes to feature branches, and a CI pipeline validates + merges.

### Named Projects Approaching This

| Project | Approach | Limitation |
|---------|----------|-----------|
| **OpenCode + Recallium** (Saurab Bhatia) | 19 agents across 4 model families, shared persistent memory via Recallium MCP | Custom setup, not a reusable framework |
| **OpenAI Symphony** | Daemon that polls issue trackers, spawns agents per issue | OpenAI-only, no cross-framework |
| **Hermes Agent** | Migrates from OpenClaw, supports OpenHands integration (proposed) | Still in development (Issue #477) |
| **Capy** | Captain/Build two-agent architecture with parallel cloud VMs | Commercial, closed source |

## 7. Unified Project Proposals

Three concrete project deliverables that compose the concepts from all three reports into buildable systems.

### Project 1: `context-curator` -- The Context Priming Engine

**What it builds:** A standalone MCP server that implements the Context Curator from the Context Priming addendum. Any coding agent that supports MCP can use it.

**Architecture:**
- MCP server (TypeScript, runs on Bun) exposing tools: `updateIntent`, `updateProjectState`, `getContextBlocks`, `recordDecision`, `recordFailure`
- Haiku-powered inference pipeline for user intent extraction
- Structured markdown block store (`.context/uir.md`, `.context/psb.md`, `.context/teb.md`, `.context/cab.md`)
- Claude Code PreCompact hook that calls `getContextBlocks` and injects the result as the compaction basis
- Semantic search over the Conversation Archaeology Block via local embeddings (all-MiniLM-L6-v2)

**Novel integration:** The Curator watches the agent's conversation stream via a PostMessage-equivalent hook and updates blocks in real-time. When compaction triggers, the agent receives curated multi-block context instead of a lossy summary.

**Stack:** Bun + Hono (MCP server), local embeddings via Qwen3-Embedding-4B (from Ice-ninja's homelab), markdown file store. Deploys as a background process alongside Claude Code.

### Project 2: `wave-scheduler` -- Intelligence-Scored Parallel Task Router

**What it builds:** A task decomposition and scheduling engine that implements automatic parallelism detection, intelligence scoring, model-tier routing, and dynamic escalation.

**Architecture:**
- Accept a high-level objective (natural language)
- Decompose into atomic tasks via Opus (structured JSON output with dependency edges)
- Construct DAG, run topological sort, extract antichains for wave scheduling
- Score each task using the rubric from Section 2 (CoT length proxy + dependency count + file touchpoint analysis)
- Allocate model tiers within a declared cost budget
- Spawn agents (Claude Code subagents, OpenHands delegates, or Hermes tasks) per wave
- Dynamic escalation: if a Haiku/Sonnet agent fails, re-attempt with the next tier, injecting failure context
- Report cost-per-task and total budget consumption

**Novel integration:** Combines Swarms' `swarm_decompose` pattern with RouteLLM's preference-trained routing and FrugalGPT's cascade escalation. The scheduler itself runs on Sonnet (cheaper than Opus for the routing logic), while the decomposition prompt runs on Opus.

**Stack:** TypeScript (Bun), LangGraph-compatible DAG execution, Claude Agent SDK for spawning. Could also emit Swarms-compatible CellTree JSON for use with the Swarms framework directly.

### Project 3: `nexus` -- The Cross-Framework Super Orchestrator

**What it builds:** The "glue layer" that composes context-curator + wave-scheduler + any combination of Claude Code, OpenHands, Hermes Agent, and Swarms into a unified orchestration system.

**Architecture:**
- A persistent process (the Nexus) that acts as the team lead
- MCP Task Board for cross-framework coordination (any MCP-speaking agent can participate)
- Integrates `context-curator` for intelligent context management
- Integrates `wave-scheduler` for task decomposition, scoring, and parallel scheduling
- Agent spawning adapters for each framework:
  - Claude Code: via CLI (`claude --agent`) or Agent SDK
  - OpenHands: via SDK's DelegateTool or CLI
  - Hermes Agent: via `delegate_task` tool or subprocess spawning
  - Swarms: via Python API (SequentialWorkflow, ConcurrentWorkflow, GraphWorkflow)
  - Local models: via Ollama API
- Git worktree manager for per-agent file isolation
- Observability dashboard (web UI): task DAG visualization, per-agent token cost, wave progress, context block inspector
- Consensus engine (from Hermes Agent's planned design): agents vote on architectural decisions before implementation proceeds

**Novel contribution:** This is the only system that would orchestrate heterogeneous agent frameworks in a single workflow with shared persistent memory and intelligent context curation. Currently, every framework is a silo. Nexus makes them composable.

**Stack:** Bun + Hono (orchestrator API + MCP server), SvelteKit (dashboard), Claude Agent SDK + OpenHands SDK + Hermes CLI (spawners), git worktrees (isolation), Qdrant (semantic memory for CAB block).

**Estimated build effort:**
- `context-curator`: ~2 weeks (MCP server + hooks + block store)
- `wave-scheduler`: ~3 weeks (decomposition + DAG + scoring + routing)
- `nexus` (MVP): ~4 weeks (MCP task board + 2 framework adapters + basic dashboard)
- Full `nexus` with all adapters + observability: ~8 weeks

### How the Three Projects Compose

```
User Objective: "Build a JWT auth system with rate limiting and E2E tests"
    |
    v
[nexus] receives objective
    |
    v
[wave-scheduler] decomposes into tasks, builds DAG, scores intelligence,
  assigns model tiers, groups into waves
    |
    v
[context-curator] generates initial Project State Block and Task Execution
  Blocks for each task
    |
    v
Wave 1: T1 (tokens.ts, Haiku, Claude Code subagent)
         T4 (ratelimit.ts, Sonnet, OpenHands delegate)
         -- parallel, different frameworks, shared context via MCP --
    |
[context-curator] updates PSB with T1/T4 results, generates TEB for T2
    |
    v
Wave 2: T2 (refresh.ts, Sonnet, Claude Code subagent)
    |
Wave 3: T3 (middleware, Sonnet, Claude Code Agent Team with T2 context)
    |
    if T3 FAILS on Sonnet:
      [wave-scheduler] escalates to Opus with failure context
    |
Wave 4: T5 (E2E tests, Haiku, Hermes Agent -- auto-generates test skill
         for future reuse)
    |
    v
[nexus] aggregates results, [context-curator] archives decisions to CAB
```

## Deliberative Refinement: Final Assessment

**R2 refinement:** Advocatus Diaboli challenged: "Three separate projects is overengineered. Just build one thing." Rebuttal accepted: the three projects are designed to be independently useful. `context-curator` alone improves any Claude Code session. `wave-scheduler` alone improves any multi-agent workflow. `nexus` is the composition layer for when you need both plus cross-framework support.

**R3 convergence:** Council agreed that the three-project decomposition follows Ice-ninja's own preference for phase-based planning. Each project is a phase. Phase 1 (`context-curator`) delivers immediate value with lowest effort. Phase 2 (`wave-scheduler`) requires Phase 1's context blocks to be maximally effective. Phase 3 (`nexus`) composes both with cross-framework adapters. **CONVERGED.**

## Sources Added by This Addendum

- RouteLLM (Ong et al., ICLR 2025): Learning-based router from preference data, 2x cost savings
- FrugalGPT (Chen et al., 2023): Cascading LLM query approach
- AutoMix (Aggarwal et al., 2024): Self-verification before routing to larger model
- DAAO: Difficulty-Aware Agent Orchestration with VAE-based difficulty estimation (Su et al., Sep 2025)
- AdaptiveLLM (Cheng et al., Jun 2025): CoT length as difficulty proxy for model routing
- GreenServ (2026): Multi-armed bandit for context-aware dynamic routing, 22% accuracy gain
- SCORE (Harvard, ICLR 2025 Workshop): Constrained optimization for model routing under cost/latency budgets
- Capy: Two-agent Captain/Build architecture with parallel cloud VMs - https://capy.ai
- Swarms swarm-mail + swarm_decompose: Event-sourced inter-agent messaging with file reservations - https://www.swarmtools.ai/docs/packages/opencode-plugin/swarm
- Agent Interlink Protocol (AIP, Feb 2026): Open standard for cross-model agent communication
- Token Arbitrage (Sep 2025): Cloud provider cost-optimized routing within orchestration layers
- Ian Paterson: "LLM Benchmark 2026" -- "Routing beats model selection" as the key finding - https://ianlpaterson.com/blog/llm-benchmark-2026-38-actual-tasks-15-models-for-2-29/
- Anthropic: "2026 Agentic Coding Trends Report" - https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf
- DAG scheduling theory: Topological sort, critical antichain, parallelism width (Dilworth's theorem)
- OpenAI Symphony (Apache-2.0, Mar 2026): Daemon service for autonomous issue resolution
- Emergent Mind: "LLM Difficulty Prediction" topic survey - https://www.emergentmind.com/topics/llm-based-difficulty-prediction
