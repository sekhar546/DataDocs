# ConvTree — Heuristic Engine & Telemetry Architecture
### Technical Design Document

---

## Table of Contents

1. [Overview](#1-overview)
2. [Heuristic Engine Architecture](#2-heuristic-engine-architecture)
3. [Signal Rules](#3-signal-rules)
4. [Scoring & Decision Model](#4-scoring--decision-model)
5. [Three-Layer Config System](#5-three-layer-config-system)
6. [Config Resolution & Caching](#6-config-resolution--caching)
7. [Calibration Triggers](#7-calibration-triggers)
8. [Agent Tree Awareness](#8-agent-tree-awareness)
9. [Engine Capability & Limitations](#9-engine-capability--limitations)
10. [Offline Telemetry — Local SQLite](#10-offline-telemetry--local-sqlite)
11. [Online Telemetry — Anonymous Global](#11-online-telemetry--anonymous-global)
12. [The Extraction Boundary](#12-the-extraction-boundary)
13. [Global Calibration Pipeline](#13-global-calibration-pipeline)
14. [The Feedback Loop — Evolution Phases](#14-the-feedback-loop--evolution-phases)
15. [Privacy Architecture](#15-privacy-architecture)
16. [Config Update Handling](#16-config-update-handling)
17. [Local Data Lifecycle](#17-local-data-lifecycle)

---

## 1. Overview

The heuristic engine is a **local, rule-based signal processor** that monitors conversation state and surfaces branching opportunities to the user. It has no model overhead — it runs in microseconds using pure logic and lightweight local computation. It never auto-branches; it detects and suggests, always leaving the decision to the user.

The engine is paired with a two-path telemetry system:

- **Offline telemetry** — rich local data collected on the user's device, used to calibrate the engine to this specific user's working style over time. Never transmitted.
- **Online telemetry** — anonymous, minimal, opt-in data aggregated globally to calibrate the baseline engine shipped to all users. Contains no conversation content. Batch transmitted on a randomized schedule.

These two paths produce two of the three layers in the engine's config system. The third layer is direct user control. All three layers are resolved into a single frozen config at session start and never changed mid-session.

```
┌─────────────────────────────────────────────────────┐
│                  Runtime (per session)              │
│                                                     │
│   User turn arrives                                 │
│         ↓                                           │
│   Heuristic engine evaluates signals                │
│   (uses frozen resolved config)                     │
│         ↓                                           │
│   Combined score > threshold?                       │
│         ↓ yes                                       │
│   Surface suggestion to user                        │
│         ↓                                           │
│   User accepts or dismisses                         │
│         ↓                                           │
│   Event logged to local SQLite                      │
│                                                     │
├─────────────────────────────────────────────────────┤
│              Between Sessions (async)               │
│                                                     │
│   Calibration pass runs →                          │
│   Local learned config updated →                   │
│   Resolved config rebuilt →                        │
│   Cached for next session                           │
└─────────────────────────────────────────────────────┘
```

---

## 2. Heuristic Engine Architecture

### 2.1 Design Principles

- **No second model.** A dedicated classifier LLM would require concurrent model execution, adding memory and compute overhead incompatible with a lightweight self-hosted solo app. Pure rule-based logic is fast, transparent, debuggable, and sufficient for the majority of detectable branching signals.
- **Separation from runtime.** The engine reads a pre-resolved config at session start and never touches the database or calibration logic during a session. All learning happens between sessions.
- **Frozen config per session.** The resolved config is fixed for the entire duration of a session. This guarantees consistent behavior and ensures branch events can be attributed to a known config state.
- **Detect and suggest, never act.** The engine has no authority to create branches. It produces a suggestion that the user consciously accepts or dismisses.

### 2.2 Engine Components

```
┌──────────────────────────────────────────────────────────┐
│                    Heuristic Engine                      │
│                                                          │
│  ┌─────────────────┐     ┌──────────────────────────┐   │
│  │  Signal Probes  │────▶│  Scoring & Combiner      │   │
│  │                 │     │                          │   │
│  │  - frustration  │     │  weighted_score =        │   │
│  │  - repetition   │     │  Σ(signal_score × weight)│   │
│  │  - stagnation   │     │                          │   │
│  │  - circular     │     │  if score > threshold:   │   │
│  │  - explicit     │     │  → emit suggestion       │   │
│  └─────────────────┘     └──────────────────────────┘   │
│           ↑                          ↓                   │
│  ┌────────────────┐      ┌───────────────────────────┐  │
│  │  Conversation  │      │  Suggestion UI Emitter    │  │
│  │  State Buffer  │      │  (non-blocking, async)    │  │
│  │  (last N turns)│      └───────────────────────────┘  │
│  └────────────────┘                  ↓                   │
│                           ┌───────────────────────────┐  │
│                           │  Event Logger             │  │
│                           │  → local SQLite           │  │
│                           └───────────────────────────┘  │
│                                                          │
│  Config: frozen resolved config (loaded at session start)│
└──────────────────────────────────────────────────────────┘
```

### 2.3 Conversation State Buffer

The engine maintains a rolling buffer of the last N turns (N = configurable, default 5). This buffer holds derived signals only — not raw message content:

```python
TurnSignal:
  turn_id:              str
  role:                 'user' | 'agent'
  message_length:       int           # character count
  sentiment_score:      float         # -1.0 to 1.0, local inference
  frustration_score:    float         # 0.0 to 1.0, rule-based
  intent_embedding:     list[float]   # local embedding vector
  contains_correction:  bool
  contains_explicit_de: bool          # dead-end language detected
  artifact_produced:    bool          # agent turn only
  agent_similarity:     float         # similarity to previous agent turn
```

Raw message text is never stored in the buffer beyond what is needed for the current turn's signal computation. Once signals are derived, the text is discarded from the buffer.

---

## 3. Signal Rules

### 3.1 Frustration Language Signal

**What it detects:** User expressing that the agent's output is wrong, misaligned, or missing the point.

**Detection logic:**
```
frustration_word_count = count of matches in user message against:
  negations:    ["no,", "no that's", "not what i", "that's not", "that is not"]
  corrections:  ["wrong", "incorrect", "missing", "you missed", "not right"]
  restarts:     ["forget that", "ignore that", "start over", "try again", "never mind"]
  frustration:  ["still not", "still failing", "same error", "same issue", "again"]

frustration_score = min(frustration_word_count / threshold_max, 1.0)
```

**Signal fires when:** `frustration_score > config.thresholds.frustration_score_min`

**Window:** Evaluated on the current user turn only, not a rolling window. The rolling trend (slope of frustration scores over last 5 turns) is computed separately as a contextual amplifier.

---

### 3.2 Semantic Repetition Signal

**What it detects:** The user restating the same intent multiple times with different wording, suggesting the agent is not converging on what the user wants.

**Detection logic:**
```
For each pair of user turns in the last repetition_window turns:
  similarity = cosine_similarity(intent_embedding[turn_i], intent_embedding[turn_j])

repetition_score = max(similarity scores across all pairs in window)
```

**Embedding model:** A local embedding model (nomic-embed-text or all-MiniLM-L6 via ONNX runtime). Runs entirely on-device, CPU-only, < 100MB model size. Embedding is computed once per user turn and stored in the buffer.

**Signal fires when:** `repetition_score > config.thresholds.semantic_similarity`

---

### 3.3 Stagnation Signal

**What it detects:** Conversation running a long time without producing a concrete output or the user acknowledging progress.

**Detection logic:**
```
turns_since_artifact = count of turns since last artifact_produced = True
turns_since_acknowledgment = count of turns since user used affirmative language:
  ["good", "that works", "perfect", "yes", "correct", "got it", "done"]

stagnation_score = turns_since_artifact / config.thresholds.stagnation_turn_count
stagnation_score = min(stagnation_score, 1.0)
```

**Signal fires when:** `stagnation_score > 0.85` (i.e., within 15% of threshold)

**Note:** This signal is highly domain-dependent. Developer sessions naturally run longer without artifacts than writer sessions. Persona-aware threshold sets (see Section 5.4) address this.

---

### 3.4 Circular Agent Signal

**What it detects:** The agent producing structurally similar outputs across consecutive turns, suggesting it's stuck in a loop rather than making genuine progress.

**Detection logic:**
```
agent_output_similarity = cosine_similarity(
  intent_embedding[current_agent_turn],
  intent_embedding[previous_agent_turn]
)

circular_score = agent_output_similarity
```

**Signal fires when:** `circular_score > config.thresholds.agent_circular_similarity`

**Combined with:** This signal is strongest when paired with the frustration signal. A circular agent AND a frustrated user is a high-confidence dead end.

---

### 3.5 Explicit Dead End Signal

**What it detects:** The user explicitly declaring the path is stuck or failed.

**Detection logic:**
```
explicit_dead_end = any match in user message against:
  declarations: ["dead end", "this isn't working", "not going anywhere"]
  commands:     ["start over", "go back", "from the beginning"]
  admissions:   ["i'm stuck", "we're stuck", "nothing is working"]
```

**Signal fires when:** Any match found. This is a hard trigger — confidence is immediately set to 1.0 regardless of other signals.

**Response:** When the explicit dead end signal fires, the suggestion is shown without waiting for a combined score threshold. The user already declared it; the product should respond immediately.

---

### 3.6 Rollup — Signal Active Matrix

At any point in the conversation, the engine maintains a matrix of which signals are active and their current scores:

```
Signal              | Active | Score | Threshold | Weight
--------------------|--------|-------|-----------|-------
frustration         | true   | 0.71  | 0.50      | 0.40
semantic_repetition | true   | 0.84  | 0.75      | 0.35
stagnation          | false  | 0.31  | 0.85      | 0.25
circular_agent      | false  | 0.22  | 0.80      | 0.25
explicit_dead_end   | false  | 0.00  | 0.01      | 1.00
```

---

## 4. Scoring & Decision Model

### 4.1 Combined Score Computation

```python
def compute_combined_score(signals: dict, config: ResolvedConfig) -> float:
    active_signals = {k: v for k, v in signals.items() if v['active']}
    
    if not active_signals:
        return 0.0
    
    # Explicit dead end overrides everything
    if signals['explicit_dead_end']['active']:
        return 1.0
    
    weighted_sum = sum(
        signals[name]['score'] * config.weights[name]
        for name in active_signals
    )
    total_weight = sum(config.weights[name] for name in active_signals)
    
    combined = weighted_sum / total_weight
    
    # Amplify if frustration trend is worsening
    if frustration_trend_slope(last_5_turns) > 0.1:
        combined = min(combined * 1.15, 1.0)
    
    return combined
```

### 4.2 Decision Threshold

```python
SUGGESTION_THRESHOLD = config.thresholds.combined_score_min  # default: 0.60

if combined_score >= SUGGESTION_THRESHOLD:
    emit_branch_suggestion(
        score=combined_score,
        active_signals=active_signals,
        confidence=combined_score
    )
```

### 4.3 Cooldown

Once a suggestion is shown, the engine enters a cooldown period. It does not show another suggestion until:
- The user takes at least 2 more turns, OR
- The explicit dead end signal fires (bypasses cooldown)

This prevents the engine from spamming suggestions in a struggling session.

---

## 5. Three-Layer Config System

### 5.1 Layer Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 3 — Local Learned Config                             │
│  Source: local calibration from this user's history        │
│  Priority: HIGHEST — overrides global when confident        │
│  Activation: only when confidence_count >= minimum          │
├─────────────────────────────────────────────────────────────┤
│  Layer 2 — User Explicit Config                             │
│  Source: user manually set in settings                      │
│  Priority: ALWAYS wins over all automation                  │
│  Activation: immediate, permanent until user changes it     │
├─────────────────────────────────────────────────────────────┤
│  Layer 1 — Global Shipped Config                            │
│  Source: Anthropic calibration pipeline, ships with update  │
│  Priority: LOWEST — the safe baseline for all users         │
│  Activation: always present, used when layers 2 & 3 absent  │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Layer 1 — Global Config Structure

```json
{
  "source": "global",
  "version": "1.4.2",
  "released_at": "2025-06-01",
  "calibrated_from_sessions": 142000,
  "thresholds": {
    "frustration_score_min": 0.50,
    "semantic_similarity": 0.81,
    "stagnation_turn_count": 7,
    "repetition_window": 3,
    "agent_circular_similarity": 0.80,
    "combined_score_min": 0.60
  },
  "weights": {
    "frustration": 0.40,
    "semantic_repetition": 0.35,
    "stagnation": 0.25,
    "circular_agent": 0.25,
    "explicit_dead_end": 1.00
  },
  "persona_overrides": {
    "developer": {
      "stagnation_turn_count": 12
    },
    "writer": {
      "stagnation_turn_count": 5
    },
    "analyst": {
      "stagnation_turn_count": 10,
      "semantic_similarity": 0.85
    }
  }
}
```

### 5.3 Layer 2 — User Explicit Config Structure

Stored in a separate file from local learned config. Never overwritten by calibration.

```json
{
  "source": "user_explicit",
  "last_modified": "2025-05-15T14:32:00Z",
  "thresholds": {
    "stagnation_turn_count": 14
  }
}
```

### 5.4 Layer 3 — Local Learned Config Structure

```json
{
  "source": "local_learned",
  "last_calibrated": "2025-05-20T09:10:00Z",
  "sessions_in_window": 47,
  "thresholds": {
    "frustration_score_min": {
      "value": 0.44,
      "confidence_count": 63,
      "min_confidence_to_activate": 20,
      "active": true
    },
    "semantic_similarity": {
      "value": 0.76,
      "confidence_count": 11,
      "min_confidence_to_activate": 20,
      "active": false
    },
    "stagnation_turn_count": {
      "value": 9,
      "confidence_count": 38,
      "min_confidence_to_activate": 20,
      "active": true
    }
  }
}
```

`active: false` means the local value exists but hasn't yet accumulated enough evidence to override the global value. The engine falls through to Layer 1 for that threshold.

---

## 6. Config Resolution & Caching

### 6.1 Resolution Logic

Run once at app startup (and again after each calibration trigger). Produces a single `ResolvedConfig` object cached in memory and persisted as a small JSON file.

```python
def resolve_config(global_cfg, user_explicit_cfg, local_learned_cfg) -> ResolvedConfig:
    resolved = {}
    
    for threshold_name in global_cfg.thresholds:
        # Layer 2: user explicit always wins
        if threshold_name in user_explicit_cfg.thresholds:
            resolved[threshold_name] = {
                'value': user_explicit_cfg.thresholds[threshold_name],
                'source': 'user_explicit'
            }
            continue
        
        # Layer 3: local learned wins when confident
        local = local_learned_cfg.thresholds.get(threshold_name)
        if local and local['active']:
            resolved[threshold_name] = {
                'value': local['value'],
                'source': 'local_learned',
                'confidence': local['confidence_count']
            }
            continue
        
        # Layer 1: global baseline fallback
        resolved[threshold_name] = {
            'value': global_cfg.thresholds[threshold_name],
            'source': 'global'
        }
    
    return ResolvedConfig(thresholds=resolved, weights=global_cfg.weights)
```

### 6.2 Session Lifecycle

```
App starts
    ↓
Resolution logic runs
    ↓
ResolvedConfig cached (memory + file)
    ↓
User opens session
    ↓
Engine loads ResolvedConfig → FROZEN for session duration
    ↓
Session runs (engine uses frozen config, events logged)
    ↓
User closes session
    ↓
Calibration pass runs on session's events
    ↓
local_learned_cfg updated
    ↓
ResolvedConfig rebuilt and re-cached
    ↓
Ready for next session
```

**Key invariant:** The ResolvedConfig object loaded at session open is never modified during the session. Any calibration results are written to the next session's config, not the current one.

---

## 7. Calibration Triggers

### 7.1 Session End Calibration

**When:** User explicitly closes a session or session times out after inactivity.  
**Scope:** Events from the just-completed session only.  
**Speed:** Milliseconds — processes one session's worth of events.  
**What it updates:** Confidence counts, true/false positive tallies in `local_engine_calibration` table.

```python
def calibrate_on_session_end(session_id: str):
    events = db.query(
        "SELECT * FROM branch_events WHERE session_id = ?",
        session_id
    )
    for event in events:
        for signal_name in event.heuristic_signals_active:
            if event.user_response == 'accepted':
                db.increment_true_positive(signal_name)
            elif event.user_response == 'dismissed':
                db.increment_false_positive(signal_name)
        
        if event.user_response == 'manual_branch':
            # Suggestion wasn't shown — potential false negative
            # Use active signal state at that moment to identify culprit
            db.increment_false_negative_candidates(event)
    
    recompute_threshold_values()
    rebuild_resolved_config()
```

### 7.2 App Startup Calibration

**When:** Every time the app starts, before any session begins.  
**Scope:** Full rolling window (last 90 days or 500 sessions).  
**Speed:** Under 1 second on any reasonable hardware.  
**What it updates:** All threshold values from scratch based on accumulated history. Also handles global config version changes.

### 7.3 Confidence Threshold Crossing

**When:** A threshold's confidence count crosses its `min_confidence_to_activate` value after a session end calibration.  
**Scope:** That specific threshold only.  
**Speed:** Milliseconds.  
**What it does:** Re-resolves that threshold in the cached ResolvedConfig. The new value activates at the next session start.

This is event-driven, not time-driven. The engine becomes progressively more personalized as individual thresholds accumulate enough evidence, one by one.

### 7.4 Manual Trigger

**When:** User initiates from settings ("Recalibrate engine").  
**Scope:** Full rolling window.  
**Speed:** Under 1 second.  
**Use cases:** After a batch of sessions, after accepting a major global config update, after manually adjusting Layer 2 overrides.

### 7.5 Long Session Soft Checkpoint

**When:** Inactivity gap of > 30 minutes within an active session.  
**Scope:** Turns completed so far in the session.  
**What it does:** Writes confidence counts and true/false positive tallies to the database. **Does NOT** rebuild the resolved config or change the frozen config for the ongoing session.  
**Purpose:** Ensures signal data isn't lost if the app is force-closed before session end.

---

## 8. Agent Tree Awareness

At session start, the agent receives a `tree_context` block injected into its system prompt. This gives the agent awareness of its position in the conversation tree and instructs it on how to behave as a participant in the branching mechanic.

### 8.1 Context Block Structure

```
### Conversation Tree Context

You are operating in a structured conversation workspace that tracks branches 
and decisions. Here is the current state of this session:

Current branch: {branch_label} (status: {status})
Branch depth: {depth} ({depth_description})
Total branches in session: {total_branches}

Prior branches explored:
{for each prior branch:}
  - {branch_label} ({status}): {one_line_summary}
    Reason abandoned: {abandonment_reason}
{end for}

### Behavioral Instructions

When you identify a point in the conversation where multiple viable approaches 
exist, present them as explicitly numbered options rather than choosing one 
unilaterally. Label each option clearly so the user can select which direction 
to explore.

If you recognize that an approach you are about to suggest was already attempted 
in a prior branch and did not succeed, flag this explicitly: state that this 
approach was tried, what the outcome was, and what you propose differently 
this time.

If you reach a point where you genuinely cannot make further progress on the 
current path, say so clearly rather than producing low-quality output or 
repeating variations of a failed approach.
```

### 8.2 Branch Summary Generation

Each branch summary (one line, included in the context block) is generated lazily when the branch is closed or marked as a dead end. Generated by the same LLM, as a quick summarization call:

```
Prompt: "Summarize this conversation branch in one sentence, focusing on 
what was attempted and why it ended. Maximum 15 words."
```

The summary is stored in the `branches` table and reused for subsequent sessions. It is never regenerated unless the user explicitly refreshes it.

### 8.3 Agent-Offered Branch Stubs

When the agent presents multiple options, the frontend detects the numbered option structure in the response and renders each option as a **branch stub** in the tree — a dashed teal node indicating an unexplored path. When the user selects an option, the stub is activated and becomes a real branch.

Detection pattern (frontend):
```
Agent response contains a numbered list where:
  - List has 2-4 items
  - Each item starts with a number or "Option"
  - Each item is a full approach description (> 20 words)
  → Render as branch stubs
```

---

## 9. Engine Capability & Limitations

### 9.1 What the Engine Does Well

- **Pattern detection for defined signals** — frustration language, repetition, stagnation, circular agent outputs, explicit declarations. Reliable and fast for the majority of obvious dead ends.
- **Consistency** — same inputs always produce the same output. Behavior is predictable, debuggable, and explainable to the user ("This was suggested because you've expressed the same intent 3 times").
- **Resource profile** — zero model overhead, microsecond execution, offline, works on any hardware. This never degrades.
- **Personalization over time** — threshold calibration pushes the accuracy ceiling upward as local evidence accumulates. A user's engine after 3 months is measurably better calibrated to them than a new user's engine.

### 9.2 Known Limitations

| Limitation | Description | Mitigation |
|---|---|---|
| Subtle context shifts | A gradual pivot that produces no frustration language or repetition | User-initiated branching always available |
| Domain-specific norms | Developer sessions look different from writer sessions — global thresholds are a compromise | Persona-aware threshold sets; local calibration narrows this gap |
| Novel dead end types | Dead ends not covered by defined rules won't be detected | New rules added as telemetry surfaces uncovered patterns |
| Nuanced language | Frustration expressed indirectly, sarcastically, or in domain jargon | Semantic similarity partially compensates; acknowledged limitation |
| First-session accuracy | Before any local calibration, engine uses global thresholds which may not fit this user | Global thresholds are a reasonable baseline; improves quickly |

### 9.3 Honest Accuracy Estimate

A well-calibrated heuristic engine is estimated to surface **65–75% of branching moments** that a trained model classifier would catch. For the product's purposes, this is sufficient — the remaining 25–35% is covered by user-initiated branching which is always available and prominently afforded. The target is to be useful, not perfect.

### 9.4 Future Evolution Path

```
Phase 1 (MVP):
  Rule-based engine, Layer 1 global config only, suggestion UI
  → Collect local telemetry silently

Phase 2:
  Layer 3 local calibration activated
  Engine personalizes to individual users
  → Collect opt-in online telemetry

Phase 3:
  Global calibration pipeline operational
  Engine improves for all users via telemetry
  Persona detection added (classify user domain from session patterns)

Phase 4 (if scale justifies):
  Fine-tune a small open-source model on the branch event dataset
  Used as a supplementary signal alongside the rule-based engine
  Never replaces the rule engine — adds a learned layer on top of it
```

---

## 10. Offline Telemetry — Local SQLite

All offline telemetry is stored in a local SQLite database (`~/.convtree/telemetry.db` or equivalent). It never leaves the device. It contains derived signals only — never raw conversation text.

### 10.1 Table: `sessions`

Top-level record for every conversation session.

```sql
CREATE TABLE sessions (
  session_id            TEXT PRIMARY KEY,
  workspace_id          TEXT NOT NULL,
  started_at            DATETIME NOT NULL,
  ended_at              DATETIME,
  total_turns           INTEGER DEFAULT 0,
  total_branches        INTEGER DEFAULT 0,
  total_dead_ends       INTEGER DEFAULT 0,
  primary_topic_hash    TEXT,       -- hash of topic embedding, NOT raw text
  model_used            TEXT,
  session_outcome       TEXT        -- 'completed' | 'abandoned' | 'dead_end' | 'branched_out'
);
```

### 10.2 Table: `turns`

One row per conversation turn. Derived signals only — no content.

```sql
CREATE TABLE turns (
  turn_id                   TEXT PRIMARY KEY,
  session_id                TEXT NOT NULL REFERENCES sessions(session_id),
  branch_id                 TEXT NOT NULL REFERENCES branches(branch_id),
  turn_index                INTEGER NOT NULL,   -- position within session
  branch_turn_index         INTEGER NOT NULL,   -- position within branch
  role                      TEXT NOT NULL,      -- 'user' | 'agent'
  timestamp                 DATETIME NOT NULL,

  -- Derived signals — never raw text
  message_length            INTEGER,            -- character count only
  sentiment_score           REAL,               -- -1.0 to 1.0
  frustration_score         REAL,               -- 0.0 to 1.0
  intent_shift_score        REAL,               -- semantic distance from prev user turn
  agent_similarity_score    REAL,               -- similarity to prev agent turn (agent turns only)
  contains_correction       BOOLEAN DEFAULT 0,
  contains_explicit_de      BOOLEAN DEFAULT 0,  -- explicit dead-end language
  artifact_produced         BOOLEAN DEFAULT 0,
  artifact_type             TEXT                -- 'code' | 'csv' | 'markdown' | 'image' | null
);
```

### 10.3 Table: `branches`

One row per branch in every session.

```sql
CREATE TABLE branches (
  branch_id                 TEXT PRIMARY KEY,
  session_id                TEXT NOT NULL REFERENCES sessions(session_id),
  parent_branch_id          TEXT,               -- null for root
  branch_point_turn_id      TEXT,               -- turn branched from
  created_at                DATETIME NOT NULL,

  -- How and why this branch was created
  trigger_type              TEXT NOT NULL,      -- 'user_explicit' | 'system_suggested'
                                                -- | 'agent_offered' | 'correction'
  trigger_signal            TEXT,               -- which heuristic rule fired, if any

  -- Structural context
  turns_back_from_tip       INTEGER,            -- how far back from last turn was branch point
  branch_depth              INTEGER NOT NULL,   -- nesting level (root = 0)
  turn_count                INTEGER DEFAULT 0,  -- turns added to this branch

  -- Outcome
  outcome                   TEXT,               -- 'active' | 'dead_end' | 'completed' | 'abandoned'
  dead_end_type             TEXT,               -- 'technical' | 'circular' | 'quality' | 'relevance'
  produced_artifact         BOOLEAN DEFAULT 0,
  resolution_time_seconds   INTEGER,
  summary_text              TEXT                -- one-line LLM-generated summary, generated lazily
);
```

### 10.4 Table: `branch_events`

Granular record of every engine firing event. The richest table for local calibration.

```sql
CREATE TABLE branch_events (
  event_id                      TEXT PRIMARY KEY,
  branch_id                     TEXT NOT NULL REFERENCES branches(branch_id),
  session_id                    TEXT NOT NULL REFERENCES sessions(session_id),
  timestamp                     DATETIME NOT NULL,

  -- What the engine observed
  signals_active                TEXT NOT NULL,    -- JSON array: ["frustration","repetition"]
  signal_scores                 TEXT NOT NULL,    -- JSON object: {"frustration":0.71,"repetition":0.84}
  combined_score                REAL NOT NULL,
  suggestion_shown              BOOLEAN NOT NULL,
  suggestion_confidence         REAL,

  -- What the user did
  user_response                 TEXT,             -- 'accepted' | 'dismissed' | 'ignored'
                                                  -- | 'manual_branch' (no suggestion was shown)
  response_delay_seconds        INTEGER,          -- time from suggestion shown to user response

  -- Session context at this moment
  turns_since_last_branch       INTEGER,
  consecutive_similar_turns     INTEGER,
  session_frustration_trend     REAL,             -- slope of frustration scores over last 5 turns
  prior_branch_outcome          TEXT,             -- outcome of last completed branch in this session
  branch_depth_at_event         INTEGER,

  -- Config state that produced this evaluation
  config_version_global         TEXT,
  config_version_local          TEXT,             -- hash of current local learned config
  active_layer_per_threshold    TEXT              -- JSON: which layer was used for each threshold
);
```

### 10.5 Table: `local_engine_calibration`

Tracks threshold state and evidence accumulation. Updated by calibration triggers.

```sql
CREATE TABLE local_engine_calibration (
  threshold_name            TEXT PRIMARY KEY,
  current_value             REAL NOT NULL,
  confidence_count          INTEGER NOT NULL DEFAULT 0,
  min_confidence_threshold  INTEGER NOT NULL DEFAULT 20,
  true_positive_count       INTEGER NOT NULL DEFAULT 0,   -- suggestion accepted
  false_positive_count      INTEGER NOT NULL DEFAULT 0,   -- suggestion dismissed
  false_negative_count      INTEGER NOT NULL DEFAULT 0,   -- user branched, no suggestion shown
  precision_rolling         REAL,                         -- true_pos / (true_pos + false_pos)
  recall_rolling            REAL,                         -- true_pos / (true_pos + false_neg)
  last_updated              DATETIME,
  source                    TEXT NOT NULL                 -- 'global_adopted' | 'local_learned'
);
```

### 10.6 Table: `session_embeddings`

Stores lightweight topic embeddings for session similarity — used to enable context-aware branching suggestions in future phases.

```sql
CREATE TABLE session_embeddings (
  session_id        TEXT PRIMARY KEY REFERENCES sessions(session_id),
  embedding_model   TEXT NOT NULL,
  embedding_vector  BLOB NOT NULL,    -- serialized float array
  created_at        DATETIME NOT NULL
);
```

---

## 11. Online Telemetry — Anonymous Global

### 11.1 Design Constraints

Every field in every online event must pass this test before inclusion:

> *"Does this field improve the accuracy of a branching threshold, and is it impossible to use it to reconstruct conversation content or identify a specific user or session?"*

Fields that fail either test are excluded. The online schema is designed from the privacy constraint outward, not trimmed from the offline schema down.

### 11.2 Bucketing Rules

All exact numeric values are bucketed before transmission:

```
Turn counts:      [1-5] [6-10] [11-20] [21-40] [41+]
Session length:   "short" (1-10) | "medium" (11-30) | "long" (31+)
Response delay:   "immediate" (<3s) | "considered" (3-15s) | "delayed" (>15s)
Turns back:       [1] [2-3] [4-6] [7+]
Platform:         "local_small" | "local_large" | "cloud"
Time of day:      "morning" | "afternoon" | "evening" | "night"  (not exact time)
```

### 11.3 Event: `branch_signal`

Emitted once per engine firing event (suggestion shown or not shown but manual branch taken).

```json
{
  "schema_version": "1.2",
  "event_type": "branch_signal",
  "event_token": "anon-evt-8f3a",

  "signal_context": {
    "signals_active": ["frustration_language", "semantic_repetition"],
    "signal_scores": {
      "frustration_language": 0.71,
      "semantic_repetition": 0.84,
      "stagnation": 0.22
    },
    "combined_score": 0.63,
    "suggestion_shown": true,
    "suggestion_confidence": 0.63
  },

  "session_context": {
    "turn_index_bucket": "10-20",
    "turns_since_last_branch": 6,
    "branch_depth": 1,
    "session_length_bucket": "medium",
    "prior_branch_outcome": "dead_end",
    "session_frustration_trend": "increasing"
  },

  "outcome": {
    "user_response": "accepted",
    "response_delay_bucket": "considered",
    "turns_back_branched": 3,
    "branch_resolved_as": "completed"
  },

  "engine_context": {
    "engine_version": "1.4.2",
    "global_config_version": "1.4.2",
    "local_overrides_active_count": 2,
    "platform_bucket": "local_small"
  }
}
```

### 11.4 Event: `dead_end`

Emitted once per dead end — either detected by heuristics or declared by the user.

```json
{
  "schema_version": "1.2",
  "event_type": "dead_end",
  "event_token": "anon-evt-2c9b",

  "detection_context": {
    "detected_by": "heuristic",
    "dead_end_type": "circular",
    "signals_present": ["semantic_repetition", "circular_agent"],
    "turns_in_dead_branch_bucket": "6-10",
    "turn_index_bucket": "15-20",
    "consecutive_agent_similar_turns": 3
  },

  "recovery_context": {
    "recovery_taken": true,
    "recovery_type": "branch_back",
    "turns_back_to_branch_point_bucket": "4-6",
    "recovery_outcome": "completed",
    "recovery_turns_bucket": "6-10"
  },

  "engine_context": {
    "engine_version": "1.4.2",
    "global_config_version": "1.4.2"
  }
}
```

### 11.5 Event: `false_signal`

Emitted when a suggestion was shown but the user dismissed it. Critical for reducing false positive rates in the global engine.

```json
{
  "schema_version": "1.2",
  "event_type": "false_signal",
  "event_token": "anon-evt-7d1f",

  "signal_context": {
    "signals_active": ["stagnation"],
    "combined_score": 0.61,
    "suggestion_confidence": 0.61
  },

  "dismissal_context": {
    "response_delay_bucket": "immediate",
    "turns_continued_after_bucket": "1-5",
    "session_outcome_after": "completed",
    "artifact_produced_after": true
  },

  "engine_context": {
    "engine_version": "1.4.2",
    "global_config_version": "1.4.2",
    "stagnation_threshold_source": "global"
  }
}
```

### 11.6 Event: `session_summary`

Emitted once per session. Gives the calibration pipeline per-session aggregate performance data.

```json
{
  "schema_version": "1.2",
  "event_type": "session_summary",
  "event_token": "anon-sess-4e8c",

  "session_stats": {
    "total_turns_bucket": "10-20",
    "total_branches": 2,
    "total_dead_ends": 1,
    "branch_acceptance_rate": 0.67,
    "suggestion_shown_count": 3,
    "session_outcome": "completed",
    "session_length_bucket": "medium"
  },

  "engine_performance": {
    "true_positives": 2,
    "false_positives": 1,
    "false_negatives": 0,
    "precision": 0.67,
    "recall": 1.0,
    "f1_score": 0.80
  },

  "config_context": {
    "global_config_version": "1.4.2",
    "local_overrides_active_count": 2,
    "platform_bucket": "local_small",
    "model_type_bucket": "local_small"
  }
}
```

---

## 12. The Extraction Boundary

The extraction boundary is a **one-way, lossy transformation** that runs locally before any data is transmitted. It is the only code path that produces the online telemetry format. No other component knows what the online schema looks like.

### 12.1 Transformation Rules

```
Local field                     →  Online field
─────────────────────────────────────────────────────────────────────
session_id (UUID)               →  anonymous event_token (session-scoped, not linkable)
turn_index (exact int)          →  turn_index_bucket (bracketed range)
timestamp (datetime)            →  dropped entirely
response_delay_seconds (exact)  →  response_delay_bucket
signal_scores (exact floats)    →  kept as-is (pure derived scores, no text)
turns_in_dead_branch (exact)    →  turns_in_dead_branch_bucket
model_used (exact name)         →  platform_bucket (local_small / local_large / cloud)
workspace_id                    →  dropped entirely
session_topic_hash              →  dropped entirely
session_embedding               →  dropped entirely
config threshold sources        →  kept (which layer — global/local/explicit — was active)
```

### 12.2 Transmission Schedule

Events are queued locally and transmitted in batches on a **randomized schedule** — not per-event. This prevents timing fingerprinting (the correlation of transmission times with session activity).

```
Batch conditions:
  - At least 10 events queued, AND
  - At least 4 hours since last transmission, AND
  - Randomized delay: additional 0–2 hours added to transmission time
  - Maximum queue hold: 72 hours (flush regardless of event count)

Transmission:
  - Single HTTPS POST to telemetry ingestion endpoint
  - Payload: JSON array of events
  - No session continuity between batches (tokens not linkable)
  - No response body expected or processed beyond HTTP status
```

### 12.3 Architectural Enforcement

The extraction pipeline is isolated as a single module with a documented interface. Any proposed addition of a new online field requires:

1. A review of whether the field passes the anonymity test
2. A documented justification for why it improves calibration
3. An update to `TELEMETRY.md` (public-facing documentation of what is collected)

This constraint is enforced by code review policy, not technical lock. The discipline is architectural, not cryptographic.

---

## 13. Global Calibration Pipeline

The global pipeline runs on Anthropic infrastructure, periodically, and produces an updated `global_config.json` shipped with product releases. It never runs on user devices.

### 13.1 What It Answers

- Which signal combinations have the highest acceptance rate?
- At what combined score threshold do users accept vs. dismiss suggestions?
- Which individual signals produce the most false positives?
- Which dead end patterns reliably lead to productive recovery?
- Do these patterns differ meaningfully by session length, branch depth, or platform type?

### 13.2 Calibration Logic (simplified)

```python
def calibrate_threshold(signal_name, events):
    # Find the combined_score value that maximizes F1
    # across all events where this signal was the primary driver
    
    accepted = [e for e in events if signal_name in e.signals_active
                and e.user_response == 'accepted']
    dismissed = [e for e in events if signal_name in e.signals_active
                 and e.user_response == 'dismissed']
    
    candidate_thresholds = [0.40, 0.45, 0.50, 0.55, 0.60, 0.65, 0.70, 0.75, 0.80]
    
    best_threshold = max(
        candidate_thresholds,
        key=lambda t: f1_score(accepted, dismissed, threshold=t)
    )
    
    return best_threshold
```

### 13.3 Output

A `global_config.json` file (see Section 5.2 structure) with updated thresholds and weights. Shipped as part of the next product release. The file is versioned and dated. Changelog entry documents what changed and by how much.

---

## 14. The Feedback Loop — Evolution Phases

### Phase 1 — MVP (manual branching only)

- Users branch explicitly, no engine suggestions yet
- Heuristic engine rules present but suggestion UI not displayed
- All branch events logged to local SQLite
- No online telemetry
- Goal: collect clean baseline data on when users actually branch

### Phase 2 — Local Calibration

- Suggestion UI enabled
- Local calibration pipeline activated (session end + startup triggers)
- Engine begins personalizing to individual users
- Opt-in online telemetry enabled
- Goal: prove that local calibration measurably improves acceptance rate vs. global baseline

### Phase 3 — Global Calibration

- Online telemetry accumulation sufficient for global calibration pipeline
- First globally-calibrated `global_config.json` shipped
- Persona detection added: session patterns classified into user type (developer, writer, analyst) to enable persona-specific threshold overrides
- Goal: demonstrate that globally-calibrated config outperforms initial hardcoded thresholds

### Phase 4 — Supplementary Learned Layer (scale-dependent)

- If consenting user base is large enough to produce a meaningful labeled dataset
- Fine-tune a small open-source model (< 1B params) on the branch event dataset
- Used as an additional signal input to the rule engine, not a replacement
- Runs locally via Ollama or ONNX
- Goal: close the 25–35% gap between rule-based accuracy and trained classifier accuracy for the most common session types

---

## 15. Privacy Architecture

### 15.1 Solo Tier Guarantee

The following guarantee is architectural, not policy-based:

```
What stays local (always):
  ├── All conversation content
  ├── All branch event logs
  ├── Heuristic calibration data
  ├── Session embeddings and topic hashes
  ├── Generated artifact files
  └── All workspace and session metadata

What leaves the device:
  └── LLM API calls to the endpoint the user configured
      (Ollama local: nothing leaves; cloud API key: only the prompt/response)

What leaves the device if user opts in:
  └── Anonymous, bucketed telemetry events (no text, no IDs, no timestamps)
      Batch transmitted, randomized schedule
```

### 15.2 Data Minimization Principle

The offline telemetry schema stores only derived signals, never source text. This is both a privacy protection and a practical engineering decision: the calibration logic only needs behavioral signals, not conversation content. Storing content would add storage overhead, create a privacy liability, and provide no calibration benefit.

### 15.3 User Telemetry Controls

In settings, under "Privacy & Telemetry":

```
Anonymous telemetry (default: OFF)
  [ ] Send anonymous branch signal events
      Helps improve suggestion timing for all users.
      No conversation content. No identifying information.

  [ ] Send session performance summaries
      Helps calibrate dead end detection globally.
      Turn counts and outcomes only.

  [ ] Send false signal feedback
      Helps reduce unwanted suggestions.
      Signal scores and dismissal context only.

  [View exactly what is sent →]   [Download my local data →]
```

Each checkbox is independent. No bundled consent. "View exactly what is sent" links to a live log of the last 10 events that would be transmitted, formatted exactly as they appear in the online schema.

---

## 16. Config Update Handling

When a new global config ships with a product update, the following logic runs at next startup:

```python
def handle_global_config_update(new_global, current_resolved, local_learned, user_explicit):
    for threshold_name in new_global.thresholds:
        
        # User explicit: never touched by updates
        if threshold_name in user_explicit.thresholds:
            continue
        
        # Local learned with high confidence: note the update, don't apply it
        local = local_learned.thresholds.get(threshold_name)
        if local and local['active']:
            # Store new global value for reference but local remains authoritative
            local['global_reference_value'] = new_global.thresholds[threshold_name]
            local['global_reference_version'] = new_global.version
            continue
        
        # Below confidence threshold: adopt new global value immediately
        # (user was already on global value anyway)
        pass  # resolved config rebuild will pick up new global value
    
    # Check for drift: if gap between local learned and new global > 20%,
    # queue a re-evaluation nudge for the user
    for threshold_name, local in local_learned.thresholds.items():
        if not local['active']:
            continue
        new_global_val = new_global.thresholds.get(threshold_name)
        if new_global_val:
            gap = abs(local['value'] - new_global_val) / new_global_val
            if gap > 0.20:
                queue_drift_nudge(threshold_name, local['value'], new_global_val)
    
    rebuild_resolved_config()
```

The drift nudge surfaces as a quiet, dismissible notification: *"A new global calibration is available for [threshold]. Your local value is [X]; the new global baseline is [Y]. Would you like to reset this threshold and let it re-learn from your recent sessions?"*

---

## 17. Local Data Lifecycle

### 17.1 Rolling Window Management

```
Raw event tables (turns, branch_events):
  Retention:  90 days OR 500 sessions (whichever limit is reached first)
  Pruning:    Oldest sessions removed first when limit reached
  On prune:   Aggregate statistics updated before row deletion

Aggregate tables (local_engine_calibration):
  Retention:  Lifetime (rows are updated in place, never deleted)
  Size:       Fixed — one row per threshold name (~15 rows maximum)

session_embeddings:
  Retention:  Last 200 sessions
  On prune:   Oldest removed first

sessions, branches tables:
  Retention:  90 days
  Note:       Turn content was never stored, so pruning is low-risk
```

### 17.2 Storage Estimate

```
Per session (average 15 turns, 2 branches):
  sessions row:           ~200 bytes
  turns rows:             ~300 bytes × 15 = 4.5 KB
  branches rows:          ~400 bytes × 2 = 0.8 KB
  branch_events rows:     ~500 bytes × 3 (avg events) = 1.5 KB
  Total per session:      ~7 KB

500 sessions rolling window:
  Raw event data:         ~3.5 MB
  session_embeddings:     ~200 sessions × 1.5 KB (768-dim float32) = ~300 KB
  Calibration table:      ~2 KB (fixed size)
  Total local DB:         ~4 MB
```

The local telemetry database stays small enough to fit on any device. No cleanup prompts, no storage warnings, no degradation over time.

### 17.3 User Data Export & Deletion

Users can at any time:
- **Export** all local telemetry as a JSON archive (`~/.convtree/export/telemetry_export.json`)
- **Delete** all local telemetry and reset the engine to global defaults
- **Inspect** the current resolved config and see which layer each threshold came from

These options are in settings under "Privacy & Telemetry". Deletion is permanent and not reversible — the UI makes this clear before confirming.
