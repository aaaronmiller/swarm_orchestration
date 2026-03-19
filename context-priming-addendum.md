---
date: 2026-03-17 17:15:00 PST
ver: 1.0.0
author: Sliither (for Ice-ninja)
model: claude-opus-4-6
tags: [context-priming, context-engineering, compaction, hierarchical-memory, curator-model, multi-block-context, user-intent, orchestration]
---
# Addendum: Context Priming Architecture

An architectural spec for replacing dumb compaction with intelligent, multi-block context curation driven by a secondary monitoring model.

## The Problem with Current Compaction

Current compaction (Claude Code's `/compact`, OpenHands' LLMSummarizingCondenser, Google ADK's SlidingWindowCompactor) follows a single pattern: when the context window fills up, summarize everything into a smaller block and continue. This has four failure modes:

**1. Uniform Lossy Compression.** Compaction treats all context equally. A critical architecture decision made 50 turns ago gets the same summarization weight as a trivial file read. The result: subtle but critical context drops out. Anthropic's own engineering blog acknowledges this: "overly aggressive compaction can result in the loss of subtle but critical context whose importance only becomes apparent later."

**2. Single Summary Block.** The compacted output is a single narrative summary. It contains everything from user intent to file paths to error traces, all flattened into one text blob. Different agent roles (planner vs. implementor vs. reviewer) need different slices of that context, but they all get the same undifferentiated summary.

**3. No Intent Preservation.** Compaction captures what happened but not why the user wanted it. The running thread of user intent (what the user is trying to accomplish across multiple commands) gets dissolved into action summaries. After compaction, the agent knows "we edited auth.ts" but not "the user is building a JWT refresh token flow because their existing session management leaks memory."

**4. Task Structure Loss.** The relationship between tasks (which are done, which are blocked, which can be parallelized) exists only implicitly in the conversation history. Compaction flattens this into a narrative, destroying the DAG structure that an orchestrator needs.

Ice-ninja's insight: a carefully curated context window, shaped by a secondary monitoring model, consistently outperforms a clean but dumb compaction. Certain tasks performed in certain orders provide structure and intent that would otherwise be missing.

## Prior Art: Who's Already Building Pieces of This?

### Confucius Code Agent (Meta/Harvard, Dec 2025)

The closest existing implementation. The Confucius SDK introduces **hierarchical working memory** with an **"Architect" agent** that monitors context and produces structured summaries. Key design:

- When the context window approaches configurable thresholds, the Architect agent (not the coding agent itself) summarizes earlier turns into a **structured plan** containing goals, decisions made, open TODOs, and critical error traces.
- The system replaces marked historical messages with this compressed summary while maintaining a **rolling window of recent messages in their original form**.
- A separate **note-taking agent** distills interaction trajectories into persistent, hierarchical Markdown notes, including **hindsight notes** that capture failure modes.
- Result: on 151 SWE-Bench-Pro tasks, reusing structured notes reduced token usage from ~104K to ~93K and increased resolve rate from 53.0% to 54.4%.

This validates Ice-ninja's core thesis: a curated summary produced by a dedicated secondary agent outperforms raw compaction.

### Anthropic's Structured Note-Taking (Feb 2026)

Anthropic's context engineering research formalizes three strategies: compaction, structured note-taking, and multi-agent architectures. Their key finding: "note-taking via a scratchpad allows agents to save information outside of the context window so that it's available to the agent." Their Pokemon-playing Claude Sonnet 4.5 demo showed the agent spontaneously developing maps, tracking objectives ("for the last 1,234 steps I've been training my Pokemon in Route 1"), and maintaining strategic notes that survived compaction. Without any prompting about memory structure, it self-organized persistent state.

### Factory.ai's Context Stack (2025-2026)

Factory treats context as a scarce, high-value resource with a "progressive distillation" pipeline: Repository Overviews (project structure, key packages, build commands) -> Semantic Search (pull relevant code at query time) -> User Style Context (individual preferences, organizational norms) -> Task Descriptions (what needs to be accomplished). Their explicit framing: "every byte of context serves a purpose, directly supporting more reliable and efficient agentic workflows." Chroma's research (Hong et al., 2025) found that "models do not use their context uniformly; their performance grows increasingly unreliable as input length grows."

### Google ADK's Tiered Context Architecture (Dec 2025)

Google's Agent Development Kit (ADK) separates **durable state (Sessions)** from **per-call views (working context)**. Three design principles: (1) Separate storage from presentation, (2) Explicit transformations via named, ordered processors, (3) Scope by default (every model call sees minimum context required). This is the most architecturally rigorous existing approach to the "what does each agent need to see?" question.

### OpenAI's State-Based Memory (2026)

Their context personalization cookbook formalizes: state object = local-first memory store (structured profile + notes). Distill memories during a run (tool call -> session notes). Consolidate session notes into global notes (dedupe + conflict resolution). Inject a well-crafted state at the start of each run (with precedence rules: latest user input -> session overrides -> global defaults).

### JetBrains Research: Observation Masking (Dec 2025, NeurIPS)

Found that **observation masking** (preserving the agent's reasoning and actions but compressing only the environment observations like file contents and tool outputs) outperforms full LLM summarization for coding agents, because a typical SE agent's turns heavily skew towards observation. A hybrid approach (observation masking + targeted summarization) achieved significant cost reduction while maintaining performance.

### Anthropic's "Effective Harnesses" Blog (2026)

Describes two critical failure modes of naive compaction: (1) The agent runs out of context mid-implementation, leaving a half-implemented feature with no documentation for the next session; the next agent "has to guess at what had happened." (2) After progress has been made, a later agent looks around, sees existing code, and declares the job done prematurely. Their solution: externalize state into scratchpad files (TODO.md, ARCHITECTURE.md) that survive compaction.

## The Context Priming Architecture

### Core Concept

A **Context Curator** (secondary model, cheaply run on Haiku or a local model) monitors the primary agent's conversation in real-time and maintains multiple structured context blocks. When compaction triggers, the Curator provides pre-structured blocks instead of the primary agent summarizing its own conversation.

This is fundamentally different from self-summarization. A model summarizing its own conversation is biased toward recency and salience. A separate observer model can maintain objective structural awareness of the entire project state.

### The Multi-Block Context System

Instead of one compacted summary, the Curator maintains **N specialized context blocks**, each serving a different purpose and potentially served to different agent roles:

#### Block 1: User Intent Register (UIR)

**What it contains:**
- Running summary of what the user is trying to accomplish (not verbatim commands, but inferred goals)
- Current high-level objective and how it relates to the project
- User's stated and inferred preferences (coding style, framework choices, error handling philosophy)
- Pending user requests that haven't been addressed yet
- Contradictions or ambiguities in user instructions (flagged for clarification)

**Example:**
```markdown
## User Intent Register
### Current Objective
Building a JWT refresh token flow for the auth module. Motivated by
memory leaks in the existing session management (user mentioned this
in turn 12). User wants silent token refresh with no UX interruption.

### Inferred Preferences
- Prefers functional patterns over class-based (observed in 3 code reviews)
- Wants error messages to be user-facing, not developer-facing
- Strongly dislikes any authentication state stored in localStorage

### Pending Requests
- [ ] "Make sure the refresh flow works with the existing middleware" (turn 47)
- [ ] "Add rate limiting to the token endpoint" (turn 52, mentioned but not started)

### Ambiguities
- User said "use Redis for sessions" in turn 15 but "keep it simple, no external deps"
  in turn 38. Likely means: use Redis only if already in the stack.
```

**Who gets it:** Orchestrator/planner model (always), coding model (on request), reviewer model (for alignment checking).

**Update frequency:** After every user message. The Curator infers intent from user commands and updates the register.

#### Block 2: Project State Block (PSB)

**What it contains:**
- Project description (auto-generated and refined over time)
- Architecture overview (key files, modules, data flow)
- Task list with status, dependencies, and parallelization markers
- Intelligence scores per task (for model-tier routing)
- Recently completed tasks (for context on what changed)
- Known anti-patterns specific to this project
- Common variables, paths, configuration values, and environment details

**Example:**
```markdown
## Project State
### Architecture
- Framework: SvelteKit + Hono + Bun
- Auth: src/lib/auth/ (JWT flow, refresh tokens)
- API: src/routes/api/ (Hono handlers)
- DB: src/lib/db/ (Drizzle ORM, PostgreSQL)

### Task DAG
- [x] T1: Create token generation utility (Haiku, score: 3)
- [x] T2: Implement refresh endpoint (Sonnet, score: 5)
- [ ] T3: Add middleware integration (Sonnet, score: 6) [blocked by T2]
- [ ] T4: Rate limiting on token endpoint (Sonnet, score: 5) [parallel with T3]
- [ ] T5: E2E test suite (Haiku, score: 3) [after T3, T4]

### Anti-patterns (this project)
- DO NOT use Express-style middleware; Hono uses a different pattern
- DO NOT import from $lib/server in client components
- The existing auth.ts has a circular dependency with user.ts; route through types.ts

### Environment
- Node: via Bun (not Node.js directly)
- Package manager: pnpm
- Test runner: vitest
- Deploy target: Cloudflare Workers
```

**Who gets it:** Orchestrator (always for task assignment), coding model (filtered to relevant section), subagents (their assigned task + dependencies only).

**Update frequency:** After every task completion or architecture-changing edit. The Curator watches for file creates/deletes, dependency changes, and config modifications.

#### Block 3: Task Execution Block (TEB)

**What it contains:**
- Current task description and acceptance criteria
- Relevant code context (only the files/functions needed for THIS task)
- Previous attempts and why they failed (hindsight notes, from Confucius pattern)
- Related completed task summaries (what was done, what interfaces were created)
- Parallelization information (what other agents are working on, what files they own)
- Known gotchas from prior tasks in this area

**Example:**
```markdown
## Current Task: T3 - Add middleware integration
### Acceptance Criteria
- Refresh token middleware runs before all /api/* routes
- Silently refreshes expired access tokens using refresh token from httpOnly cookie
- Returns 401 only when refresh token is also expired
- Does NOT run on /api/auth/login or /api/auth/register

### Relevant Code
- src/lib/auth/tokens.ts (created in T1: generateAccessToken, generateRefreshToken, verifyToken)
- src/lib/auth/refresh.ts (created in T2: handleRefresh endpoint)
- src/routes/api/+hooks.server.ts (WHERE TO ADD middleware)

### Previous Attempts
- Attempt 1 (turn 34): Used Express-style next() pattern. FAILED because
  Hono uses c.next() and the context object is different.

### Parallel Work
- Agent B is working on T4 (rate limiting) in src/lib/auth/ratelimit.ts
  DO NOT edit this file. Agent B will export rateLimiter for T3 to import.
```

**Who gets it:** The specific coding agent assigned to this task. This is the "hot" context that drives immediate work.

**Update frequency:** At task assignment and after every failed attempt.

#### Block 4: Conversation Archaeology Block (CAB)

**What it contains:**
- Compressed history of significant decisions and their rationale
- Key code snippets that were discussed but not yet implemented
- Resolved debates (so the agent doesn't re-litigate settled decisions)
- Links to relevant external resources mentioned in conversation
- Timeline of major events (what happened when, for orientation after compaction)

**Who gets it:** Any agent that needs to understand "why did we do it this way?" Low priority block; loaded on demand rather than always present.

**Update frequency:** Continuously, but with aggressive deduplication.

### How the Curator Works

```
User Message
    |
    v
+-------------------+
| Primary Agent     |  (Opus/Sonnet: does the actual work)
| (visible session) |
+-------------------+
    |  conversation stream (read-only tap)
    v
+-------------------+
| Context Curator   |  (Haiku or local model: monitors and structures)
| (shadow process)  |
+-------------------+
    |
    |--- Updates UIR after every user message
    |--- Updates PSB after task completions and file changes
    |--- Updates TEB when tasks are assigned or fail
    |--- Updates CAB continuously with deduplication
    |
    v
+-------------------+       +-------------------+
| Block Store       |       | Compaction Hook   |
| (structured       | <---- | (PreCompact hook  |
|  markdown files)  |       |  in Claude Code)  |
+-------------------+       +-------------------+
    |
    v
On compaction trigger:
    1. Curator assembles relevant blocks for the current agent role
    2. Blocks replace the default compaction summary
    3. Primary agent continues with curated context, not lossy summary
```

### Implementation with Claude Code Hooks

Claude Code's PreCompact hook is the integration point. When compaction triggers:

```javascript
// hooks/pre-compact.js
// This hook fires before compaction. It reads the Curator's blocks
// and provides them as the compaction basis instead of raw summarization.

export default async function preCompact(context) {
  const uir = await readFile('.claude/context/user-intent.md');
  const psb = await readFile('.claude/context/project-state.md');
  const teb = await readFile('.claude/context/current-task.md');

  // Compose the primed context based on current role
  const primedContext = [
    '## User Intent\n' + uir,
    '## Project State\n' + psb,
    '## Current Task\n' + teb,
  ].join('\n\n');

  // Return this as the compaction basis
  return { preserveContent: primedContext };
}
```

The Curator itself runs as a background subagent on Haiku (cheap, fast) or as a local model via Ollama. It reads the conversation stream (via session logs or a hook on every message) and updates the block store files.

### Why Multiple Blocks, Not One Summary

The key insight: **different agent roles need different context slices.**

| Agent Role | Blocks Served | Why |
|-----------|--------------|-----|
| Orchestrator/Planner | UIR + PSB | Needs to know what user wants and what's left to do; doesn't need code details |
| Implementation Agent | TEB + (relevant PSB subset) | Needs current task, code context, and anti-patterns; doesn't need full user intent history |
| Review Agent | UIR + TEB + CAB | Needs to verify implementation matches user intent and understand past decisions |
| Debug Agent | TEB + CAB + error-relevant PSB | Needs failed attempts, error traces, and project-specific gotchas |

This is the "compiled view" pattern from Google ADK applied to curated context blocks.

### The Reusability Property

A primed context block, once curated, can be reused across multiple compaction events and even across sessions. The Project State Block (PSB) changes slowly (only when architecture changes). The User Intent Register (UIR) evolves but carries forward. Only the Task Execution Block (TEB) is truly ephemeral.

This means: if you start a new session tomorrow on the same project, the Curator loads the stored blocks and the agent begins with a context window that already contains the project's architecture, known anti-patterns, user preferences, and task state. No "catching up" period. No re-reading CLAUDE.md and hoping the agent infers the right thing.

## Integration with Multi-Agent Orchestration

Context Priming directly enhances the Super Orchestrator architecture from the main report:

**Task Decomposition:** The PSB's task DAG with intelligence scores feeds directly into the wave scheduler. The Curator keeps this updated as tasks complete, so the orchestrator always has a current view.

**Agent Assignment:** Each spawned agent receives only the blocks relevant to its role. Implementation agents get TEB + PSB subset. Review agents get UIR + TEB. This is minimum-necessary context, which is Google ADK's "scope by default" principle.

**Cross-Agent Knowledge Transfer:** When Agent A completes T2 and Agent B needs T2's output for T3, the Curator updates T3's TEB with T2's results before Agent B starts. No orchestrator relay needed; no summarization loss.

**Cost Optimization:** The Curator runs on Haiku (~$0.03/task). The blocks it produces save the primary Opus/Sonnet agent from wasting tokens on re-discovering project structure, re-inferring user intent, and re-reading files that haven't changed. Net effect: higher quality context at lower total token cost.

## Deliberative Refinement: V(8,3,1) Assessment

**Profile:** V(8,3,1) Standard | **Council:** Expert Council (7 agents) | **Mode:** REFINE | **Strategy:** LINEAR

### R1: Initial Assessment

**Agent 1 (Systems Architect):** The multi-block architecture is sound. Confucius CCA validates the "Architect agent as summarizer" pattern with measured results (10% token reduction, 1.4% resolve improvement). The block separation maps cleanly to Google ADK's "separate storage from presentation" principle. **Concern:** The Curator running as a background Haiku process reading conversation logs introduces latency. If compaction triggers before the Curator finishes updating, the blocks will be stale.

**Agent 2 (Context Researcher):** Anthropic's own blog validates that "the art of compaction lies in the selection of what to keep versus what to discard." The UIR (User Intent Register) is novel. No existing system explicitly tracks inferred user intent as a separate, persistent artifact. Closest analog is OpenAI's state-based memory with "precedence rules" (latest user input -> session overrides -> global defaults). **Enhancement:** UIR should also track user frustration signals and repeated requests (indicator that the agent isn't getting it).

**Agent 3 (Cost Analyst):** Running a Haiku curator alongside an Opus primary adds ~$0.10-$0.20 per session in Haiku costs. But the token savings from not re-reading files, not re-inferring project structure, and not losing critical decisions during compaction likely save 15-30% of total Opus tokens. Net positive ROI on any session longer than ~30 turns.

**Agent 4 (Practitioner):** The PreCompact hook integration is real and works today. Claude Code hooks fire on SubagentStart, SubagentStop, PreToolUse, and PreCompact. The gap is that there's no official "conversation stream tap" for the Curator to read from in real-time. Current workaround: the Curator reads session logs or uses `/stats` output. A cleaner integration would be a PostMessage hook.

**Agent 5 (Security):** No sensitive data concerns with the block store (it's local markdown files in .claude/context/). The Curator should NOT have write access to the codebase. It's read-only for conversation + file system state.

**Agent 6 (Adversarial):** "Isn't this just CLAUDE.md with extra steps?" No. CLAUDE.md is static, human-authored, and project-scoped. The Curator's blocks are dynamic, machine-generated, conversation-aware, and session-scoped. CLAUDE.md tells the agent how to behave. The Curator's blocks tell the agent what's happening right now and why.

**Agent 7 (Integration):** JetBrains' observation masking research suggests the TEB should preserve the agent's reasoning chain verbatim but compress environment observations. The Curator should not summarize the agent's own reasoning, only the observations (file reads, tool outputs, test results).

### R2: Refinement

Unanimous agreement on architecture. Two additions:
1. Add a **Frustration Detector** to the UIR that flags repeated user requests (the user asked for X three times means the agent isn't delivering X).
2. The TEB should use observation masking (JetBrains pattern): preserve reasoning chains, compress only environment outputs.
3. The block store should support semantic search (for the CAB block) so agents can query "why did we choose Hono over Express?" rather than reading the entire archaeology.

### R3: Convergence

**CONVERGED.** The Context Priming architecture is validated by: Confucius CCA (Architect agent as structured summarizer), Anthropic's context engineering research (scratchpad + note-taking), Google ADK (tiered context with scope-by-default), Factory.ai (progressive distillation), JetBrains (observation masking), and OpenAI (state-based memory with precedence rules). The novel contribution is the multi-block system with role-based serving and the explicit User Intent Register.

## Sources

- Anthropic: "Effective Context Engineering for AI Agents" (2026) - https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Anthropic: "Effective Harnesses for Long-Running Agents" (2026) - https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- Google Developers Blog: "Architecting Efficient Context-Aware Multi-Agent Framework for Production" (Dec 2025) - https://developers.googleblog.com/architecting-efficient-context-aware-multi-agent-framework-for-production/
- Confucius Code Agent (Meta/Harvard): arXiv 2512.10398 (Dec 2025)
- LangChain: "Context Engineering for Agents" (Oct 2025) - https://blog.langchain.com/context-engineering-for-agents/
- Factory.ai: "The Context Window Problem" - https://factory.ai/news/context-window-problem
- Will Larson: "Building an Internal Agent: Context Window Compaction" (Dec 2025) - https://lethain.com/agents-context-compaction/
- JetBrains Research: "Cutting Through the Noise: Smarter Context Management" (NeurIPS 2025) - https://blog.jetbrains.com/research/2025/12/efficient-context-management/
- Martin Fowler: "Context Engineering for Coding Agents" - https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html
- OpenAI: "Context Engineering for Personalization" (2026) - https://developers.openai.com/cookbook/examples/agents_sdk/context_personalization/
- Google ADK: Context Compression Docs - https://google.github.io/adk-docs/context/compaction/
- Chroma Research: "Context Rot" (Hong et al., 2025) - Measuring context utilization degradation across 18 LLMs
- SFEIR Institute: "Context Management Optimization Guide" (2026) - PreCompact hooks, structured prompt fidelity (92% vs 71%)
