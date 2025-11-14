---
layout: post
title: "Automating Compliance Evidence Review with an Agentic Architecture (and Why I Will Probably Not Use This Architecture Again)"
date: 2025-11-10 07:07:07 +0100
---

*For all code discussed here, visit the Github repository:*  
https://github.com/Sput/compliance_app

Audits are notorious for being a mix of detective work and paperwork. Every compliance framework—SOC 2, PCI DSS, ISO 27001, HIPAA, GDPR—requires collecting “evidence” that proves a security control is implemented and operating correctly. The problem: finding, uploading, classifying, and reviewing hundreds of artifacts is slow, manual, and inconsistent.

This application streamlines that process from **artifact upload to human-approved classification**, using **Next.js**, **Supabase**, and a **Python-based agentic backend**. It’s small enough to deploy quickly but architected to grow from deterministic heuristics to multi-agent reasoning as your review needs scale. *(Scroll to the last section for why I’m not convinced I’ll use this architecture again.)*

---

## What the App Does

At its core, the system is a **compliance evidence pipeline**:

1. **Upload** an artifact (PDF, image, or text file)  
2. **Extract** text and parse metadata (system, date)  
3. **Classify** it against framework controls via deterministic + agentic steps  
4. **Review** the recommendation in a simple dashboard  

The result is a repeatable, transparent workflow that turns unstructured evidence into structured, reviewable data—with traceability and explainability at every step.

---

## The Agentic Architecture

The system uses a **hierarchical supervisor-worker agent model**, common in agentic designs. A **supervisor** orchestrates three narrow sub-agents:

1. **Supervisor Agent** — Oversees all subordinate actions  
   ![Supervisor Agent](/images/Screenshot%202025-11-13%20at%208.25.57%20AM.png)

2. **Date Guard** — Verifies that the artifact’s date falls within the audit window  
   - Output: `{ status: PASS|FAIL, parsed_date, reason }`  
   ![Date Guard](/images/Screenshot%202025-11-13%20at%208.15.54%20AM.png)

3. **Action Describer** — Summarizes what the document demonstrates in ≤120 words  
   - Output: `{ actions_summary }`  
   ![Action Describer](/images/Screenshot%202025-11-13%20at%208.17.20%20AM%201.png)

4. **Control Assigner** — Chooses the best-matching control and provides rationale  
   - Output: `{ control_id, rationale }`  
   ![Control Assigner](/images/Screenshot%202025-11-13%20at%208.17.41%20AM.png)

The execution chain is:

**Date Guard → (if PASS) Action Describer → (if PASS) Control Assigner**

This ensures inexpensive, deterministic checks run before expensive LLM steps.

---

## Architecture Overview

### Frontend — Next.js + Supabase

The Next.js front-end provides:

- **Drag-and-drop uploader** with Supabase Storage integration  
- **Results dashboard** showing extracted text, parsed fields, and candidate controls  
- **Review interface** for Accept/Reject decisions  
- **Status cards** for evidence lifecycle  

Authentication uses **Supabase Auth**. Evidence, metadata, and classifications are stored in **Supabase Postgres**.

---

### Backend — Python Processing Service

The Python backend (FastAPI or CLI runner) performs:

1. **OCR extraction** (provider-agnostic)  
2. **Parsing** key fields (system, evidence_date)  
3. **Classification** into one or more framework controls  
4. **Supervisor orchestration** (agentic mode)  

All results persist via Supabase REST or Postgres connections.

---

## Evidence Lifecycle

1. **Uploaded → Processing → Classified → Needs Review → Accepted/Rejected**  
2. **Date Guard precheck** — auto-rejects out-of-range evidence  
3. **Control Assigner** recommends a control + rationale  
4. **Human reviewer** sees parsed data and can:
   - Accept  
   - Reject (with reason)  
   - Request changes  
5. **Optional auto-accept** for high-confidence cases  

This keeps humans in control, with agents assisting rather than replacing reviewers.

---

## Data Model Highlights

| Table              | Purpose                     | Key Fields |
|-------------------|-----------------------------|------------|
| **audits**        | Audit metadata              | audit_start, audit_end |
| **evidence**      | Uploaded artifacts          | file_ref, extracted_text, system, evidence_date, status |
| **classifications** | Candidate control matches | framework_id, control_id, confidence |
| **reviews**       | Reviewer decisions          | review_status, reviewed_by, reviewed_at, rationale |

All agent outputs are **JSON-contracted** for auditability.

---

## Observability and Guardrails

- **Deterministic date checks** (no LLM needed)  
- **Structured JSON outputs** from every agent  
- **Full tracing** of intermediate steps for audits  
- **Safety limits**: token bounds, fallbacks, circuit breakers  

This ensures decisions are defendable during audits.

---

## Roadmap

- UI improvements  
- Expanded OCR support  
- More detailed rejection reasons  

---

## Why I’ll Think Twice Before Using This Architecture Again

The hierarchical agent model has been heavily promoted in the AI-dev world. I tried it because I wanted to understand the trade-offs.

Here’s what I found:

- **It adds a lot of complexity.**  
  The logical flow is brittle unless you add guardrails everywhere.

- **The guardrails are just as complex as traditional code.**  
  Many “agent guardrails” ended up being fully deterministic functions anyway.

- **Function calling would have been simpler.**  
  A clean set of tools/functions + a small router might outperform a whole agentic chain.

Two ways I *might* be wrong:

1. Agents shine when the application grows substantially and reusable behaviors become valuable.  
2. I may simply need more experience designing resilient multi-agent systems.

But today, for this problem, **traditional function-calling would have been faster, simpler, and more reliable.**
