# AI Clinical Audit — Product Requirements Document

**Owner:** Atin Mittra | **Tech Lead:** Conor  
**Status:** Draft for stakeholder review  
**Related:** MHLA Form Automation Milestone 2 (WPC-802)

---

## Problem

Premier customers' post-evaluation paperwork goes through a single clinical auditor with a **7-day turnaround**. Verizon (new Premier customer) would overwhelm this bottleneck. We need to scale clinical audit capacity without scaling headcount 1:1.

---

## Solution

Insert an **AI audit step** to assist the clinical auditor. Every time a provider submits a document, the AI reviews it and produces audit notes that help the clinical auditor work more efficiently. The AI will take one of three actions:

1. **Add comments directly in DocuSign and send back to provider** — When the submission fails to meet baseline clinical audit guidelines (scoring threshold not met), AI returns specific correctable comments directly in DocuSign, reusing the existing "changes requested" loop.
2. **Escalate to human clinical auditor with audit notes** — When the submission passes the audit threshold or AI is uncertain, it escalates with structured audit notes to accelerate the human auditor's review.
3. **Escalate to human auditor with "No audit notes needed"** — If AI audit produces no findings, it escalates with a note indicating no issues detected, for auditor confirmation.

**Key principle:** All submissions ultimately go to the human clinical auditor for final sign-off. The AI reduces the auditor's workload by pre-screening, producing structured notes, or flagging items that need clarification, not by auto-approving.

---

## Audit Flow

Every time a provider submits or resubmits a document:

1. Form enters the human clinical auditor's queue in the system
2. AI workflow runs asynchronously to review the completed form
3. AI generates a scoring/assessment (based on clinical guidelines provided by clinical team)
4. AI either:
   - Produces correctable comments and triggers the "send back to provider" loop (if score falls below threshold)
   - Produces structured audit notes and escalates to the human auditor (if score passes or is uncertain)
   - Produces no findings and escalates with "no audit notes needed" confirmation
5. Human auditor reviews the form with AI-generated notes/comments to inform their final decision

**Timing:** In most cases, the AI audit will complete before the human auditor reviews the form, providing them with pre-populated assessment data. The form remains in the human queue throughout.

---

## Current State (Milestone 2 — Already Built, Reused As-Is)

| Component | Location | Notes |
|-----------|----------|-------|
| Form status state machine | `rotom/app/services/mhla/form_transition.rb` | Existing; no new states required |
| Premier tier gate | `contract_terms.mhla`; checked in `rotom/app/jobs/docusign/import_signed_form_document_job.rb` | Routes Premier forms to human audit |
| DocuSign envelope | `rotom/app/services/docusign/envelope_service.rb` | Provider signs and form enters audit queue |
| Envelope webhook to status transitions | `rotom/app/jobs/docusign/process_connect_event_job.rb` | Routes to audit when signed |
| Decline and regenerate | `rotom/app/jobs/docusign/regenerate_after_review_job.rb` | Used for human-auditor rejections; will be reused for AI send-back cases |
| Existing LLM patterns | `llm-orchestrator-api/workflows/pdf_field_annotator/` | Pre-send field detection pattern to be reused by AI audit workflow |

---

## Where We Need to Build

| # | Change | Repo / File |
|---|--------|-------------|
| 1 | New LLM workflow that audits a **completed** form for correctness (missing fields, inconsistent answers, non-compliant responses) against clinical guidelines. Returns: audit notes + confidence score + inline comments for provider corrections (if below threshold) | New workflow under `llm-orchestrator-api/workflows/` (e.g. `form_completion_audit/`), following the `pdf_field_annotator` pattern: guardrail node → structured extraction → scoring logic |
| 2 | Asynchronous job that: (a) waits for provider signature, (b) calls AI audit workflow, (c) routes result to either provider send-back or human auditor escalation | New job in `rotom/app/jobs/docusign/` (e.g. `run_ai_audit_job.rb`), triggered by envelope webhook after signature |
| 3 | Write AI-generated comments onto the DocuSign envelope for the provider's "send back" case (reuse existing tab-writing infrastructure) | `rotom/app/services/docusign/annotations_to_tabs.rb` (existing pattern, extended) |
| 4 | Store AI verdict, confidence score, raw provider input (form data), and audit notes for: (a) audit trail/QA, (b) future model enhancements, (c) human auditor visibility | New columns on `mhla_form_completions` or new `mhla_ai_audits` table to capture: `ai_score`, `ai_notes`, `ai_confidence`, `ai_action`, `raw_provider_input` |
| 5 | Care Support / auditor visibility into what the AI assessed and why (show AI notes in Zendesk, provider portal, auditor dashboard) | `rotom/app/jobs/mhla_zendesk_comment_job.rb` (new comment templates for AI assessments) |
| 6 | Fail-safe guard: unknown/low-confidence AI outcomes always escalate to human auditor (never send back to provider) | `rotom/app/services/mhla/cutover_guard.rb` pattern — apply same "conservative default" rule |

---

## Scoring / Threshold Mechanism

The clinical team will define what "passes" the AI audit. We will implement a scoring mechanism where:

- **Prompt input:** Clinical guidelines, completeness rules, compliance requirements per form type
- **Scoring output:** AI produces a score/verdict. TBD whether this is binary (pass/fail) or spectrum (0-10 grade) — clinical team to decide
- **Threshold logic:** If score falls below threshold → send back to provider with correctable comments. If score passes or is uncertain → escalate to human auditor with notes
- **Comments:** AI always produces specific, actionable comments (either for provider correction or for auditor context)

---

## Requirements

### Functional

- AI runs automatically once the provider's envelope reaches `signed` status
- AI audits **every provider submission and resubmission** (not just first pass)
- Three and only three AI outcomes: (a) send back with comments, (b) escalate with audit notes, (c) escalate with no-notes-needed flag
- Every send-back includes specific, actionable comments in DocuSign the provider can address directly
- Every AI decision is logged with reasoning/evidence for audit trail and QA
- Human auditor can always override AI assessment and make final call
- AI assessment is available to the auditor before or as they review the form

### Non-Functional / Safety

- **Fail closed:** Any low-confidence, ambiguous, or error case escalates to human auditor with flagged uncertainty
- **No auto-approval:** AI never auto-releases a form to the member. All forms must receive final human auditor sign-off
- No behavior change for non-Premier or non-enrolled customers
- **Accuracy validated against historical auditor decisions** before AI scoring threshold is deployed to production
- Ongoing QA sampling of AI-assessed forms post-launch to catch model drift

---

## Phasing & Effort

| Phase | Work | Duration |
|-------|------|----------|
| 1 | Spec + prompt design with clinical stakeholders | TBD |
| 2 | Build AI workflow + Rotom integration (new job, LLM workflow, score logic, comment routing) | 1-2 weeks |
| 3 | Deploy to test environment; iterate prompt with clinical team until satisfied | TBD |
| 4 | Validation: Audit threshold accuracy against historical human auditor decisions (~10 test forms of varying compliance) | TBD |
| 5 | Controlled live rollout (Verizon + other opted-in Premier customers) | TBD |

**Total engineering effort:** ~2 weeks. **Schedule risk:** Prompt refinement and threshold validation depend on clinical team availability and volume of historical audit data.

---

## Critical Open Questions

- Should the AI scoring be binary (pass/fail) or a spectrum (0-10 grade)?
- What confidence threshold triggers "escalate to auditor" vs. "send back to provider"?
- Do we have enough historical (form, auditor-decision) pairs to validate the AI scoring before going live?
- Should the human auditor's queue be prioritized (AI-flagged items first) once Verizon volume arrives?
- What level of visibility should the provider have into why the AI sent them back (detailed comments vs. summary)?

---

## Success Metrics

- Clinical auditor average review time per form decreases by 30-50% within 8 weeks of go-live
- AI audit accuracy ≥ 95% alignment with subsequent human auditor decisions (shadow mode validation)
- Verizon post-evaluation turnaround improves to ≤2 days
- Zero unlogged AI decisions; 100% audit trail coverage
- No increase in provider escalations or compliance issues post-launch
- Human auditor confidence in AI assessments measured via periodic surveys
