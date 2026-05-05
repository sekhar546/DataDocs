# DataPlumbr — Product Vision

> *Built by a data engineer who's lived the pain. For data engineers who live it daily.*

---

## The Problem

Data engineering teams are the invisible infrastructure that modern companies depend on. When pipelines break — and they always break — teams scramble: digging through logs, posting in Slack threads, doing manual triage with no structure, writing post-mortems in Confluence documents nobody reads. A week later, the same pipeline breaks the same way.

The tools available today force a bad choice:

**Too expensive for mid-market teams**
Monte Carlo, Acceldata — $100K+/year, built for enterprise teams with dedicated platform engineers and multi-month implementation timelines. A 15-person data engineering team at a regional healthcare company or a fintech scaleup cannot justify this.

**Too raw for production use**
Great Expectations, Elementary — powerful open-source frameworks, but you have to build the product on top of them yourself. That means a platform team, ongoing maintenance, and months before you get value.

**No longer available as focused products**
Bigeye — the best mid-market option — was acquired by Collibra in 2023. Metaplane was acquired by Datadog in April 2025. Their former customers are actively looking for a replacement.

**The result:** Mid-market data engineering teams (5–30 engineers, $500K–$5M data infrastructure spend) have no focused, affordable platform for pipeline reliability. They feel the pain more acutely than anyone — pipeline failures have compliance consequences, business impact, and reputation cost — but have been systematically ignored by the vendor landscape.

---

## The Solution

DataPlumbr is a modular data lifecycle platform that meets mid-market data engineering teams where they are, delivers day-one value, and grows with them as their maturity increases.

### Three layers. One platform. One upgrade journey.

```
Layer 3: AI Modeler        ← AI-generated models, dbt scaffolding, lineage  [parked]
Layer 2: Data Contracts    ← Producer-consumer agreements, SLA enforcement   [parked]
Layer 1: Reliability       ← Pipeline monitoring, incidents, post-mortems    [ACTIVE]
```

Teams start with Layer 1 and unlock the next layer as their needs grow. No rip-and-replace. No new vendor. The platform compounds in value the longer it's used.

---

## Layer 1 — Reliability Platform (Active Build)

The entry point and the core product. Treats broken data pipelines the way mature engineering teams treat broken software — with structured detection, triage, response, and learning.

**What it does:**

- Monitors pipeline health across Airflow, Prefect, and custom schedulers
- Detects failures, anomalies, and SLA breaches in real time
- Routes incidents to the right on-call engineer via Slack, PagerDuty, or email
- Provides a structured triage workflow — not just an alert, but a guided response process
- Captures post-mortems in a structured format and surfaces recurring patterns
- Tracks mean time to resolution (MTTR) and pipeline reliability scores over time

**What it replaces:** A Slack channel called `#data-incidents`, a Confluence page called "Pipeline Runbook" that nobody updates, and a team ritual of blame-and-fix with no memory.

---

## Layer 2 — Data Contracts (Planned)

Formal agreements between data producers and consumers, enforced at the pipeline level. A producer commits to a schema, a freshness SLA, and a quality standard. When any commitment is broken, Layer 1 opens an incident automatically.

This is the key integration that competitors don't have: quality failures become reliability incidents. The two layers are connected, not separate tools.

---

## Layer 3 — AI Modeler (Planned)

AI-assisted data model design, dbt scaffold generation, and intelligent lineage mapping. Helps data engineers design better data architecture from the start — not discover problems after the fact.

---

## Who It's For

### The Company

- 200–2,000 employees
- $500K–$5M annual data infrastructure spend
- Running production data pipelines daily
- Cannot justify a $100K+ enterprise contract but has outgrown open-source frameworks
- Any industry — regulated industries (healthcare, fintech, insurance) are the highest-pain segment due to compliance and business consequences of bad data

### The Team

- 5–30 data engineers
- Running on AWS, Snowflake or Databricks, Airflow or Prefect, dbt
- No dedicated platform team to maintain their own observability tooling
- Pipeline failures are a recurring operational headache, not a rare edge case

### The Buyer

- Head of Data Engineering, VP of Data, or Senior Data Engineering Lead
- Has budget authority for tooling up to ~$5K/month
- Primary pain: team time wasted on reactive firefighting
- Primary desire: reliability they can report upward without embarrassment

---

## Competitive Differentiation

### vs. Enterprise Tools (Monte Carlo, Acceldata)

- No minimum $100K contract — accessible from the first month
- No warehouse query tax — monitoring that doesn't scale your Snowflake or Databricks bill
- Self-serve setup — no 3-month implementation project, no professional services required
- Designed for teams of 5–30, not teams of 300+

### vs. Mid-Market (Soda, Atlan)

- Pipeline reliability (Layer 1) is the entry point — not just table-level data quality
- Data contracts are tied to pipeline execution, not standalone quality checks
- Single platform with a clear upgrade arc — reliability → contracts → AI modeling
- Priced for the team, not per seat

### vs. Open Source (Great Expectations, Elementary)

- Managed SaaS — no engineering overhead to maintain the platform itself
- Structured incident response layer — not just test results, but triage and resolution workflows
- Works out of the box — not a framework you have to build a product around

---

## Business Model

SaaS subscription — annual billing primary, monthly available.

| Tier | Includes | Price (target) | Team size |
|---|---|---|---|
| Starter | Layer 1 — core monitoring + alerting | ~$500–1,000/month | Up to 10 engineers |
| Growth | Layer 1 full + Layer 2 alpha | ~$2,000–3,500/month | Up to 30 engineers |
| Scale | All layers + priority support | ~$5,000+/month | 30+ engineers |

Pricing is per data team, not per seat. A team of 25 engineers pays the same as a team of 12 at the same tier. This removes the friction that per-seat enterprise pricing creates and aligns cost with the value delivered — reliable pipelines for the whole team.

**Year one target:** 20 customers at an average of $2,500/month = **$600K ARR.**

---

## The Vision

DataPlumbr becomes the operating system for mid-market data engineering teams.

In the same way that PagerDuty became the default for software incident management, DataPlumbr becomes the default for data pipeline incident management — building deep muscle memory in how teams respond to, learn from, and prevent data failures. The reliability foundation (Layer 1) earns trust that unlocks the governance layer (Layer 2) and the intelligence layer (Layer 3).

The compound effect: a team that starts with DataPlumbr for pipeline monitoring grows into a team that uses DataPlumbr to enforce data contracts across their organisation and design AI-ready data models. Switching cost grows with each layer. Revenue per customer grows with each layer.

Winning mid-market positions DataPlumbr to move upmarket — a proven pattern in enterprise SaaS. The goal is not to start by competing with Monte Carlo. The goal is to win the customers Monte Carlo ignores, build product credibility and revenue, and meet the market as it moves toward us.

---

## Founder Context

Built by a solo data engineer with 12 years of experience running production pipelines across healthcare and financial services industries. The product is built by the person who would buy it — someone who has lived through every failure mode DataPlumbr is designed to solve. Not a domain expert in healthcare business or financial products. A domain expert in data pipeline failure, triage, and recovery — and what it costs a team when those processes are ad hoc.

---

*DataPlumbr — dataplumbr.com | dataplumbr.dev*
