# Claude-Powered Loops — Runbook & Reference

A collection of self-inspecting, self-correcting agentic loops built with the Claude API and Claude Code. Each loop follows the same pattern: **inspect context → propose or create work → check itself → revise** — with human-approval gates and guardrails for sensitive data baked in throughout.

---

## Table of Contents

1. [What's in this repo](#whats-in-this-repo)
2. [Setup](#setup)
3. [Loop catalogue](#loop-catalogue)
   - [Repo Onboarding Agent](#1-repo-onboarding-agent)
   - [Bug Triage Loop](#2-bug-triage-loop)
   - [Proposal Response Assembler](#3-proposal-response-assembler)
   - [Slack Thread → Decision Brief](#4-slack-thread--decision-brief)
   - [Design Review Loop](#5-design-review-loop)
   - [Test Generation Loop](#6-test-generation-loop)
4. [Shared guardrails](#shared-guardrails)
5. [Eval rubric](#eval-rubric)
6. [Publishing to Agent Commons](#publishing-to-agent-commons)
7. [Before / after demo](#before--after-demo)

---

## What's in this repo

Each loop is a runnable Claude Code skill or workflow script. They share:

- A **self-check step** where Claude evaluates its own output before surfacing it to a human
- An **allow/deny runbook** (see [Shared guardrails](#shared-guardrails)) that every loop respects
- A **human-approval gate** before any write, post, or publish action
- A **structured eval** so you can measure improvement over time

```text
.
├── loops/
│   ├── repo-onboarding/
│   ├── bug-triage/
│   ├── proposal-assembler/
│   ├── slack-to-brief/
│   ├── design-review/
│   └── test-generation/
├── sample-data/
│   ├── sample-repo/          # Minimal fake repo for onboarding demo
│   ├── sample-bugs.json      # 10 realistic bug reports
│   ├── sample-rfp.md         # Sanitized RFP excerpt
│   ├── sample-slack.json     # Sanitized Slack thread export
│   ├── sample-figma.json     # Design spec excerpt
│   └── sample-src/           # Small JS module for test-gen demo
├── evals/
│   └── rubric.md             # Shared scoring rubric
└── README.md                 # This file
```

---

## Setup

### Prerequisites

- **Claude Code CLI** — `npm install -g @anthropic-ai/claude-code` — [docs](https://docs.anthropic.com/claude-code)
- **Anthropic API key** — set in your environment: `export ANTHROPIC_API_KEY=sk-ant-...`
- **Python 3.10+** — for eval scripts and the Slack sanitizer
- **Node 18+** — for workflow scripts

### Install

```bash
git clone <your-repo-url> claude-loops
cd claude-loops
pip install -r requirements.txt   # anthropic, rich, python-dotenv
npm install                        # @anthropic-ai/sdk
```

### Environment

Create a `.env` file (never commit this):

```env
ANTHROPIC_API_KEY=sk-ant-...
SLACK_EXPORT_DIR=./sample-data/sample-slack.json
HUMAN_APPROVER_EMAIL=you@yourcompany.com
```

### Run any loop

```bash
# General pattern
claude /loop-name [--input <file>] [--dry-run] [--approve auto|manual]

# Examples
claude /repo-onboarding --input sample-data/sample-repo
claude /bug-triage --input sample-data/sample-bugs.json --approve manual
claude /proposal-assembler --input sample-data/sample-rfp.md
```

Pass `--dry-run` to see the proposed output without writing or posting anything.
Pass `--approve auto` only in CI environments where every output has already been reviewed upstream.

---

## Loop Catalogue

### 1. Repo Onboarding Agent

**What it does:** Reads a repository, generates a structured onboarding brief (architecture, key entry points, local dev setup, gotchas), self-reviews for accuracy and completeness, then asks a human to approve before writing `ONBOARDING.md`.

**Command:**
```bash
claude /repo-onboarding --input <path-to-repo>
```

**Steps:**
1. **Inspect** — reads `README`, `package.json`/`pyproject.toml`, directory tree, key source files
2. **Draft** — produces a structured brief (see template below)
3. **Self-check** — Claude re-reads the repo against its draft and flags anything missing or wrong
4. **Human gate** — shows diff, waits for `y/n`
5. **Write** — commits `ONBOARDING.md` if approved

**Output template:**

```markdown
## What this project does (2 sentences)
## How to run it locally
## Key files and where to start reading
## Architecture in plain language
## Common gotchas
## Who to ask when stuck
```

**Sample data:** `sample-data/sample-repo/` — a minimal Node/Python monorepo with intentional gaps so the self-check step has something to catch.

**Failure handling:**
- If the repo is too large (>500 files), the agent reads only the top-level and asks the human to narrow the scope.
- If `git` is unavailable, it falls back to filesystem scan only and notes the limitation in the brief.

---

### 2. Bug Triage Loop

**What it does:** Takes a batch of raw bug reports, scores each by severity and reproducibility, groups related bugs, drafts a triage comment per bug, self-reviews for missing reproduction steps or mislabelled severity, then posts to GitHub Issues only after human approval.

**Command:**
```bash
claude /bug-triage --input sample-data/sample-bugs.json --approve manual
```

**Steps:**
1. **Ingest** — parses bug reports (JSON or plain text)
2. **Score** — severity (P0–P3), reproducibility (confirmed/likely/unclear), affected surface
3. **Draft** — triage comment per bug + grouping suggestions for duplicates
4. **Self-check** — Claude asks itself: "Would a senior engineer be satisfied with this triage?" and flags any P0 labelled without a reproduction path
5. **Human gate** — table view of all proposed labels/comments, approve individually or in bulk
6. **Post** — writes to GitHub Issues (or appends to `triage-output.md` in dry-run mode)

**Sample data:** `sample-data/sample-bugs.json` — 10 reports ranging from clear P0 crashes to vague UX complaints with no steps to reproduce.

**Guardrail:** Any bug containing words like `password`, `token`, `PII`, or `SSN` is automatically flagged for manual review and excluded from the auto-post batch.

---

### 3. Proposal Response Assembler

**What it does:** Reads an RFP or client brief, extracts the explicit requirements, drafts a response section-by-section, self-reviews against the original requirements (checking nothing was missed), then produces a final Word-compatible Markdown document for human editing.

**Command:**
```bash
claude /proposal-assembler --input sample-data/sample-rfp.md
```

**Steps:**
1. **Extract requirements** — numbered list of every stated requirement and evaluation criterion
2. **Draft response** — one section per requirement, with evidence from your company's past work (provide as context or it will note gaps)
3. **Compliance matrix** — table mapping every RFP requirement → response section → confidence level
4. **Self-check** — Claude reads both documents and calls out any requirement with no clear response
5. **Human gate** — diff view + compliance matrix; human edits before finalising
6. **Output** — `proposal-response.md` + `compliance-matrix.csv`

**Sample data:** `sample-data/sample-rfp.md` — a 400-word sanitised RFP excerpt with 12 numbered requirements.

**Guardrail:** The assembler will not include pricing, headcount, or named client references without explicit human input — it leaves clearly marked `[FILL IN: ...]` placeholders instead.

---

### 4. Slack Thread → Decision Brief

**What it does:** Takes a raw Slack thread export, strips all PII (names → roles, emails → `[redacted]`, phone numbers → `[redacted]`), extracts the decision made, the alternatives considered, the dissenting views, and the agreed next steps, then formats a decision brief for an async audience.

**Command:**
```bash
claude /slack-to-brief --input sample-data/sample-slack.json
```

**Steps:**
1. **Sanitise** — Python script (`loops/slack-to-brief/sanitize.py`) strips PII before Claude ever sees the data
2. **Extract** — Claude reads the sanitised thread and extracts: decision, context, alternatives, dissent, next steps, owner, due date
3. **Draft brief** — structured 1-page document
4. **Self-check** — Claude asks: "Is there anything in this brief that could re-identify a participant?" and flags it
5. **Human gate** — shows brief + a diff of what was redacted; human confirms before sharing
6. **Output** — `decision-brief.md`

**Sample data:** `sample-data/sample-slack.json` — a 30-message thread about a fictional architecture decision, with fake names and a planted email address to verify sanitisation.

**Sanitiser rules (configurable in `sanitize.py`):**
- All `@mentions` → `@[role]` based on a role-map you provide, or `@[participant-N]` if unmapped
- Email addresses → `[email redacted]`
- Phone numbers → `[phone redacted]`
- Names in the blocklist → `[name redacted]`

**Guardrail:** If the sanitiser confidence score drops below 0.85 (too many unknowns to safely redact), the loop stops and asks the human to review the raw thread manually.

---

### 5. Design Review Loop

**What it does:** Takes a Figma export or design spec, evaluates it against a checklist (accessibility, consistency with design system, copy quality, edge cases covered), produces a structured review with pass/fail per criterion, proposes specific change suggestions, self-reviews to make sure suggestions are actionable, then posts findings as a Figma comment thread or a Markdown report.

**Command:**
```bash
claude /design-review --input sample-data/sample-figma.json [--checklist loops/design-review/checklist.md]
```

**Steps:**
1. **Ingest** — reads design spec / Figma JSON export
2. **Evaluate** — scores each checklist item: Pass / Needs Work / Fail
3. **Draft findings** — for each Fail or Needs Work, a specific, actionable suggestion
4. **Self-check** — Claude asks: "Is every suggestion specific enough that a designer could act on it without a follow-up conversation?"
5. **Human gate** — rendered checklist + suggestions; reviewer can promote, demote, or edit before posting
6. **Output** — `design-review.md` or Figma comment payload

**Default checklist categories:**
- Accessibility (contrast, touch targets, screen-reader labels)
- Consistency (spacing, colour tokens, typography scale)
- Copy (clarity, tone, truncation behaviour)
- Edge cases (empty states, error states, loading states, long content)

**Sample data:** `sample-data/sample-figma.json` — a spec for a fictional dashboard with three intentional issues planted (missing empty state, off-brand colour, no error state on a form).

---

### 6. Test Generation Loop

**What it does:** Takes a source file or module, reads the function signatures and docstrings, generates unit tests, runs them, reads the failure output, revises the tests, and repeats until all pass or a maximum iteration count is reached.

**Command:**
```bash
claude /test-generation --input sample-data/sample-src/utils.js [--framework jest|pytest|vitest]
```

**Steps:**
1. **Read** — parses source file, extracts functions and their signatures
2. **Generate** — writes an initial test file covering happy path, edge cases, and known error conditions
3. **Run** — executes the test suite and captures output
4. **Self-check / revise** — Claude reads failures, diagnoses root cause (bad test vs. real bug), and either fixes the test or flags a suspected bug to the human
5. **Iterate** — repeats up to 5 times (configurable via `--max-iterations`)
6. **Human gate** — final test file shown for approval before writing to disk
7. **Output** — `utils.test.js` (or equivalent)

**Sample data:** `sample-data/sample-src/utils.js` — a small JS utility module with 8 functions, including two with subtle edge-case behaviour (empty array handling, timezone-sensitive date formatting).

**Guardrail:** The loop will not overwrite an existing test file without an explicit `--overwrite` flag. It writes to `utils.generated.test.js` by default so existing tests are never at risk.

---

## Shared Guardrails

Every loop in this repo respects the following allow/deny rules. These are enforced in `loops/shared/guardrails.js` and checked before any write, post, or publish step.

### Allow list (safe to do automatically)
- Read any file in the working directory
- Write files to `./output/` in dry-run mode
- Generate Markdown, JSON, or CSV output
- Run tests in a sandboxed environment
- Call the Anthropic API

### Deny list (always requires human approval)
- Write or overwrite any file outside `./output/`
- Post to GitHub, Slack, Figma, or any external service
- Include pricing, headcount, or named clients in output
- Process any file matching `*.env`, `*secret*`, `*credential*`, `*token*`
- Proceed if PII sanitiser confidence < 0.85

### Sensitive data rules
1. **Never pass raw PII to the API.** Run the sanitiser first; Claude only ever sees the sanitised version.
2. **API key in env only.** No hardcoded keys; the `.env` file is in `.gitignore`.
3. **Audit log.** Every loop writes an append-only `audit.log` with timestamp, loop name, input hash, and what action was approved or denied.

### Human approval flow

```text
Claude proposes action
       ↓
[APPROVAL REQUIRED] shown in terminal with full proposed output
       ↓
Human types y (approve), n (reject), or e (edit)
       ↓
   Approved? → Execute
   Rejected?  → Log reason, stop loop
   Edit?      → Open $EDITOR with proposed output, re-confirm after save
```

### Failure handling
- **API timeout / rate limit:** exponential backoff, 3 retries, then graceful exit with a clear message
- **Self-check fails 3 times:** loop halts and surfaces the last attempt to the human with a "could not self-resolve" warning
- **Missing input:** descriptive error with expected format, not a stack trace
- **Partial completion:** progress is saved to `./output/.checkpoint.json` so the loop can be resumed with `--resume`

---

## Eval Rubric

Use this rubric to score any loop output before merging or publishing it. Score each criterion 1–5.

| Criterion | 1 (Poor) | 3 (Acceptable) | 5 (Excellent) |
|-----------|----------|----------------|---------------|
| **Accuracy** | Multiple factual errors | Mostly correct, 1–2 minor issues | No errors, verified against source |
| **Completeness** | Key sections missing | All sections present, some thin | All sections present and substantive |
| **Actionability** | Vague, no next step | Next steps implied | Specific, assigned, time-boxed |
| **Safety** | PII or sensitive data exposed | No PII but some risk flags | Fully sanitised, guardrails logged |
| **Self-check quality** | Loop didn't catch obvious issues | Caught most issues | Caught all issues including subtle ones |
| **Human gate respected** | Action taken without approval | Gate shown but not enforced | Gate enforced, edit option offered |

**Minimum acceptable score:** 3 on every criterion before merging. Any criterion scoring 1 is a blocker.

Run the automated portion of the eval:
```bash
python evals/run_eval.py --loop <loop-name> --output ./output/<output-file>
```

This checks: no PII patterns in output, required sections present, audit log entry exists, no files written outside `./output/` without a logged approval.

---

## Publishing to Agent Commons

[Agent Commons](https://agentcommons.ai) is a shared registry of tested, reusable agent patterns. To publish a loop from this repo:

1. **Verify the eval rubric** — every criterion must score ≥ 3
2. **Run the reproducibility check:**
   ```bash
   claude /reproduce-check --loop <loop-name>
   ```
   This replays the loop on the sample data and compares output to the reference output in `evals/<loop-name>/reference-output/`. Pass rate must be ≥ 80%.
3. **Package the loop:**
   ```bash
   python scripts/package_loop.py <loop-name>
   # produces loops/<loop-name>/<loop-name>.agent.json
   ```
4. **Publish:**
   ```bash
   claude /publish-to-commons --package loops/<loop-name>/<loop-name>.agent.json
   ```
   This is a human-gated step — it will show you the package manifest and ask for confirmation before pushing.

---

## Before / After Demo

Each loop ships with a before/after comparison in `sample-data/<loop-name>/`:

| Loop | Before | After |
|------|--------|-------|
| Repo Onboarding | Raw `README.md` (sparse, no local-dev steps) | Structured `ONBOARDING.md` with architecture map, gotchas, and setup commands |
| Bug Triage | 10 unstructured bug reports in a spreadsheet | Labelled, grouped, commented issues with severity scores and duplicate links |
| Proposal Assembler | Blank response doc + RFP PDF | Compliance matrix + full draft response with `[FILL IN]` placeholders clearly marked |
| Slack → Brief | 30-message thread, impossible to skim | 1-page decision brief, fully sanitised, async-shareable |
| Design Review | "Looks good to me" Slack message | Structured checklist with 3 specific, actionable findings |
| Test Generation | 0 tests on a 200-line utility module | 24 passing unit tests covering happy path, edge cases, and error conditions |

To run the full before/after demo for all loops:
```bash
python scripts/run_demo.py --all --approve manual
```

Outputs land in `./demo-outputs/` with a side-by-side `demo-report.md`.

---

## Contributing

1. Fork the repo
2. Add your loop under `loops/<your-loop-name>/`
3. Include sample data in `sample-data/<your-loop-name>/`
4. Add at least one eval in `evals/<your-loop-name>/`
5. Ensure the shared guardrails pass: `python scripts/check_guardrails.py <your-loop-name>`
6. Open a PR — the CI pipeline runs the reproducibility check automatically

---

*Built by Sarah Baker · Robots and Pencils · 2026*
