# ArcVault Intake Pipeline — Prompt Documentation

---

## Call 1 — Classification

### System Prompt
```
You are a support ticket classifier for ArcVault, a B2B SaaS platform.
Analyze inbound customer messages and classify them accurately.
Respond with valid JSON only. No markdown, no explanation.
```

### User Prompt
```
Classify the customer message below.

Source: {{source}}
Message: {{raw_message}}

Return JSON with these exact fields:
- category: one of Bug Report, Feature Request, Billing Issue, Technical Question, Incident/Outage
- priority: Low, Medium, or High
- confidence_score: float between 0.0 and 1.0

Priority rules:
  High = service down, multiple users affected, data loss risk, billing error
  Medium = single user blocked, compliance need, billing discrepancy
  Low = general inquiry, feature suggestion, non-urgent question

Confidence rules:
  0.90-1.0 = message clearly maps to one category
  0.70-0.89 = mostly clear with minor ambiguity
  0.50-0.69 = could reasonably fit multiple categories
  below 0.50 = insufficient information to classify
```

### Why I structured it this way

The most important structural decision was enumerating the exact allowed values for `category` and `priority` directly in the prompt. Without this, language models will hallucinate plausible-sounding but non-standard category names ("Login Issue", "Account Problem") that break downstream routing logic. Explicit enums eliminate that failure mode entirely. The confidence scoring guidelines were added because LLMs are overconfident by default — without anchoring buckets, most outputs cluster above 0.85 regardless of actual ambiguity, which makes the escalation threshold meaningless. I kept the system prompt short and stable deliberately: in a production system, system prompts are the natural boundary for prompt caching, so minimizing changes to them reduces cache misses. The main tradeoff I made was omitting few-shot examples. Adding 2–3 labeled examples would meaningfully improve accuracy on ambiguous inputs — Sample 4 (SSO question) sits on the boundary between "Technical Question" and "Feature Request" and a well-chosen example would anchor that case. With more time, I would add examples and measure classification consistency across 20–30 synthetic inputs before considering the prompt stable.

---

## Call 2 — Enrichment + Summary

### System Prompt
```
You are a support ticket enrichment agent for ArcVault, a B2B SaaS platform.
Extract structured information to help the receiving team act immediately.
Respond with valid JSON only. No markdown, no explanation.
```

### User Prompt
```
Extract structured information from the customer message below.

Source: {{source}}
Message: {{raw_message}}
Category: {{category}}
Priority: {{priority}}

Return JSON with these exact fields:
- core_issue: one sentence describing what the customer needs
- identifiers: object with account_id, invoice_number, error_code, other (each extracted or null)
- urgency_signal: one of Immediate, Same-day, Next-business-day, No urgency
- billing_delta: numeric dollar difference if a billing discrepancy is mentioned, else null
- escalation_keywords_found: array of any matching phrases found: outage, down for all users, multiple users affected
- summary: 2-3 sentences for the receiving team, written in third person, actionable tone

Urgency rules:
  Immediate = outage, multiple users affected, security incident
  Same-day = single user blocked, billing dispute
  Next-business-day = compliance need, integration evaluation
  No urgency = general question, feature suggestion
```

### Why I structured it this way

The key structural decision was injecting `category` and `priority` from Call 1 into this prompt rather than starting cold. This anchors the model's framing — a Bug Report summary should read like an incident handoff note, while a Feature Request summary should read like a product intake note. Without this context injection, summaries are generic and the receiving team has to re-read the raw message anyway. I requested `billing_delta` as a bare number (not a string) because the Code node needs to evaluate `billing_delta > 500` — extracting it as a typed number eliminates a parsing step and reduces the chance of a string like "$260" slipping through. I used LLM-based detection for `escalation_keywords_found` rather than a hardcoded regex because the model catches semantic equivalents: "the platform is completely down for our team" carries the same urgency as "down for all users" even without the exact phrase. The tradeoff is that LLMs can occasionally produce false positives or miss edge cases that a regex would catch deterministically — in production I would layer both, using the LLM for semantic coverage and a regex as a hard backstop for SLA-triggering terms. With more time, I would also separate the `summary` field into its own dedicated call: summaries have high output variance, and a prompt focused solely on writing the handoff note — with a few labeled examples — would produce more consistent, actionable results than bundling it with entity extraction.
