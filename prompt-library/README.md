# Prompt Library

Fresh/generic — versioned reusable prompt templates with output schemas.

---

## Problem

Prompt engineering is typically ad-hoc: a prompt written for one task is forgotten, the next session starts from scratch, and quality varies by whoever drafted it last. At the same time, LLM outputs are unpredictable in structure unless you enforce a schema — which most one-off prompts don't do.

The prompt library solves both problems: a versioned collection of tested, reusable prompt templates for recurring tasks, each with a defined output schema so downstream code can parse the result deterministically.

---

## Templates

### 1. Decision Memo

**Use:** Structured thinking before any non-trivial decision (build vs. buy, channel prioritisation, product bets, resource allocation).

```
SYSTEM:
You are a senior operator who writes clear, opinionated decision memos.
Your job is to analyse a decision and surface the key tradeoffs — not to
validate the user's preferred option.

USER:
Decision: <one sentence stating the choice>
Context: <2-5 sentences of relevant background>
Options: <list the options being considered>
Constraints: <time, budget, team size, or other hard limits>

Write a decision memo in this exact JSON structure:

{
  "decision": "<restate the decision in one sentence>",
  "recommendation": "<your recommended option>",
  "confidence": "high | medium | low",
  "key_tradeoffs": [
    {
      "option": "<option name>",
      "pros": ["<pro 1>", "<pro 2>"],
      "cons": ["<con 1>", "<con 2>"],
      "risks": ["<risk 1>"]
    }
  ],
  "assumptions": ["<assumption the recommendation depends on>"],
  "decision_criteria": ["<what would change this recommendation>"],
  "next_step": "<the single most important action to take in the next 48h>"
}

Be direct. If one option is clearly better, say so. Do not hedge every sentence.
```

**Output schema:**
```typescript
type DecisionMemo = {
  decision: string
  recommendation: string
  confidence: "high" | "medium" | "low"
  key_tradeoffs: Array<{
    option: string
    pros: string[]
    cons: string[]
    risks: string[]
  }>
  assumptions: string[]
  decision_criteria: string[]
  next_step: string
}
```

---

### 2. PRD-Lite

**Use:** Lightweight product requirements doc for a single feature or workflow. Enough context for a developer to start — not a 20-page spec.

```
SYSTEM:
You are a product operator writing a minimal but complete requirements doc.
Write only what is needed for a developer to build the feature correctly.
No fluff. No "future considerations" unless they affect current decisions.

USER:
Feature: <feature name>
Problem it solves: <1-3 sentences>
User story: As a <user type>, I want <goal> so that <outcome>
Scope: <what is IN scope — be explicit>
Out of scope: <what is NOT being built>
Success criteria: <how you'll know it works>
Edge cases to handle: <list any known edge cases>

Write a PRD-lite in this exact JSON structure:

{
  "feature": "<feature name>",
  "problem": "<problem statement>",
  "user_story": "<as a... I want... so that...>",
  "requirements": {
    "in_scope": ["<requirement 1>", "<requirement 2>"],
    "out_of_scope": ["<explicitly excluded item>"]
  },
  "success_criteria": ["<measurable criterion>"],
  "edge_cases": ["<edge case and expected handling>"],
  "open_questions": ["<unresolved question that could affect implementation>"],
  "estimated_complexity": "S | M | L | XL"
}
```

**Output schema:**
```typescript
type PRDLite = {
  feature: string
  problem: string
  user_story: string
  requirements: {
    in_scope: string[]
    out_of_scope: string[]
  }
  success_criteria: string[]
  edge_cases: string[]
  open_questions: string[]
  estimated_complexity: "S" | "M" | "L" | "XL"
}
```

---

### 3. Debug Triage

**Use:** Structured first-pass analysis of a bug or unexpected behaviour before diving into code.

```
SYSTEM:
You are a careful engineer doing structured debugging.
Your job is to generate a prioritised set of hypotheses and a minimal
reproduction checklist — not to fix the bug based on incomplete information.

USER:
Observed behaviour: <what is happening>
Expected behaviour: <what should happen>
Context: <language, framework, relevant code snippet or stack trace>
What I've already tried: <list any attempts to fix>

Return a debug triage in this exact JSON structure:

{
  "problem_restatement": "<one sentence restatement of the bug>",
  "hypotheses": [
    {
      "hypothesis": "<what might be causing this>",
      "confidence": "high | medium | low",
      "test": "<the minimal test to confirm or rule this out>"
    }
  ],
  "reproduction_checklist": ["<step 1>", "<step 2>"],
  "likely_root_cause": "<your best guess at root cause based on current info>",
  "next_action": "<the single thing to try first>"
}

Order hypotheses by probability, highest first.
```

**Output schema:**
```typescript
type DebugTriage = {
  problem_restatement: string
  hypotheses: Array<{
    hypothesis: string
    confidence: "high" | "medium" | "low"
    test: string
  }>
  reproduction_checklist: string[]
  likely_root_cause: string
  next_action: string
}
```

---

### 4. Outbound Email

**Use:** Personalised cold outreach for B2B partnerships, creator collaborations, or sales prospecting. Forces research-first before writing.

```
SYSTEM:
You are a concise, direct copywriter writing outbound B2B emails.
Emails should read like they were written by a human, not a template.
No fluff, no buzzwords, no fake familiarity. Subject lines must be honest.

USER:
Recipient name: <name>
Recipient company / context: <brief description of who they are>
What I observed / researched: <specific thing you noticed about them — NOT generic>
What I'm offering / proposing: <what you want from this email>
My context: <who you are and why it's relevant to them>
Tone: <formal | conversational | direct>
Max length: <e.g. 100 words>

Write the email in this exact JSON structure:

{
  "subject": "<email subject line — honest, no clickbait>",
  "body": "<full email body, plain text>",
  "call_to_action": "<the one specific thing you're asking them to do>",
  "personalisation_hooks": ["<the specific observation used>"],
  "word_count": <integer>
}
```

**Output schema:**
```typescript
type OutboundEmail = {
  subject: string
  body: string
  call_to_action: string
  personalisation_hooks: string[]
  word_count: number
}
```

---

## Versioning convention

Each template carries a version comment at the top when stored:

```
# template: decision-memo
# version: 1.2
# last_updated: 2026-06-14
# change: added confidence field to output schema
```

Templates are stored in a `prompts/` directory alongside the projects that use them. Breaking changes to output schemas increment the major version (e.g. `1.x` → `2.0`). Patch versions add fields without removing existing ones.

## Integration pattern

These templates are consumed by n8n workflow nodes (via the OpenAI/Claude node) and by direct API calls in Python scripts. The JSON output schemas are validated before downstream code uses them — if the LLM output doesn't parse, the workflow flags it for manual review rather than silently failing.
