# ConvTree — Product Specification
> Working title. Final name TBD. Built on a fork of Open WebUI.

---

## 1. Product Overview

ConvTree is an open-source, self-hostable AI conversation workspace that treats every session as a **structured work artifact** rather than a linear chat log. The core innovation is a **conversation tree** — every session is a directed graph of turns, branches, and decisions. Artifacts produced by the agent (code, files, reports) are captured and organized automatically. The product ships in two tiers: a fully-featured solo tier (free, open source) and a team tier (paid, open core).

**Base:** Fork of [Open WebUI](https://github.com/open-webui/open-webui) (Python/FastAPI backend, SvelteKit frontend).  
**GitHub repo to fork:** `open-webui/open-webui` → rename to `convtree` (or chosen name).  
**Primary deployment:** Docker-first, self-hosted. Solo tier also supports local Electron-style desktop packaging.

---

## 2. Product Principles

- **Solo tier is complete and free.** No features that serve only the individual are paywalled. Ever.
- **Team tier is where the business lives.** Collaboration, accounts, shared workspaces, SSO.
- **Nothing leaves the device by default.** All conversation data, branch history, and telemetry stays local unless the user explicitly opts in to anonymous telemetry.
- **The tree is the primary UI.** Not a secondary panel. Not a hidden feature. The conversation-as-tree is the product's identity.
- **Detect and suggest, never auto-execute.** The heuristic engine surfaces branch opportunities. The user decides.

---

## 3. Deployment Tiers

### Solo Tier (Open Source / Free)
- Single user, no login required
- Local Docker container or desktop app
- All data on local filesystem
- Connects to user-configured LLM endpoints (Ollama local, or API keys for cloud models)
- Full conversation tree view and navigation
- Full artifact capture and session file management
- Workspace and session organization
- Session summary and export
- Conversation templates
- Local search across sessions
- Local knowledge base per workspace
- Heuristic branching engine (locally calibrated)
- Anonymous telemetry: opt-in only, off by default

### Team Tier (Open Core / Paid)
Everything in solo, plus:
- User accounts and authentication
- Shared workspaces with role-based access
- Collaborative branching (another user forks from a shared session node)
- Admin dashboard (usage, members, audit logs)
- SSO / SAML / OIDC
- Google Workspace and Microsoft 365 import/export
- Managed hosting option (future)
- Priority support and SLAs

---

## 4. Core Feature: Conversation Tree

### 4.1 What a Branch Is
A branch is a **semantic divergence** in problem-solving direction, not just a message edit. The tree captures meaningful path changes, not every keystroke. The branch is the atomic unit of the tree; a session is a collection of branches rooted at a shared origin.

### 4.2 Branch Trigger Types

| Trigger Type | Description | Detection |
|---|---|---|
| `user_explicit` | User goes back to a node and takes a different direction | User action on tree node |
| `agent_offered` | Agent presents multiple viable paths as branch stubs | Agent response structured as options |
| `goal_shift` | User's intent meaningfully changes mid-conversation | Heuristic engine + user confirmation |
| `correction` | User identifies a wrong assumption made several turns back | User action or heuristic |

### 4.3 Dead End Types

| Dead End Type | Description | Detection Signals |
|---|---|---|
| `technical` | Agent hits a hard capability limit | Agent signals failure explicitly |
| `circular` | Conversation repeating same intent with no progress | Semantic similarity score, frustration language |
| `quality` | Outputs not converging toward user's goal | Manual declaration primarily |
| `relevance` | Approach is working but solving the wrong problem | Manual declaration |

### 4.4 Dead End Recovery
1. Zoom the tree out and show the full path that led to the dead end
2. Highlight candidate branch points (last open decision node, node where wrong assumption entered)
3. Explain why the dead end was detected (if system-detected)
4. Preserve the dead end branch — never delete it
5. User explicitly chooses a branch point to fork from

### 4.5 Branch UX Rules
- Dead ends are **never auto-deleted**. They are labeled and preserved for auditability.
- The system **suggests** branches, never auto-creates them without user confirmation.
- Every node has a "branch from here" affordance in the UI.
- The current branch is always the primary visual focus.

---

## 5. UI/UX Specification

### 5.1 Layout Structure

```
┌──────────────────────────────────────────────────────────────┐
│  Header: logo · workspace name                    [↗] [+]    │
├──────────────────────────────────────────────────────────┬───┤
│  Peek bar: OTHER BRANCHES  [B1·dead end↑] [B2·done↑]    │ f │
├──────────────────────────────────────────────────────────┤ i │
│  [branch  │  Conversation messages (current branch only) │ l │
│   lanes   │                                              │ e │
│   44px]   │  turn: You — message content                 │ s │
│           │  turn: Agent — response + artifact card      │   │
│           │                                              │ ← │
│           │                                              │ h │
│           │                                              │ o │
│           │                                              │ v │
├──────────────────────────────────────────────────────────┤ e │
│  [message input field]                             [↑]   │ r │
├──────────────────────────────────────────────────────────┴───┤
│  Status bar: [● model-name ▾]  |  Branch N · turn X · state  │
│                                              [Expand tree ⌥] │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 Branch Lane Panel (left, 44px)
- Integrated into the conversation scroll — no separate scroll context
- Each lane is a vertical SVG line (10-12px lane width, 3 lanes max visible by default)
- **Current branch:** thick line (2px), full opacity, full height
- **Other branches:** thin line (1px), ~28% opacity, terminates at their endpoint
- **Dead end terminal:** ✕ marker in muted red at endpoint
- **Completed terminal:** hollow dot in muted green at endpoint
- **Fork point:** small filled dot on current branch lane at the turn where branches diverged
- Lane positions are mapped to actual pixel heights of turn messages (redrawn on resize and message add)

### 5.3 Peek Bar (above conversation, below header)
- Shows only the **endpoints** of other branches as small chips
- Chips: colored dot + branch label + status + ↑ arrow
- Clicking a chip switches to that branch
- Hidden when only one branch exists

### 5.4 Files Panel (right, 36px collapsed → 168px on hover)
- Collapsed: shows file type icons only (py, sql, md, csv badges)
- Hover: expands with 150ms ease transition, shows file names and node origin
- Collapses immediately on mouse leave
- Files are linked to the branch and turn that produced them

### 5.5 Status Bar (bottom, 32px)
- **Left:** Model indicator — colored dot + model name, clickable → opens model switcher dropdown (above the bar)
- **Center:** Current branch name + turn count + status
- **Right:** "Expand tree" button → opens full-session tree overlay

### 5.6 Tree Expand Overlay
- Covers the body area (between header and status bar)
- Shows the full session tree as a navigable SVG graph
- Clicking a branch node navigates to it and closes the overlay
- Dead ends, completed branches, active branches, agent option stubs all color-coded

### 5.7 Color Coding

| Node / Branch State | Color | Hex |
|---|---|---|
| Root | Neutral grey | `#888780` |
| Active branch (current) | Blue | `#378ADD` |
| Completed branch | Sage green | `#639922` |
| Dead end — technical | Terracotta red | `#E24B4A` |
| Dead end — circular | Amber | `#EF9F27` |
| Dead end — relevance | Muted purple | `#7F77DD` |
| Correction branch | Coral | `#D85A30` |
| Agent-offered options | Teal | `#1D9E75` |

**Rule:** Only the current node/branch is rendered at full opacity. All others use ~28–40% opacity. The tree is ambient, not dominant.

---

## 6. Artifact Capture

Every file, code block, or structured output produced by the agent is automatically captured into the session files panel:
- Named, timestamped, and linked to the branch + turn that produced them
- Versioned if regenerated (both versions kept)
- User can rename, download, or pin them
- Pinned artifacts survive session archival

**Supported artifact types:** code files (any language), markdown, CSV, JSON, images, PDF.

---

## 7. Workspace & Session Organization

- Sessions are grouped into **Workspaces** (user-created, named, e.g. "DataGuardian Build")
- Each workspace has: name, description, shared system prompt, shared knowledge files, model config
- Sessions within a workspace display: tree thumbnail, artifact count, one-line summary, last active timestamp
- Search is workspace-scoped by default, with an option to search all workspaces

---

## 8. Heuristic Branching Engine

### 8.1 Architecture
A **rule-based signal engine** that runs locally. No second LLM. No cloud calls. Runs in microseconds. Operates between turns, never mid-turn.

### 8.2 Signal Rules (initial set)

```
frustration_language:
  condition: correction/negation words in last 3 user turns > threshold
  signals: ["no", "that's not", "wrong", "not what I meant", "try again", "forget that"]

semantic_repetition:
  condition: cosine similarity between last 3 user messages > threshold
  model: local embedding (nomic-embed-text or all-MiniLM-L6 via ONNX)

stagnation:
  condition: N turns elapsed without a new artifact or user acknowledgment

circular_agent:
  condition: agent outputs have high similarity across consecutive turns

explicit_dead_end:
  condition: user uses dead-end language
  signals: ["dead end", "stuck", "not working", "start over", "go back"]
```

### 8.3 Config Structure (resolved at session start, frozen during session)

```json
{
  "source": "resolved",
  "version": "1.4.2",
  "thresholds": {
    "frustration_word_count": 2,
    "semantic_similarity": 0.81,
    "stagnation_turn_count": 7,
    "repetition_window": 3
  },
  "weights": {
    "frustration": 0.4,
    "repetition": 0.35,
    "stagnation": 0.25
  }
}
```

### 8.4 Three-Layer Config Priority

```
Layer 3 — Local learned config      (highest priority, activates after confidence minimum)
Layer 2 — Local explicit config     (user manual overrides, always wins over automation)
Layer 1 — Global shipped config     (baseline, lowest priority)
```

Resolution logic (runs at app startup, produces a frozen resolved config for the session):
1. If `user_explicit` override exists for threshold → use it (Layer 2)
2. Else if `local_learned` value exists AND `confidence >= minimum` → use it (Layer 3)
3. Else → use `global_shipped` value (Layer 1)

### 8.5 Calibration Triggers

| Trigger | Scope | Speed |
|---|---|---|
| Session end | Last session's events | Milliseconds |
| App startup | Full rolling window (90 days / 500 sessions) | < 1 second |
| Confidence threshold crossing | Single threshold, background | Milliseconds |
| Manual (settings) | Full rolling window | < 1 second |

**Rule:** Config is **never updated mid-session.** The session starts with a frozen resolved config and keeps it until the session ends.

### 8.6 Agent Tree Awareness
The agent receives a tree-context block in its system prompt at session start:

```
You are operating in a branched conversation session.
Current branch: Branch 3 (active)
Branch depth: 1 (direct child of root)
Previous branches tried: 2
  - Branch 1 (dead end — circular): schema validation approach, abandoned turn 6
  - Branch 2 (completed): transform layer rewrite, resolved turn 11

When you identify a point where multiple viable approaches exist, present them
as numbered options rather than choosing one unilaterally. If you recognize an
approach that was already attempted in a prior branch, flag this explicitly
rather than repeating it.
```

---

## 9. Telemetry & Data Architecture

### 9.1 Offline Telemetry (local SQLite, never transmitted)

**Core tables:**
- `sessions` — session metadata, outcome, turn counts, branch counts
- `turns` — per-turn signals (sentiment score, frustration score, intent shift score, artifact produced) — NO raw text
- `branches` — branch metadata, trigger type, depth, outcome, resolution time
- `branch_events` — granular events: signals active, user response (accepted/dismissed/ignored), response delay
- `local_engine_calibration` — per-threshold: current value, confidence count, true/false positive tallies

**Rolling window:** Raw events kept for 90 days or 500 sessions (whichever comes first). Older data summarized into aggregates and pruned.

### 9.2 Online Telemetry (anonymous, opt-in, batch transmitted)

**Event types:**
- `branch_signal` — which rules fired, combined score, user response (accepted/dismissed), session context buckets
- `dead_end` — detection context (which signals), recovery taken, recovery outcome
- `false_signal` — rules that fired when user dismissed suggestion, session outcome after dismissal
- `session_summary` — aggregate engine performance (precision, recall), config versions

**Privacy rules enforced at extraction layer (one-way transformation, irreversible):**
- All timestamps → session-relative offsets, bucketed time-of-day
- All IDs → anonymous event-scoped tokens (not linkable across events)
- All exact values → bucketed ranges
- All content signals → computed scores only, no text
- Batch transmitted on random schedule (not per-event, prevents timing fingerprinting)
- The extraction pipeline is the **only code path** that produces online telemetry format

### 9.3 Global Calibration Pipeline (server-side, periodic)
- Consumes anonymous telemetry from consenting users globally
- Computes optimal thresholds per rule based on acceptance/rejection rates
- Outputs a single `global_config.json` shipped with product updates
- Never runs on user devices — output only (the config file) reaches users

---

## 10. Data Privacy Guarantee (Solo Tier)

```
Conversation content          → Local SQLite only, never transmitted
Branch event logs             → Local SQLite only
Heuristic pattern store       → Local SQLite only
Generated artifacts/files     → Local filesystem, user-controlled path
Session embeddings            → Local vector store (ChromaDB local mode)
LLM calls                     → Only to the endpoint the user configured
```

No telemetry, no analytics, no outbound calls except to the user's configured LLM endpoint. This is an architectural guarantee, not a policy.

---

## 11. MVP Scope

### In MVP
1. **Conversation tree view** — git-graph-style branch lanes integrated in the conversation scroll, peek bar for branch navigation, expand-tree overlay for full navigation
2. **Artifact capture** — automatic capture of code blocks and file outputs into a session files panel, linked to branch and turn
3. **Workspace + session organization** — workspaces with grouped sessions, session cards with tree thumbnail and artifact count
4. **Heuristic engine (rule-based, Layer 1 only)** — initial rule set, suggestion nudge UI, user accept/dismiss
5. **Local telemetry logging** — branch events written to local SQLite (no online telemetry in MVP)
6. **Model switcher** — status bar model indicator, quick switcher dropdown, supports Ollama + API key providers
7. **Solo deployment** — Docker Compose single-container, no auth required

### Explicitly Out of MVP
- Online telemetry and global calibration pipeline
- Local calibration (Layer 3 learning) — log the data, build the calibration later
- Conversation templates
- Semantic search across sessions
- Team tier (accounts, collaboration, permissions)
- Google Workspace / M365 integration
- Managed hosting

---

## 12. Tech Stack (inheriting from Open WebUI fork)

| Layer | Technology |
|---|---|
| Backend | Python, FastAPI |
| Frontend | SvelteKit |
| Database | SQLite (local), expandable to PostgreSQL for team tier |
| Vector store | ChromaDB (local mode) |
| Tree visualization | Custom SVG renderer (no D3 — hand-written, ~100 lines) |
| Branch lane animation | CSS transitions only |
| Local embeddings | nomic-embed-text or all-MiniLM-L6 via ONNX runtime |
| Containerization | Docker Compose |
| Desktop packaging | Tauri (future, for no-Docker solo option) |

---

## 13. Open Source & Licensing

- **Core (solo features):** AGPL-3.0 — same as Open WebUI base
- **Team tier features:** Commercial license (dual-license model)
- **Contributor License Agreement (CLA):** Required for contributions — enables the dual-license model
- **Telemetry:** Explicit opt-in, granular consent, documented in PRIVACY.md

---

## 14. Key Open Decisions (to discuss)

- Final product name (ConvTree is a working title from UI mockups)
- Whether to diverge from Open WebUI's frontend framework (SvelteKit) or stay with it
- How deep to go on the Open WebUI fork vs. rewrite approach — assess at fork time
- Embedding model choice for local semantic similarity (ONNX vs. Ollama-served)
- Desktop packaging strategy — Docker-only MVP or include Tauri from the start
