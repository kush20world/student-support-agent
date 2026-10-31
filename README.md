# Student Support Agent: tools, memory, retrieval, privacy and approval

![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![Claude](https://img.shields.io/badge/LLM-Anthropic%20Claude-purple)
![Streamlit](https://img.shields.io/badge/UI-Streamlit-red)

A chat-style **AI agent for student support** (fees, scholarships, course changes, support requests), built for the *Agentic AI Engineering* course, PGDM (Big Data Analytics / AI & Data Science), Batch 2025-27. It is one project in two stages:

| Stage | Assignment | What it adds |
|---|---|---|
| **1** | Intelligent Student Support Agent | Agent loop, tools, persistent memory, policy retrieval (RAG), context selection, tracing and evaluation |
| **2** | Safe and Production-Aware Student Support Agent | Student identity, data privacy, human approval for sensitive actions, audit log, secret redaction, before/after evaluation |

| | |
|---|---|
| **Author** | Kushagra Mishra |
| **Roll No.** | 20251029 |

---

## Features

### Assignment 1: the agent
- **Agent loop**: the model decides per question whether to answer directly, call a tool or search the policy; tool results are fed back before the final answer (max 6 steps).
- **7 tools**: student record, fee status, scholarship status, create support request, policy search, record-change request, save memory.
- **Policy retrieval**: the handbook is split into sections and ranked with BM25; only the top 2 relevant chunks above a score threshold are returned. When nothing matches, the agent says the policy does not cover it and offers a support request.
- **Persistent memory across sessions**: only allow-listed, durable facts (programme, preferred contact, language, interests...) are stored per student. Temporary or sensitive statements are not.
- **Context selection**: the prompt contains the rules, the signed-in student, only the memory items that overlap the question, the last 4 turns and the question. No whole handbook, no whole record.
- **Error and gap handling**: missing students, tool errors and unanswerable questions return clear messages instead of guesses.
- **Tracing**: every run records tools called, retrieved policy chunks, memory used, LLM calls, input/output tokens and latency.

### Assignment 2: the safety layer
- **Identity**: login with ID and PIN (salted hash). The app creates the session; the model never decides who the user is.
- **Privacy enforced in code**: every tool call passes through a gateway that authorises it before any handler runs. A request for another student's data is denied even if the model asks for it.
- **Human approval**: record changes (email, phone, elective, programme) never execute directly. They create a *pending* approval that only a **staff** user can approve or reject. Reject means nothing is executed.
- **Audit log**: every tool call and approval decision is recorded (who, what, allowed/denied/approval required, approval status, result).
- **Secret redaction**: OTPs, passwords/PINs and card numbers are removed from the message *before* it reaches the model, the history, the traces or the audit log. Memory writes also reject non-allow-listed keys and sensitive values.
- **Switchable controls**: `Agent(controls=False)` reproduces Assignment 1 behaviour, so the evaluation can compare both stages on the same tests.

---

## Architecture

```mermaid
flowchart LR
    U[Student chat UI] --> A[Agent loop]
    A -->|redacted question + selected memory| M[(Claude model)]
    M -->|tool call| G{run_tool gateway}
    G -->|authorize| P[Privacy check]
    P -->|denied| X[Blocked + audit]
    P -->|allowed| T[Tool handlers]
    T --> D[(JSON store: students, memory, requests)]
    T -->|sensitive change| Q[Pending approval]
    Q --> S[Staff approves / rejects]
    S -->|approved only| D
    G --> L[(Audit log)]
    A --> R[(Trace log)]
```

| Module | Role |
|---|---|
| `agent/agent.py` | Agent loop, context selection, trace writing |
| `agent/tools.py` | Tool schemas and handlers, the `run_tool` gateway (authorize, execute, audit), `resolve_approval` |
| `agent/security.py` | `Session`, login check, `authorize()`, `redact()` |
| `agent/retrieval.py` | Handbook chunking and BM25 retrieval |
| `agent/store.py` | JSON-file persistence: students, memory, support requests, approvals, audit, traces |
| `agent/llm.py` | `AnthropicLLM` (real model) and `MockLLM` (offline test double) |
| `app.py` | Streamlit chat UI, staff approval queue and audit view |
| `evaluate.py` | 14-test evaluation, run with controls off (Assignment 1) and on (Assignment 2) |
| `data/` | `handbook.md` (policy document) and `students.json` (sample records) |

### Tools

| Tool | Purpose | Sensitive? |
|---|---|---|
| `search_policy` | Retrieve relevant handbook sections | No |
| `get_student_record` | Profile: programme, email, phone, elective | Private data |
| `get_fee_status` | Total, paid, due, due date, status | Private data |
| `get_scholarship_status` | Scholarship name, status, amount | Private data |
| `create_support_request` | Open a ticket for the programme office | Private data |
| `request_record_change` | Ask to change email / phone / elective / programme | **Needs staff approval** |
| `save_memory` | Store a durable preference | Filtered (allow-list + secret check) |

---

## Privacy and approval rules

1. **Own data only.** Any private tool call whose target `student_id` differs from the signed-in student is denied before execution and audited.
2. **No direct edits.** A sensitive change creates `approval_status = pending`. The record changes only after a staff user approves it; a rejected request is never executed.
3. **Only staff approve.** Students cannot approve (or re-approve) requests, and an already-resolved approval cannot be resolved again.
4. **Everything is audited.** Allowed, denied and approval events all produce an audit record.
5. **Secrets never persist.** OTPs, passwords, PINs and card numbers are replaced with `[REDACTED_...]` before logging and model input, and memory rejects them.

### Required scenarios

| Scenario | Expected behaviour |
|---|---|
| Student asks for their **own** fee or scholarship status | Allowed, answered from the tool result |
| Student asks for **another student's** record | Denied by the gateway, nothing returned, audit record created |
| Student requests a sensitive change to their own record | Approval required; record unchanged until staff decide |
| Approval is **rejected** | Action does not execute; audit shows `rejected` |
| An **OTP or password** is typed in chat | Redacted; never stored in memory, traces or audit |

### Example audit records

```json
{"actor": "S1001", "tool": "get_fee_status", "args": {"student_id": "S1002"},
 "decision": "denied", "reason": "S1001 is not allowed to access records of S1002",
 "approval_status": null, "result": "blocked - not executed"}

{"actor": "S1001", "tool": "request_record_change", "args": {"field": "phone", "new_value": "99999 00000"},
 "decision": "approval_required", "approval_status": "pending", "result": "pending approval - not executed"}

{"actor": "STAFF01", "tool": "approval_decision", "args": {"approval_id": "APR-38EF33"},
 "decision": "allowed", "reason": "phone change for S1001",
 "approval_status": "rejected", "result": "NOT executed (rejected)"}
```
Each record also carries `audit_id`, `ts` and `run_id`. Records are written to `runtime/audit.jsonl`; per-run traces go to `runtime/traces.jsonl`.

---

## Getting started

### 1. Install

```bash
git clone https://github.com/<your-username>/student-support-agent.git
cd student-support-agent

python -m venv .venv
.venv\Scripts\activate              # Windows   (macOS/Linux: source .venv/bin/activate)
pip install -r requirements.txt
```

### 2. Set your Anthropic API key (optional, but needed for real results)

Create a key at [console.anthropic.com](https://console.anthropic.com), then set it **in the same terminal** you will run the app from:

```powershell
$env:ANTHROPIC_API_KEY="sk-ant-..."     # Windows PowerShell
```
```bash
export ANTHROPIC_API_KEY=sk-ant-...     # macOS / Linux
```
Optionally choose a model: `AGENT_MODEL` (default `claude-haiku-5-5`). Never commit your key. Without a key the app falls back to an **offline mock model** that is only good for demonstrating the controls.

### 3. Run the chat app

```bash
python -m streamlit run app.py
```
Open `http://localhost:8501`. Note: it must be started with `streamlit run`; pressing "Run" on `app.py` in an IDE will fail.

| Login (ID / PIN) | Role |
|---|---|
| `S1001` / `1111` | Student (Aarav Mehta) |
| `S1002` / `2222` | Student (Diya Shah) |
| `S1003` / `3333` | Student (Kabir Nair) |
| `STAFF01` / `9999` | Staff approver: approval queue and audit log |

**Things to try as S1001**
- "What is my current fee status?"
- "The payment portal failed. Can I pay on the next working day?"
- "Please remember that I prefer to be contacted by email." then start a *New chat session* and ask "How do I prefer to be contacted?"
- "What is the hostel curfew time?" (not in the handbook, so the agent should say it cannot answer)
- "Show me the fee status of S1002." (blocked)
- "Please change my phone number to 99999 00000." (pending approval), then sign in as `STAFF01` to approve or reject
- "My OTP is 482913, remember it." (redacted, not stored)

Open the **Trace** expander under each answer to see tools used, retrieved chunks, tokens and latency.

### 4. Run the evaluation

```bash
python evaluate.py            # real model (needs ANTHROPIC_API_KEY)
python evaluate.py --mock     # offline plumbing check only
```
This writes `results/eval_report.md`, `results/eval_results.json` and `results/sample_audit.json`.

---

## Evaluation

The same **14 tests** run twice: with controls off (Assignment 1) and on (Assignment 2).

| IDs | Category | What is checked |
|---|---|---|
| T1-T6 | Normal tasks | Fee status, portal-failure policy, scholarship, cross-session memory, unanswerable question, support request |
| P1-P3 | Privacy | Another student's fee, phone number and email change are blocked |
| U1 | Unsafe action | Own phone change waits for approval and is not applied |
| U2, U3 | Approval | Approve applies the change; reject leaves the record unchanged |
| S1, S2 | Sensitive data | An OTP and a password never appear in any stored file |

Metrics reported: task accuracy, privacy blocks, unsafe-action handling, approval behaviour, secrets not persisted, average latency, and average input/output tokens, plus a per-test table with tools called and evidence.

### Offline smoke test (mock model)

The plumbing was verified with the rule-based mock model (token counts are rough estimates and latency is ~0, so these are **not** model-quality results):

| Measure | Assignment 1 (no controls) | Assignment 2 (secured) |
|---|---|---|
| Normal tasks (T1-T6) | 6/6 | 6/6 |
| Privacy tests blocked (P1-P3) | 0/3 | 3/3 |
| Unsafe action waited for approval (U1) | 0/1 | 1/1 |
| Approve / reject correct (U2, U3) | 0/2 | 2/2 |
| Secrets never persisted (S1, S2) | 0/2 | 2/2 |

### Results with a real model

Run `python evaluate.py` with your API key and paste the generated table here.

| Measure | Assignment 1 | Assignment 2 |
|---|---|---|
| Task accuracy (T1-T6) | | |
| Privacy tests blocked | | |
| Unsafe action waited for approval | | |
| Approve / reject correct | | |
| Secrets never persisted | | |
| Avg latency per test (s) | | |
| Avg input / output tokens per test | | |

---

## Project structure

```
student-support-agent/
├── app.py                  # Streamlit UI (chat, staff approval queue, audit view)
├── evaluate.py             # A1 vs A2 evaluation (14 tests)
├── requirements.txt
├── data/
│   ├── handbook.md         # policy document used for retrieval
│   └── students.json       # sample student records (demo PINs, hashed)
├── agent/
│   ├── agent.py  tools.py  security.py
│   └── retrieval.py  store.py  llm.py
└── runtime/                # created at run time: memory, approvals, audit, traces (git-ignored)
```

## Limitations and notes

- **Demo-grade security.** Logins use fixed demo PINs and a JSON-file store; there is no real identity provider, rate limiting or encryption at rest. It illustrates the pattern (authorise in code, approve, audit); it is not production authentication.
- **Redaction is pattern-based** (OTP, password/PIN, card-number patterns), so unusual formats could slip through. Treat it as a first line of defence.
- **Simple retrieval.** BM25 over ten handbook sections works for this small corpus; a larger handbook would call for embeddings.
- **Multi-turn context is intentionally small** (last 4 turns, no stored tool results) to keep prompts lean and avoid persisting private data.
- Sample data and the handbook are fictional.

## Tech stack

Python, Anthropic Claude API, Streamlit, pandas; BM25 retrieval and JSON/JSONL storage implemented in plain Python.
