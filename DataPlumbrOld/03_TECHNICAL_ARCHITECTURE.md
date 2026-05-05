# Datastack Platform — Technical Architecture

---

## Overview

Datastack is built as a **monorepo** containing three independently deployable product modules and a shared platform foundation. All modules share the same authentication, billing, notification, and SDK layer. Customers purchase access to modules individually via feature-flagged subscriptions.

---

## High-Level Architecture

```mermaid
flowchart TB
    subgraph Customers["Customer Interfaces"]
        WEB["Web Dashboard\nReact + TypeScript"]
        API["REST API"]
        SDK["Python SDK"]
        CLI["CLI Tool"]
    end

    subgraph Gateway["API Gateway (Node.js / Fastify)"]
        AUTH["Auth middleware"]
        ROUTE["Request router"]
        RATE["Rate limiter"]
        FLAGS["Feature flag enforcer"]
    end

    subgraph Modules["Service Modules"]
        R["🔴 Reliability Service\nFastAPI + PySpark"]
        C["🟢 Contracts Service\nFastAPI"]
        AI["🔵 AI Modeler Service\nFastAPI + LLM"]
    end

    subgraph Shared["Shared Packages"]
        AUTHPKG["auth"]
        BILLING["billing"]
        NOTIF["notifications"]
        SDKPKG["sdk"]
        DB["db"]
    end

    subgraph Infra["AWS Infrastructure"]
        ECS["ECS Fargate"]
        RDS["PostgreSQL RDS"]
        REDIS["ElastiCache Redis"]
        S3["S3"]
        SQS["SQS"]
        CW["CloudWatch"]
    end

    Customers --> Gateway
    Gateway --> Modules
    Modules --> Shared
    Shared --> Infra
```

---

## Repository Structure

```
datastack-platform/                  ← monorepo root
│
├── apps/
│   ├── api-gateway/                 ← Node.js / Fastify
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── feature-flags.ts
│   │   │   │   └── rate-limit.ts
│   │   │   └── index.ts
│   │   └── Dockerfile
│   │
│   ├── web-dashboard/               ← React + TypeScript + Vite
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── reliability/
│   │   │   │   ├── contracts/
│   │   │   │   └── ai-modeler/
│   │   │   ├── shared/
│   │   │   └── App.tsx
│   │   └── Dockerfile
│   │
│   ├── reliability/                 ← Python / FastAPI — ACTIVE
│   │   ├── src/
│   │   │   ├── api/
│   │   │   ├── detection/           ← anomaly detection engine
│   │   │   ├── triage/              ← root cause classifier
│   │   │   ├── connectors/          ← Airflow, dbt, Snowflake
│   │   │   └── main.py
│   │   ├── tests/
│   │   └── Dockerfile
│   │
│   ├── contracts/                   ← Python / FastAPI — SCAFFOLDED
│   │   └── src/
│   │       └── main.py              ← stub only
│   │
│   └── ai-modeler/                  ← Python / FastAPI — SCAFFOLDED
│       └── src/
│           └── main.py              ← stub only
│
├── packages/
│   ├── auth/                        ← JWT, API keys, OAuth
│   ├── billing/                     ← Stripe, feature flags
│   ├── notifications/               ← Slack, email, PagerDuty
│   ├── sdk/                         ← Python customer SDK
│   ├── db/                          ← schemas, migrations, ORM
│   └── shared-types/                ← TypeScript + Python types
│
├── infra/
│   ├── terraform/
│   │   ├── modules/
│   │   │   ├── ecs/
│   │   │   ├── rds/
│   │   │   ├── redis/
│   │   │   ├── s3/
│   │   │   └── sqs/
│   │   ├── environments/
│   │   │   ├── dev/
│   │   │   ├── staging/
│   │   │   └── prod/
│   │   └── main.tf
│   └── docker/
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── deploy-staging.yml
│       └── deploy-prod.yml
│
├── docs/                            ← architecture decision records
├── CLAUDE.md                        ← Claude Code context file
├── turbo.json                       ← Turborepo config
└── package.json                     ← monorepo root
```

---

## Core Data Model

### Multi-tenancy First

Every table carries `tenant_id` from day one. No retrofitting later.

```mermaid
erDiagram
    TENANTS {
        uuid id PK
        string name
        string slug
        string plan
        timestamp created_at
    }

    USERS {
        uuid id PK
        uuid tenant_id FK
        string email
        string role
        timestamp created_at
    }

    PIPELINES {
        uuid id PK
        uuid tenant_id FK
        string name
        string source_type
        string owner_id
        string status
        timestamp last_run_at
    }

    PIPELINE_RUNS {
        uuid id PK
        uuid pipeline_id FK
        uuid tenant_id FK
        string status
        timestamp started_at
        timestamp finished_at
        jsonb metrics
    }

    INCIDENTS {
        uuid id PK
        uuid tenant_id FK
        uuid pipeline_id FK
        string severity
        string root_cause_type
        string status
        timestamp detected_at
        timestamp resolved_at
    }

    INCIDENT_EVENTS {
        uuid id PK
        uuid incident_id FK
        string event_type
        string actor
        text notes
        timestamp created_at
    }

    FEATURE_FLAGS {
        uuid id PK
        uuid tenant_id FK
        string flag_name
        boolean enabled
        timestamp updated_at
    }

    TENANTS ||--o{ USERS : "has"
    TENANTS ||--o{ PIPELINES : "owns"
    TENANTS ||--o{ FEATURE_FLAGS : "has"
    PIPELINES ||--o{ PIPELINE_RUNS : "produces"
    PIPELINES ||--o{ INCIDENTS : "triggers"
    INCIDENTS ||--o{ INCIDENT_EVENTS : "has"
```

---

## Reliability Module — Internal Architecture

```mermaid
flowchart TD
    subgraph Connectors["Connector Layer"]
        AF["Airflow connector"]
        DBT["dbt connector"]
        SF["Snowflake connector"]
        CUST["Custom webhook"]
    end

    subgraph Ingestion["Ingestion Layer"]
        NORM["Normalizer\nStandard pipeline run schema"]
        STORE["Run store\nPostgreSQL"]
    end

    subgraph Detection["Detection Engine"]
        STAT["Statistical baseline\nrolling mean ± std dev"]
        ML["ML anomaly detection\nIsolation Forest / PySpark"]
        RULES["Rule engine\ncustomer-defined thresholds"]
    end

    subgraph Triage["Triage Engine"]
        CLASS["Root cause classifier"]
        SRC["Source issue?"]
        SCHEMA["Schema drift?"]
        TRANS["Transform bug?"]
    end

    subgraph Incident["Incident Manager"]
        CREATE["Create incident"]
        ROUTE["Route to owner"]
        TRACK["Track resolution"]
        PM["Auto post-mortem"]
    end

    subgraph Notify["Notification Layer"]
        SLACK["Slack"]
        EMAIL["Email"]
        PD["PagerDuty"]
        WEBHOOK["Webhook"]
    end

    Connectors --> NORM --> STORE
    STORE --> STAT & ML & RULES
    STAT & ML & RULES --> CLASS
    CLASS --> SRC & SCHEMA & TRANS
    SRC & SCHEMA & TRANS --> CREATE
    CREATE --> ROUTE --> TRACK --> PM
    ROUTE --> Notify
```

---

## Authentication & Authorization Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant A as Auth Package
    participant F as Feature Flags
    participant S as Service Module

    C->>G: Request + JWT / API Key
    G->>A: Validate token
    A-->>G: Token valid + tenant_id + user_id
    G->>F: Check feature flags for tenant_id
    F-->>G: Allowed modules for this tenant
    G->>G: Does route match allowed modules?
    alt Not allowed
        G-->>C: 403 Forbidden
    else Allowed
        G->>S: Forward request with tenant context
        S-->>G: Response
        G-->>C: Response
    end
```

---

## Event-Driven Communication Between Modules

Modules never call each other directly. All cross-module communication flows through SQS events. This keeps modules loosely coupled and independently deployable.

```mermaid
flowchart LR
    subgraph R["Reliability Module"]
        RE["Incident detected"]
    end

    subgraph SQS["AWS SQS"]
        Q1["reliability-events queue"]
        Q2["contracts-events queue"]
    end

    subgraph C["Contracts Module"]
        CE["Contract violation handler"]
    end

    subgraph AI["AI Modeler Module"]
        AIE["Lineage update handler"]
    end

    RE -->|"publish event"| Q1
    Q1 -->|"subscribe"| CE
    Q1 -->|"subscribe"| AIE
    CE -->|"publish event"| Q2
    Q2 -->|"subscribe"| RE
```

---

## Infrastructure — AWS Architecture

```mermaid
flowchart TB
    subgraph Internet["Internet"]
        USR["Users / SDK / API clients"]
    end

    subgraph AWS["AWS — ap-northeast / ca-central"]
        ALB["Application Load Balancer"]

        subgraph ECS["ECS Fargate Cluster"]
            GW["api-gateway task"]
            RT["reliability task"]
            CT["contracts task"]
            AT["ai-modeler task"]
            WEB["web-dashboard task"]
        end

        subgraph Data["Data Layer"]
            RDS[("PostgreSQL RDS\nMulti-AZ")]
            REDIS[("ElastiCache Redis")]
            S3[("S3 Buckets\nlogs + artifacts")]
        end

        subgraph Async["Async Layer"]
            SQS["SQS Queues"]
        end

        subgraph Observability["Observability"]
            CW["CloudWatch Logs + Metrics"]
            OTEL["OpenTelemetry traces"]
        end
    end

    USR --> ALB --> GW
    GW --> RT & CT & AT & WEB
    RT & CT & AT --> RDS & REDIS & S3
    RT & CT & AT --> SQS
    ECS --> CW & OTEL
```

---

## CI/CD Pipeline

```mermaid
flowchart LR
    DEV["Developer\npushes code"] --> GH["GitHub PR"]
    GH --> CI["GitHub Actions CI"]

    subgraph CI["CI Checks"]
        LINT["Lint + type check"]
        TEST["Unit tests"]
        BUILD["Docker build"]
        SEC["Security scan"]
    end

    CI -->|"PR merged to main"| STAGING["Deploy to Staging\nauto"]
    STAGING --> SMOKE["Smoke tests"]
    SMOKE -->|"manual approval"| PROD["Deploy to Production"]

    PROD --> CW["CloudWatch\nalerts"]
    CW -->|"on error"| ROLLBACK["Auto rollback"]
```

---

## Tech Stack Summary

| Layer | Technology | Rationale |
|---|---|---|
| **Monorepo tooling** | Turborepo | Best-in-class for mixed Python/JS monorepos |
| **API Gateway** | Node.js + Fastify | Lightweight, fast routing, great TypeScript support |
| **Service modules** | Python + FastAPI | Founder's core language, excellent data ecosystem |
| **Anomaly detection** | PySpark + Scikit-learn | Distributed at scale, founder's deep expertise |
| **Frontend** | React + TypeScript + Vite | Industry standard, fast dev loop |
| **Styling** | TailwindCSS | Rapid UI development solo |
| **API data fetching** | TanStack Query | Best-in-class async state management |
| **Database** | PostgreSQL (RDS) | Relational, robust, multi-tenant friendly |
| **Cache / pub-sub** | Redis (ElastiCache) | Fast reads, pub-sub for live updates |
| **Async events** | SQS | Reliable, managed, decouples modules |
| **Object storage** | S3 | Logs, artifacts, pipeline snapshots |
| **Compute** | ECS Fargate | No Kubernetes overhead for a solo founder |
| **IaC** | Terraform | Reproducible, version-controlled infra |
| **CI/CD** | GitHub Actions | Free tier generous, tight GitHub integration |
| **Observability** | CloudWatch + OpenTelemetry | AWS-native + vendor-neutral tracing |

---

## Architectural Principles

| Principle | Implementation |
|---|---|
| **Multi-tenancy from day one** | `tenant_id` on every table, enforced in ORM base class |
| **Loose module coupling** | Modules communicate via SQS events only, never direct imports |
| **Feature flag access control** | Module access gated by `feature_flags` table per tenant |
| **Observability before features** | Logging, metrics, tracing configured in Phase 0, before any feature code |
| **Database isolation** | Each module owns its PostgreSQL schema, no cross-module joins |
| **Shared packages versioned independently** | Turborepo workspaces, each package has own `package.json` / `pyproject.toml` |
| **Deploy each module independently** | Separate ECS task definitions, separate Docker images, separate CI jobs |

---

## Phase 0 — What to Build First

Before any product features, the following foundation must exist:

```mermaid
flowchart TD
    A["1. Monorepo init\nTurborepo + workspace config"] -->
    B["2. Shared packages skeleton\nauth / billing / db / notifications / sdk"]
    B --> C["3. Database setup\nPostgreSQL schema + migrations + ORM base"]
    C --> D["4. API Gateway\nRouting + auth middleware + feature flag check"]
    D --> E["5. Web Dashboard shell\nReact app + auth flow + module routing"]
    E --> F["6. CI/CD\nGitHub Actions — lint, test, build, deploy"]
    F --> G["7. AWS dev environment\nTerraform — ECS, RDS, Redis, S3, SQS"]
    G --> H["✅ Foundation complete\nReady to build Reliability Module features"]
```

---

*Questions? Raise an issue in Linear or review the session history in `CLAUDE.md`.*
