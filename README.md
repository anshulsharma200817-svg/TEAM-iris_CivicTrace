# CivicTrace — Traceable, Accountable Municipal Grievance System

> **AI Synergy Hackathon 2026 · Round 1 · Team IRIS**

CivicTrace is a unified grievance lifecycle platform that gives every citizen complaint a unique identity, a recorded owner, an explainable deadline, and a verifiable outcome — across every channel through which it enters the system.

---

## Table of Contents

1. [The Problem](#the-problem)
2. [Problem Statement](#problem-statement)
3. [Our Solution](#our-solution)
4. [Key Features](#key-features)
5. [Complaint Lifecycle](#complaint-lifecycle)
6. [AI vs. Rules — Where We Draw the Line](#ai-vs-rules--where-we-draw-the-line)
7. [Tech Stack](#tech-stack)
8. [Project Structure](#project-structure)
9. [Getting Started](#getting-started)
10. [Design Principles](#design-principles)
11. [Hackathon Scenarios Tested](#hackathon-scenarios-tested)
12. [Team](#team)

---

## The Problem

Municipal complaint systems fail citizens and staff in two compounding ways:

### Problem A — Lack of transparency and accountability
A citizen registers a complaint but cannot find out:
- **Who** is responsible for fixing it
- **What** action has been taken
- **Why** it is delayed
- **Whether** the resolution is genuine

### Problem B — Duplicate complaints and fragmented registration
The same pothole, broken water pipe, or garbage pile may be reported via phone, WhatsApp, a web portal, or an in-person office visit. These reports often become separate, disconnected records — confusing staff and wasting municipal resources.

### The Underlying Failure
Both problems share one root cause: **there is no reliable, unified lifecycle for a complaint across all the channels through which it enters the system.** A complaint that changes departments loses its history. Duplicate reports create parallel work orders for the same physical problem. Citizens have no window into the process.

---

## Problem Statement

> *Citizens lack transparent, accountable tracking of municipal grievances across fragmented reporting channels. They cannot reliably identify the department responsible, understand actions taken, or know when a complaint is delayed or resolved. Meanwhile, municipal staff struggle to connect duplicate reports and maintain a consistent, auditable record of each incident.*

---

## Our Solution

CivicTrace enforces a single, traceable complaint lifecycle from first report to verified resolution. A complaint ID follows the case regardless of which department handles it, which channel received it, or how many other citizens reported the same incident.

---

## Key Features

| Feature | Description |
|---|---|
| **Omni-channel intake** | Web portal, phone, WhatsApp/SMS, in-person office — all feed one system |
| **Unique Case ID** | Generated at intake; follows the complaint for its entire life |
| **Duplicate detection** | AI-assisted text similarity + location/time proximity; uncertain matches go to human review |
| **Dynamic deadlines** | Category- and severity-appropriate SLAs, not a single fixed timer |
| **Ownership tracking** | Every case has a named responsible officer and department at all times |
| **Citizen status timeline** | Event-driven progress view; only real system events generate updates |
| **Overdue escalation** | Rules-based alerts push to supervisors and higher authorities automatically |
| **Resolution verification** | Evidence required to close; citizen can contest an incorrect closure |
| **Full audit log** | Every action, reassignment, and status change is permanently recorded |
| **Multilingual support** | Hindi, English, and mixed-language descriptions handled by NLP model |

---

## Complaint Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│  INTAKE                                                          │
│  Web · Phone · WhatsApp/SMS · In-Person                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STANDARDISE & IDENTIFY                                          │
│  Location · Category · Description · Timestamp · Evidence       │
│  ──────────────────────────────────────────────────────         │
│  Generate unique Case ID  (e.g. GRV-10482)                      │
│  Send confirmation to citizen                                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  DUPLICATE / RELATED-COMPLAINT DETECTION                         │
│  Compare: location radius · category · time window · text sim.  │
│  ──────────────────────────────────────────────────────         │
│  High confidence → link to canonical incident                    │
│  Low confidence  → queue for human review                        │
│  All linked reports preserved; no evidence discarded            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  ROUTING & OWNERSHIP ASSIGNMENT                                  │
│  Rules engine selects responsible department                     │
│  AI suggests when category is ambiguous → human confirms        │
│  Named officer assigned; SLA deadline set by category/severity  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  ACTIVE TRACKING & ESCALATION                                    │
│  Officer logs actions; system records every event               │
│  Approaching deadline → reminder to officer                      │
│  Deadline breached    → automatic escalation to supervisor       │
│  Second breach        → escalation to higher authority          │
│  Citizen-facing timeline updated on every real event            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  RESOLUTION & VERIFICATION                                       │
│  Officer submits evidence (photo, report)                        │
│  Supervisor review for complex/high-severity cases              │
│  Citizen notified; given window to contest closure              │
│  Case closed only after verification; audit log sealed          │
└─────────────────────────────────────────────────────────────────┘
```

### What the citizen sees at each stage

| Milestone | Citizen timeline entry |
|---|---|
| Complaint registered | Case GRV-10482 confirmed · Sent via SMS/email |
| Assigned | Sanitation Dept · Officer name · Assignment date |
| In progress | Inspection scheduled · Expected date shown |
| Delayed | Delay reason · New expected date · Escalation note if applicable |
| Resolved | Resolution evidence · Option to contest if incorrect |

> **Design rule:** Only real system events trigger a timeline update. No status label moves without a corresponding logged action.

---

## AI vs. Rules — Where We Draw the Line

The case instructions require human review, auditability, and a route to correct AI errors. We honour that explicitly:

| Capability | Approach |
|---|---|
| Generate Case ID, record timestamps | Deterministic backend logic |
| Enforce SLA deadlines | Rules-based workflow engine |
| Identify likely duplicate complaints | AI text similarity + location/time heuristics; uncertain → human review |
| Understand Hindi/English/mixed descriptions | NLP language model; low-confidence outputs flagged |
| Suggest responsible department | Rules first; AI suggestion for ambiguous categories; human confirmation |
| Escalate overdue cases | Auditable workflow rules — no AI involvement |
| Confirm physical resolution | Human evidence submission only — AI never closes a case |
| Challenge incorrect closures | Citizen-initiated; reviewed by supervisor |

> **AI never silently rejects, closes, or merges complaints.** Every AI-influenced decision surfaces a confidence score and a human review queue when that score is below threshold.

---

## Tech Stack

> *(Indicative — to be finalised during development)*

| Layer | Technology |
|---|---|
| Frontend | React / Next.js — citizen portal & officer dashboard |
| Mobile / WhatsApp | Twilio / Meta WhatsApp Business API |
| Backend API | Node.js (Express) or Python (FastAPI) |
| Database | PostgreSQL — complaint records, audit log |
| Workflow engine | Custom rules engine (deadline tracking, escalation) |
| AI — duplicate detection | Sentence-BERT embeddings + geospatial distance |
| AI — NLP / language | Google Gemini / multilingual transformer |
| AI — department routing | Rule layer + ML classifier with human fallback |
| Auth | Role-based (citizen / officer / supervisor / admin) |
| Notifications | SMS, email, WhatsApp outbound |

---

## Project Structure

```
TEAM-iris_CivicTrace/
├── docs/
│   ├── architecture.md       # Full system architecture & AI workflow
│   └── decisions/            # Architecture decision records (ADRs)
├── src/
│   ├── intake/               # Channel adapters (web, SMS, WhatsApp, in-person)
│   ├── core/                 # Case lifecycle engine, ID generation, audit log
│   ├── ai/                   # Duplicate detection, NLP, routing suggestions
│   ├── workflow/             # Deadline engine, escalation rules
│   ├── notifications/        # SMS, email, WhatsApp outbound
│   ├── api/                  # REST API routes
│   └── dashboard/            # Officer & supervisor UI
├── tests/
│   ├── scenarios/            # End-to-end lifecycle scenarios
│   └── unit/
├── architecture.md           # ← You are here (project root copy)
├── README.md                 # ← You are here
└── package.json / pyproject.toml
```

---

## Getting Started

```bash
# Clone the repository
git clone https://github.com/your-org/TEAM-iris_CivicTrace.git
cd TEAM-iris_CivicTrace

# Install dependencies (example for Node.js backend)
npm install

# Copy environment template and fill in credentials
cp .env.example .env

# Start development server
npm run dev
```

Detailed setup instructions for each service (database, AI models, notification providers) will be added in `/docs/setup.md` once the tech stack is finalised.

---

## Design Principles

1. **One complaint, one truth.** Every intake channel writes to the same lifecycle. Reassignment never erases history.
2. **AI assists; humans decide.** No automated system silently closes or merges complaints. Every AI output is auditable.
3. **Transparency must be earned, not performed.** Status updates reflect real logged events — never manufactured progress.
4. **Deadlines are context-aware.** An exposed electrical wire and a damaged park bench are not on the same SLA.
5. **Linked ≠ deleted.** Duplicate detection links related reports to a canonical incident but preserves every individual report, timestamp, and piece of evidence.
6. **Every citizen can contest.** A wrongly closed case must have a clear route for the citizen to challenge it.

---

## Hackathon Scenarios Tested

The prototype must pass these four scenarios before Round 1 submission:

- [ ] **Cross-channel duplicate:** Same incident reported via web portal and WhatsApp. Should link to one canonical case; both citizens notified.
- [ ] **Reassignment without history loss:** Case transferred between departments. Full action history intact; same Case ID.
- [ ] **Overdue without resolution:** SLA breached. Supervisor auto-notified; escalation logged; case remains open.
- [ ] **Uncertain duplicate kept separate:** Low-confidence similarity match goes to human review; not auto-merged.
- [ ] **Citizen contests closure:** Case wrongly marked resolved. Citizen challenges; case reopened; event logged.

---

## Team

**Team IRIS** — AI Synergy Hackathon 2026

---

> *Note on the 23% reassignment figure cited in the case file: this statistic is from a simulated sample provided by the hackathon organisers, not a verified measurement from a real municipality. It is treated here as illustrative evidence, not a general empirical claim.*