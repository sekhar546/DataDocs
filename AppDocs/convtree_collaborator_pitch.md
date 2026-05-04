# ConvTree
### A new kind of AI workspace — where conversations become structured thinking

---

## The Problem

Every AI chat app today has the same flaw. Conversations are a single scrolling log. You ask, the agent answers, you ask again. When the agent goes down the wrong path, you lose everything. When you want to try a different approach, you start over. When a session ends and you want to reference what was decided — and why — you're scrolling through walls of text trying to reconstruct the reasoning.

The conversations we have with AI agents aren't actually linear. They branch. We make decisions, hit dead ends, try different approaches, correct wrong assumptions. We produce files, code, reports. All of that structure is real — but current tools throw it away and hand you a flat log instead.

---

## The Idea

What if the conversation was a **tree**?

Not a visualization bolted on as an afterthought. The tree *is* the interface. Every session is a directed graph — turns, decisions, branches, dead ends, outputs. You can see where you've been, why a path failed, and where the productive branches went. You can navigate back to any decision point and try a different direction without losing the work you've already done.

And every file the agent produces — code, reports, spreadsheets, analyses — is automatically saved and linked to the exact point in the conversation that created it. Not lost in the scroll. Organized, named, and ready to use.

This is ConvTree.

---

## How It Works

**The conversation panel** looks familiar — you talk to the AI, it responds. But on the left side, running alongside your messages like a git graph, you see branch lanes. Thin colored lines showing the shape of your session. The current branch is prominent. Other branches are quiet indicators you can navigate to in one click.

**When the conversation hits a dead end** — because it goes in circles, or the agent can't do what's needed, or you realize you're solving the wrong problem — the product surfaces it. It shows you exactly where in the tree the path diverged, highlights the last good decision point, and lets you branch from there cleanly. The dead end isn't deleted. It stays in the tree as a record of what was tried.

**As you work**, every code block, file, report, or structured output the agent produces is captured automatically into a session files panel. Each file is linked to the branch and the turn that produced it. Version history is preserved if you regenerate something.

**Over time**, the product learns your working style locally — how far back you tend to branch, what signals precede your dead ends — and calibrates its suggestions accordingly. All on your device. Nothing leaves your machine.

---

## Who It's For

**Individual developers and data engineers** who use AI agents for real work — debugging pipelines, writing code, analyzing data, drafting documentation. People who've felt the frustration of a 40-turn session collapsing into an unusable scroll. People who want to use AI as a thinking tool, not just a chat box.

**Teams in structured domains** — healthcare, finance, legal, data engineering — where AI-assisted work needs to be auditable. Where you need to be able to show what was tried, what decisions were made, and why. Where data staying on your infrastructure isn't optional.

**Open source contributors and power users** who want a tool they can extend, self-host, and trust.

---

## What Makes It Different

Most AI chat tools are optimized for the casual user — fast answers to simple questions. ConvTree is optimized for the user doing *serious work* — multi-hour sessions, complex problems, iterated outputs.

The specific gap: no current product treats the **conversation itself as a structured artifact**. Git changed software by making every code change trackable and reversible. ConvTree does the same for AI work sessions.

There's also a practical differentiator for teams: the product can be deployed entirely on your own infrastructure with no data leaving your environment. In industries with data residency requirements — healthcare, finance, government — this isn't a nice-to-have. It's the only acceptable option.

---

## The Product Model

**Solo tier: free, open source, forever.**

If you're working alone, the full product is yours. No paywalls on features that only affect you. No account required. Run it locally with Docker in minutes. This isn't a stripped-down trial — it's the complete product for individual use.

The philosophy: individual users are the community. The community is what makes the product worth using. Crippling the solo experience to push upgrades would poison the thing that makes the product grow.

**Team tier: paid, for organizations.**

When a team needs shared workspaces, user accounts, collaboration on session trees, admin controls, SSO, and integrations with Google Workspace or Microsoft 365 — that's where the business model lives. Organizations can self-host it on their own infrastructure (license) or use managed hosting when that becomes available.

The line between free and paid is principled and simple: **if the feature only serves you, it's free. If the feature requires other people to be in the room, that's the team tier.**

---

## Where We Are

The product architecture is fully designed. Core decisions made:

- **Branching mechanics** — four trigger types, four dead end types, detection-and-suggest model (never auto-branch)
- **Heuristic engine** — local, rule-based, zero model overhead, calibrates to individual users over time using local telemetry, improves globally via opt-in anonymous telemetry
- **Privacy model** — architectural guarantee for the solo tier: nothing leaves the device except calls to the user's own configured LLM endpoint
- **UI/UX** — git-graph-style branch lanes integrated in the conversation scroll, hover-expand files panel, status bar with model switcher, expand-tree overlay for full navigation
- **Data architecture** — local SQLite for behavioral telemetry (no conversation content stored), three-layer engine config (global baseline / user explicit / local learned)
- **Business model** — open core, AGPL solo tier, commercial license for team features

**MVP scope is defined:**
1. Conversation tree view with branch lanes
2. Artifact capture and session files panel
3. Workspace and session organization
4. Heuristic engine with suggestion UI
5. Local telemetry logging
6. Model switcher in status bar
7. Docker-based solo deployment

**Base:** Fork of Open WebUI (Python/FastAPI + SvelteKit). Significant divergence in the frontend — the tree view replaces the linear chat paradigm at the core.

---

## The Opportunity

The space is real and underserved. Nobody has built the conversation-as-tree paradigm for end users in a clean, general-purpose way. The adjacent tools — Open WebUI, LibreChat, LangFlow — are all either linear chat interfaces or agent pipeline builders. None of them treat the session itself as a structured artifact worth organizing and preserving.

The open source community around AI tools is large and growing. A product with a genuinely novel paradigm, a trustworthy open core model, and local-first privacy will find its audience quickly — especially among developers and data engineers who already self-host their tools.

The forks-and-competition question has a clean answer: the moat isn't the code. It's the calibration data from consenting users improving the heuristic engine, the community ecosystem of templates and integrations, and the velocity of a focused team shipping against a clear vision.

---

## What's Needed

A collaborator who can contribute to one or more of:

- **Frontend (SvelteKit)** — the tree visualization and branch lane mechanics are the product's signature feature. Getting this right is the highest-priority engineering work.
- **Backend (Python/FastAPI)** — the artifact capture layer, workspace data model, and heuristic engine integration.
- **Product thinking** — there's more to design (templates, session export, search, team tier UX). A second mind on the product decisions would be valuable.
- **Open source community building** — documentation, contribution guidelines, community presence.

The product has a solo founder with 12 years of healthcare data engineering experience and a clear, fully-specified vision. What's needed now is execution.

---

*This document is a product overview for discussion. A full technical specification covering architecture, data models, engine design, telemetry schemas, and UI spec is available separately.*
