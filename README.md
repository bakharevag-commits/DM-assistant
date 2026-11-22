# 🚀 DM-assistant — Flow & Delivery Analytics Engine

**DM-assistant** is an intelligent flow analytics and delivery management tool designed for Delivery Managers, Scrum Masters, and Team Leads. Simply paste a link to your agile board and provide your session MCP token to get instant visibility into delivery speed, bottlenecks, team workload, and board hygiene—backed by actionable recommendations from **Kanban, Scrum, Lean, and DORA** practices.

---

## ✨ Key Features

* **Dual-Mode Board Detection (Scrum & Kanban):** Automatically detects active sprints (`Scrum Mode` with carry-over tracking) or switches to continuous flow (`Kanban Mode` with 14/30-day rolling windows).
* **Epic Time-to-Market (TTM) Breakdown:** Tracks end-to-end delivery across `Discovery`, `Waiting Delivery`, and `Delivery` stages against a configurable **60-day SLA**.
* **Task & Bug Delivery Time / Age:** Measures active delivery speed against a **30-day SLA**, strictly separating completed items (included in averages) from active items (tracked as `Delivery Age`).
* **Data Hygiene Quarantine:** Isolates cards with missing start dates (`start_date = null`), unmapped workflow stages, or active-sprint items left in `Backlog` so they never distort team SLA metrics.
* **WIP & Stagnation Alerts:** Highlights personal overload (`1 person = 1 active task` rule) and items stagnating in a single status for more than **2 days**.
* **Actionable Insights Engine:** Provides context-aware coaching playbooks when bottlenecks, high carry-over (`> 25%`), or WIP overloads are detected.
* **Zero-Persistence Token Security:** Your MCP server token lives strictly in memory during your active browser session and is never written to any database.

---

## 🏗 System Architecture

```mermaid
flowchart TB
    classDef external fill:#1e293b,stroke:#475569,stroke-width:2px,color:#f8fafc
    classDef security fill:#31102f,stroke:#ec4899,stroke-width:2px,color:#fce7f3
    classDef ingestion fill:#1e1b4b,stroke:#6366f1,stroke-width:2px,color:#e0e7ff
    classDef hygiene fill:#451a03,stroke:#f59e0b,stroke-width:2px,color:#fef3c7
    classDef engine fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#d1fae5
    classDef storage fill:#172554,stroke:#3b82f6,stroke-width:2px,color:#dbeafe
    classDef frontend fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f0f9ff

    subgraph EXT ["1. External Issue Tracker Layer"]
        direction LR
        TRACKER[(Issue Tracker API<br/>Epics, Tasks, Bugs,<br/>Statuses, Metadata)]:::external
        GATEWAY[Integration Gateway /<br/>MCP Adapter]:::external
        TRACKER <--> GATEWAY
    end

    subgraph AUTH ["2. Ephemeral Session Security (Zero-Persistence)"]
        direction LR
        SESSION[Session Token Vault<br/>In-Memory / sessionStorage only<br/>Auto-wiped on session end]:::security
    end

    subgraph ING ["3. Data Ingestion & Normalization"]
        direction TB
        PARSER[Board URL Parser<br/>Extract Board & Project ID]:::ingestion
        MODE{Board Mode Detection<br/>has_active_sprint?}:::ingestion
        SCRUM[Scrum Mode<br/>Sprint boundaries & Carry-over]:::ingestion
        KANBAN[Kanban Mode<br/>Continuous flow, 14/30d windows]:::ingestion
        MAPPER[Status-to-Stage Mapper<br/>Backlog / Discovery / Waiting / Delivery]:::ingestion

        PARSER --> MODE
        MODE -->|Yes| SCRUM
        MODE -->|No| KANBAN
        SCRUM --> MAPPER
        KANBAN --> MAPPER
    end

    subgraph HYG ["4. Data Hygiene Filter"]
        direction TB
        FILTER{Data Quality Gate<br/>start_date & Stage validation}:::hygiene
        QUARANTINE[Hygiene Quarantine<br/>• Missing start_date<br/>• Active sprint item in Backlog<br/>• Unmapped workflow status]:::hygiene
        CLEAN[Validated Flow Stream<br/>Eligible for SLA metrics]:::engine

        FILTER -->|Hygiene violation| QUARANTINE
        FILTER -->|Valid data| CLEAN
    end

    subgraph CORE ["5. Flow Analytics Engine"]
        direction TB
        TTM[Epic Time-to-Market SLA 60d<br/>Discovery + Waiting + Delivery]:::engine
        DELIVERY[Task Delivery Time & Age SLA 30d<br/>Completed vs Active items]:::engine
        BOTTLENECK[Bottleneck Detector<br/>Stage median time, Carry-over > 25%]:::engine
        FLOW[WIP & Stagnation Monitor<br/>Personal WIP > 1, Stagnation > 2d]:::engine
        ADVISOR[Actionable Insights Engine<br/>Kanban, Scrum, Lean & DORA rules]:::engine

        CLEAN --> TTM & DELIVERY & BOTTLENECK & FLOW
        TTM & DELIVERY & BOTTLENECK & FLOW --> ADVISOR
    end

    subgraph DB ["6. Persistence & Knowledge Base (No Tokens Stored)"]
        direction LR
        PG[(PostgreSQL<br/>• Anonymized board mappings<br/>• Aggregated trend snapshots<br/>• Insight feedback 👍/👎)]:::storage
        CONFIG[(Insights Config JSON/YAML<br/>Extensible rules &<br/>best-practice playbooks)]:::storage

        CONFIG <--> ADVISOR
        PG <--> MAPPER
        CORE --> PG
        QUARANTINE --> PG
    end

    subgraph UI ["7. Presentation Layer (Frontend UI)"]
        direction TB
        LOGIN[Onboarding Prompt<br/>Board URL + Session MCP Token]:::frontend
        DASH[Flow Dashboard<br/>KPI Cards & Trend Sparklines]:::frontend
        BADGES[Interactive Filter Badges<br/>WIP Overload / Stagnation / Hygiene]:::frontend
        TOOLTIPS[Transparent Formula Tooltips<br/>Timestamps, SLA breakdown & Tips]:::frontend

        LOGIN --> DASH
        DASH --- BADGES --- TOOLTIPS
    end

    LOGIN -->|Transient Token| SESSION
    LOGIN -->|Board URL| PARSER
    SESSION -.->|Bearer Auth per request| GATEWAY
    GATEWAY --> PARSER
    MAPPER --> FILTER
    QUARANTINE --> BADGES
    ADVISOR --> DASH
    PG --> DASH
```

---

## 📊 Metrics & Methodology

| Metric | Target Entity | Start Trigger | End Trigger | Default SLA |
| :--- | :--- | :--- | :--- | :--- |
| **Time to Market (TTM)** | Epics | First transition into `Discovery` (or `Delivery` if skipped) | Final transition to `Done` | **≤ 60 days** |
| **Delivery Time** | Tasks & Bugs | `start_date` (or first entry into active sprint / `Delivery`) | Final transition to `Done` | **≤ 30 days** |
| **Delivery Age** | Open Tasks / Bugs | `start_date` | Current timestamp (excluded from SLA averages) | Warning at **> 24 days** |
| **Personal WIP** | Team Members | Active items in `Delivery` stage per assignee | Transition out of `Delivery` | **1 item / person** |
| **Status Stagnation** | Active Items | Last status change timestamp | Next status transition | **≤ 2 days** |

---

## 🔒 Security & Privacy

* **Zero-Persistence Authentication:** User MCP tokens are stored exclusively in ephemeral session memory (`In-Memory / sessionStorage`) and passed per request via Bearer authorization. Tokens are never logged or persisted in PostgreSQL.
* **Anonymized Storage:** The database stores only structural status-to-stage mappings, aggregated historical trend snapshots, and anonymous usefulness ratings (`👍 / 👎`) for coaching insights.

---

## UI Preview & Demo

To protect corporate data and internal workflows, UI screenshots are available upon request during the interview process.

The interface includes:

* Real-time flow metrics dashboard (TTM, Lead Time, WIP).
* Automated bottleneck and stagnation alerts.
* Personal overload tracking (1 person = 1 active task rule).
