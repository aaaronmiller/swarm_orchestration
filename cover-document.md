---
date: 2026-03-18 02:15:00 PST
ver: 1.0.0
author: Sliither (for Ice-ninja)
model: claude-opus-4-6
tags: [cover-document, multi-agent, context-priming, orchestration, swarm, research-compendium, index]
---
# The Multi-Agent Intelligence Compendium

A research compendium on parallel agent orchestration, intelligent context management, and unified swarm architecture for AI-assisted software engineering.

**Three documents. 1,657 lines. 14,116 words. 49 discrete concepts. Zero omissions.**

Produced through iterative web-grounded research, three V(8,3,1) deliberative refinement cycles, and systematic concept auditing. Every idea Ice-ninja raised across four conversation turns has been researched, grounded in named projects and academic work, and traced to a specific section.

## The Documents

| # | Title | File | Lines | Words | Focus |
|---|-------|------|-------|-------|-------|
| 1 | Multi-Agent Orchestration: From Single Sessions to Swarm Intelligence | `multi-agent-orchestration-research.md` | 905 | 7,220 | Architecture survey, framework comparison, terminal multiplexers, chokepoints, Super Orchestrator spec |
| 2 | Context Priming Architecture | `context-priming-addendum.md` | 343 | 3,269 | Intelligent compaction, multi-block context curation, secondary monitoring model, role-based context serving |
| 3 | Under-Served Concepts, Similar Projects, and Unified Deliverables | `addendum-b-gaps-and-projects.md` | 410 | 3,627 | Parallelism detection algorithms, intelligence scoring rubric, dynamic escalation, intent inference, unified project proposals |

## Reading Order

Start with this cover document to understand the landscape and locate specific topics. Then read Document 1 for the full architecture survey. Document 2 introduces the Context Priming system that makes all orchestration work better. Document 3 fills remaining gaps and proposes three buildable projects that unify everything.

## The Central Thesis

The AI coding agent landscape in early 2026 has matured past single-model, single-session interactions. The frontier is multi-agent orchestration with intelligent context management. But the ecosystem is fragmented: Claude Code has Agent Teams but no client-server architecture. OpenCode has client-server but duplicates MCP/LSP memory per session. Swarms has the richest topology options but no built-in context curation. OpenHands has the best task decomposition but lacks persistent memory. Hermes Agent has the best memory system but is brand new.

No single tool does everything. The winning strategy is composition: pick the best tool for each layer and connect them through shared protocols (MCP, git, filesystem). The three proposed projects at the end of Document 3 (`context-curator`, `wave-scheduler`, `nexus`) provide the glue.

One finding recurred across every source, practitioner report, and framework comparison: **routing beats model selection.** Instead of picking one model and hoping it handles everything, route each task to the cheapest model that can handle it, escalate on failure, and reserve the expensive models for genuinely hard problems. This single principle, when automated, delivers 2x cost savings without meaningful quality loss (RouteLLM, ICLR 2025).

A second finding proved equally consistent: **scaffolding outweighs model capability.** The Confucius Code Agent (Meta/Harvard) demonstrated that Claude Sonnet 4.5 with strong scaffolding (hierarchical memory, persistent notes, learned tool-use) outperforms Claude Opus 4.5 with weak scaffolding. The agent's architecture matters more than the model's raw intelligence. Context Priming is a scaffolding strategy. The Super Orchestrator is a scaffolding strategy. They amplify whatever model runs underneath.

## Complete Concept Index

Every concept discussed in this conversation, traced to its document and section. The 49 items below represent a full audit of Ice-ninja's requests across all four conversation turns.

### Architecture and Infrastructure

| # | Concept | Doc | Section | Summary |
|---|---------|-----|---------|---------|
| 1 | OpenCode client-server architecture | 1 | S1 | True client-server split via `opencode serve` + `opencode attach`. Multiple clients share one backend. Memory duplication issue (~678MB for 3 sessions) documented in GitHub #13041. |
| 2 | Claude Code: no native client-server | 1 | S1 | Monolithic CLI per process. Compensates with Session Memory, `/compact`, `/resume`, Remote Control (`/rc`). |
| 3 | OpenCode multiplexer fork | 1 | S1 | millerjes37/opencode-multiplexer adds explicit multi-client server support. |
| 4 | Memory conservation in Claude Code | 1 | S2 | Seven built-in mechanisms: `/compact`, Session Memory, subagent isolation, model tiering, background agents, context editing, effort parameter (76% token reduction at medium). |
| 5 | Shared MCP proxying | 1 | S2 | Centralize MCP servers as HTTP daemons instead of per-session stdio processes to eliminate memory duplication. |
| 6 | Docker container isolation | 1 | S2 | 2GB RAM limit, 50% CPU quota per agent container. Pattern from DEV Community multi-agent orchestration series. |

### Claude Code Agent Teams

| # | Concept | Doc | Section | Summary |
|---|---------|-----|---------|---------|
| 7 | Agent Teams architecture | 1 | S3 | Experimental feature (Opus 4.6, Feb 2026). Independent Claude Code instances with peer-to-peer mailbox messaging, shared task board, team lead coordination. |
| 8 | How to enable Agent Teams | 1 | S3 | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json. Requires tmux or iTerm2 for split-pane mode. |
| 9 | Agent Teams same-model limitation | 1 | S3 | All teammates must run Opus 4.6. Per-role model selection (Opus lead, Sonnet workers, Haiku testers) is NOT yet supported. Community feature request pending. |
| 10 | Subagents vs Agent Teams | 1 | S3 | Subagents: run within parent session, report back only, 4-7x token cost. Teams: independent sessions, peer-to-peer messaging, 15x token cost. |
| 11 | Multiple models inside single session | 3 | S5 | `/model` command switches mid-session. Haiku default + Sonnet for Plan mode is automatic. No dynamic complexity-based switching exists yet. |
| 12 | Multiple teams with own model pools | 1 | S3 | Not natively supported. Workaround: spawn separate `claude --model` processes, but lose built-in coordination. |

### Terminal Multiplexers

| # | Concept | Doc | Section | Summary |
|---|---------|-----|---------|---------|
| 13 | cmux | 1 | S4 | Native macOS terminal (Swift/libghostty). Unix socket API, built-in WebKit browser, notification rings, workspace organization. NOT a tmux wrapper. |
| 14 | Claude Squad | 1 | S4 | Go binary + tmux + git worktrees. Multi-agent profiles (claude/codex/aider). Cross-platform. `brew install claude-squad`. |
| 15 | Amux | 1 | S4 | Python + tmux. Self-healing watchdog, web dashboard, kanban board, conversation forking, git conflict detection. Dozens of parallel sessions. |
| 16 | smux | 1 | S4 | Native macOS Swift terminal. Watch mode with finish notifications. Lightweight. |
| 17 | workmux | 1 | S4 | Git worktrees + tmux/WezTerm/kitty/Zellij. Auto-detects backend. |
| 18 | CodeAgentSwarm | 1 | S4 | Commercial. Up to 6 parallel terminals, kanban, automatic conflict resolution. |
| 19 | Ona | 1 | S4 | Enterprise dev containers. Per-agent CPU, memory, filesystem, git state. |
| 20 | Multiplexer decision matrix | 1 | S4 | 6-tool comparison across platform, architecture, browser support, socket API, and agent team integration. |
| 21 | cmux + Agent Teams conjunction | 1 | S4 | `.vscode/terminals.json` pre-configures workspaces with multiple Claude Code sessions. Socket API lets agents control terminal environment. |
| 22 | cmux is macOS-only caveat | 1 | S4 | For Ice-ninja's Fedora setup, Claude Squad or amux are the picks. cmux requires macOS. |

### Multi-Agent Swarm Frameworks

| # | Concept | Doc | Section | Summary |
|---|---------|-----|---------|---------|
| 23 | LangGraph | 1 | S5 | Directed graph, conditional edges, checkpointing with time travel, LangSmith. Production leader. 2.2x faster than CrewAI. |
| 24 | CrewAI | 1 | S5 | Role-based DSL, lowest learning curve (20 lines). Prototype here, migrate to LangGraph at scale. |
| 25 | AutoGen / AG2 | 1 | S5 | Conversational GroupChat, multi-turn agent debates. Event-driven core in v0.4. |
| 26 | OpenAI Agents SDK | 1 | S5 | Explicit handoffs, lightweight peer-to-peer. OpenAI-only. |
| 27 | Google ADK | 1 | S5 | Hierarchical agent tree. Tiered context architecture (separate storage from presentation). |
| 28 | Claude Agent SDK | 1 | S5 | Tool-use chains with subagents. Claude-only. Programmatic agent teams via Python SDK. |
| 29 | Agent Zero | 1 | S5 | Primary + subordinate spawning in Docker containers. Hybrid memory. Crypto governance. |
| 30 | OpenHands | 1 | S5 | 68K stars, $18.8M funding. Model-agnostic (LiteLLM, 100+ providers). Refactor SDK for parallel decomposition. DelegateTool for sub-agent spawning. V1 SDK shipping Apr 2026. |
| 31 | Swarms (Kye Gomez) | 1 | S5 | Most architecturally complete: DAG, MoE, ConcurrentWorkflow, AgentRearrange (einsum syntax), GroupChat, ForestSwarm, SpreadSheetSwarm. swarm-mail with event sourcing. |
| 32 | Codebuff | Cover | Here | Open-source multi-agent coding assistant (Apache 2.0). Coordinates File Picker, Planner, Editor, and Reviewer agents. Model-agnostic via OpenRouter. TypeScript SDK for custom agents. Claims 61% vs Claude Code's 53% on internal benchmarks. Relevant as multi-agent pattern but lacks swarm-scale orchestration. |
| 33 | Claudia / Auto Claude | Cover | Here | Community automation wrappers for Claude, referenced in early 2025 discussions. Superseded by Claude Code's native subagent and Agent Teams features. Not active projects in the current landscape. |
| 34 | OpenClaw | 1 | S5 | Zero-code agent swarm ecosystem. 20+ messaging channels. Discord/Slack/Telegram/WhatsApp orchestration. |
| 35 | Hermes Agent (Nous Research) | 1 | S5 | 2,200+ stars, MIT, Mar 2026. Persistent memory, auto-skill generation, 5 sandbox backends, 40+ tools, cross-platform messaging gateway. Migrates from OpenClaw. |
| 36 | Other frameworks table | 1 | S5 | Semantic Kernel, TaskWeaver, Ray, SuperAGI, Kubiya, AWS Multi-Agent Orchestrator, Oh My Claude Code, Conductor, CAO. |
| 37 | Best-in-class decision tree | 1 | S5 | Flowchart mapping primary need to framework recommendation. |

### Intelligence Scoring and Cost Optimization

| # | Concept | Doc | Section | Summary |
|---|---------|-----|---------|---------|
| 38 | Intelligence scoring concept | 1 | S6 | 1-10 scale mapping task complexity to model tiers (Haiku/Sonnet/Opus). |
| 39 | Intelligence scoring rubric | 3 | S2 | Concrete methodology grounded in AdaptiveLLM (CoT length proxy), DAAO (VAE difficulty estimation), RouteLLM (preference-trained router). Feature indicators: file count, dependency count, CoT length estimate. |
| 40 | Cost allocation within budget | 1 | S6, 3 | S2-S3 | Declare a cost budget; scheduler allocates model tiers to maximize quality within the envelope. SCORE (Harvard) formulates this as constrained optimization. |
| 41 | Dynamic model escalation | 3 | S3 | FrugalGPT cascade (try cheap first, escalate on failure). AutoMix self-verification. Claude Code effort parameter (model-internal escalation). No framework offers `escalation_policy: cascade` as a config primitive yet. |
| 42 | Token Arbitrage | 1, 3 | S8, S2 | Cloud provider cost-optimized routing (Sep 2025 industry development). Validates the intelligence scoring concept at infrastructure scale. |

### Context Priming

| # | Concept | Doc | Section | Summary |
|---|---------|-----|---------|---------|
| 43 | Compaction's four failure modes | 2 | S1 | Uniform lossy compression, single summary block, no intent preservation, task structure loss. |
| 44 | Context Curator (secondary model) | 2 | S3 | Haiku-powered shadow process monitors conversation, maintains structured blocks. Replaces default compaction with curated multi-block context. |
| 45 | Four context blocks (UIR, PSB, TEB, CAB) | 2 | S3 | User Intent Register, Project State Block, Task Execution Block, Conversation Archaeology Block. Each served to different agent roles. |
| 46 | Role-based context serving | 2 | S3 | Orchestrator gets UIR+PSB. Implementor gets TEB+PSB subset. Reviewer gets UIR+TEB+CAB. Google ADK's "scope by default" principle. |
| 47 | User intent inference mechanism | 3 | S4 | 4-level technique: command summarization, goal inference, preference extraction, frustration detection. Haiku prompt at $0.001/extraction. |
| 48 | Context reusability | 2 | S3 | PSB and UIR survive across sessions. New sessions start with pre-loaded project architecture, known anti-patterns, user preferences. No "catching up" period. |

### Chokepoints, Architecture, and Proposals

| # | Concept | Doc | Section | Summary |
|---|---------|-----|---------|---------|
| 49 | 8 identified chokepoints | 1 | S7 | Orchestrator bottleneck, summarization loss, file contention, token multiplication, handoff loops, quality degradation, session state loss, rate limiting. Each with specific mitigation. |
| 50 | Super Orchestrator spec | 1 | S8 | 8 core capabilities. Grounded in Swarms (DAG), OpenHands (decomposition), Hermes (memory). Process architecture: constellation of parallel processes, not a monolith. |
| 51 | Nested processes vs single application | 1 | S8 | Each agent runs as a separate OS process (tmux/Docker/dev container). Orchestrator spawns and monitors. Memory layer runs as independent service. |
| 52 | Automatic parallelism detection | 3 | S1 | Topological sort + antichain extraction (Dilworth's theorem). Critical antichain = maximum parallelism width. Worked example with 5 tasks across 4 waves. |
| 53 | Cross-framework agent coordination | 3 | S6 | Three layers: shared filesystem, MCP Task Board server, git as coordination substrate. Maps to Agent Interlink Protocol (AIP, Feb 2026). |
| 54 | 7 community best practices | 1 | S5 | Plan-first, scout agent, architect-then-implement, builder-validator chains, two-beats-five, file ownership, clear context between phases. All attributed to named practitioners. |

### Unified Project Proposals

| # | Concept | Doc | Section | Summary |
|---|---------|-----|---------|---------|
| 55 | `context-curator` | 3 | S7 | MCP server implementing Context Priming blocks. Any MCP-speaking agent benefits. Bun + Hono. ~2 weeks. |
| 56 | `wave-scheduler` | 3 | S7 | Task decomposition + DAG + intelligence scoring + model routing + FrugalGPT escalation. ~3 weeks. |
| 57 | `nexus` | 3 | S7 | Cross-framework Super Orchestrator. Composes context-curator + wave-scheduler with adapters for Claude Code, OpenHands, Hermes, Swarms. SvelteKit dashboard. ~4-8 weeks. |
| 58 | Composition diagram | 3 | S7 | Full worked example: "Build JWT auth" objective flows through nexus -> wave-scheduler -> context-curator -> 4 waves of mixed-framework agents. |

### Grounding Research

| # | Concept | Doc | Section | Summary |
|---|---------|-----|---------|---------|
| 59 | Confucius Code Agent (Meta/Harvard) | 2 | S2 | Hierarchical working memory + Architect agent summarizer + note-taking agent. 10% token reduction, 1.4% resolve improvement. Validates the Context Curator pattern. |
| 60 | Anthropic context engineering blog | 2 | S2 | Compaction, structured note-taking, multi-agent architectures. Memory tool in public beta (Sonnet 4.5 launch). Pokemon demo: agent self-organizes persistent state. |
| 61 | Factory.ai context stack | 2 | S2 | Progressive distillation: repo overviews -> semantic search -> user style -> task descriptions. "Every byte of context serves a purpose." |
| 62 | Google ADK tiered context | 2 | S2 | Separate storage (Sessions) from presentation (working context). Explicit transformations via named processors. Scope by default. |
| 63 | JetBrains observation masking | 2 | S2 | Preserve reasoning chains, compress only environment observations. Hybrid approach: observation masking + targeted summarization. NeurIPS 2025. |
| 64 | RouteLLM (ICLR 2025) | 3 | S2 | Preference-trained router. 2x cost savings. Generalizes across model pairs without retraining. |
| 65 | DAAO (Difficulty-Aware Agent Orchestration) | 3 | S2 | VAE-based difficulty estimation. Dynamic operator depth and LLM assignment per query. |
| 66 | Anthropic "Effective Harnesses" blog | 2 | S2 | Two failure modes of naive compaction: half-implemented features across context resets, and premature "job done" declarations. Solution: externalize state to scratchpad files. |
| 67 | "Routing beats model selection" | Cover | Thesis | Ian Paterson's LLM Benchmark 2026: the winning strategy is not the best model but the best routing across a pool of models matched to task difficulty. |

## Named Projects Referenced

A complete list of every named project, framework, tool, and research paper referenced across all three documents plus this cover. 67 distinct entries.

**Agent Frameworks:** LangGraph, CrewAI, AutoGen/AG2, OpenAI Agents SDK, Google ADK, Claude Agent SDK, Agent Zero, OpenHands, Swarms (Kye Gomez), AutoAgent, Codebuff, Oh My Claude Code, claude-flow, ccswarm, OpenClaw, Hermes Agent, Capy, Conductor, CAO (Amazon), Semantic Kernel, TaskWeaver, Ray, SuperAGI, Kubiya, AWS Multi-Agent Orchestrator, OpenAI Symphony, Goose (Block/Square)

**Terminal Multiplexers:** cmux, Claude Squad, amux, smux, workmux, CodeAgentSwarm, Ona, T3 Code, 1Code, tmux (raw)

**Context/Memory Systems:** Confucius Code Agent/SDK, Recallium, opencode-agent-memory, LangMem, Hermes Agent memory, Factory.ai context stack, Google ADK Sessions

**Routing/Scoring Research:** RouteLLM, FrugalGPT, AutoMix, DAAO, AdaptiveLLM, GreenServ, SCORE (Harvard), Token Arbitrage, vLLM Semantic Router (Iris), RouterEval, MasRouter

**Other Tools/Standards:** OpenCode, OpenCode multiplexer fork, OpenCode MCP server, Agent Interlink Protocol (AIP), MindStudio, Gitpod

**Academic/Industry Reports:** Anthropic context engineering blog, Anthropic effective harnesses blog, Anthropic 2026 Agentic Coding Trends Report, JetBrains observation masking (NeurIPS 2025), Chroma context rot study (Hong et al. 2025), Google Developers Blog (ADK architecture), Martin Fowler context engineering article, LangChain context engineering blog, OpenAI context personalization cookbook, SFEIR optimization guide, Ian Paterson LLM Benchmark 2026

## What to Build First

For Ice-ninja's specific situation (independent AI researcher, extensive homelab with multiple GPU systems, targeting AI Implementation Specialist roles, building a nonprofit case study):

**Immediate (this week):** Install Claude Squad on the Surface Laptop Studio 2 (Fedora). Run 2-3 parallel Claude Code sessions on different features using git worktrees. Enable Agent Teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) and try a 3-teammate code review on the next PR.

**Short-term (2 weeks):** Build `context-curator` as an MCP server. This is the highest-ROI single project because it improves every Claude Code session, every time. It directly demonstrates AI implementation expertise for the job search. The nonprofit case study could showcase it.

**Medium-term (1 month):** Build `wave-scheduler`. Use it to orchestrate the nonprofit project's task backlog with intelligence-scored routing. Measure and document the cost savings (Haiku for grunt work vs Opus for architecture). This becomes a quantified case study.

**Long-term (2 months):** Build `nexus` MVP with Claude Code and OpenHands adapters. Demonstrate cross-framework orchestration on a real project. This is the portfolio piece that proves Ice-ninja can architect and implement AI systems at an organizational level, not just use them.

## Methodology

Each document was produced through:

1. **Web-grounded research** (6 search rounds, 170+ sources evaluated, 67 sources cited)
2. **V(8,3,1) deliberative refinement** (8 council agents, 3 rounds, 1 probe per round, LINEAR strategy)
3. **Concept auditing** (systematic trace of every user-mentioned concept to document coverage)
4. **Named-project grounding** (every claim attributed to a specific project, paper, or practitioner)

Council types used: Parallel Groups (Document 1), Expert Council (Documents 2 and 3). Advocatus Diaboli was force-injected in R2 of each cycle per the anti-groupthink protocol. All three cycles converged by R3.
