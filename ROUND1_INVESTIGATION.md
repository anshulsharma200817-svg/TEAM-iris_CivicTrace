# CivicTrace — Round 1 Investigation Record

> **AI Synergy Hackathon 2026 · Round 1 · Team IRIS**
>
> This document is the primary Round 1 submission evidence.
> It demonstrates all five required criteria **explicitly and in order**:
>
> 1. Problem discovery before solution generation
> 2. Purposeful AI use for investigation, reasoning, comparison, research and design
> 3. Human ownership of consequential decisions
> 4. Clear separation of facts, assumptions, inferences, unknowns and external evidence
> 5. A feasible scope for the on-campus Build Round

---

## Table of Contents

1. [Criterion 1 — Problem Discovery](#criterion-1--problem-discovery-before-solution-generation)
2. [Criterion 2 — Purposeful AI Use](#criterion-2--purposeful-ai-use)
3. [Criterion 3 — Human Ownership of Decisions](#criterion-3--human-ownership-of-consequential-decisions)
4. [Criterion 4 — Epistemic Separation](#criterion-4--epistemic-separation)
5. [Criterion 5 — Feasible Build-Round Scope](#criterion-5--feasible-build-round-scope)

---

## Criterion 1 — Problem Discovery Before Solution Generation

### 1.1 What we were given

The case file presented a scenario about municipal grievance management in an Indian urban context. We were not given a solution to validate — we were given a situation to investigate.

Our first action was to resist the instinct to immediately propose a portal or an app. Instead we asked: **what is actually failing, and for whom?**

### 1.2 How we investigated before proposing anything

We followed this sequence deliberately:

```
Step 1 — Read the case file without writing any solution ideas
Step 2 — List every failure mode described or implied
Step 3 — Identify who bears the cost of each failure
Step 4 — Look for the common root cause beneath the surface symptoms
Step 5 — Compare competing problem framings
Step 6 — Choose one framing and justify the choice
Step 7 — Only then begin thinking about system design
```

We did not open a code editor or draw architecture diagrams until Step 7 was complete.

### 1.3 Failure modes we identified

From the case file and general knowledge of municipal systems in India, we catalogued these distinct failure modes:

| ID | Failure mode | Who bears the cost |
|---|---|---|
| F-01 | Citizen cannot find out who is responsible for their complaint | Citizen |
| F-02 | Complaint is closed without genuine resolution | Citizen |
| F-03 | Complaint is delayed with no explanation given | Citizen |
| F-04 | Same physical problem reported multiple times; creates parallel work orders | Municipal staff |
| F-05 | Complaint history is lost when the case is reassigned to another department | Both |
| F-06 | No auditable record of what actions were taken and by whom | Municipal leadership |
| F-07 | Status labels ("In Process") are moved without corresponding real-world actions | Citizen |
| F-08 | Channels operate independently; a WhatsApp report and a web report for the same drain overflow become unrelated records | Municipal staff |
| F-09 | Urgency is not differentiated; a broken park bench and an exposed electrical wire follow the same queue | Citizen, public safety |
| F-10 | Citizens have no route to challenge an incorrect closure | Citizen |

### 1.4 The three candidate problem framings we compared

Before choosing our problem statement, we explicitly compared three framings:

---

**Framing A — "Citizens cannot track their complaints"**

> *The problem is that citizens submit complaints and receive no follow-up.*

- **Strength:** Immediately relatable; maps to a citizen-facing product.
- **Weakness:** Treats the symptom, not the cause. A tracking portal on top of a broken back-end is cosmetic. Citizens would see a timeline that doesn't move — which is arguably worse than silence.
- **Verdict:** Too narrow. It addresses F-01 but ignores F-04, F-05, F-06, F-08.

---

**Framing B — "Duplicate complaints waste municipal resources"**

> *The problem is that the same incident is reported multiple times through different channels, creating redundant work.*

- **Strength:** Quantifiable; the case file's simulated data suggests a meaningful fraction of cases are reassigned (23% figure — see note in §4.1).
- **Weakness:** Duplicate detection is a supporting capability, not a primary experience. A citizen does not experience "duplicate detection failure" — they experience "nobody fixed my drain."
- **Verdict:** Too narrow and too internal. Real enough to include, but should not be the primary frame.

---

**Framing C — "There is no reliable, unified lifecycle for a complaint"**

> *The problem is that a complaint has no single identity, no recorded owner, and no verifiable outcome as it moves across channels and departments.*

- **Strength:** Explains all ten failure modes above. F-01 through F-10 are all consequences of this single structural absence. Duplicate reports exist because there is no canonical record to link to. History is lost because the lifecycle is not enforced across reassignments.
- **Weakness:** Harder to explain in a one-sentence pitch. Requires a more sophisticated solution.
- **Verdict:** **Selected.** This framing is the most honest description of the underlying failure and the most defensible choice.

### 1.5 Chosen problem statement

> *Citizens lack transparent, accountable tracking of municipal grievances across fragmented reporting channels. They cannot reliably identify the department responsible, understand actions taken, or know when a complaint is delayed or resolved. Meanwhile, municipal staff struggle to connect duplicate reports and maintain a consistent, auditable record of each incident.*

This statement describes the **structural failure** (no unified lifecycle), names **both affected groups** (citizens and staff), and deliberately avoids prescribing a technology solution.

### 1.6 What we ruled out before building

Before designing anything, we explicitly ruled out:

| Rejected idea | Why we rejected it |
|---|---|
| Build a new complaint portal only | A new intake channel on top of broken back-end processes solves nothing |
| Use AI to auto-close resolved complaints | The case file explicitly prohibits silent AI closure; it also erodes trust |
| One fixed SLA deadline for all issue types | A broken park bench and an exposed electrical wire cannot share a deadline |
| Delete duplicate reports after merging | Linked reports are evidence; deleting them removes audit trail and individual notifications |
| Use AI for complaint routing without human review | Routing errors waste time and damage citizen trust; human confirmation is non-negotiable |

---

## Criterion 2 — Purposeful AI Use

We divided every system capability into one of three categories: **deterministic logic**, **rules-based workflow**, or **AI-assisted with human oversight**. AI is applied only where it genuinely adds value over simpler approaches.

### 2.1 Where AI is used, and why it is justified

#### AI Role 1 — Duplicate and related-complaint detection

**Why AI and not rules?**

A citizen reporting "गंदा पानी बह रहा है मेन रोड के पास" and another reporting "dirty water overflow near main road junction" describe the same physical event in different languages. A rule-based string-match fails completely. A geospatial radius check alone would create false positives for unrelated issues near each other.

AI adds value specifically here because:
- Text similarity across languages and phrasings is a task suited to embedding models
- The combination of spatial proximity + temporal proximity + semantic similarity requires a learned weighted function, not hand-coded thresholds

**How AI is used:**
- Sentence-BERT (multilingual) encodes complaint descriptions as embeddings
- Cosine similarity scores semantic closeness
- Combined with geospatial distance and time-window filtering
- Output is a confidence score, not a binary decision
- High-confidence suggestions surface to officer as a pre-filled recommendation (overridable)
- Medium-confidence cases go to a human review queue for side-by-side comparison
- AI never merges two cases; a human always makes the final call

**What AI does NOT do here:**
- It does not delete any report
- It does not automatically merge cases
- It does not notify citizens of a merge without human confirmation

---

#### AI Role 2 — Multilingual description processing

**Why AI and not rules?**

Municipal complaints in Indian cities arrive in Hindi, English, Hinglish, regional languages, and mixed scripts. A rules-based keyword extractor would require exhaustive manual vocabulary lists for each language and fail on any new phrasing. A language model generalises to novel expressions.

**How AI is used:**
- Language detection (Hindi / English / Hinglish / other)
- Translation to English working copy — original text always preserved
- Named-entity extraction: location mentions, issue type keywords, urgency signals
- Each extracted field carries a confidence score
- Low-confidence fields are flagged for officer review; the officer edits, not the AI

**What AI does NOT do here:**
- It does not replace the original text
- It does not make routing decisions based on extracted entities alone
- Low-confidence extractions are shown as suggestions, not as confirmed data

---

#### AI Role 3 — Department routing suggestion

**Why AI and not rules alone?**

Most complaints map cleanly to a department: potholes → Public Works, garbage → Sanitation. A rule layer handles these deterministically. But some complaints are genuinely ambiguous: a flooded road may involve both Public Works (road surface) and Water & Sanitation (drainage). A multi-class ML classifier surfaces the ranked possibilities when the rule layer cannot give a single answer.

**How AI is used:**
- Rule layer runs first and handles the majority of cases deterministically
- Ambiguous or multi-category complaints are passed to an ML classifier
- Classifier outputs a ranked department list with confidence scores
- High-confidence result: pre-fills the routing field in the officer dashboard
- Low-confidence result: shows top 3 suggestions; officer selects one
- Urgency signals (exposed wire, gas leak) trigger mandatory supervisor review regardless of confidence

**What AI does NOT do here:**
- It does not route cases autonomously
- It does not notify departments without a human officer confirming the assignment
- The human selection — not the AI suggestion — is what enters the audit log

### 2.2 Where AI is deliberately excluded

| Capability | Why AI is excluded |
|---|---|
| Generate Case ID and timestamps | Deterministic; AI adds no value and would be slower |
| Enforce SLA deadlines | Rules-based; deadlines must be predictable and auditable |
| Escalate overdue cases | Rules-based; escalation must be consistent and explainable |
| Confirm physical resolution | Human evidence required; AI cannot verify a physical fix |
| Close or reject complaints | Explicitly prohibited by the case instructions |
| Citizen challenge review | Must be handled by a human supervisor |

### 2.3 How AI was used in the investigation phase (not just the system)

AI assistance was also used during Round 1 investigation itself, in these specific ways:

| Investigation task | AI used for | Human decision |
|---|---|---|
| Comparing the three problem framings | Articulating trade-offs of each framing | Team selected Framing C after discussion |
| Reviewing Indian municipal complaint system literature | Summarising known failure patterns | Team validated against the case file |
| Drafting the problem statement | Generating candidate phrasings | Team edited and selected the final wording |
| Identifying which failure modes were symptoms vs. root causes | Generating a causal map of failure modes | Team reviewed and accepted the causal chain |
| Suggesting technical approaches for duplicate detection | Listing NLP similarity methods and their trade-offs | Team selected Sentence-BERT approach after evaluating options |
| Drafting SLA tier suggestions | Proposing deadline tiers by category | Team reviewed; rejected the original flat 1–2 day structure |

In every case, the AI output was reviewed by the team, questioned, and either accepted, modified, or rejected before being included in any document.

---

## Criterion 3 — Human Ownership of Consequential Decisions

This section documents every decision point where AI produces a suggestion, and who makes the actual consequential choice.

### 3.1 Decision point map

```
                    AI output                    Human gate
                    ─────────                    ──────────

Duplicate          Confidence score +         Officer: accept link /
detection          candidate match list  ──►  keep separate /
                                              send to supervisor review

Department         Ranked department list ──►  Officer: confirm /
routing            with reasoning note         override / escalate
suggestion                                     to supervisor

NLP field          Extracted entities    ──►  Officer: confirm /
extraction         with confidence score       edit each flagged field

Resolution         n/a (AI not involved) ──►  Officer: submit evidence
verification                                   Supervisor: approve (high-severity)
                                              Citizen: accept / contest

Escalation         n/a (rules-based)     ──►  Supervisor: review /
                                              reassign / close
```

### 3.2 Specific decisions the team made during investigation

These are decisions the **human team** made, some after explicitly rejecting an AI suggestion:

---

**Decision 1: Reject one fixed SLA deadline**

*AI suggestion:* Use a simple 1–2 day deadline for small issues and 1 week for larger ones.

*Team response:* Rejected. A single-tier deadline treats an exposed electrical cable (public safety risk) identically to a damaged park bench. We replaced this with a category-and-severity-weighted SLA table and made the values configurable so municipalities can adjust without code changes.

*Rationale:* Accountability requires deadlines that reflect actual urgency. A fixed deadline creates the illusion of accountability without its substance.

---

**Decision 2: Reject auto-deletion of duplicate reports**

*Initial instinct:* When two reports are confirmed as duplicates, keep only the canonical one.

*Team response:* Rejected. Each linked report represents a real citizen, a real timestamp, and potentially real evidence (photographs, location). Deleting linked reports removes audit trail and means some citizens receive no notifications. We replaced this with a `duplicate_links` relational table that preserves every original report while linking it to the canonical case.

*Rationale:* Fifty reports of the same drain overflow are evidence of severity. Evidence must not be discarded.

---

**Decision 3: Reject AI-only routing with no human gate**

*Initial design sketch:* AI classifier routes complaint to department automatically.

*Team response:* Rejected after reviewing the case file requirement for human review and auditability. Routing errors delay resolution and damage citizen trust. Even high-confidence routing suggestions require officer confirmation before any assignment is made official.

*Rationale:* The officer's confirmation — not the AI's suggestion — is the consequential decision. The audit log records what a human decided, with an `ai_involved` flag indicating when AI informed that decision.

---

**Decision 4: Confirm that the 23% reassignment figure is illustrative, not empirical**

*Initial approach:* Cite the figure as a real statistic.

*Team response:* The case file states this is from a simulated sample. We explicitly note this in documentation (see §4.1) and do not present it as a general real-world measurement.

*Rationale:* Epistemic honesty. Using a simulated statistic as if it were empirical evidence would undermine the credibility of the analysis.

---

**Decision 5: Choose the citizen-facing status timeline as the primary accountability mechanism**

*AI suggestion during design:* Build a real-time dashboard for citizens.

*Team response:* Modified. A real-time dashboard implies continuous activity. Instead, we chose an event-driven timeline that only updates when a real system event is logged. This prevents the interface from manufacturing the appearance of progress.

*Rationale:* "Transparency must be earned, not performed." A status label that moves without a corresponding logged action is deceptive, not transparent.

---

## Criterion 4 — Epistemic Separation

This section clearly marks what we know, what we assume, what we inferred, what we do not know, and what external evidence we brought in.

### 4.1 Case facts — explicitly stated in the hackathon case file

These are claims we treat as given for the purposes of this submission. We do not independently verify them.

| # | Fact |
|---|---|
| CF-01 | Municipal grievances arrive through multiple channels: phone, WhatsApp, web portal, in-person |
| CF-02 | Reports for the same physical incident may become separate records across channels |
| CF-03 | The simulated sample shows ~23% of cases involve reassignment between departments *(note: simulated, not real-world data)* |
| CF-04 | The case requires that AI must not silently close or reject complaints |
| CF-05 | The case requires human review, auditability, and a route to correct AI errors |
| CF-06 | The context involves Hindi, English, and mixed-language complaint descriptions |

> ⚠️ **Note on CF-03:** The 23% reassignment figure is explicitly from a simulated dataset provided by the hackathon organisers. It is used here as an illustrative data point, not as an empirically verified measurement about any real municipality. All claims in this submission that rely on it are conditional on its illustrative nature.

### 4.2 Assumptions — things we believe are true but cannot verify from the case file alone

| # | Assumption | Confidence | What would change if wrong |
|---|---|---|---|
| A-01 | Indian municipalities have named officers who can be assigned as accountable owners of cases | Medium | Ownership model would need to map to a different organisational unit |
| A-02 | Citizens have access to SMS at minimum; smartphone and internet access varies | Medium-High | If SMS is not reliable, notification model needs revision |
| A-03 | Duplicate complaints are more often the result of channel fragmentation than malicious intent | High | Fraud detection layer would need to be added |
| A-04 | Municipal staff currently lack a unified dashboard — they operate per-department | Medium | Integration complexity changes significantly if partial systems exist |
| A-05 | A complaint genuinely resolved in 72 hours is an achievable target for medium-severity issues | Low-Medium | SLA tiers need municipal validation before being set |

### 4.3 Inferences — conclusions we reached by reasoning from facts and assumptions

| # | Inference | Chain of reasoning |
|---|---|---|
| I-01 | Complaint history is lost during reassignment because channels create independent records | CF-01 + CF-02 + A-04 → each channel has its own record; reassignment moves the case without moving its history |
| I-02 | A citizen-facing timeline built on top of a broken back-end would show no movement | F-07 + Framing A analysis → surface transparency on a dysfunctional process = theatre |
| I-03 | Duplicate detection is a support capability, not the primary problem | Framing B analysis → citizens experience the drain overflow, not the duplicate record |
| I-04 | A single fixed deadline creates systemic under-prioritisation of safety issues | A-05 + F-09 → if all cases share one deadline, safety issues are treated the same as cosmetic ones |
| I-05 | Deletion of linked duplicate reports removes audit evidence and breaks per-citizen notifications | A-03 + CF-04 → preserving original reports is both ethically required and practically necessary |

### 4.4 Unknowns — questions we cannot answer without further investigation

| # | Unknown | Why it matters | How we would resolve it |
|---|---|---|---|
| U-01 | What existing systems (if any) are in use at the target municipality? | Integration vs. greenfield changes build complexity significantly | Stakeholder interviews; system audit |
| U-02 | What is the actual distribution of complaint types by category and severity? | This determines whether our SLA tier model is realistic | Historical data request from municipality |
| U-03 | What percentage of duplicate reports arrive within a detectable time and location window? | Determines how useful the AI duplicate detector will be in practice | Pilot dataset analysis |
| U-04 | Do municipal officers have reliable smartphone/internet access during field work? | Determines feasibility of mobile-based action logging | Field visit and device audit |
| U-05 | What is the legal and regulatory framework for complaint handling in the target state? | SLA deadlines and escalation rules may need to comply with existing law | Legal research; RTI inquiry |
| U-06 | How will residents without digital access (no smartphone, no SMS) interact with the system? | Inclusion requirement; the in-person channel must remain fully functional | Offline-first design review |

### 4.5 External evidence — research or knowledge brought in from outside the case file

| # | Evidence | Source type | How it informs the design |
|---|---|---|---|
| E-01 | Indian cities commonly use WhatsApp as an informal grievance channel alongside official portals | General knowledge of Indian civic tech landscape | Validates multi-channel intake requirement |
| E-02 | Sentence-BERT produces strong multilingual embeddings for Hindi-English text similarity | NLP research literature | Informs duplicate detection algorithm choice |
| E-03 | Immutable audit logs are standard practice in government digital systems (e.g. NIC e-Gov guidelines) | Indian government digital standards | Informs append-only audit_log table design |
| E-04 | Service Level Agreements in Indian municipalities are mandated by some state government rules (e.g. Maharashtra Right to Public Services Act 2015) | Public policy knowledge | Supports configurable SLA design; legal compliance may be required |
| E-05 | False duplicate merges in complaint systems cause citizen distrust and legal complications | General knowledge of complaint management systems | Justifies human-gated merge decision (ADR-002) |

---

## Criterion 5 — Feasible Build-Round Scope

This section defines what we will actually build in the on-campus Build Round, and what we are deliberately deferring. The goal is a working demonstration, not a production system.

### 5.1 Build Round constraints (assumed)

- Team size: ~4–6 students
- Duration: 24–48 hours on-campus
- Environment: laptops, shared WiFi, no production infrastructure
- Deliverable: a working demo that can be run end-to-end

### 5.2 MVP — what we will build

The Build Round prototype will demonstrate the full complaint lifecycle through these four core capabilities:

---

#### MVP Capability 1 — Unified complaint intake with Case ID

**What it does:** A web form that accepts complaint text, location, category, and optional photo. Generates a unique Case ID (format: `GRV-YYYYMMDD-NNNNNN`). Returns an immediate acknowledgement with the Case ID.

**Why it proves the concept:** Shows that a complaint is registered as a single canonical record regardless of how it enters. The Case ID is the spine of the entire system.

**Build complexity:** Low. Standard form → REST API → PostgreSQL insert → Case ID returned.

---

#### MVP Capability 2 — Citizen-facing status timeline

**What it does:** A public URL (`/track/GRV-20261013-004821`) showing every logged event for a case in chronological order. Events include: registered, assigned, action taken, overdue, resolved, contested.

**Why it proves the concept:** Demonstrates that transparency is event-driven, not manufactured. Shows the full history of a case without any fake status movement.

**Build complexity:** Low-Medium. Query `case_events` table ordered by `created_at`; render as a timeline component.

---

#### MVP Capability 3 — Officer dashboard with AI routing suggestion

**What it does:** A login-gated dashboard showing assigned cases with SLA countdown. For each case, an AI-suggested department is displayed (pre-filled, editable). Officer logs actions and confirms or overrides the suggestion. Confirmation is logged with `ai_involved = true` and the officer's chosen department.

**Why it proves the concept:** Demonstrates human ownership of AI-assisted routing. Shows that the AI suggestion and the human decision are two distinct, auditable events.

**Build complexity:** Medium. Requires auth, case queue, AI call (Gemini API for routing suggestion), audit log write on every action.

---

#### MVP Capability 4 — AI duplicate detection with human review queue

**What it does:** When a new complaint is submitted, the system computes a similarity score against recent open cases in the same geographic area. If score ≥ HIGH threshold, it surfaces a "possible duplicate" banner on the officer dashboard. The officer sees both cases side by side and clicks "Link" or "Keep separate". Either action is logged.

**Why it proves the concept:** Demonstrates that AI assists but does not decide. The merge is only created when a human confirms it. Linked cases remain independently queryable.

**Build complexity:** Medium-High. Requires embedding computation (can use Gemini embeddings API), cosine similarity, side-by-side UI. In Build Round, may simplify to keyword + location matching if embedding latency is a constraint.

---

### 5.3 What we are deferring (out of MVP scope)

| Deferred feature | Reason for deferral | Path to production |
|---|---|---|
| WhatsApp / SMS intake | Requires Twilio/Meta API credentials and webhooks | Add as a channel adapter post-MVP |
| Phone / IVR intake | Requires voice infrastructure | Out of scope for student hackathon |
| Automated SLA escalation emails | Requires background job runner and email service | Add cron job + SendGrid post-MVP |
| ML routing classifier (trained model) | Requires labelled training data we do not have | Replace with Gemini prompt-based suggestion in MVP |
| Production auth (OAuth / OTP) | Build Round uses simple session auth | Replace with proper auth pre-production |
| Encryption at rest for contact details | Infrastructure concern | Add at database layer pre-production |
| Offline / low-connectivity support | Significant frontend complexity | Progressive Web App phase 2 |
| Citizen mobile app | Responsive web covers this for MVP | Native app phase 2 |

### 5.4 Five scenarios the MVP must pass

Before the Build Round demo, the prototype must successfully run through all five of these scenarios end-to-end:

| # | Scenario | Pass condition |
|---|---|---|
| S-01 | **Cross-channel duplicate** | Complaint A submitted via web; Complaint B (same incident) submitted again. AI flags possible duplicate. Officer links them. Both citizens' timelines show updates for the canonical case. |
| S-02 | **Reassignment without history loss** | Complaint assigned to Dept A. Dept A reassigns to Dept B. Officer in Dept B sees full prior history. Case ID unchanged. Citizen timeline shows reassignment event. |
| S-03 | **Overdue without resolution** | Complaint SLA breaches. `is_overdue` flag set. Supervisor dashboard shows alert. Case is NOT closed. Officer must log a reason. |
| S-04 | **Uncertain duplicate kept separate** | Two complaints with medium similarity score. No auto-link. Human review queue shows them side-by-side. Officer clicks "Keep separate". Both cases remain independent. Decision logged. |
| S-05 | **Citizen contests closure** | Officer marks complaint resolved. Citizen uses contest link. Case status changes to `CONTESTED`. Supervisor receives in-app alert. Case re-opened. Event logged. |

### 5.5 Technology choices for the Build Round

These are finalised for the Build Round (not indicative):

| Layer | Choice | Reason |
|---|---|---|
| Frontend | React + Vite | Fast setup; team familiarity |
| Backend API | Python FastAPI | Clean async support; team familiarity |
| Database | PostgreSQL (local Docker) | Full SQL; append-only audit log pattern |
| AI — routing suggestion | Google Gemini API (prompt-based) | No training data needed; fast integration |
| AI — duplicate detection | Gemini Embeddings API + cosine similarity | Multilingual; no model hosting needed |
| Auth | JWT with simple session (dev mode) | Sufficient for demo |
| Notifications | In-app only (MVP) | No external API dependencies for demo |

### 5.6 Build Round milestone plan

```
Hour 00–04  Database schema + API skeleton + Case Engine (Case ID, audit log)
Hour 04–08  Complaint intake form + citizen status timeline (Capabilities 1 & 2)
Hour 08–14  Officer dashboard + routing suggestion (Capability 3)
Hour 14–20  Duplicate detection + human review queue (Capability 4)
Hour 20–24  Run all 5 scenarios end-to-end; fix blockers; prepare demo
Hour 24+    Polish UI; record demo video; write demo script
```

---

## Summary — Round 1 Criteria Checklist

| Criterion | Where demonstrated |
|---|---|
| ✅ Problem discovery before solution generation | §1.2 investigation sequence; §1.3 failure mode catalogue; §1.4 framing comparison; §1.6 ruled-out ideas |
| ✅ Purposeful AI use for investigation and design | §2.1 three AI roles with justification; §2.2 deliberate exclusions; §2.3 AI use in investigation itself |
| ✅ Human ownership of consequential decisions | §3.1 decision point map; §3.2 five specific team decisions, including cases where AI was rejected or modified |
| ✅ Epistemic separation | §4.1 case facts (with caveat on 23% figure); §4.2 assumptions with confidence levels; §4.3 inferences with reasoning chains; §4.4 unknowns with resolution paths; §4.5 external evidence with sources |
| ✅ Feasible Build-Round scope | §5.2 four MVP capabilities with complexity ratings; §5.3 explicit deferral list; §5.4 five pass/fail test scenarios; §5.5 finalised tech stack; §5.6 hourly milestone plan |

---

*CivicTrace · Team IRIS · AI Synergy Hackathon 2026 · Round 1*
