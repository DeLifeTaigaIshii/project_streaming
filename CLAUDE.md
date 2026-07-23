# project_streaming

## Purpose

This project exists to grow the user's skills as a **data engineer**, and more broadly across IT/software
engineering. It is a learning project, not a product.

**Guiding principle: architecture and technology stack take priority over the end deliverable.**
When a decision must be made between "what's simplest to satisfy the use case" and "what teaches the most
about the required architecture/stack," prefer the latter. The demo e-commerce site described below exists
only to produce realistic data for the pipeline — it is not the point of the project.

Two areas are the primary focus and should get the most design discussion and depth:

1. **Streaming data processing with Kafka**
2. **A modern, AI-augmented ELT pipeline**

### Required technology (non-negotiable)

- **AWS** — cloud platform for compute, streaming, and storage
- **Snowflake** — data warehouse
- **Fivetran** — managed batch ELT (Extract/Load)

## Use Case: Demo E-Commerce Site

A demo e-commerce application is built as part of this project's scope, solely to generate realistic
transactional and behavioral data.

- **Backend (API server)** — the primary component; produces:
  - Clickstream / cart / order **events** → streamed to Kafka
  - Customer / product **master data** → written to RDS PostgreSQL
- **Frontend (UI)** — a simple browsing/checkout UI. Low priority; Claude may implement this with
  autonomy (see Collaboration Style below).
- **Hosting**: Amazon ECS on Fargate

## Architecture Overview

```
                 ┌────────────────────────┐
                 │   Demo EC App (ECS)     │
                 │  backend API + frontend │
                 └───────────┬─────────────┘
                    events    │    master data
                 (click/cart/ │  (customers/products)
                    order)    ▼
                    ┌─────────┐        ┌──────────────────┐
                    │  Kafka  │        │  RDS PostgreSQL   │
                    │ (MSK    │        └─────────┬──────────┘
                    │Serverless)                 │ Fivetran (CDC)
                    └────┬────┘                  │
             Snowflake Connector                 ▼
              for Kafka (Kafka Connect)   Snowflake RAW schema
                    │                              │
                    └──────────────┬───────────────┘
                                   ▼
                         Snowflake RAW → STG → MART
                              (dbt models)
                     AI-enriched via Snowflake Cortex
                    (SENTIMENT / SUMMARIZE / COMPLETE /
                           EMBED_TEXT, etc.)
```

## Component Decisions

| Component | Choice | Rationale |
|---|---|---|
| Kafka runtime | **Amazon MSK Serverless** | Satisfies the AWS requirement with managed operations (IAM auth, VPC integration) while avoiding idle broker cost; better fit than self-hosting given the free-tier/trial cost policy. |
| Kafka → Snowflake ingestion | **Snowflake Connector for Kafka** (via Kafka Connect) | Simpler, more direct path than Kafka → S3 → Snowpipe; teaches Kafka Connect + Snowpipe Streaming. |
| Batch ELT source DB | **Amazon RDS for PostgreSQL** (not Aurora) | RDS is AWS Free Tier eligible (db.t3/t4g.micro, 750 hrs/month); Aurora has no free tier and costs more even at minimum size. Chosen to fit the cost policy below. |
| Batch ELT tool | **Fivetran** (Postgres connector, CDC/logical replication) | Required tool; ingests master/dimension data (customers, products) from RDS into Snowflake RAW. |
| Transformation | **dbt** | Industry-standard RAW → STG → MART modeling; also where Snowflake Cortex calls are integrated. |
| AI / LLM integration | **Snowflake Cortex** (`SENTIMENT`, `SUMMARIZE`, `COMPLETE`, `EMBED_TEXT`, etc.) | Runs natively in Snowflake SQL/Snowpark; no separate AI infra to manage. |
| IaC | **Terraform** | Provisions AWS resources (MSK, RDS, ECS/Fargate, IAM, VPC) and, where practical, Snowflake objects. |
| Orchestration | **Not yet decided** | Deferred until the core pipeline (Kafka, Fivetran, dbt, Cortex) is working end-to-end. Candidates to revisit: Airflow, Dagster, MWAA. |
| App hosting | **ECS on Fargate** | Broadly applicable AWS container skills (ECR/ECS/ALB/Fargate) without server/OS management overhead. |

## Repository Structure

```
project_streaming/
├── CLAUDE.md
├── README.md
├── infra/          # Terraform: AWS + Snowflake resources
├── app/
│   ├── backend/     # EC site API — emits Kafka events, writes RDS master data
│   └── frontend/    # Simple UI (low priority, Claude-driven)
├── streaming/       # Kafka topic schemas, Kafka Connect / Snowflake Connector config
├── dbt/             # dbt project: RAW → STG → MART, Cortex-based models
└── docs/            # architecture notes, decisions
```

This structure may evolve as the project develops; treat it as a starting point, not a constraint.

## Cost Policy

All required services (AWS, Snowflake, Fivetran) are paid. **Policy: stay within free tiers and trial
credits wherever possible.**

- Prefer resources with a free tier (RDS over Aurora; MSK Serverless's low-throughput pricing over
  Provisioned's always-on brokers).
- Be mindful of Fivetran's free-plan row/connector limits and Snowflake trial credit consumption.
- Prefer tearing down non-essential AWS resources (`terraform destroy` / stopping ECS services) between
  active work sessions rather than leaving things running continuously, unless the user says otherwise.
- Flag to the user when a design choice has a real cost implication, rather than silently picking the
  more expensive option.

## Collaboration Style

This project has three areas of Claude's behavior, each with distinct rules: **Architecture**,
**Coding**, and **Commands**.

### Architecture — Claude is the user's consulting partner

- Applies to overall system/component design: Kafka, Fivetran, Snowflake, dbt, Terraform, Cortex, and
  how they fit together.
- When proposing a design, weigh three axes: **learning value, cost, and effort**, and propose what
  seems optimal considering all three.
- Discuss and reach agreement with the user before treating a decision as settled.
- Once agreed, summarize the outcome in **Japanese prose plus ASCII art** (diagrams of the
  architecture/data flow) **in the chat response**. This is a conversational summary, not a documentation
  deliverable — see Documentation Language below for what happens when a decision is written into project
  docs.
- Don't silently make architectural choices; surface them for discussion first.

### Coding — the user is the primary author

- Except for the frontend UI, **the user writes the code**. Claude's main role here is answering
  questions.
- This applies to backend/application code as well as **Terraform and dbt files** — infrastructure and
  transformation code is treated as "coding," not "architecture," and is user-driven.
- Claude does **not** write source code directly into project files for this code (no Write/Edit tool
  use). Sample code and code review/corrections are given as **text in the chat response only** — never
  applied directly to files.
- The user may ask Claude for sample code or to review/correct code they wrote; Claude should do so, but
  always as chat text, leaving the user to apply it.
- When producing sample code or reviewing code, consult official documentation (Kafka, Snowflake, dbt,
  Terraform, AWS, Fivetran, etc.) rather than relying solely on memory, to avoid giving inaccurate
  information.
- **Proactively flag cost or security issues**, even without an explicit review request — e.g. a public
  S3 bucket, an unexpectedly expensive instance/resource type, hardcoded credentials. The restriction on
  Claude is specifically about not autonomously writing code or executing commands; it is not a mandate
  to stay silent about risks noticed along the way.

### Frontend UI — the one exception to "user codes"

- Claude may implement this directly and autonomously, using its own judgment, with minimal explanation.
- Prioritize speed over teaching value here — it exists only to support the data platform's use case.

### Commands — the user runs them; Claude may sandbox-test first

- In principle, **the user executes commands**, not Claude.
- When the user asks Claude to "provide the correct command," Claude gives a command that's ready to run
  as-is.
- If Claude is unsure the command is correct, or an error seems likely, Claude should first test it in
  a **local-only sandbox** before handing it to the user, to minimize confusion. This pre-check is the
  one form of unprompted command execution that's allowed.
- This sandboxing is strictly local: it must never touch real AWS/Snowflake/Fivetran resources — no
  `apply`, no real API calls, and no read-only calls against live cloud resources either. Local checks
  only (e.g. `terraform validate`, syntax/dry-run checks against local state).
- **Diagnostic/investigative command execution is different and is not allowed unprompted.** When the
  user reports an error or asks "why is X failing," Claude must not run commands on its own to
  investigate — even non-destructive, local ones. Instead, Claude should tell the user which command to
  run and ask them to share the output, unless the user explicitly tells Claude to go ahead and execute
  something itself.

## Documentation Language

English, for both this file and other project documentation. Code identifiers and comments also follow
standard English convention.

## Open / Deferred Decisions

- Orchestration tool (Airflow / Dagster / MWAA / none) — revisit after the core pipeline works.
- Exact dbt model structure and Cortex use cases — to be designed together once RAW data is flowing.
