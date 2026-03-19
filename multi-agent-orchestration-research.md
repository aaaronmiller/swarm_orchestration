---
date: 2026-03-17 15:45:00 PST
ver: 2.0.0
author: Sliither (for Ice-ninja)
model: claude-opus-4-6
tags: [multi-agent, claude-code, orchestration, swarm, subagents, agent-teams, terminal-multiplexer, task-decomposition, model-routing, super-orchestrator, cmux, hermes-agent, openhands]
---
# Multi-Agent Orchestration: From Single Sessions to Swarm Intelligence

A comprehensive research report covering Claude Code parallel execution, terminal multiplexers, agent swarm frameworks, model-tier routing, and a proposed "Super Orchestrator" architecture.

## 1. OpenCode's Client-Server Architecture vs Claude Code

### OpenCode: True Client-Server Separation

OpenCode implements a genuine client-server split. When you run `opencode`, it starts both a TUI (client) and an HTTP server (backend). The server exposes an OpenAPI 3.1 spec, meaning any compatible client can connect to it.

**Key commands:**

```bash
# Start a standalone server
opencode serve --hostname 0.0.0.0 --port 4096

# Connect a TUI client to a remote server
opencode attach http://opencode.local:4096

# Web interface + TUI sharing the same sessions
opencode web
```

The architecture permits multiple clients hitting a single backend, which means you can drive one server from a browser, a TUI, and an IDE plugin simultaneously, all sharing session state. An mDNS feature lets you broadcast the server on your local network so other machines can discover it automatically.

**Memory implications:** Each OpenCode session still spawns its own MCP servers and LSP instances. A GitHub issue (anomalyco/opencode#13041) documents that 3 concurrent sessions on the same codebase produced ~678MB of duplicated MCP process memory and ~160MB in duplicated LSP instances. The client-server split helps with UI overhead, but the per-session tool-chain duplication remains a real memory problem.

A fork called `opencode-multiplexer` (millerjes37/opencode-multiplexer on GitHub) adds explicit multi-client server support where multiple TUI clients connect to a single persistent server.

### Claude Code: No Native Client-Server Split

Claude Code does not implement a client-server architecture. Each `claude` process is a monolithic CLI that manages its own context window, API connections, and tool invocations. There is no `claude serve` command or HTTP API to attach multiple frontends to.

**What Claude Code does offer instead:**

- **Session Memory:** Automatic cross-session context persistence (available on Anthropic's first-party API, Pro/Max subscriptions). Sessions write structured summaries including status, completed work, and a work log. New sessions load relevant past summaries as background knowledge.
- **`/compact` command:** Instant context compaction that loads a summary into a fresh context window, reclaiming token space without losing continuity.
- **`/resume` and `/rename`:** Resume prior sessions by name with `claude --resume session-name`.
- **Remote Control:** The `/rc` command generates a QR code for monitoring sessions from a phone or other device, but this is observation-only, not a multi-client architecture.

**Bottom line:** OpenCode wins on architectural flexibility for multi-client scenarios. Claude Code compensates with richer native orchestration (subagents, agent teams) and session memory.

## 2. Memory Conservation in Claude Code

### Built-in Mechanisms

| Method | What It Does | Token Savings |
|--------|-------------|---------------|
| `/compact` | Instant compaction; writes summary, loads into fresh context | Reclaims full context minus summary |
| Session Memory | Auto-recalls relevant past session summaries at startup | Avoids re-explaining project context |
| Subagent isolation | Each subagent gets a fresh context; only returns a summary to parent | Prevents verbose tool output from polluting main context |
| Model tiering | Route Explore subagents to Haiku (cheapest), implementation to Sonnet, planning to Opus | 40-50% cost reduction vs. uniform Opus |
| `run_in_background` | Background subagents via Ctrl+B; results surface asynchronously | Main session stays unbloated |
| Context editing | Models remove less relevant information while preserving what matters | Dynamic context optimization |
| Effort parameter (Opus 4.5+) | Set to "medium" effort: matches Sonnet performance with 76% fewer tokens | Dramatic token reduction on simpler tasks |

### Model Routing for Cost Control

Claude Code's subagent system permits per-agent model selection via YAML frontmatter:

```yaml
---
name: code-reviewer
description: Expert code review specialist
model: sonnet     # Options: haiku, sonnet, opus, inherit, or full model ID
---
```

The recommended routing pattern:

- **Haiku:** Explore subagent (read-only codebase search), file searches, quick questions. ~$0.03/task.
- **Sonnet:** General implementation, standard coding, moderate complexity. ~$0.75/task.
- **Opus 4.6:** Complex reasoning, architecture decisions, agent team coordination. ~$2.00/task.

A common production pattern: run the main orchestrator on Opus for planning while subagents handle focused tasks on Sonnet or Haiku. This cuts costs significantly without sacrificing quality on well-scoped subagent work.

### External Memory Conservation

- **Git worktrees:** Isolate each agent's working directory so file locks and merge conflicts don't cascade.
- **Docker containers:** Limit each agent to 2GB RAM, 50% CPU quota (see DEV Community multi-agent orchestration pattern).
- **Shared MCP proxying:** Run MCP servers as centralized HTTP daemons rather than per-session stdio processes. This eliminates the duplication problem OpenCode suffers from.

## 3. Claude Code Agent Teams (The New Hotness)

### What They Are

Agent Teams shipped as an experimental feature with Opus 4.6 in February 2026. Unlike subagents (which run within a single session and report back), agent teams are fully independent Claude Code instances that:

- Each get their own context window (up to 1M tokens per teammate)
- Communicate peer-to-peer via a mailbox system (JSON messages appended to inbox files)
- Share a file-backed task board with states and dependencies
- Are coordinated by a "team lead" session

### How to Enable

```json
// In settings.json or environment:
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "CLAUDE_CODE_SPAWN_BACKEND": "tmux"
  },
  "teammateMode": "tmux"
}
```

Requires tmux (or iTerm2 on macOS) for split-pane display mode. Without tmux, agent teams still work in "in-process" mode (agents run but you can't see them in separate panes).

### Architecture

```
Team Lead (your main session)
 |
 |--- SharedTaskList (.claude/tasks/)
 |       task-1.lock  (Agent A claimed)
 |       task-2.lock  (Agent B claimed)
 |       task-3.pending
 |
 |--- Mailbox System (peer-to-peer messaging)
 |       inbox-agent-a.json
 |       inbox-agent-b.json
 |
 |--- Teammate A (own context, own pane)
 |--- Teammate B (own context, own pane)
 |--- Teammate C (own context, own pane)
```

Key operations: TeamCreate, TeamSpawn, TeamMessage, TeamTaskCreate, TeamTaskUpdate, TeamDelete.

### When to Use vs. When Not To

**Best use cases:**
- Research and review (multiple perspectives in parallel)
- New modules/features (each teammate owns separate files)
- Debugging with competing hypotheses
- Cross-layer coordination (frontend + backend + tests)

**Avoid for:**
- Sequential tasks with heavy dependencies
- Same-file edits (creates merge conflicts)
- Simple tasks (subagents are cheaper)

### Practical Findings from the Community

- **Keep teams small:** 2-3 agents with narrow scope consistently outperform larger teams with broad mandates.
- **Plan-first is non-negotiable:** Enforce `plan mode -> approval -> execution`. Letting agents jump to code without an approved plan produces garbage.
- **Clear context between phases:** Start fresh after each team run. Stale context bleeds across phases.
- **Token cost:** Agent teams use ~15x more tokens than single-agent sessions. On Max plan ($200/month), you get roughly 8-10 complex team tasks in a 5-hour window.
- **Model limitation:** As of March 2026, all agents in a team run the same model (Opus 4.6 required). Per-role model selection (Opus for planning, Sonnet for implementation, Haiku for testing) is a community feature request that is not yet supported.

### Subagents vs. Agent Teams: Decision Matrix

| Criteria | Subagents | Agent Teams |
|----------|-----------|-------------|
| Communication | Report back to parent only | Peer-to-peer messaging |
| Context | Share parent's context window | Independent context per agent |
| Inter-agent awareness | None | Shared task board + mailbox |
| Use case | Focused, self-contained work | Collaborative, cross-cutting work |
| Cost | Moderate (4-7x single session) | High (15x single session) |
| Model flexibility | Per-subagent model selection | Same model for all (Opus 4.6) |
| Observability | Limited | tmux panes, task list inspection |

## 4. Running Multiple Separate Claude Code Sessions

### Method 1: Raw Terminal Multiplexing

The simplest approach: open multiple terminals (or tmux panes) and run `claude` in each one, pointed at different directories or git worktrees.

```bash
# Create worktrees for isolation
git worktree add ../feature-auth feature-auth
git worktree add ../feature-dashboard feature-dashboard

# Terminal 1
cd ../feature-auth && claude

# Terminal 2
cd ../feature-dashboard && claude
```

Each session is fully independent. No coordination, no shared state. You manage context manually.

### Method 2: Claude Squad (`cs`)

**GitHub:** smtg-ai/claude-squad
**Install:** `brew install claude-squad` or `curl -fsSL https://raw.githubusercontent.com/smtg-ai/claude-squad/main/install.sh | bash`

Claude Squad is a TUI app that manages multiple Claude Code (and Codex, Aider, OpenCode) sessions. It uses tmux for isolated terminal sessions and git worktrees for codebase isolation.

Key features:
- Profile picker for different agents (claude, codex, aider)
- Auto-accept/YOLO mode for background completion
- Git worktree isolation per session
- Session preview and management from a single interface

```json
// ~/.claude-squad/config.json
{
  "default_program": "claude",
  "profiles": [
    { "name": "claude", "program": "claude" },
    { "name": "codex", "program": "codex" },
    { "name": "aider", "program": "aider --model ollama_chat/gemma3:1b" }
  ]
}
```

### Method 3: Amux (Agent Multiplexer)

**GitHub:** mixpeek/amux
**Install:** `git clone https://github.com/mixpeek/amux && cd amux && ./install.sh`

An open-source Claude Code agent multiplexer that runs dozens of parallel sessions unattended from your browser or phone. Features:

- Self-healing watchdog (auto-compacts context, restarts on corruption, unblocks stuck prompts)
- Web dashboard with session cards, live terminal peek, file explorer
- Shared kanban board for task coordination
- Conversation forking (clone session history to new branches)
- Git conflict detection with one-click isolation
- Built-in cron for scheduled commands
- PWA with offline support
- Single ~23,000-line Python file

```bash
amux register myproject --dir ~/Dev/myproject --yolo
amux start myproject
amux serve  # Web dashboard at https://localhost:8822
amux exec myproject -- "implement the auth module"
```

### Method 4: smux (macOS Native)

**Website:** smux-terminal.vercel.app

A native macOS terminal multiplexer built in Swift, specifically designed for running AI coding agents in parallel. Features watch mode that notifies you when an agent finishes. Lightweight, fast, free, and open source.

### Method 5: workmux

**GitHub:** raine/workmux

Tightly couples git worktrees with tmux window management. Supports tmux, WezTerm, kitty, and Zellij backends. Auto-detects the multiplexer from environment variables.

### Method 6: CodeAgentSwarm

**Website:** codeagentswarm.com

A commercial tool (works on top of Claude Code subscription) that manages up to 6 parallel terminals with a kanban board, task-to-terminal assignment, dynamic title updates, automatic conflict resolution, and notifications when agents finish or need attention.

### Method 7: Ona (Enterprise)

**Website:** ona.com

Enterprise-grade dev containers where each agent gets its own CPU, memory, file system, git state, and services. Uses the Dev Container spec with Claude Code feature integration.

### cmux (The Native macOS Agent Terminal)

**Website:** cmux.dev | **GitHub:** open source, free

cmux is a native macOS terminal application built on libghostty (Mitchell Hashimoto's terminal rendering engine). It is NOT a tmux wrapper or fork. It is a ground-up GUI terminal designed specifically for AI coding agent workflows.

**Architecture:** Native Swift + AppKit application with WebKit for embedded browser, BondSplit for layout management, and a Unix domain socket API for programmatic control. All inter-process communication happens via JSON messages over the socket, giving any agent harness (Claude Code, Codex, OpenCode, Aider, Gemini CLI, Kiro) zero-latency control of the terminal environment.

**Key differentiators from tmux-based solutions:**

- **No prefix keys or config files.** GUI-native vertical tabs, split panes, and workspaces.
- **Built-in browser.** Agents can open WebKit browser panes, navigate pages, inspect DOM, click elements, fill forms, and take screenshots without leaving the terminal. This collapses the "agent suggests URL -> you switch to Chrome -> you copy info back" loop.
- **Notification rings.** When a process in any pane needs attention, cmux shows visual rings around the pane, unread badges in the sidebar, a popover notification, and a macOS desktop notification. Fires automatically via standard OSC 9/99/777 escape sequences, or via Claude Code hooks.
- **Socket API for agent control.** Any agent can programmatically create panes, send commands, read screen content, open browser tabs, and report progress:

```bash
# Agent discovers its context
cmux identify --json
cmux list-workspaces
cmux list-panes

# Agent opens a browser split and inspects a page
cmux browser surface:1 open-split --direction right
cmux browser surface:2 navigate "https://localhost:3000"
cmux browser surface:2 snapshot --compact
cmux browser surface:2 click "button.submit"
cmux browser surface:2 screenshot --out /tmp/result.png

# Agent creates parallel work panes
cmux new-pane --direction down
cmux send surface:3 "npm test"
```

- **Workspace organization.** Window > Workspace (sidebar tab) > Pane (split region) > Surface (terminal tab in pane). Each workspace can hold a complete project context with multiple agents.
- **Memory efficiency.** Native macOS app avoids Electron overhead. When running resource-intensive AI models, cmux itself consumes minimal RAM compared to VS Code or Electron-based terminals.
- **Skills integration.** A community-maintained skill file (markdown with CLI examples) teaches agents how to use cmux proactively. The agent reads it at session start and can then split panes, fan out parallel builds, open browser verifications, and report progress without explicit user instruction.
- **Ghostty keybindings.** Reads from `~/.config/ghostty/config` for terminal shortcuts. cmux-specific shortcuts (workspaces, splits, browser, notifications) are customized in Settings.

**How it works with Claude Code Agent Teams:**

cmux's `.vscode/terminals.json` integration can pre-configure workspaces with multiple Claude Code sessions:

```json
{
  "workspace": "my-project",
  "tabs": [
    { "name": "Claude 1", "commands": ["cd ~/project", "claude --dangerously-skip-permissions"] },
    { "name": "Claude 2", "commands": ["cd ~/project", "claude --dangerously-skip-permissions"] },
    { "name": "Claude 3", "commands": ["cd ~/project", "claude --dangerously-skip-permissions"] },
    { "name": "Console", "focus": true, "commands": ["cd ~/project"] }
  ]
}
```

**vs. Claude Squad / amux:** Claude Squad and amux are tmux-based wrappers that work on any platform. cmux is macOS-only but provides a fundamentally richer integration surface (browser, socket API, native notifications). For Ice-ninja's Fedora setup, Claude Squad or amux remains the better choice. For macOS workflows, cmux is the current apex tool.

### Multiplexer Decision Matrix

| Tool | Platform | Architecture | Browser | Socket API | Agent Teams | Best For |
|------|----------|-------------|---------|------------|-------------|----------|
| **cmux** | macOS only | Native Swift/libghostty | Built-in WebKit | Yes (Unix socket) | Via terminals.json | macOS power users, browser-in-loop workflows |
| **Claude Squad** | Any (tmux) | Go binary + tmux + git worktrees | No | No | Manual | Cross-platform, multi-agent (claude/codex/aider) |
| **amux** | Any (tmux) | Python + tmux | Web dashboard | HTTP API | Kanban board | Unattended batch operations, remote monitoring |
| **smux** | macOS only | Native Swift | No | No | Watch mode notifications | Lightweight macOS agent parallelism |
| **workmux** | Any (tmux/WezTerm/kitty/Zellij) | Rust binary | No | No | Auto-detect backend | Git worktree-centric workflows |
| **tmux raw** | Any | C binary | No | No | Manual | Maximum flexibility, minimum abstraction |

## 5. Multi-Agent Swarm Frameworks: Comprehensive Survey

### Tier 1: Production-Ready Frameworks

#### LangGraph (by LangChain)
- **Architecture:** Directed graph with conditional edges, typed state channels
- **Strengths:** Built-in checkpointing with time travel, visual graph editor, LangSmith integration for debugging
- **Model support:** Fully model-agnostic
- **Task decomposition:** Native graph-based task routing with conditional branching
- **Cost optimization:** Route to cheaper models at specific nodes
- **Community:** Largest ecosystem; most production deployments
- **Benchmark:** 2.2x faster than CrewAI, most token-efficient (2,589 tokens vs 3,187 for LangChain)

#### CrewAI
- **Architecture:** Role-based crews with configurable process types (sequential, hierarchical)
- **Strengths:** Lowest learning curve (20 lines to start), built-in delegation, role-based DSL
- **Model support:** Fully model-agnostic
- **Task decomposition:** Define roles and tasks declaratively; crew handles sequencing
- **Limitations:** No built-in checkpointing, coarse-grained error handling, limited agent-to-agent direct messaging
- **Common trajectory:** Teams prototype with CrewAI, migrate to LangGraph at scale

#### Microsoft AutoGen / AG2
- **Architecture:** Conversational agent teams with GroupChat pattern
- **Strengths:** Multi-turn agent debates for output refinement, event-driven core (v0.4/AG2), pluggable orchestration
- **Model support:** Model-agnostic, includes specialized agents (WebSurfer, Coder)
- **Unique:** Agents interact through multi-turn conversations, not just task handoffs
- **Enterprise adoption:** Led by Microsoft; strong in regulated environments

#### OpenAI Agents SDK (Swarm)
- **Architecture:** Explicit handoffs between agents, lightweight peer-to-peer transfers
- **Model support:** OpenAI models only
- **Strengths:** Clean, opinionated API; easy to understand handoff model
- **Limitations:** Vendor lock-in, ephemeral context variables by default

#### Google ADK (Agent Development Kit)
- **Architecture:** Hierarchical agent tree
- **Model support:** Optimized for Gemini but supports others
- **Strengths:** Tight integration with Google Cloud services

#### Claude Agent SDK
- **Architecture:** Tool-use chains with subagents
- **Model support:** Claude models only
- **Strengths:** Native integration with Claude Code's tool ecosystem, MCP server support
- **Programmatic agent teams:** Can orchestrate agent teams via Python SDK

### Tier 2: Autonomous/Swarm-Oriented Frameworks

#### Agent Zero
- **Architecture:** Primary agent + subordinate spawning with specialized roles
- **Execution:** Each agent runs in an isolated Docker container (full Linux env)
- **Model support:** OpenAI, Anthropic, local models, Venice.ai
- **Task decomposition:** Primary agent decomposes complex tasks, assigns to subordinates with per-agent prompts, toolsets, and execution contexts
- **Memory:** Hybrid memory system (short-term + long-term)
- **Governance:** Open-source with crypto-based community governance (Ethereum BASE L2)
- **Best for:** Distributed autonomous workflows, not interactive development

#### OpenHands (formerly OpenDevin)
- **GitHub:** 68.6K+ stars, $18.8M Series A funding (All Hands AI)
- **Architecture:** Open platform for AI software developers as generalist agents with sandboxed Docker runtimes
- **Model support:** Model-agnostic via LiteLLM (100+ providers including Nous, DeepSeek, Qwen, Llama, Claude, GPT, Ollama local models). Prompt-based fallback for non-function-calling models.
- **Task decomposition:** Refactor SDK (private beta) with dependency analysis tools that break codebases into manageable pieces based on directory boundaries and inter-component dependencies. Fixers and verifiers define how code should be changed and validated.
- **Sub-agent delegation:** Parent agent spawns multiple sub-agents for parallel processing via DelegateTool. Each sub-agent runs independently with its own conversation context. Blocking parallel execution implemented as a standard tool in openhands.tools.
- **Context management:** LLMSummarizingCondenser replaces old conversation history with summaries to prevent context overflow (~2x cost reduction). Event sourcing via immutable event log enables replay, recovery, and incremental persistence.
- **Security:** LLM-based risk assessment (Low/Medium/High) with configurable confirmation policy for dangerous commands.
- **V1 SDK (March 2026):** Moving from mandatory Docker to optional sandboxing, LocalWorkspace by default. Skills can be loaded from .openhands/skills/ or compatible formats (.cursorrules, agents.md).
- **Benchmark:** 77.6% on SWE-bench Verified (own harness, Claude 3.5 Sonnet Thinking). Planning Agent lets you switch between Plan Mode and Code Mode with structured PLAN.md output.
- **Best for:** Model-agnostic parallel coding, enterprise refactoring at scale (1000s of parallel agent runs), teams needing Docker-sandboxed execution

#### Swarms (by Kye Gomez)
- **GitHub:** kyegomez/swarms | Enterprise-grade production-ready multi-agent orchestration
- **Architecture:** The most architecturally complete swarm framework. Implements ALL major orchestration patterns natively:
  - **SequentialWorkflow:** Linear chain; output of one agent becomes input for the next
  - **ConcurrentWorkflow:** Agents run tasks simultaneously for maximum throughput
  - **AgentRearrange:** einsum-inspired syntax for defining complex non-linear agent relationships (e.g., `a -> b, c` for fan-out)
  - **GraphWorkflow:** DAG-based orchestration with nodes as agents
  - **MixtureOfAgents:** Multiple expert agents in parallel, outputs synthesized (MoE pattern)
  - **GroupChat:** Conversational collaboration with dynamic speaker selection
  - **SpreadSheetSwarm:** Orchestrates agents with a director who creates plans and distributes tasks to specialized workers with feedback loops
  - **ForestSwarm:** Tree-of-agents for dynamic task routing and decision-making
- **Task decomposition:** Built-in `swarm_decompose` tool with strategy selection (file-based, feature-based, risk-based), CASS history queries, and CellTreeSchema validation. Detects file conflicts and instruction conflicts automatically.
- **Model support:** Fully model-agnostic
- **Inter-agent messaging:** swarm-mail package with file reservations backed by event sourcing. Agents can reserve file paths, send progress updates, and report blockers.
- **Best for:** Maximum flexibility in swarm topology. The closest existing framework to the "Super Orchestrator" vision. If you need DAG scheduling + MoE routing + concurrent execution, this is it.

#### AutoAgent
- **Architecture:** Zero-code LLM agent framework, draws from OpenAI Swarm + Magentic-One
- **Strengths:** Three-agent user mode design, fully automated agent construction

#### Oh My Claude Code
- **Architecture:** Five execution modes for Claude Code orchestration
- **Strengths:** Pre-built swarm patterns, beginner-friendly, one of the earliest community solutions

#### claude-flow
- **Swarm orchestration for Claude Code**
- Pre-dates native Agent Teams; uses git worktree isolation

#### ccswarm
- **Git worktree isolation** for parallel Claude Code instances
- Lightweight approach, pre-dates Agent Teams

### Tier 3: Chat-Platform Agent Swarms

#### OpenClaw
- **Architecture:** Zero-code autonomous AI agent swarm ecosystem
- **Channels:** Discord, Slack, Telegram, WhatsApp, Signal, Teams, Matrix, and 20+ more
- **Model support:** Claude Opus 4.6, GPT-5.1 Codex, Ollama local models, OpenRouter
- **Task delegation:** Multi-agent specialized teams with sub-agent patterns
- **Deployment:** AgentSwarm app as primary local runtime for chat-first orchestration
- **Best for:** Deploying agentic swarms managed via enterprise chat platforms

#### Hermes Agent (by Nous Research)
- **GitHub:** NousResearch/hermes-agent | 2,200+ stars, MIT license, launched March 2026
- **Architecture:** Persistent personal agent that lives on your server, builds memory across sessions, auto-generates reusable skills, and reaches you through multiple messaging platforms
- **Model support:** Nous Portal (OAuth), OpenRouter (200+ models), OpenAI, or any custom endpoint. Fully model-agnostic.
- **Task delegation:** `delegate_task` tool spawns isolated child agents with restricted toolsets and separate terminal sessions. Batch mode supports up to 3 parallel tasks. Can also spawn full Hermes instances as subprocesses for independent long-running work.
- **Persistent memory:** Multi-level system that mimics procedural learning. When Hermes solves a complex task, it synthesizes the experience into searchable markdown files following the agentskills.io open standard. Skills are auto-generated and retrieved when similar problems arise.
- **Sandboxing:** Five backends (local, Docker, SSH, Singularity, Modal). Docker containers get read-only root filesystems, dropped Linux capabilities, PID limits, and namespace isolation.
- **Messaging gateway:** Telegram, Discord, Slack, WhatsApp, Signal, and CLI from a single gateway. Start on one platform, pick up on another.
- **40+ built-in tools:** Web search, browser automation, vision, image generation, text-to-speech, code execution, subagent delegation, memory, task planning, cron scheduling, multi-model reasoning.
- **OpenClaw migration:** Auto-detects `~/.openclaw` and offers to migrate settings, memories, skills, and API keys. `hermes claw migrate` handles the transition.
- **Research integration:** Creates thousands of tool-calling trajectories in parallel with automatic checkpointing, exports in ShareGPT format for model fine-tuning, integrates with Atropos for RL on agent behaviors.
- **Planned features (from GitHub issues):** Symphony-style autonomous issue resolution (polls GitHub Issues, claims work, creates isolated workspaces), consensus/voting engine for multi-agent decisions (majority, supermajority, weighted, quorum), adversarial debate mode, and multi-agent workflow DAG engine.
- **Best for:** Developers wanting a persistent learning agent with cross-platform reach, no vendor lock-in, and the strongest open-source memory system available

### Frameworks Ice-ninja Was Missing

Beyond the ones mentioned (Agent Zero, OpenHands, Codebuff, Claudia, Auto Claude):

| Framework | Key Differentiator |
|-----------|-------------------|
| **LangGraph** | Graph-based state management, production leader |
| **CrewAI** | Easiest onboarding, role-based |
| **AutoGen/AG2** | Conversational multi-agent debates |
| **Google ADK** | Hierarchical agent trees, Gemini-optimized |
| **Semantic Kernel** | Microsoft enterprise orchestration engine |
| **TaskWeaver** | Coding-based task execution with modular decomposition |
| **Ray** | Distributed computing framework for scaling agent workloads |
| **SuperAGI** | Open-source agent operations platform |
| **Kubiya** | Secure, flexible agent operations for DevOps |
| **AWS Multi-Agent Orchestrator** | Cost-efficient routing with DynamoDB state persistence |
| **Oh My Claude Code** | 5-mode Claude Code orchestration |
| **Ona** | Enterprise dev containers for agent parallelism |
| **Conductor** | Free tool focusing on independent agent workspaces |
| **CAO (CLI Agent Orchestrator)** | Amazon's sophisticated MCP-based solution |

### Best-in-Class: What to Use When (Decision Tree)

```
START: What is your primary need?
 |
 |-- "I need production-grade orchestration with full control"
 |     -> LangGraph (graph workflows, checkpointing, LangSmith observability)
 |
 |-- "I want to prototype multi-agent fast"
 |     -> CrewAI (20 lines to start, role-based DSL)
 |     -> Graduate to LangGraph when you need checkpointing
 |
 |-- "I need Claude Code parallel execution"
 |     -> Agent Teams (native, experimental, requires Opus 4.6)
 |     -> Claude Squad/amux for manual parallelism on any model
 |
 |-- "I need model-agnostic coding agents at scale"
 |     -> OpenHands (68K stars, Refactor SDK, 1000s of parallel runs)
 |
 |-- "I want a persistent personal agent that learns"
 |     -> Hermes Agent (auto-skill gen, cross-platform messaging, 5 sandbox backends)
 |
 |-- "I need maximum swarm topology flexibility"
 |     -> Swarms by Kye Gomez (DAG, MoE, ConcurrentWorkflow, AgentRearrange)
 |
 |-- "I want agents debating and refining outputs"
 |     -> AutoGen/AG2 (GroupChat, multi-turn conversation)
 |
 |-- "I need chat-platform managed agent swarms"
 |     -> OpenClaw (20+ messaging channels, zero-code)
 |     -> Hermes Agent (migrates from OpenClaw, adds persistent memory)
 |
 |-- "I need enterprise compliance and governance"
 |     -> AWS Multi-Agent Orchestrator + DynamoDB
 |     -> Semantic Kernel (Microsoft ecosystem)
```

### Community-Validated Best Practices (Consensus Findings)

These patterns emerged from practitioners running parallel agents in production. They represent the current strongest methodology:

**1. Plan-First is Non-Negotiable.** Every successful multi-agent workflow starts with a plan phase. Enforce `plan mode -> human approval -> execution`. Letting agents jump to code without an approved plan produces garbage. The plan gives you a checkpoint before committing expensive tokens. (Sources: Michael Habib, alexop.dev, multiple Medium authors.)

**2. Scout Agent Pattern.** Before committing to a full implementation, send a read-only "scout" agent to explore the problem space. It finds the sticky bits (missing dependencies, unexpected file structures, API changes) so you don't waste full implementation agents on avoidable failures. (Source: Josh Bleecher Snyder, "The 7 Prompting Habits of Highly Effective Engineers.")

**3. Architect-Then-Implement.** Have an architect agent iterate on a plan, then fresh implementation agents receive the plan and execute. The architect never touches code; implementors never plan. This separation prevents context pollution. (Source: Jesse Vincent, "How I'm Using Coding Agents," September 2025.)

**4. Builder-Validator Chains.** Every implementation agent is paired with a review agent. Adversarial cross-model review (Claude reviews GPT's code, GPT reviews Claude's) catches more bugs than same-model review. (Source: Saurab Bhatia's 19-agent setup.)

**5. Two Agents Beat Five.** Small, focused teams (2-3 agents) consistently outperform larger teams with broad scope. Too many agents create coordination overhead that exceeds the parallelism benefit. (Source: Michael Habib's Kafka project experiment.)

**6. File Ownership is Mandatory.** If Agent A owns `src/api/` and Agent B owns `src/frontend/`, they never conflict. Two agents editing the same file is the single most common failure mode. Git worktrees enforce this at the filesystem level. (Source: unanimous across all Agent Teams practitioners.)

**7. Clear Context Between Phases.** After each team run, start fresh. Stale context from a prior phase bleeds into the next one and causes bizarre failures. (Source: Michael Habib, Heeki Park.)

## 6. Intelligence-Score-Based Model Routing

### The Concept

Assign each task in a decomposed todo list an "intelligence score" (1-10) representing the cognitive complexity. Map scores to model tiers:

| Score | Complexity | Model Tier | Approximate Cost |
|-------|-----------|------------|-----------------|
| 1-3 | File reads, grep, format checks | Haiku 4.5 | $0.03/task |
| 4-6 | Standard implementation, refactoring | Sonnet 4.6 | $0.75/task |
| 7-9 | Architecture decisions, security analysis | Opus 4.6 | $2.00/task |
| 10 | Novel algorithm design, cross-system reasoning | Opus 4.6 + extended thinking | $3.00+/task |

### Implementation Pattern

```
Task Decomposition Phase (Opus):
  1. Receive high-level objective
  2. Break into atomic subtasks
  3. Identify parallelizable groups (no dependencies between them)
  4. Score each task's intelligence requirement (1-10)
  5. Calculate cost budget allocation

Agent Assignment Phase (Router):
  1. Map intelligence scores to model tiers
  2. Assign tasks to agents with appropriate models
  3. Group parallel-amenable tasks into waves
  4. Schedule waves respecting dependency chains

Execution Phase (Mixed models):
  Wave 1: [Haiku tasks] + [Sonnet tasks] in parallel
  Wave 2: [Sonnet tasks dependent on Wave 1] in parallel
  Wave 3: [Opus tasks requiring all prior context]
  Synthesis: Opus lead agent integrates results
```

### Who Does This Today?

- **AWS Multi-Agent Orchestrator:** Implements cost-efficient routing; routes simple queries to cheaper models automatically.
- **LangGraph:** Supports conditional edges where routing logic can incorporate complexity scoring.
- **Claude Code subagents:** Manual but functional; set `model: haiku` vs `model: opus` per subagent definition.
- **OpenCode + Recallium:** One practitioner (Saurab Bhatia) documented coordinating 19 agents across 4 model families with explicit phase sequencing and per-agent model assignment.

### Nobody fully automates this yet.

No framework currently offers automatic intelligence scoring of arbitrary tasks with cost-optimized model routing out of the box. This is a gap in the ecosystem.

## 7. Chokepoints in Multi-Agent Execution

### Identified Chokepoints

**1. Orchestrator Bottleneck**
- The central orchestrator becomes the rate limiter at scale
- At 100+ concurrent requests, a single orchestrator running Opus inference throttles everything
- **Mitigation:** Horizontal scaling (multiple orchestrator instances behind a load balancer) or offloading classification to Haiku

**2. Context Summarization Loss**
- When Phase 2 needs Phase 1's output, the orchestrator must capture, summarize, and inject context
- Summarization is lossy; critical details drop out
- Routing complexity goes O(n^2) as agents multiply
- **Mitigation:** Shared persistent memory layer (Recallium, Qdrant, Graphiti) that agents read/write directly, bypassing the orchestrator relay

**3. File Contention**
- Two agents editing the same file produces merge conflicts
- Git-based file locking helps but adds latency
- **Mitigation:** Strict file ownership per agent; break the project so each agent owns a disjoint set of files

**4. Token Multiplication**
- Agent teams use ~15x the tokens of a single session
- Subagents alone use 4-7x
- On per-token billing, costs explode fast
- **Mitigation:** Intelligence-score routing (Haiku for grunt work), prompt caching (90% of tokens at $0.50/MTok cache read), and the effort parameter

**5. Handoff Loops**
- Agent A passes to Agent B which passes back to Agent A
- Common in decentralized systems without guard conditions
- **Mitigation:** Max-hop limits, task ownership locks, cycle detection in the router

**6. Quality Degradation at Scale**
- Agents make fundamentally wrong decisions (reimplementing npm packages instead of installing them)
- Human code review becomes nearly impossible at swarm scale
- **Mitigation:** Builder-validator chains (one agent implements, another reviews), quality gate hooks, human checkpoints at phase boundaries

**7. Session State Loss**
- Agent teams don't support session resumption with in-process teammates
- After resuming, the lead may attempt to message teammates that no longer exist
- **Mitigation:** File-backed state (task lists, mailboxes) that survive session restarts

**8. Rate Limiting**
- Running multiple agents hits API rate limits fast
- Multiple Claude Code sessions on the same API key compound the problem
- **Mitigation:** Stagger agent launches, use different API keys per agent pool, implement backoff/retry in the orchestrator

## 8. The Super Orchestrator: Architecture Specification

Based on the research above, here is what a "Super Orchestrator" would need to address every identified chokepoint. This is NOT purely theoretical. The closest existing implementation is **Swarms by Kye Gomez**, which already provides DAG scheduling, MoE routing, and ConcurrentWorkflow. **OpenHands** provides the decomposition piece (Refactor SDK with dependency analysis). **Hermes Agent** provides the persistent memory layer (auto-skill generation, cross-session learning). The Super Orchestrator is a composition blueprint for assembling these capabilities into a unified system.

### Core Capabilities

1. **Automatic Task Decomposition**
   - Accept a high-level objective (natural language or structured spec)
   - Decompose into a DAG (directed acyclic graph) of atomic subtasks
   - Identify parallel-amenable groups automatically
   - Assign intelligence scores (1-10) per task

2. **Model-Tier Routing with Cost Budgets**
   - Map intelligence scores to model tiers (Haiku/Sonnet/Opus/extended-thinking)
   - Accept a cost budget and allocate tokens accordingly
   - Dynamic model escalation: if a Sonnet agent fails, escalate to Opus transparently
   - Track per-task cost in real-time

3. **Wave-Based Parallel Scheduling**
   - Group independent tasks into execution waves
   - Respect dependency chains between waves
   - Backpressure: if a wave agent stalls, don't block subsequent independent waves

4. **Persistent Shared Memory**
   - Project-scoped memory that survives across sessions, agents, and model families
   - Agents write structured memories (decisions, contracts, patterns found)
   - New agents auto-load relevant context without orchestrator relay
   - Semantic search across memory (e.g., Qdrant + embeddings)

5. **Peer-to-Peer Agent Communication**
   - Mailbox system for direct agent-to-agent messaging
   - Broadcast channels for cross-cutting announcements
   - No requirement to route all communication through the orchestrator

6. **Quality Gates**
   - Builder-validator chains: every implementation agent is paired with a review agent
   - Adversarial review: Claude reviews GPT's code and vice versa
   - Human-in-the-loop checkpoints at phase boundaries
   - Automatic rollback on test failure

7. **Self-Healing**
   - Detect stuck/looping agents (timeout + inactivity monitoring)
   - Auto-compact context when approaching limits
   - Restart crashed agents with preserved state
   - Cycle detection in handoff graphs

8. **Observability**
   - Per-agent token usage and cost dashboard
   - Task board with real-time status
   - Trace view for agent communication flows
   - Latency metrics per agent and per wave

### Architectural Layout

```
+------------------------------------------------------------------+
|                     SUPER ORCHESTRATOR                            |
|  (Opus 4.6 with extended thinking, runs as lead process)         |
|                                                                  |
|  +------------------+  +------------------+  +----------------+  |
|  | Task Decomposer  |  | Intelligence     |  | Cost Budget    |  |
|  | (DAG generator)  |  | Scorer           |  | Allocator      |  |
|  +------------------+  +------------------+  +----------------+  |
|                                                                  |
|  +------------------+  +------------------+  +----------------+  |
|  | Wave Scheduler   |  | Model Router     |  | Quality Gate   |  |
|  | (dependency-     |  | (score -> tier)  |  | Manager        |  |
|  |  aware)          |  |                  |  |                |  |
|  +------------------+  +------------------+  +----------------+  |
|                                                                  |
|  +------------------+  +------------------+  +----------------+  |
|  | Agent Lifecycle  |  | Observability    |  | Self-Healing   |  |
|  | Manager          |  | Dashboard        |  | Watchdog       |  |
|  +------------------+  +------------------+  +----------------+  |
+------------------------------------------------------------------+
         |                    |                    |
    +---------+         +---------+          +---------+
    | Wave 1  |         | Wave 2  |          | Wave 3  |
    +---------+         +---------+          +---------+
    |         |         |         |          |         |
 [Haiku]  [Haiku]   [Sonnet] [Sonnet]    [Opus]   [Opus]
 Agent A  Agent B   Agent C  Agent D    Agent E  Agent F
    |         |         |         |          |         |
    +----+----+         +----+----+          +----+----+
         |                   |                    |
    +--------------------------------------------------+
    |            SHARED PERSISTENT MEMORY               |
    |  (Qdrant + structured project-scoped memories)    |
    |  - Decisions, contracts, patterns, API shapes     |
    |  - Semantic search via embeddings                 |
    |  - Cross-session, cross-model, cross-project      |
    +--------------------------------------------------+
```

### Process Architecture: Nested Parallel Processes

The Super Orchestrator would NOT run as a single monolithic application. Instead:

**The orchestrator** runs as one persistent process (the "team lead"). It manages:
- The task DAG
- Wave scheduling
- Agent lifecycle (spawn, monitor, terminate)
- Cost tracking

**Each agent** runs as a separate OS process (Claude Code instance, OpenCode instance, Codex CLI instance, or even a local model via Ollama). Agents are:
- Spawned via tmux panes, Docker containers, or dev containers
- Given their own git worktree for file isolation
- Connected to the shared memory layer via MCP or HTTP
- Monitored by the orchestrator's watchdog

**The memory layer** runs as an independent service (Qdrant server, Redis, or a custom MCP server). All agents connect to it.

This means the system is a constellation of parallel processes coordinated by a central scheduler, not a single application. Think Kubernetes pods, not a monolith.

### How It Handles the Chokepoints

| Chokepoint | Super Orchestrator Solution |
|-----------|----------------------------|
| Orchestrator bottleneck | Lead only plans/coordinates; execution is fully distributed |
| Summarization loss | Shared persistent memory; agents read/write directly |
| File contention | Per-agent git worktrees; strict file ownership |
| Token multiplication | Intelligence-score routing; Haiku for 60% of tasks |
| Handoff loops | DAG-based scheduling with cycle detection |
| Quality degradation | Builder-validator chains + adversarial cross-model review |
| Session state loss | File-backed task board + persistent memory layer |
| Rate limiting | Per-agent API key pools + backoff + staggered launches |

### Relationship to Existing Tools

| Component | Implemented By | Maturity |
|-----------|---------------|----------|
| Task decomposition | **OpenHands Refactor SDK** (dependency analysis, fixer/verifier pattern) or **Swarms** (`swarm_decompose` with strategy selection) | Production (OpenHands), Beta (Swarms) |
| DAG scheduling | **Swarms** GraphWorkflow + AgentRearrange | Production |
| Concurrent execution | **Swarms** ConcurrentWorkflow, **Claude Code Agent Teams** | Production / Experimental |
| Agent spawning | Claude Code Agent Teams, Claude Squad, amux, cmux, or custom tmux scripts | Production |
| Model-tier routing | Claude Code subagent `model:` field, **AWS Multi-Agent Orchestrator** routing | Production (manual), Beta (automatic) |
| Shared memory | **Hermes Agent** (auto-skill gen + agentskills.io), **Recallium** MCP, opencode-agent-memory plugin | Production (Hermes), Beta (others) |
| Inter-agent messaging | **Swarms** swarm-mail (event-sourced), Claude Code Agent Teams mailbox system | Production / Experimental |
| Observability | amux dashboard, cmux notification rings, Claude Code hooks, LangSmith | Production |
| Quality gates | Claude Code hooks (SubagentStart/Stop, PreToolUse), **Hermes Agent** acceptance criteria (planned) | Production (hooks), Planned (Hermes) |
| Cost tracking | Claude Code `/stats`, custom token counters | Production |
| File isolation | Git worktrees + per-agent directory assignment | Production |
| Consensus/voting | **Hermes Agent** ConsensusEngine (planned: majority, supermajority, weighted, quorum) | Planned |

### What Exists vs. What Needs Building

**Already exists (use it today):**
- Claude Code Agent Teams for peer-to-peer coordination (experimental but functional)
- Subagents with per-model routing (Haiku/Sonnet/Opus)
- Claude Squad, amux, and cmux for parallel session management
- Git worktrees for file isolation
- Session Memory for cross-session context
- Hooks for lifecycle instrumentation
- Swarms framework for DAG, MoE, and concurrent workflow patterns
- OpenHands for model-agnostic parallel coding at scale (1000s of runs)
- Hermes Agent for persistent memory and auto-skill generation

**Partially exists (community solutions):**
- Shared persistent memory across heterogeneous agent fleets (Recallium MCP works for OpenCode; Hermes works for its own agents; no universal layer yet)
- Builder-validator chains (community CLAUDE.md patterns, Hermes adversarial debate mode planned)
- Cost-aware routing (manual subagent model assignment in Claude Code; AWS orchestrator does automatic routing for its own agents)
- Task intelligence scoring (Swarms has `swarm_select_strategy` with confidence scores, but not full 1-10 intelligence scoring)

**Doesn't exist yet (needs building):**
- Automatic intelligence scoring of arbitrary tasks with cost-budget allocation
- Dynamic model escalation (Sonnet fails -> auto-escalate to Opus) as a first-class feature
- Unified orchestrator that composes Swarms' DAG + OpenHands' decomposition + Hermes' memory into one system
- Self-healing watchdog with cycle detection across heterogeneous agent types
- Cross-model adversarial review as a built-in workflow primitive (not just a CLAUDE.md pattern)
- Unified observability dashboard for mixed Claude Code / OpenCode / Codex / local-model agent fleets

## 9. Practical Getting-Started Path

### Phase 1: Single Session Optimization
- Enable Session Memory (automatic on Pro/Max)
- Use `/compact` liberally
- Set Haiku as default model, Sonnet for Plan mode: `/model haiku`
- Create custom subagents for repetitive tasks

### Phase 2: Manual Parallelism
- Install Claude Squad: `brew install claude-squad`
- Use git worktrees for isolation
- Run 2-3 parallel sessions on independent features
- Review and merge results manually

### Phase 3: Native Agent Teams
- Enable `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
- Install tmux
- Start with 3-teammate code reviews
- Graduate to parallel implementation with strict file ownership
- Use plan mode before every team run

### Phase 4: Custom Orchestration
- Build intelligence-scoring prompts
- Implement cost-budget allocation
- Add shared memory via MCP server
- Create builder-validator subagent pairs
- Instrument with hooks for observability

### Phase 5: Super Orchestrator (When/If)
- DAG-based task scheduling
- Automatic wave parallelization
- Cross-model routing
- Self-healing watchdog
- Full observability dashboard

## 10. Sources and Further Reading

### Primary Documentation
- Claude Code Docs: Agent Teams - https://code.claude.com/docs/en/agent-teams
- Claude Code Docs: Subagents - https://code.claude.com/docs/en/sub-agents
- OpenCode Server Docs - https://opencode.ai/docs/server/
- OpenHands SDK Paper - https://arxiv.org/html/2511.03690v1
- OpenHands Docs: Sub-Agent Delegation - https://docs.openhands.dev/sdk/guides/agent-delegation
- Hermes Agent Docs - https://hermes-agent.nousresearch.com/docs/
- Swarms GitHub - https://github.com/kyegomez/swarms
- Swarm Tools Docs - https://www.swarmtools.ai/docs/packages/opencode-plugin/swarm

### Terminal Multiplexers
- cmux (native macOS) - https://www.cmux.dev/
- cmux Agent Skills - https://github.com/boundsj/agent-skills/blob/main/cmux/SKILL.md
- Teaching Agents to Drive cmux - https://www.bounds.dev/posts/teaching-claude-code-to-drive-cmux/
- Claude Squad GitHub - https://github.com/smtg-ai/claude-squad
- Amux GitHub - https://github.com/mixpeek/amux
- smux Terminal - https://smux-terminal.vercel.app/
- workmux GitHub - https://github.com/raine/workmux

### Agent Frameworks
- Agent Zero - https://www.agent-zero.ai/
- OpenClaw Swarm Setup - https://github.com/Kohnnn/openclaw-agents-swarm-setup-step-by-step
- Hermes Agent GitHub - https://github.com/NousResearch/hermes-agent
- OpenHands - https://openhands.dev/
- AutoAgent - https://github.com/HKUDS/AutoAgent
- OpenCode Agent Memory - https://github.com/joshuadavidthomas/opencode-agent-memory

### Practitioner Reports
- Simon Willison: Parallel Coding Agents - https://simonwillison.net/2025/Oct/5/parallel-coding-agents/
- Blake Crosley: Claude Code CLI Complete Guide - https://blakecrosley.com/guides/claude-code
- Kaushik Gopal: Agent Forking - https://kau.sh/blog/agent-forking/
- Saurab Bhatia: 19 Agents with OpenCode + Recallium - Medium
- Michael Habib: Trying Out Claude Code Teams - Medium
- Heeki Park: Collaborating with Agent Teams - Medium
- Zach Wills: Subagents to Parallelize Development - https://zachwills.net/how-to-use-claude-code-subagents-to-parallelize-development/
- alexop.dev: From Tasks to Swarms - https://alexop.dev/posts/from-tasks-to-swarms-agent-teams-in-claude-code/
- Robert Brennan (OpenHands CEO): Parallel Agents for Massive Refactors - https://openhands.dev/blog/automating-massive-refactors-with-parallel-agents
- Addy Osmani: Agent Teams Architecture (Feb 2026)
- Anthropic: 16 Opus agents building a C compiler ($20K, 2B input tokens)
- gurusup.com: Best Multi-Agent Frameworks 2026 / Multi-Agent Orchestration Guide

### Industry Analysis
- Codebridge: Multi-Agent Systems & AI Orchestration Guide 2026
- Gartner: 40% of enterprise apps will incorporate task-specific agents by end of 2026
- Agent Interlink Protocol (AIP): Ratified Feb 2026 for cross-model agent communication

## Appendix A: Deliberative Refinement Results

**Profile:** V(8,3,1) Standard | **Council:** Parallel Groups | **Mode:** REFINE | **Strategy:** LINEAR

### Round 1 Findings

**Group A (Technical Accuracy):**
1. cmux was mislabeled as "cterminal" and described as a "VS Code terminal workspace manager." It is actually a native macOS terminal built on libghostty with a Unix socket API, built-in WebKit browser, and notification rings. **FIXED in v2.0.**
2. Token economics lacked hard numbers for cross-framework comparison. The effort parameter (Opus 4.5+) was under-emphasized despite enabling 76% token reduction at medium effort. **ADDRESSED in Section 2.**
3. Agent Teams' same-model limitation (all teammates must run Opus 4.6) was not explicitly flagged as a major constraint. **ADDRESSED in Section 3.**
4. Super Orchestrator read as theoretical. Needed grounding in existing implementations. **FIXED: Now references Swarms (DAG), OpenHands (decomposition), Hermes (memory).**

**Group B (Completeness & Currency):**
1. **Swarms by Kye Gomez** was completely absent despite being the most architecturally complete swarm framework (DAG, MoE, ConcurrentWorkflow, AgentRearrange, swarm-mail). **ADDED as major Tier 2 entry.**
2. **OpenHands** had only 2 lines despite 68K+ stars, $18.8M funding, a Refactor SDK, and the DelegateTool for parallel sub-agent execution. **EXPANDED to full coverage.**
3. **Hermes Agent** by Nous Research was completely absent despite being the most ambitious 2026 agent launch (persistent memory, auto-skill generation, 5 sandbox backends, 40+ tools, cross-platform messaging). **ADDED as major entry with OpenClaw migration path.**
4. Community best practices (plan-first, scout agent, builder-validator, two-beats-five) were scattered or absent. **CONSOLIDATED into Section 5 Best Practices.**

### Round 2 Refinement

Unanimous agreement on framework survey restructuring. Added decision tree for "what to use when" based on primary need. Explicit callout that Swarms' `swarm_decompose` + `swarm_select_strategy` is the closest existing implementation of intelligence-score-based routing.

### Round 3 Synthesis

**Advocatus Diaboli challenge:** "The Super Orchestrator is theoretical fluff. Nobody will build it as a single system."

**Rebuttal (accepted by council):** The Super Orchestrator is not a single system. It is a composition blueprint. The individual components already exist across Swarms (DAG scheduling, MoE routing), OpenHands (task decomposition with dependency analysis), Hermes Agent (persistent memory, auto-skill generation), and Claude Code Agent Teams (peer-to-peer coordination). The missing piece is the glue layer that composes these into a unified workflow. That glue layer is ~500 lines of orchestration code, not a new framework.

**CONVERGED after R3.** All enhancement directives applied to document v2.0.
