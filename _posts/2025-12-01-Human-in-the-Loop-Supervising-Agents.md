---
layout: post
title: "Human in the Loop Supervising Agents"
date: 2025-12-01 07:07:07 +0100
---


# Summary of Human in the Loop (HITL) Agent Architecture Implementation

I first built this application as a supervisor-worker agent , outlined in a blog post here: https://sput.github.io/2025/11/10/Agentic-Compliance-Evidence-Review.html, where a supervisor (python function) called workers (also python functions) to execute code to review evidence. I wanted to try a different solution and so re-built that functionality as a Human in the Loop (HITL) agent model. Essentially under this model a human plays the role of the supervisor, rather than a software agent. The worker agents still carry out reviews the same, but a human approves each step. In the real world this would not be considered "agentic", humans have supervised computer output for decades. The way you would most likely implement this solution is a hybrid along with a supervisor-worker model. The supervisor would control the workers most of the time, while a human would be needed for approval at critical stages. I made this one 100% purely human run for demostration purposes. 

# What is Human-in-the-Loop?
A human-in-the-loop (HITL) agent model is a workflow where AI agents perform the heavy lifting—extracting facts, drafting summaries, and proposing decisions-while explicitly pausing at defined
  checkpoints for a human to review, edit, and approve before the process advances. The agent routes tasks based on confidence thresholds and rules, explains its recommendations, and incorporates
  human feedback to correct errors or handle edge cases. This loop—propose → review → correct → apply → log—keeps decision authority with people, adds guardrails for safety and compliance, and
  produces an auditable trail of what changed and why. Practically, it speeds repetitive steps, reduces variance across reviewers, and escalates only uncertain or high‑risk items. Over time,
  captured feedback becomes training signal to improve the agent’s prompts, policies, or models, steadily increasing accuracy while preserving accountability.

Human-in-the-loop (HITL) classification brings structure, speed, and confidence to how your team reviews compliance evidence. Instead of free‑form reviews that vary by analyst, the workflow
  guides everyone through the same short sequence of checkpoints—capturing the text, validating the date against the audit window, summarizing actions, suggesting likely controls, and finalizing a
  classification. The result is fewer back‑and‑forths, faster cycle times, and decisions that are easier to explain to auditors and stakeholders.
  
  From a user’s perspective, the experience is simple and guided. An analyst uploads a file and the system proposes a first pass: it extracts key details, checks dates against the
  relevant audit period, and drafts a concise summary. It then recommends control candidates based on the evidence, and the analyst confirms or edits each step before moving on. Each decision
  is recorded. The human stays in control; the system does the heavy lifting and documents the rationale.


  ---

## The Agentic Architecture

Below is a summary of the **-worker agents**, the parts of the application that perform all data checks. A **human** orchestrates three narrow sub-agents:

1. **Date Guard** — Verifies that the artifact’s date falls within the audit window  
   - Output: `{ status: PASS|FAIL, parsed_date, reason }`  
   ![Date Guard](/images/Screenshot%202025-11-13%20at%208.15.54%20AM.png)

2. **Action Describer** — Summarizes what the document demonstrates in ≤120 words  
   - Output: `{ actions_summary }`  
   ![Action Describer](/images/Screenshot%202025-11-13%20at%208.17.20%20AM%201.png)

3. **Control Assigner** — Chooses a ranked list of candidates for the best-matching control and provides rationale  
   - Output: `{ control_id, rationale }`  
   ![Control Assigner](/images/Screenshot%202025-11-13%20at%208.17.41%20AM.png)

The execution chain is:

**Date Guard → (if PASS) Action Describer → (if PASS) Control Assigner**

This ensures inexpensive, deterministic checks run before expensive LLM steps.

# Thoughts on this Architecture
When I implemented this solution as a supervisor-worker I concluded that maybe agents were a solution in search of a problem, but I think after creating this implementation, that if you were to hybridize these two architectures, have a supervisor perform most checks, and have a human interact at critical checkpoints, that agents start to make more sense.
