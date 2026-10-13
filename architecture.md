# CivicTrace — System Architecture & AI Workflow

> **AI Synergy Hackathon 2026 · Round 1 · Team IRIS**

This document describes the complete technical architecture and AI workflow of CivicTrace: a unified, traceable municipal grievance lifecycle platform.

---

## Table of Contents

1. [Architectural Goals](#architectural-goals)
2. [High-Level System Overview](#high-level-system-overview)
3. [Component Architecture](#component-architecture)
   - [Intake Layer](#1-intake-layer)
   - [Standardisation & Case Engine](#2-standardisation--case-engine)
   - [AI Pipeline](#3-ai-pipeline)
   - [Workflow & Escalation Engine](#4-workflow--escalation-engine)
   - [Data Layer](#5-data-layer)
   - [Notification Service](#6-notification-service)
   - [Presentation Layer](#7-presentation-layer)
4. [AI Workflow — Detailed](#ai-workflow--detailed)
   - [Duplicate & Related-Complaint Detection](#duplicate--related-complaint-detection)
   - [Multilingual NLP Processing](#multilingual-nlp-processing)
   - [Department Routing Suggestion](#department-routing-suggestion)
5. [Data Models](#data-models)
6. [Escalation Rules](#escalation-rules)
7. [Audit & Auditability](#audit--auditability)
8. [Security & Roles](#security--roles)
9. [Deployment Topology](#deployment-topology)
10. [Key Design Decisions](#key-design-decisions)

---

## Architectural Goals

| Goal | Constraint |
|---|---|
| Single source of truth for every complaint | All channels write to one lifecycle record |
| AI assists; humans decide | No automated system closes or rejects complaints |
| Every action is auditable | Immutable audit log; no silent state changes |
| Transparent citizen timeline | Only real events generate timeline entries |
| Graceful degradation | System remains functional if AI components are unavailable |

---

## High-Level System Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          INTAKE CHANNELS                                  │
│   Web Portal   │   WhatsApp / SMS   │   Phone (IVR)   │   In-Person     │
└────────┬───────┴──────────┬─────────┴────────┬────────┴───────┬─────────┘
         │                  │                   │                │
         └──────────────────┴───────────────────┴────────────────┘
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │    INTAKE ADAPTER      │
                         │  (normalise payload)   │
                         └───────────┬────────────┘
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │    CASE ENGINE         │◄──── Audit Log
                         │  Generate Case ID      │
                         │  Persist canonical     │
                         │  record                │
                         └───────────┬────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                 │
                    ▼                ▼                 ▼
           ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
           │ AI PIPELINE   │ │  WORKFLOW    │ │  NOTIFICATION    │
           │               │ │  ENGINE      │ │  SERVICE         │
           │ • NLP         │ │              │ │                  │
           │ • Duplicate   │ │ • SLA calc.  │ │ • SMS            │
           │   detection   │ │ • Routing    │ │ • Email          │
           │ • Dept.       │ │ • Escalation │ │ • WhatsApp out   │
           │   routing     │ │ • Deadlines  │ │                  │
           └──────┬────────┘ └──────┬───────┘ └──────────────────┘
                  │                 │
                  └────────┬────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │      DATA LAYER        │
              │  PostgreSQL            │
              │  • complaints          │
              │  • case_events         │
              │  • duplicate_links     │
              │  • audit_log           │
              └────────────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
       ┌─────────┐  ┌──────────┐  ┌──────────┐
       │ Citizen │  │ Officer  │  │Supervisor│
       │ Portal  │  │Dashboard │  │Dashboard │
       └─────────┘  └──────────┘  └──────────┘
```

---

## Component Architecture

### 1. Intake Layer

**Purpose:** Receive complaint data from any channel and normalise it into a standard internal payload.

```
Channel          Adapter                 Normalised Payload
─────────────────────────────────────────────────────────────
Web portal    →  REST endpoint        →  { source, raw_text,
WhatsApp      →  Webhook handler            location, category,
SMS           →  Twilio handler             media_urls,
Phone (IVR)   →  Voice-to-text + form       reporter_contact,
In-person     →  Staff data-entry form      timestamp }
```

**Adapter responsibilities:**
- Parse and sanitise raw input
- Detect input language (for NLP routing)
- Extract or request: location (GPS co-ordinates or address), category hint, supporting media
- Forward normalised payload to the Case Engine via internal API call
- Return a preliminary acknowledgement to the citizen *immediately*, before any AI processing

> **Design rule:** Acknowledgement is never blocked on AI processing. The citizen gets a response at intake; the Case ID is confirmed in a follow-up notification once the pipeline completes.

---

### 2. Standardisation & Case Engine

**Purpose:** The authoritative owner of the complaint record.

**Responsibilities:**

| Action | Detail |
|---|---|
| Generate Case ID | Format: `GRV-{YYYYMMDD}-{6-digit-seq}` — e.g. `GRV-20261013-004821` |
| Persist canonical record | Written to `complaints` table before AI pipeline starts |
| Invoke AI pipeline | Asynchronously; case is usable before AI returns results |
| Apply routing result | Sets `assigned_department` and `responsible_officer` after human confirmation |
| Apply SLA | Inserts deadline into `case_events`; workflow engine picks up |
| Record every state change | Every mutation appends an immutable row to `audit_log` |

**Case states:**

```
REGISTERED → DUPLICATE_REVIEW → ASSIGNED → IN_PROGRESS → PENDING_VERIFICATION
                                                        → OVERDUE (parallel flag)
                                                        → ESCALATED (parallel flag)
                              → RESOLVED
                              → CONTESTED → RE_OPENED → …
                              → CLOSED
```

---

### 3. AI Pipeline

Runs asynchronously after the canonical record is persisted. Three sub-components run in parallel where possible.

#### 3a. NLP Pre-processor

```
Input:  raw_text (any language)
Output: normalised_text (English), detected_language, confidence, entities

Steps:
  1. Language detection  →  Hindi / English / Hinglish / other
  2. Translation         →  English working copy (original preserved)
  3. Named-entity extraction:
       - Location mentions
       - Issue keywords  (pothole, water, electricity, garbage…)
       - Urgency signals (dangerous, emergency, smell, exposed wire…)
  4. Confidence score attached to every field
  5. Low-confidence fields → flagged for human review in officer dashboard
```

**Model:** Multilingual transformer (e.g. Google Gemini or mBERT fine-tuned on municipal domain). Original language text is always preserved; translated text is a working copy only.

---

#### 3b. Duplicate & Related-Complaint Detector

This is the most critical AI component. Its output influences whether a new case is created or linked.

```
Input:  normalised_text, location (lat/lng), category, timestamp
Output: [ { case_id, similarity_score, match_type, confidence } ]

Algorithm:
  Step 1 — Spatial filter
    Retrieve open cases within configurable radius
    (default: 100 m for street-level issues; 500 m for area-level issues)

  Step 2 — Temporal filter
    Narrow to cases opened within configurable window
    (default: 7 days; shorter for time-sensitive categories)

  Step 3 — Category match
    Exact or fuzzy category comparison
    Different categories → similarity floor reduced

  Step 4 — Semantic similarity
    Encode complaint text with Sentence-BERT (multilingual)
    Compute cosine similarity against candidate set

  Step 5 — Combined score
    score = α·spatial + β·temporal + γ·category + δ·semantic
    (weights tuned on labelled municipal data; configurable)

  Step 6 — Threshold routing
    score ≥ HIGH_THRESHOLD  →  suggest auto-link  (human can override)
    MEDIUM ≤ score < HIGH   →  queue for human review
    score < MEDIUM          →  treat as independent case
```

**Critical invariant:** Linking a report to a canonical case never deletes the original report. The `duplicate_links` table preserves every linked report, its reporter, its timestamp, and its media. All linked reporters receive status updates for the canonical case.

**Human review queue:** Any case flagged as a possible duplicate at medium confidence is held in a review dashboard. A supervisor sees both cases side by side and makes the merge/keep decision. Their decision is logged.

---

#### 3c. Department Routing Suggester

```
Input:  category, entities, location (ward / zone), urgency_signals
Output: { suggested_department, confidence, reasoning_note }

Algorithm:
  Step 1 — Rule layer (exhaustive for known categories)
    Pothole / road damage      → Public Works
    Water pipe / drainage      → Water & Sanitation
    Street light / electricity → Electricity Board
    Garbage / waste            → Sanitation
    Tree / park                → Horticulture
    …

  Step 2 — ML classifier (for ambiguous or multi-category complaints)
    Trained on historical routing decisions
    Outputs ranked department list with confidence scores

  Step 3 — Threshold check
    confidence ≥ HIGH  →  pre-fill routing in officer dashboard (human confirms)
    confidence < HIGH  →  show top 3 suggestions; officer selects

  Step 4 — Urgency override
    High-urgency signals (exposed wire, gas leak, structural collapse)
    → Force human supervisor review before routing, regardless of confidence
```

**Design rule:** The routing suggester *never routes autonomously*. It populates a pre-filled field that a human officer confirms or overrides. The human's final selection is what enters the audit log.

---

### 4. Workflow & Escalation Engine

Pure rules-based; no AI. This engine is the guarantor of accountability.

#### SLA Deadline Assignment

```
Category         Severity      SLA deadline
─────────────────────────────────────────────────────
Urgent safety    Critical      4 hours
Water / drainage High          24 hours
Road / pothole   Medium        72 hours
Street light     Medium        72 hours
Garbage          Low           48 hours
Park / tree      Low           7 days
Default          —             48 hours
```

SLA values are stored in a configuration table — not hard-coded — so municipalities can adjust them without a code deployment.

#### Escalation Rules

```
T = SLA deadline

T - 4h   →  Reminder notification to assigned officer
T + 0    →  Case flagged OVERDUE; supervisor notified
T + 24h  →  Case flagged ESCALATED (Level 1); department head notified
T + 72h  →  Case flagged ESCALATED (Level 2); municipal commissioner notified
```

Each escalation is a logged event. Escalation does **not** reassign the case or change its history — it adds a supervisory notification layer on top. The case history is preserved intact.

#### Reassignment Rule

When a case is transferred between departments:
- Original assignment record is **not deleted** — it is closed with a timestamp and reason
- New assignment is appended as a new event
- Case ID and all prior history remain attached
- Citizen is notified of the transfer

---

### 5. Data Layer

**Database:** PostgreSQL

#### Core Tables

```sql
-- The canonical complaint record
complaints (
  case_id          TEXT PRIMARY KEY,   -- GRV-YYYYMMDD-NNNNNN
  channel          TEXT,               -- web | whatsapp | sms | phone | inperson
  reporter_id      UUID,
  reporter_contact TEXT,
  raw_text         TEXT,
  normalised_text  TEXT,               -- English working copy
  original_lang    TEXT,
  location_lat     NUMERIC,
  location_lng     NUMERIC,
  location_text    TEXT,
  category         TEXT,
  severity         TEXT,
  status           TEXT,               -- state machine value
  assigned_dept    TEXT,
  assigned_officer UUID,
  sla_deadline     TIMESTAMPTZ,
  is_overdue       BOOLEAN DEFAULT FALSE,
  is_escalated     BOOLEAN DEFAULT FALSE,
  created_at       TIMESTAMPTZ DEFAULT now(),
  resolved_at      TIMESTAMPTZ,
  closed_at        TIMESTAMPTZ
)

-- Immutable event log — append-only
case_events (
  event_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id          TEXT REFERENCES complaints(case_id),
  event_type       TEXT,    -- REGISTERED | ASSIGNED | ACTION_TAKEN |
                            -- ESCALATED | OVERDUE | RESOLVED | CONTESTED |
                            -- REASSIGNED | CLOSED | DUPLICATE_LINKED | …
  actor_id         UUID,    -- officer / system / citizen
  actor_role       TEXT,    -- citizen | officer | supervisor | system
  description      TEXT,
  evidence_urls    TEXT[],
  ai_involved      BOOLEAN DEFAULT FALSE,
  ai_confidence    NUMERIC,
  created_at       TIMESTAMPTZ DEFAULT now()
)

-- Links related reports to a canonical case
duplicate_links (
  link_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  canonical_case_id    TEXT REFERENCES complaints(case_id),
  linked_case_id       TEXT REFERENCES complaints(case_id),
  similarity_score     NUMERIC,
  match_type           TEXT,   -- auto_high | human_reviewed
  reviewed_by          UUID,
  reviewed_at          TIMESTAMPTZ,
  created_at           TIMESTAMPTZ DEFAULT now()
)

-- Immutable audit trail (separate from events for compliance)
audit_log (
  log_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id          TEXT,
  action           TEXT,
  before_state     JSONB,
  after_state      JSONB,
  performed_by     UUID,
  performed_at     TIMESTAMPTZ DEFAULT now(),
  ip_address       TEXT,
  session_id       TEXT
)
```

---

### 6. Notification Service

Outbound notifications are triggered by case events — never by timers alone.

| Trigger event | Recipient | Channel |
|---|---|---|
| Case registered | Citizen | SMS + email + WhatsApp |
| Case assigned | Citizen | SMS |
| Case overdue (approaching) | Officer | In-app + email |
| Case overdue (breached) | Supervisor | Email + in-app |
| Case escalated L1 | Department head | Email |
| Case escalated L2 | Commissioner | Email + SMS |
| Case resolved | Citizen | SMS + email; includes contest link |
| Case contested | Supervisor | In-app alert |
| Case re-opened | Officer + Citizen | In-app + SMS |

**Citizen notification content rules:**
- Always include Case ID
- Always include current status in plain language
- Always include next expected action or date (when known)
- Never include internal department names or jargon
- Include a link to the citizen status timeline for every notification

---

### 7. Presentation Layer

#### Citizen Portal
- Complaint submission form (multi-step, location picker, photo upload)
- Case status timeline (event-driven, no manufactured progress)
- Notification history
- Contest resolution form

#### Officer Dashboard
- Assigned case queue with SLA countdown
- AI routing suggestion (pre-filled, editable)
- Action log entry form
- Duplicate review queue (side-by-side comparison)
- Evidence upload for resolution

#### Supervisor Dashboard
- Department-wide case overview
- Overdue and escalated case alerts
- Duplicate merge/keep decisions
- Reassignment controls
- Audit log viewer

---

## AI Workflow — Detailed

### Duplicate & Related-Complaint Detection

```
┌─────────────────────────────────────────────────────────────────┐
│  New complaint arrives                                           │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Spatial + temporal  │
                    │  candidate retrieval │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Category filter     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Sentence-BERT       │
                    │  embedding + cosine  │
                    │  similarity          │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Combined score      │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                 │
        score ≥ HIGH     MEDIUM ≤ score      score < LOW
              │            < HIGH                │
              ▼                │                 ▼
    ┌──────────────────┐       │      ┌──────────────────────┐
    │ Suggest auto-link│       │      │  New independent case│
    │ Officer can      │       ▼      └──────────────────────┘
    │ override         │  ┌──────────────────┐
    └──────────────────┘  │ Human review     │
                          │ queue (supervisor│
                          │ side-by-side)    │
                          └──────────────────┘
```

### Multilingual NLP Processing

```
Raw text (any language)
        │
        ▼
Language detection
        │
        ├── Hindi / Hinglish → translate to English (preserve original)
        ├── English          → pass through
        └── Other            → translate to English; flag for review
        │
        ▼
Named-entity extraction
  • Location mentions  →  reconcile with location field
  • Issue keywords     →  assist category assignment
  • Urgency signals    →  influence SLA tier and routing
        │
        ▼
Confidence scoring per field
        │
        ├── High confidence  →  populate field; officer can edit
        └── Low confidence   →  flag field; officer must review
```

### Department Routing Suggestion

```
Category + entities + location + urgency
        │
        ▼
Rule layer (deterministic, exhaustive for known categories)
        │
        ├── Match found, single department  →  pre-fill with HIGH confidence
        ├── Match found, ambiguous          →  pass to ML classifier
        └── No match                        →  pass to ML classifier
        │
        ▼ (when ML classifier invoked)
ML classifier output: ranked departments with confidence scores
        │
        ├── Top-1 confidence ≥ HIGH  →  pre-fill; human confirms
        └── Top-1 confidence < HIGH  →  show top 3; human selects
        │
        ▼
Urgency override check
        │
        └── Critical urgency  →  supervisor review before any routing
        │
        ▼
Human officer confirms or overrides → selection logged in audit_log
```

---

## Data Models

### Case ID Format

```
GRV - YYYYMMDD - NNNNNN
 │       │          │
 │       │          └── 6-digit zero-padded daily sequence
 │       └──────────── intake date (UTC)
 └──────────────────── fixed prefix "GRV" (Grievance)

Example: GRV-20261013-004821
```

### Complaint Status State Machine

```
                        ┌─────────────────────────┐
                        │        REGISTERED         │
                        └────────────┬────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                 │
                    ▼                ▼                 │
           ┌──────────────┐  ┌──────────────┐         │
           │ DUPLICATE_   │  │   ASSIGNED   │◄─────────┘
           │ REVIEW       │  └──────┬───────┘
           └──────┬───────┘         │
                  │                 ▼
                  │        ┌──────────────┐
                  │        │ IN_PROGRESS  │◄──────────────────┐
                  │        └──────┬───────┘                   │
                  │               │                           │
                  │   ┌───────────┼──────────────┐            │
                  │   │           │              │            │
                  │   ▼           ▼              ▼            │
                  │ OVERDUE  ESCALATED   PENDING_VERIFICATION  │
                  │   │           │              │            │
                  │   │           │              ▼            │
                  │   │           │         CONTESTED ─────────┘
                  │   │           │              │
                  └───┴───────────┴──────────────┘
                                     │
                                     ▼
                               ┌──────────┐
                               │  CLOSED  │
                               └──────────┘
```

---

## Escalation Rules

```
┌────────────────────────────────────────────────────────────┐
│  SLA DEADLINE                                               │
│                                                             │
│  T - 4h  ──►  Reminder to assigned officer                 │
│  T + 0   ──►  OVERDUE flag set; supervisor notified        │
│  T + 24h ──►  ESCALATED L1; department head notified       │
│  T + 72h ──►  ESCALATED L2; commissioner notified         │
│                                                             │
│  Escalation is additive — it does NOT reassign the case    │
│  or erase prior history. It adds a notification layer.     │
└────────────────────────────────────────────────────────────┘
```

All thresholds are stored in a `sla_config` table. Municipalities configure them without code changes.

---

## Audit & Auditability

Every action in the system produces an immutable audit record. The `audit_log` table is:

- **Append-only** — rows are never updated or deleted
- **Actor-stamped** — every row records the user or system process that caused it
- **State-diffed** — `before_state` and `after_state` columns record what changed in JSON
- **AI-tagged** — the `ai_involved` flag and `ai_confidence` score are recorded whenever an AI suggestion influenced an outcome

**Queries the audit log supports:**
- Who routed this complaint to this department, and when?
- Was this merge decision made by AI or a human?
- How many times was this case reassigned, and why?
- When did the SLA breach occur and who was notified?
- Did the citizen contest the closure, and what was the outcome?

---

## Security & Roles

| Role | Permissions |
|---|---|
| **Citizen** | Submit complaints · View own case timeline · Contest resolution |
| **Officer** | View/update assigned cases · Log actions · Submit resolution evidence · Review duplicate queue |
| **Supervisor** | Officer permissions + approve merges + manage escalations + reassign cases |
| **Admin** | Full access · Configure SLA rules · Manage users and departments |
| **System** | Internal service account for automated workflow events only |

All endpoints are authenticated with JWT. Role is embedded in the token and verified at the API gateway.

Sensitive fields (reporter contact details) are encrypted at rest and accessible only to assigned officer and above.

---

## Deployment Topology

```
Internet
   │
   ▼
┌──────────────┐
│  CDN / WAF   │  (Static assets, DDoS protection)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  API Gateway │  (Auth, rate limiting, routing)
└──────┬───────┘
       │
       ├──► Intake Service
       ├──► Case Engine
       ├──► AI Pipeline Service (async, queue-based)
       ├──► Workflow Engine (cron + event-driven)
       └──► Notification Service
              │
              ▼
       ┌──────────────┐
       │  PostgreSQL  │  (Primary + read replica)
       └──────────────┘
              │
              ▼
       ┌──────────────┐
       │  Message     │  (Redis Streams or RabbitMQ)
       │  Queue       │  (AI pipeline, notifications)
       └──────────────┘
```

**AI components run as a separate service** so that their unavailability degrades gracefully — complaints are still registered, confirmed, and queued for manual review if the AI pipeline is offline.

---

## Key Design Decisions

### ADR-001: AI pipeline is asynchronous and non-blocking
**Decision:** Citizens receive a Case ID and acknowledgement before any AI processing completes.
**Rationale:** AI processing (especially language models) can be slow. Blocking the citizen acknowledgement on AI results would harm user experience and trust. Cases are usable from the moment they are persisted.

### ADR-002: Duplicate detection never auto-merges below high-confidence threshold
**Decision:** Only matches scoring above `HIGH_THRESHOLD` are surfaced as suggestions to officers. Medium-confidence matches go to a human review queue.
**Rationale:** A false merge is worse than a false negative. If two separate incidents are wrongly merged, one citizen loses their complaint history and notifications. Human review is cheap; wrongful merges are costly.

### ADR-003: AI routing suggestion is always human-confirmed
**Decision:** The routing suggester populates a pre-filled field; the assigned officer confirms or overrides. The human selection is what enters the audit log.
**Rationale:** The hackathon case explicitly requires human review and auditability for AI-influenced decisions. Routing a complaint to the wrong department wastes time and erodes citizen trust.

### ADR-004: SLA values are configuration, not code
**Decision:** All deadline and escalation thresholds live in a `sla_config` database table.
**Rationale:** Different municipalities have different rules. Different issue categories have different urgency profiles. Hardcoding SLAs would make the system inflexible and require code changes for routine administrative updates.

### ADR-005: Original language text is always preserved
**Decision:** Translation produces a working copy. The `raw_text` column always stores the original text exactly as submitted.
**Rationale:** Translation models make errors, especially for Hinglish and domain-specific terminology. Officers who speak the original language must be able to read the complaint as submitted. The original is also legally safer as the authoritative record.

### ADR-006: Linked complaints are never deleted
**Decision:** `duplicate_links` records a relationship between a linked report and a canonical case. The linked complaint record itself is never deleted or overwritten.
**Rationale:** Fifty citizens reporting the same drain overflow are evidence of the problem's severity. Their individual reports, timestamps, and contact details must be preserved for notifications and audit purposes.

---

*CivicTrace · Team IRIS · AI Synergy Hackathon 2026 · Round 1*
