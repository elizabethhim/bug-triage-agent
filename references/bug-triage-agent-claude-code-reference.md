# Bug Triage Agent — Claude Code Reference

## Context

Elizabeth is building this for an Arize AX take-home assignment. Arize is an AI observability/evaluation platform. The assignment has 3 parts:

1. **Build an agent** and instrument it with Arize AX tracing
2. **Try both workflows** — development (dataset + evals) and observability (tracing). Share feedback on product experience.
3. **Product proposal** — what features should Arize build to increase AX adoption for developers building agent applications? Include MVP scope, mocks, and how you'd present to an engineering team.

**Submission to:** mmende@arize.com, aparna@arize.com, aman@arize.com, sdelucia@arize.com
**Deadline:** May 15/16, 2026

---

## What We're Building

A **Bug Triage Agent** that takes bug reports (simulating a Slack #bugs channel), investigates them using tools, and classifies them. The fictional product being triaged is **TaskFlow AI** — a project management SaaS with AI features.

### Why This Concept
- Multi-step agent with tool calls, routing, and classification → produces rich, interesting traces in Arize
- Failure cases are debuggable (misclassification, wrong tool choice, stale cache reasoning) → compelling for observability demo
- AI features in the fictional product create AI-debugging-AI scenarios → relevant to Arize's users
- Authentic to Elizabeth's PM experience (she's seen these Slack channels at Highwire and Stell)

---

## Architecture

### Agent Pattern
Raw agent loop (no framework). Reasons:
- Easier to instrument with Arize tracing
- Easier to explain every design decision in the presentation
- Frameworks (LangGraph, CrewAI) add abstractions that obscure what's happening — opposite of what you want when demoing an observability tool

### LLM Provider
**Anthropic (Claude)** — not OpenAI like the Arize cookbook examples. Reasons:
- Authentic to how Elizabeth works (built Elizabot with Anthropic API)
- Differentiates from other candidates who'll copy the cookbook pattern
- Arize supports Anthropic via `openinference-instrumentation-anthropic`

### Agent Loop Structure
```
1. Receive bug report as input
2. LLM decides which tool to call based on the report
3. Execute tool, return result to LLM
4. LLM decides: call another tool or produce final classification
5. Loop until LLM outputs final investigation summary
```

### Tools (4 total)
1. **search_product_docs** — searches TaskFlow AI's documentation to understand intended behavior
2. **pull_error_logs** — pulls simulated error/warning logs filtered by service or keyword
3. **search_past_issues** — searches issue tracker for known bugs, duplicates, past resolutions
4. **check_test_coverage** — checks if the affected feature area has e2e tests and their status

All tools are Python functions over simulated data (dictionaries/lists). This is the expected pattern — the Arize cookbook does the same thing with their customer support agent.

### Classification Categories
- **bug** — genuine software defect
- **intended_behavior** — working as designed (may need UX improvement)
- **user_error** — user doing something wrong or lacking permissions
- **needs_more_info** — report too vague
- **duplicate** — matches existing issue

### System Prompt
The agent is instructed to investigate using tools, then classify with: confidence level, investigation summary, evidence cited, and recommended next step.

---

## Simulated Data

**All simulated data is in the companion design doc: `taskflow-ai-bug-triage-agent-design.md`**

That file contains:
- `PRODUCT_DOCS` — 10 documentation entries covering all TaskFlow AI features
- `ERROR_LOGS` — 14 log entries across 8 services, matching the test bug reports
- `PAST_ISSUES` — 7 past bugs (mix of resolved and open), some matching test reports
- `TEST_REGISTRY` — e2e test coverage for 6 feature areas
- `TOOLS` — JSON schemas for all 4 tools
- `SYSTEM_PROMPT` — the full agent system prompt
- `TEST_BUG_REPORTS` — 8 test reports with ground truth labels
- Eval templates for classification accuracy, tool selection, and investigation quality

---

## Test Bug Reports Summary

| # | Reporter | Issue | Ground Truth | Key Signal |
|---|---------|-------|-------------|------------|
| 1 | Account Manager | Export fails for 25K rows | intended_behavior | Docs say 10K limit, BUG-234 resolved |
| 2 | Engineer | AI summarizer references deleted task | bug | Known open bug BUG-267 |
| 3 | Customer Success | SSO redirect loop with Okta | user_error | HTTP vs HTTPS mismatch in ACS URL |
| 4 | Engineering Manager | Auto-assign overloading one person | bug | Stale cache from Redis congestion, BUG-301 open |
| 5 | Designer | File download gives 403 | intended_behavior | Signed URL expired after 1hr, BUG-156 resolved |
| 6 | Intern | AI summarize button doesn't work | user_error | Guest role can't use AI features |
| 7 | Tech Lead | Slack notifications stopped | user_error | OAuth revoked, webhook degraded, BUG-312 resolved |
| 8 | VP Engineering | Weekly digest late and sparse | intended_behavior | 412 tasks triggered timeout fallback |

These exercise different trace shapes: report-6 only needs 1 tool call, others need 3-4.

---

## Build Order

### Step 1: Environment Setup
```bash
pip install anthropic arize-otel openinference-instrumentation-anthropic jupyter
```
- Python 3.10+
- Jupyter notebook (or .py script — notebook preferred for the deliverable since Arize cookbooks use Colab)
- Need: Anthropic API key, Arize space ID + API key

### Step 2: Copy Simulated Data
Paste PRODUCT_DOCS, ERROR_LOGS, PAST_ISSUES, TEST_REGISTRY from the design doc into the notebook.

### Step 3: Implement Tool Functions
Write the 4 Python functions that search/filter the simulated data. Simple keyword matching is fine — this isn't a RAG demo.

### Step 4: Set Up Arize Tracing
```python
from arize.otel import register
from openinference.instrumentation.anthropic import AnthropicInstrumentor

tracer_provider = register(
    space_id="YOUR_SPACE_ID",
    api_key="YOUR_API_KEY",
    project_name="bug-triage-agent",
)
AnthropicInstrumentor().instrument(tracer_provider=tracer_provider)
```

### Step 5: Build the Agent Loop
- System prompt + tool definitions + while loop
- On each iteration: call Claude, check if it wants to use a tool, execute tool, feed result back
- Loop until Claude gives a final text response (the investigation summary)

### Step 6: Run All 8 Test Reports
Run each through the agent. Inspect traces in Arize AX. Screenshot interesting traces.

### Step 7: Create Dataset in Arize
Upload test reports + ground truth labels as a dataset for eval workflows.

### Step 8: Run Evals
Use the eval templates from the design doc with Arize's eval tools (llm_classify or similar).

### Step 9: Take Notes on Product Experience
Document friction points, onboarding issues, what worked, what didn't — this feeds Part 2 and Part 3 of the assignment.

---

## Key Patterns from Arize's Cookbook

Their customer support agent example uses:
- OpenAI function calling with tool schemas as JSON
- `tool_choice="required"` to ensure a function is always called
- `arize-otel` + `OpenAIInstrumentor` for tracing (we use `AnthropicInstrumentor`)
- `llm_classify` from `phoenix.evals` for running evaluations
- Eval templates that output single-word labels ("correct" / "incorrect")
- Datasets uploaded to Arize for experiment tracking

Mirror this structure but adapt for Anthropic's tool use API format and our bug triage use case.

---

## Deliverable Format

- **Part 1:** Jupyter notebook (or link to hosted version) with the agent code
- **Part 2:** 1-2 slides with screenshots and observations on product experience (dev workflow + observability workflow)
- **Part 3:** Product proposal — slide deck (5-8 slides) with problem, "why now," MVP scope, low-fi mocks, architecture diagram, success metrics
- Optional: working prototype of the proposed feature

---

## Notes for the Presentation

- The agent concept is already creative and differentiated — don't overthink it
- Spend zero time in the presentation explaining the fictional product (TaskFlow AI) — it's self-explanatory
- Spend maximum time on: trace walkthrough, agent reasoning, eval results, and the product proposal
- The product proposal should be informed by real friction encountered during Parts 1 and 2
- Elizabeth's differentiator: most PM candidates submit a doc. She submits a doc AND a functional prototype.
