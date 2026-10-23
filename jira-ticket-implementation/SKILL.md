---
name: jira-ticket-implementation
description: 'Take a JIRA ticket (URL or key like PROJ-12345), read the ticket title, description, key identifiers, and comments, classify as bug or story, validate the ask against this codebase, propose an implementation roadmap, outline file-level changes, present code changes with business + technical justification, and apply them only after explicit user approval. Use when the user shares a JIRA link, ticket key, or says things like "work on this ticket", "fix this bug", "implement this story", "investigate this ticket", "plan a fix for PROJ-XXXXX".'
argument-hint: '<JIRA URL or ticket key, e.g. PROJ-12345>'
---

# JIRA Ticket Implementation Workflow (Generic)

End-to-end workflow for turning a JIRA ticket into a validated, well-justified code change in **any backend codebase**. The workflow has **explicit user-confirmation gates** — do not skip past them.

> **Before using this skill in a new repo**, do a project-wide find-and-replace on `<jira_mcp_tool>` with the actual MCP JIRA tool prefix available in your workspace (e.g. `mcp_jira`, `mcp_adp_jira`, `mcp_acme_jira`, etc.). Every JIRA tool reference below uses `<jira_mcp_tool>_<action>`.

## When to Use

- User pastes a JIRA URL or ticket key (e.g. `PROJ-12345`, `https://jira.../browse/PROJ-12345`)
- User says: "work on this ticket", "fix this bug", "implement this story", "investigate this ticket", "plan a fix for ..."
- User wants a structured plan + code change for a defect or enhancement tracked in JIRA

Do NOT use for:
- Creating new JIRA tickets
- Posting standalone comments on tickets (unrelated to a code change)
- Pure code questions where no JIRA ticket is involved

## Prerequisites

Before starting, load the repo's onboarding / context documentation so the analysis and code changes match the project's conventions.

Look for and load only the file(s) relevant to the ticket's domain. Common locations to check:

- `AGENTS.md` at the repo root
- `.github/copilot-instructions.md` (may itself point to a memory bank of topic files, e.g. `.github/copilot/*.md`)
- `README.md`, `CONTRIBUTING.md`, `ARCHITECTURE.md`, `docs/`
- Any project-specific memory bank the repo maintains (e.g. topic files covering product, tech stack, module structure, code patterns, architectural decisions)

Load only what is relevant to the ticket — don't pull everything.

## Required Tools

- `<jira_mcp_tool>_retrieve_ticket_details` — fetch title, description, fields
- `<jira_mcp_tool>_retrieve_ticket_comments` — fetch comments for context
- `<jira_mcp_tool>_download_single_attachment` — pull attached sample files if relevant
- `<jira_mcp_tool>_add_comment_to_ticket` — post the post-implementation summary comment (Stage 8)
- `renderMermaidDiagram` — render flow / sequence diagrams in Stage 2 when helpful (if available)
- A code-exploration subagent (if available in the workspace, e.g. `Explore`) — for thorough codebase investigation, including the Stage 4 regression-impact check. If no subagent exists, fall back to `grep_search` / `vscode_listCodeUsages` / targeted `read_file`.

## Procedure

The workflow has **9 stages** with **5 hard confirmation gates**. Each gate requires explicit user "yes / go ahead / approved" before continuing.

---

### Stage 1 — Ingest the ticket

1. Extract the ticket key from the input (URL → strip to `PROJ-XXXXX`).
2. Fetch ticket details and comments in parallel.
3. From the description and comments, extract:
   - **Issue type** (Bug / Story / Task / Spike) — from the `issuetype` field
   - **Primary domain identifiers** — any stable IDs the ticket uses to point at the affected data / entity / request (e.g. request ID, correlation ID, tenant ID, customer ID, order ID, transaction ID, run ID, UUID). If the repo uses a specific ID pattern (regex), use it. If multiple identifiers are present, list them all. These IDs are used later to locate a matching sample / fixture / log entry in the workspace when one is needed for repro.
   - **Business / functional context** — the domain-specific fields the ticket template asks for (e.g. module, component, environment, client / tenant, feature area). Extract whatever the project's ticket template calls out.
   - **Attached sample files** — list every attachment with filename + size. Do not download yet.
   - **Linked tickets** — if present in other systems (e.g. support-desk ticket, incident ticket, parent epic), mention them; do not fetch unless the primary ticket lacks context.

4. If any of the above are missing AND seem important, ask the user before proceeding (do not assume).

5. **Attachment / sample fetching** — at the end of Stage 1, decide whether you need an attached file or a workspace sample (looked up by one of the primary domain identifiers) to understand the issue. If yes:
   - **Workspace sample first**: `file_search` / `grep_search` for the identifier or related filename in the repo (e.g. fixtures, sample inputs, test data folders, `*_sample.*`, `*_temp.*`).
   - **JIRA attachment fallback**: use `<jira_mcp_tool>_download_single_attachment` for the specific file you need (don't bulk-download).
   - **Linked-ticket fallback**: only if the primary ticket + its attachments still don't give enough context, follow linked tickets.

---

### Stage 2 — Build understanding of the problem

Write a short summary (no more than ~10 lines) covering:

- **For a Bug**: what the user/client says is broken, what the expected behavior is, what the actual behavior is, and the specific data condition or input that triggers it.
- **For a Story / Enhancement**: what new capability is being requested, what the inputs and outputs are, and which part of the system it touches.

**Add a pictorial depiction** alongside the text summary whenever it sharpens understanding (most non-trivial tickets benefit from one). Pick the lightest format that fits:

- **Mermaid flowchart / sequence diagram** — for control flow, expected vs. actual paths, or a request lifecycle. Render with `renderMermaidDiagram` when helpful.
- **ASCII before/after table or tree** — for data-shape changes (e.g. JSON field rename, DB column added, config restructured).
- **Side-by-side example** — for input/output transformations (show one concrete row of the ticket's actual data going in, and what should come out).

Always pair the diagram with a concrete **example** drawn from the ticket itself — use the real identifiers and a snippet of the actual offending input where available. Synthetic examples are a fallback only.

Example shape for a Bug:

> **Summary**: For `<component / module>`, `<operation>` produces `<incorrect outcome>` instead of `<expected outcome>` under `<specific condition>`.
>
> **Expected vs. Actual** (request `abc-123-...`):
>
> ```mermaid
> flowchart LR
>   A[Input branch 1] --> M[Merge / process]
>   B[Input branch 2] --> M
>   M -->|Expected| E[Result: X separate from Y]
>   M -->|Actual| X[Result: X + Y collapsed]
> ```
>
> **Trigger condition**: any request where `<field>` has value `<X>` while `<other field>` is also present.

Do not start investigating code yet — this stage is pure ticket comprehension; the diagram describes *what the ticket says*, not *what the code does*.

> **Story branch**: For Story / Enhancement tickets, after Stage 3 (validation) the workflow diverges — see [Story Variant](#story-variant--design-doc-first) below. Bugs continue with Stage 4 as written.

---

### Stage 3 — Validate the ask against the codebase

Use the exploration subagent (thoroughness: **medium** or **thorough** depending on scope), or `grep_search` / `vscode_listCodeUsages` / `read_file` if no subagent is available, to investigate:

- **For a Bug**: locate the module/class/function the bug lives in. Read the actual logic. Confirm whether the reported behavior matches the code. Identify the root cause. Note any data assumptions the code makes that the ticket's input violates.
- **For a Story**: locate the closest existing implementation (similar feature, similar surface, similar transformation). Identify reusable components, base classes, or utilities. Map the new feature onto the existing architecture.

Report back to the user:

1. **Is the ask valid?** (For bugs: is this actually a bug, a misuse, or expected behavior? For stories: is the proposed change feasible with current architecture?)
2. **Root cause / architectural fit** (in 3-6 lines)
3. **Files / classes / functions involved** (linked)

If the ask is **not valid** (e.g. the reported behavior is actually correct, or the data is malformed upstream), stop here and explain — do not propose a fix until the user redirects.

---

### Stage 4 — Propose an implementation roadmap  →  GATE 1

Present a roadmap with:

- The approach in 3-5 numbered steps
- **Alternative approaches**, if any exist, with one-line trade-off per alternative
- Risks (regression surface, areas affected, tests that may need updates)
- Whether the change is local or cross-cutting

**Regression-impact check** (mandatory, one per proposed approach):

Before presenting each approach, use the exploration subagent (or targeted `grep_search` / `vscode_listCodeUsages`) to:

1. **Enumerate every consumer** of the code being changed — call sites, subclasses, shared utilities, and tests that exercise the touched function/class. If the change sits in a base class or shared utility, list every downstream component that flows through it.
2. **For each enumerated consumer, evaluate whether the proposed change alters its behavior** and attach an explicit verdict. Do not just list the consumer and stop — reason through the data shape / input condition each consumer relies on and state one of:
   - **Not affected** — explain *why* in one line (e.g. "new branch only triggers when `<field>` is present; consumer X never emits it, so its path is unchanged").
   - **Affected but acceptable** — explain why the altered behavior is still correct for that consumer.
   - **Affected and risky** — call it out and propose a concrete mitigation (or recommend a different approach).
3. **Existing test coverage** — name the test files protecting each affected path; flag thin or missing coverage.

Present the finding inline as a short **Regression risk** block per approach. Every enumerated consumer must carry a verdict — no bare names.

> **Regression risk (Approach 1)**:
> Touched: `<function>` in `<path/to/file>`.
>
> - `<ConsumerA>` — **affected, intended** (this is the fix target).
> - `<ConsumerB>` — **not affected**: new branch keys off `<field>`, which this consumer never sets, so the function falls through to the existing path.
> - `<ConsumerC>` — **not affected**: this consumer sets `<field> == <value>` only; the new branch is a no-op for that value.
> - Subclasses / callers found via `vscode_listCodeUsages` — none override the changed method, so no hidden consumer.
>
> Existing coverage: `<test file 1>`, `<test file 2>` — both continue to exercise the unchanged path.
> Mitigation: default to existing logic when `<field>` is `None` or unrecognized.

Rules:

- An approach **cannot be presented** until every enumerated consumer has an explicit verdict attached.
- If any consumer lands on "Affected and risky" with no clear mitigation, say so and recommend a safer alternative.
- If the regression surface is genuinely empty (purely local change, no external callers), say "No external consumers — change is local to `<file>`" rather than omitting the block.

Ask the user explicitly:
> "Which approach do you want me to take? Or do you want me to revise?"

**Stop. Wait for approval.** Do not move to Stage 5 until the user picks an approach (or modifies one).

---

### Stage 5 — Outline file-level changes  →  GATE 2

Once the approach is approved, list every file you will touch:

| File | Change type | What changes | Why |
|------|-------------|--------------|-----|
| `path/to/file` | modify | one-line summary | one-line reason |
| `tests/...` | add | new test for X | covers Y |

Include:
- **Tests** you will add or update — pick the right level (unit / integration / component / e2e / contract) for the repo's test layout
- Any **config / constants / fixtures** that need updates

Ask the user explicitly:
> "Does this list of changes look right? Anything to add or drop?"

**Stop. Wait for approval.**

---

### Stage 6 — Present code changes with justification  →  GATE 3

For each change, present:

1. **Business understanding (plain English)** — what the user/client gets out of this change, with a concrete example using the ticket's actual data / identifiers when possible.
2. **Technical justification** — why this code shape (design pattern choice, why an existing helper is reused, why a specific class is extended).
3. **The exact code change** — the actual `diff`-style before/after for each edit, no `...existing code...` placeholders that hide important context.

Group changes by file. Keep each change minimal and focused on the ticket — no opportunistic refactors.

Ask the user explicitly:
> "Approve these changes? I'll apply them and then suggest running the relevant test suite."

**Stop. Wait for approval.**

---

### Stage 7 — Apply, validate, and report

After explicit approval:

1. Apply edits using `multi_replace_string_in_file` where multiple edits cluster in one file.
2. Run `get_errors` on every edited file.
3. **Test coverage decision**:
   - **Small fix** (one or two files, narrow logic change): ask the user whether they want a test added; do not add unprompted.
   - **Big fix** (multiple files, behavioral change, new branch in logic, cross-cutting): first scan existing tests for coverage of the changed code paths. Report which tests already cover it. If gaps exist, ask the user whether to add tests and at which level.
4. Suggest the right test command for the repo (whatever the project uses — `make test-*`, `npm test`, `pytest`, `go test`, `mvn test`, etc.). Do not run it automatically unless the user asks.
5. Summarize in 2-3 lines: what changed, which tests to run, any follow-ups (e.g. "consider adding a regression sample under `<tests path>`").

Do not create branches or open PRs — the user handles those separately.

---

### Stage 8 — Draft JIRA summary comment  →  GATE 4

Once edits are applied and tests are evaluated (Stage 7 complete), draft a **brief** summary comment for the JIRA ticket. Keep it short — a few lines, not a wall of text.

Structure:

- **What changed (plain English)** — one or two sentences on the user-visible outcome, using the ticket's context and identifiers.
- **Technical gist** — one or two sentences naming the file(s) / class(es) / function(s) touched and the nature of the change.
- **Tests** — one line on what was added/updated or which existing suite covers it.

Example:

> Resolved `<problem>` for `<component / area>`. Added `<mechanism>` in `<Class.method>` at `<path>` so `<condition>` is handled correctly, with a fallback to existing behavior when `<field>` is absent. Covered by a new case in `<test file>`; existing suites for adjacent components still pass.

Show the drafted comment inline and ask explicitly:
> "Post this comment to the ticket? (yes / edit / skip)"

**Stop. Wait for confirmation.**

- On **yes**: post via `<jira_mcp_tool>_add_comment_to_ticket`.
- On **edit**: revise per user feedback and re-confirm before posting.
- On **skip**: do not post; end the workflow.

Do not post until the user says yes.

---

### Stage 9 — Memory-bank / context-docs reconciliation  →  GATE 5

After the JIRA comment step (Stage 8) is resolved, check whether this change introduced anything that a *future* agent working an unrelated ticket would be materially misled or slowed down without knowing. The bar is deliberately high — most tickets, especially Bugs, will not qualify.

**Trigger criteria** (propose an update only if the change hits at least one):

- **Product / functional docs** — a new capability, a new supported input type, a new integration surface, or a meaningful shift in what an existing capability covers.
- **Structure / architecture docs** — a new top-level module, a new base class, a new entry point, a renamed/moved core module, or an updated component diagram.
- **Tech / stack docs** — a new library dependency, a language version change, a new env var, a new infra component, or a changed external integration.
- **Patterns docs** — a new cross-cutting code pattern (e.g. new strategy registry, new error-handling convention, new logging shape, new validation approach) that future code should follow.
- **Decisions docs** — an architectural decision worth recording, a deprecation, or a newly discovered "known issue".

**Explicit non-triggers** (do NOT propose updates for these):

- Bug fixes that don't change architecture or public surface
- New test additions on their own
- A single function added inside an existing class
- Refactors that don't change public surface
- Anything ticket-specific (e.g. "PROJ-12345 fixed X") — that belongs in JIRA, not the memory bank
- One-off data quirks for a single user / tenant / customer

**Default to skip on doubt.** If you are not confident the change clears the threshold, do not propose an update — silently end the workflow. Better to under-propose than to spam.

If a trigger is hit:

1. Identify the specific context-doc file(s) and the exact section to change.
2. Draft a **diff** — the actual lines to add / modify, kept as terse as the rest of the file (these files are brevity-first; they are loaded into context automatically).
3. State in one line *why* this rises above the threshold (which trigger it matches, and what a future agent would miss without it).

Show the proposal inline and ask explicitly:
> "Update the memory bank / context docs with this? (yes / edit / skip)"

**Stop. Wait for confirmation.**

- On **yes**: apply the edit with `multi_replace_string_in_file` (or `replace_string_in_file` for a single block). Run `get_errors` only if a code-aware file was touched (markdown context files usually don't need it).
- On **edit**: revise per user feedback and re-confirm before applying.
- On **skip**: do not edit; end the workflow.

Do not edit any context-doc file until the user says yes.

---

## Story Variant — Design Doc First

Stories need shared understanding before any code is written. Replace Stages 4–7 with this sequence for Story / Enhancement tickets:

### Story Stage A — Design doc  →  GATE S1

Produce a short design doc covering:

1. **Goal** — one sentence on the user-facing outcome.
2. **Scope** — explicitly in-scope and out-of-scope items.
3. **Inputs / Outputs** — data shapes, surfaces affected, example payloads if relevant.
4. **Proposed architecture** — which existing classes/modules are extended, which new ones are added, how it slots into the existing pattern (singleton service, strategy per variant, base class + subclass, handler registry, etc.).
5. **Alternatives considered** — one-line pros/cons for each.
6. **Open questions** — anything ambiguous that needs the user's input.

Present the design doc inline in chat. Ask:
> "Does the design match your understanding? Any changes before we plan the code?"

**Stop. Wait for approval / amendments.** Iterate the design doc as needed.

### Story Stage B — Code-change pass

Once the design doc is approved, run the original **Stage 5 (file outline) → Stage 6 (code changes) → Stage 7 (apply)** sequence with their gates intact.

---

## Standing Rules for Every Code Change

These apply at Stages 5, 6, and 7 — every time:

1. **Follow the repo's language, version pins, and style conventions.** Match the surrounding code (formatter, linter, type-hint style, import ordering). Do not introduce syntax or features unsupported by the repo's declared language version.
2. **Reuse over duplication.** Before writing a new function, search the codebase for an existing one doing the same or similar thing. Prefer extending or calling existing utilities. Use the exploration subagent if unsure where similar code lives.
3. **Appropriate design patterns.** Match the surrounding code's pattern (e.g. singleton services, strategy per variant, base class + subclass, handler registry, dependency injection). Do not introduce a new pattern unless it is clearly better and the user agrees.
4. **Minimal scope.** Only change what the ticket requires. No drive-by refactors, no extra error handling for impossible cases, no new abstractions for one-time operations.
5. **Comments**: minimal. Only when the *why* is non-obvious (hidden constraint, workaround, subtle invariant). No "added for PROJ-XXXXX" tags.
6. **Type annotations / signatures** on new/changed function signatures where the repo's language and style supports them. Match existing style.
7. **Ask, don't assume.** If the ticket is ambiguous, if there are multiple plausible interpretations, or if you cannot find the area of code referenced, ask the user before guessing.

## Confirmation Gate Summary

**Bugs:**

| Gate | After Stage | Question to ask | Do not proceed without |
|------|-------------|-----------------|------------------------|
| 1    | 4 — Roadmap | "Which approach?" | Approach chosen |
| 2    | 5 — File outline | "Does this list look right?" | "Yes" / amendments |
| 3    | 6 — Code changes | "Approve these changes?" | "Yes" / amendments |
| 4    | 8 — JIRA summary comment | "Post this comment to the ticket?" | "Yes" / edits / skip |
| 5    | 9 — Memory-bank / context-docs reconciliation (only if a trigger is hit) | "Update the memory bank with this?" | "Yes" / edits / skip |

**Stories:**

| Gate | After Stage | Question to ask | Do not proceed without |
|------|-------------|-----------------|------------------------|
| S1   | A — Design doc | "Does the design match your understanding?" | Design approved |
| 2    | 5 — File outline | "Does this list look right?" | "Yes" / amendments |
| 3    | 6 — Code changes | "Approve these changes?" | "Yes" / amendments |
| 4    | 8 — JIRA summary comment | "Post this comment to the ticket?" | "Yes" / edits / skip |
| 5    | 9 — Memory-bank / context-docs reconciliation (Stories should evaluate this; most will hit a trigger) | "Update the memory bank with this?" | "Yes" / edits / skip |

## Anti-patterns to Avoid

- Skipping Stage 3 and jumping straight to a fix because the ticket "looks obvious"
- Presenting code changes before the user has approved the roadmap
- Bundling multiple unrelated fixes into one ticket's changes
- Inventing a new utility module when an existing one in the repo already covers it
- Adding type hints / docstrings / formatting changes to lines you didn't otherwise modify
- Running tests automatically without the user asking (long suites; let the user decide)
- Proposing memory-bank / context-doc updates for ticket-specific details, bug fixes that don't change architecture, or anything that doesn't clearly clear the Stage 9 trigger bar
- Editing context-doc files without explicit Gate 5 approval
