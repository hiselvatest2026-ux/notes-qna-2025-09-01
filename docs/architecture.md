# Refund Leakage Hunter + SLA Enforcement (Multi‑Agent RAG)

## 1) Goal
- Recover missed refunds/credits (flights + hotels + ancillaries)
- Enforce supplier SLAs automatically with evidence-backed claims
- Improve CSAT via proactive notifications

## 2) Scope
- Back-office automation (no customer UI change)
- Flights: schedule change refunds, unused ancillaries, downgrades, duplicate charges
- Hotels: downgrade/overbook/amenity failure/late check-in
- Claims submission + reconciliation

## 3) Core Components
- Orchestrator (crewAI): routes tasks, manages state, retries
- Ingestion/ETL: PNRs, hotel reservations, ancillaries, flight status, PMS signals, chats/emails, payments/refunds
- Vector Store (RAG): contracts, SLAs, fare/waiver rules, policy memos, internal SOPs
- Detector Agents:
  - Flight Detector, Hotel Detector, Payment/Refund Detector
- Evidence Builder: compiles proof pack (logs, screenshots, clauses, timestamps)
- Claims Agent: submits to supplier portals/APIs/email; tracks SLAs
- Reconciliation Agent: matches credits/cash to claims; closes ledger loop
- Compliance/Policy Agent: approve/block/arbitrate; PII governance
- Notification/Reporting: ops dashboards, finance exports, client statements

## 4) High-Level Architecture
```mermaid
flowchart LR
  subgraph Sources
    A[Bookings (PNR/Hotel)] -->|ETL| I
    B[Ancillaries] -->|ETL| I
    C[Flight Status/Schedule] -->|ETL| I
    D[PMS/Hotel Events] -->|ETL| I
    E[Chats/Emails/CRM] -->|ETL| I
    F[Payments/Refunds] -->|ETL| I
    G[Supplier Contracts/SLAs/Rules] -->|Index| VS
  end

  I[Ingestion & Normalization] --> DS[(Ops Data Lake)]
  VS[(Vector Store/RAG Index)]

  subgraph Agents (crewAI)
    ORCH[Orchestrator]
    DETF[Detector - Flights]
    DETH[Detector - Hotels]
    DETP[Detector - Payments]
    POL[Compliance/Policy]
    EV[Evidence Builder]
    CL[Claims Agent]
    RC[Reconciliation]
  end

  DS --> ORCH
  VS <---> ORCH

  ORCH --> DETF
  ORCH --> DETH
  ORCH --> DETP

  DETF --> EV
  DETH --> EV
  DETP --> EV
  EV --> POL
  POL -->|approve| CL
  CL --> SUP[Supplier Portals/APIs/Email]
  SUP --> RC
  RC --> FIN[Finance/ERP]
  RC --> REP[Ops Dashboards/Exports]
## 5a) Data Flow (Hotels example)
```mermaid
sequenceDiagram
  participant ETL as ETL/Normalizer
  participant DS as Ops Data Lake
  participant ORCH as Orchestrator
  participant DETH as Hotel Detector
  participant RAG as RAG Index
  participant EV as Evidence Builder
  participant POL as Policy Agent
  participant CL as Claims Agent
  participant SUP as Hotel Portal/Email
  participant RC as Reconciliation

  ETL->>DS: Load reservations + PMS/OTA events + comms logs
  ORCH->>DS: Fetch candidates (stays with issues)
  ORCH->>DETH: Evaluate SLA breach (downgrade/overbook/amenity/late room)
  DETH->>RAG: Retrieve contract clause/compensation rules
  DETH->>ORCH: Entitlement suggestion (amount, nights affected)
  ORCH->>EV: Build proof (PMS snapshots, chat/email, photos, timestamps, clauses)
  EV->>POL: Policy/legal check (thresholds, confidence, PII)
  POL-->>ORCH: Approved (amount, clause cited)
  ORCH->>CL: Submit claim (portal/API/email template)
  CL->>SUP: Send packet + track SLA
  SUP-->>RC: Credit/Cash posted
  RC-->>ORCH: Reconcile → ledger → close claim
4a) Hotels SLA Detection (logic view)
flowchart TD
  A[Stay & PMS Events] --> B[Normalize & Correlate]
  B --> C{Issue Detected?}
  C -- No --> Z[Exit]
  C -- Yes --> D[Hotel Detector]
  D --> E[RAG: Contract/Rate Rules]
  E --> F[Compute Entitlement]
  F --> G[Evidence Builder]
  G --> H[Policy/Legal Review]
  H -- Reject --> Z
  H -- Approve --> I[Claims Agent]
  I --> J[Hotel Portal/API/Email]
  J --> K[Reconciliation]
  K --> L[Close Claim & Report]

6) Agent Responsibilities
Orchestrator: task routing, idempotency, backoff/retries, audit trail
Detectors: domain-specific heuristics + LLM reasoning w/ citations
Evidence Builder: assemble artifacts, normalize timestamps, compute entitlement
Policy/Compliance: legal checks, thresholds, human-in-loop rules
Claims: templated submission, SLA tracking, escalations
Reconciliation: match credits to claims, post to ledger, exceptions queue
7) Tech Choices (example)
ETL: Airflow/Temporal + dbt; CDC from OLTP
Storage: OLTP (Postgres/MySQL), Data Lake (S3/Delta), Cache (Redis)
Vector DB: pgvector/Weaviate/Pinecone; embeddings for contracts/policies
Agents: crewAI (Python), LangChain (optional tooling), Playwright (portal automation)
Services: FastAPI/NestJS; Celery/Temporal workers
Observability: OpenTelemetry, Grafana, Sentry
Security: JWT/OIDC, KMS, per-tenant encryption, row-level ACLs
8) RAG Corpus
Supplier contracts & SLAs (PDF/Docx → chunk + index)
Fare rules/waiver memos
Internal SOPs/playbooks
Historic claims & win/loss rationale
Legal/policy templates
9) Controls & Governance
PII minimization; store only necessary fields
Redaction in vector store
Human review thresholds (e.g., claims > ₹X or low confidence)
Full audit log (inputs, citations, outputs, actions)
10) KPIs
Recovery ₹ / eligible ₹, hit rate (% won), time-to-refund
Margin bps lift, DSO improvement, CSAT delta, contacts per booking
Supplier-wise performance; false-positive rate
11) Rollout Plan (8–12 weeks)
Phase 1 (Weeks 1–4): Flights – schedule changes & unused ancillaries → manual send
Phase 2 (Weeks 5–8): Hotels – downgrade/overbook/amenity failures → email/portal automations
Phase 3 (Weeks 9–12): Reconciliation auto‑match, SLA escalations, reporting
12) Risks & Mitigations
Portal changes → e2e tests + resilient selectors
Supplier throttling → rate limits + backoff
False positives → conservative rules + human-in-loop
Contract variance → per‑supplier rule packs in RAG EOF
