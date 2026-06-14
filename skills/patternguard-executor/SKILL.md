---
name: patternguard-executor
description: Use when the user wants to make a code change, apply a fix, implement a feature, or modify files in a codebase that may have cross-cutting patterns. This is the only skill the user needs to invoke — it handles planning, execution, and review internally. Do NOT use for read-only tasks like explaining code or answering questions.
---

# patternguard-executor

You are the single entry point for applying changes to an agent-operated codebase. You orchestrate the full cycle: plan when needed, execute with bounded scope, review for consistency. The user always invokes you — you invoke patternguard-planner and patternguard-reviewer internally.

## When you are invoked

- User asks you to execute a change, apply a fix, or implement a feature
- You receive a task envelope from an orchestrator (you are a subagent)

---

## Workflow lock

If you are a subagent receiving a task envelope, skip this section. Subagents do not create, clear, or write `.patternguard/lock`.

If you are the top-level executor, acquire `.patternguard/lock` before invoking patternguard-planner or writing any PatternGuard state.

- Ensure `.patternguard/` exists, then create `.patternguard/lock` only if it does not already exist. Use an atomic create operation when available; never overwrite an existing lock.
- The lock records at least: `owner`, `task`, `started_at`, and `state: active`.
- If `.patternguard/lock` already exists and is not yours, fail fast. Report the lock owner, task, and start time from the file if present. Do not wait, retry, invoke patternguard-planner, or write PatternGuard state.
- If the lock appears stale, report that it may be stale and ask the user before clearing it. Do not clear it automatically.
- Keep the lock while planning, executing, reviewing, and updating `.patternguard/` state. Remove only your own lock when the top-level workflow is complete or blocked and you are returning control to the user.

---

## Step 1 — Read .patternguard/patternguard-handoff.yaml

Before reading any PatternGuard state, check whether `.patternguard/` exists.

- If `.patternguard/` does not exist, invoke patternguard-planner immediately to create it and produce the current state files. Do not create or write root `CODEBASE_INDEX.yaml`, `IMPACT_MAP.yaml`, or `patternguard-handoff.yaml`.
- If legacy root `CODEBASE_INDEX.yaml`, `IMPACT_MAP.yaml`, or `patternguard-handoff.yaml` exists, invoke patternguard-planner immediately to migrate state into `.patternguard/`. Root `IMPACT_MAP.yaml` must be split into `.patternguard/changes/index.yaml` and per-change `.patternguard/changes/<change-type>.yaml` files before execution continues.
- After patternguard-planner returns, re-enter Step 1 and read only `.patternguard/` state. Treat root PatternGuard files as legacy artifacts, not authoritative state.

If `.patternguard/patternguard-handoff.yaml` exists, read it before doing anything else. It tells you:
- What the task is
- Which files are in scope
- Whether the affected pattern is template-backed (`template:` plus `generator.command`)
- The current `retry_count`
- Any `header_repairs` the planner found for indexed files missing required PatternGuard headers

If no handoff note exists, check for `.patternguard/CODEBASE_INDEX.yaml`. If neither exists, invoke patternguard-planner before continuing (see below).

---

## Step 2 — Check index freshness and generated file guard

Before selecting a mode, verify the index is not stale and that no generated files are being targeted directly. Skip this step only if you are a subagent receiving a task envelope.

1. Read every file path listed in `.patternguard/CODEBASE_INDEX.yaml`
2. For each declared pattern, scan the same directories those files live in for files that match the same naming convention (e.g. if `routes/payments.js` is in the index, check for other `routes/*.js` files)
3. **Header scan** — grep files in those directories for `// Patterns: <pattern-name>`. Any file that declares the pattern in its header but is absent from the index is drift.
4. **Required header check** — every file listed in `.patternguard/CODEBASE_INDEX.yaml` must contain:
   - `// Patterns: <pattern-name>`
   - `// Index: .patternguard/CODEBASE_INDEX.yaml`
   If a listed non-generated file is missing one or both lines, add a header-only repair to the current execution scope. If a generated file is missing the full generated header and the pattern has both `template:` and `generator.command`, repair the template or generator output path, then regenerate. If the generator command is missing, invoke patternguard-planner to keep the pattern in duplicated mode or flag generator setup before continuing. Do not finish execution while indexed files are missing PatternGuard headers.
5. If you find files that match a known pattern but are absent from the index → **pause code edits immediately**. Report:
   > "Index drift detected: found `<path>` matching pattern `<name>` but not declared in `.patternguard/CODEBASE_INDEX.yaml`."
   Then invoke patternguard-planner to update the index before continuing.
6. **Generated file guard** — for each file the task will touch, check whether it contains a `// Code generated` or `// DO NOT EDIT` header.
   - If it does, look up the pattern in `.patternguard/CODEBASE_INDEX.yaml` to find both `template:` and `generator.command`.
   - If both exist, halt immediately and report:
     > "Target file `<path>` is generated. To change cross-cutting logic, edit the template at `<template-path>` and run the generator. Do not edit this file directly."
   - If the index does not contain both `template:` and `generator.command` for that pattern, halt immediately and invoke patternguard-planner because the generated-file metadata and generator setup are inconsistent.
   - Do not proceed until the user redirects the task to the template or the planner repairs the metadata.
7. If the index matches the filesystem and no generated files are targeted → continue to mode selection

---

## Step 3 — Select your mode

Choose exactly one mode. Do not skip this step.

### Invoke patternguard-planner
**When:** No `.patternguard/CODEBASE_INDEX.yaml` exists, index drift was detected, or an ESCALATE signal requires re-planning.

**Action:** Call patternguard-planner with the user's task as context. patternguard-planner will inspect relevant project structure and likely pattern locations, classify patterns, ensure `.patternguard/` and `.patternguard/changes/` exist, migrate any legacy root PatternGuard files, split root `IMPACT_MAP.yaml` if present, and produce `.patternguard/CODEBASE_INDEX.yaml`, `.patternguard/changes/index.yaml`, relevant `.patternguard/changes/<change-type>.yaml` files, and `.patternguard/patternguard-handoff.yaml`. patternguard-planner will surface its plan to the user for visibility before returning, without waiting for confirmation unless the user explicitly asked for a planning-only stop. Once it returns, re-enter Step 1.

### Solo
**When:** The task touches a single file AND that file has no siblings in `.patternguard/CODEBASE_INDEX.yaml`.

**Before selecting Solo, you must:**
1. Read `.patternguard/CODEBASE_INDEX.yaml`
2. Search every pattern group for the target file path
3. If the file appears in any group that contains other files → **Solo is not permitted. Use Orchestrator.**
4. If the file is not in the index, or its pattern group lists only itself → Solo is permitted

Make the change directly. No subagents. Update `.patternguard/patternguard-handoff.yaml` when done. Then pass the result to patternguard-reviewer.

If the target file is listed in `.patternguard/CODEBASE_INDEX.yaml`, ensure it contains the required `Patterns` and `Index` header before reporting done. Header repair is part of the Solo change, not optional cleanup.

### Orchestrator
**When:** The task touches multiple files, OR the target file has siblings in the index, OR you were directed to use this mode.

**Action:**
1. Read `.patternguard/CODEBASE_INDEX.yaml` and `.patternguard/patternguard-handoff.yaml`
2. Apply or schedule all `header_repairs` from the handoff note. Treat header repairs as part of the declared scope and include them in the scope summary. Do not proceed to review while a scoped indexed file is missing its required PatternGuard header.
3. For each affected pattern, check whether `.patternguard/CODEBASE_INDEX.yaml` declares both `template:` and `generator.command`. Classify the execution path before execution:
   - **Template-backed pattern:** edit the template once, run the generator command, and review the generated output files. Do not spawn edit subagents.
   - **Template candidate only:** if `template_candidate: true` is present but `generator.command` is missing, keep duplicated execution and mention that generator setup is still required before template mode is available.
   - **Duplicated pattern:** spawn one edit subagent per scoped file after surfacing the scope summary.
4. Count the total files in scope across all affected patterns. Read `.patternguard/changes/index.yaml` if it exists, select the most relevant change entry by `name`, `impact`, and affected pattern, then read the referenced `.patternguard/changes/<change-type>.yaml` file. Extract the matching rationale, test targets, and likely user-visible behavior for each scoped path. If the changes index or detailed impact file is missing, incomplete, or does not clearly match the task, infer the affected behavior from `.patternguard/CODEBASE_INDEX.yaml`, `.patternguard/patternguard-handoff.yaml`, and the surrounding code you have read; label any uncertain inference as `inferred`.
   Before execution, surface the count, every scoped path, the execution path, and the likely feature/behavior impact to the user:
   > "Scope: **N files** across **M pattern(s)**:
   >
   > - `<pattern-name>`:
   >   - execution: `<edit template at template/path and regenerate>` OR `<spawn one subagent per file>`
   >   - `<path/filename>` — affects `<feature or user-visible behavior>`; risk: `<what could break if this change is wrong>`; tests: `<test path or manual scenario>`
   >   - `<path/filename>` — affects `<feature or user-visible behavior>`; risk: `<what could break if this change is wrong>`; tests: `<test path or manual scenario>`
   >
   > Why these files are included: `<rationale from .patternguard/changes/<change-type>.yaml or inferred rationale>`
   >
   > Continuing without a confirmation gate."
   Do not wait for confirmation. Continue unless the user has explicitly asked to pause, stop, or only plan.
5. Follow the execution path selected above for each affected pattern:

   **If a template exists:**
   - Edit `generator/templates/<name>.tmpl` with the required change
   - Run the generator (e.g. `go generate ./...` or the project-specific equivalent)
   - Read each generated output file to verify the change appears correctly and the full generated PatternGuard header is present
   - Pass all generated files directly to patternguard-reviewer — no edit subagents are spawned, the edit already happened via the generator

   **If no template exists (duplicated files):**
   - Build one task envelope per file:
     ```yaml
     envelope:
       file: <path>
       task: <what to change in this file specifically>
       pattern: <pattern-name>
       description: <pattern description from .patternguard/CODEBASE_INDEX.yaml>
       siblings: [<other files in the same pattern>]
       required_header:
         - "// Patterns: <pattern-name>"
         - "// Index: .patternguard/CODEBASE_INDEX.yaml"
       retry_count: <current retry_count from handoff note>
     ```
   - Spawn one patternguard-executor subagent per envelope. Each subagent must add or repair `required_header` in its assigned file before reporting done.
   - Collect results

6. Pass all results to patternguard-reviewer
7. **If reviewer returns `changes-needed`:**
   - Read the current `retry_count` from `.patternguard/patternguard-handoff.yaml`
   - Increment it by 1
   - Write the updated value back
   - **If `retry_count` is now 3 or higher → invoke patternguard-planner** with the escalation reason. patternguard-planner will re-evaluate the index and surface an updated plan to the user. Once patternguard-planner returns a new `.patternguard/patternguard-handoff.yaml`, reset `retry_count: 0` and re-enter execution.
   - Otherwise → apply the reviewer's feedback. If the pattern has a template, re-edit the template and regenerate. If duplicated, re-run the affected subagents.
8. **If reviewer returns `approved`:**
   - Update `.patternguard/CODEBASE_INDEX.yaml`, `.patternguard/changes/index.yaml`, and the relevant `.patternguard/changes/<change-type>.yaml` file if the change added or modified a pattern, introduced a reusable impact category, changed a blast radius, or changed the impact details for an existing category
   - Check whether extraction conditions have been met (see below)
   - Report done
9. **If reviewer returns `ESCALATE`:** Invoke patternguard-planner with the escalation reason. Same flow as step 7 above.

### Subagent
**When:** You received a task envelope from an orchestrator.
**Action:** Read the envelope. Make exactly the change described for your assigned file. If the envelope contains `required_header`, add or repair that header in your assigned file before reporting done. Do not touch sibling files — those belong to other subagents. Report your result back (done / blocked / ESCALATE).

---

## After successful execution — check extraction conditions

After a successful change, look at `.patternguard/changes/index.yaml` and the relevant `.patternguard/changes/<change-type>.yaml` file for the affected pattern. Check whether at least 2 of these are true:

1. The same set of files appears in `touch` for 3 or more different change types (stable blast radius)
2. The instructions sent to each subagent were identical with no meaningful variation per file (mechanical repetition)
3. `.patternguard/CODEBASE_INDEX.yaml` shows this pattern exists across 3+ sibling files with no template yet (divergence risk)

If 2 or more are true, surface this to the user before closing:
> "Pattern `<name>` has been changed mechanically across the same N files multiple times. Consider running patternguard-planner to mark it as a template candidate, or to promote it to template mode if a generator command already exists."

Do not extract automatically. If no generator exists, keep using duplicated execution and surface the generator setup gap only. Let the user decide.

---

## What you must not do

- Do not choose Solo if the target file has siblings in the index
- Do not read or write root `CODEBASE_INDEX.yaml`, `IMPACT_MAP.yaml`, or `patternguard-handoff.yaml` as authoritative state — invoke patternguard-planner to migrate them into `.patternguard/`
- Do not continue looping after `retry_count` reaches 3 — invoke patternguard-planner instead
- Do not start a top-level workflow while `.patternguard/lock` is owned by another workflow — fail fast instead
- Do not clear a `.patternguard/lock` you did not create unless the user explicitly confirms it is stale
- Do not write to `.patternguard/CODEBASE_INDEX.yaml`, `.patternguard/changes/index.yaml`, or `.patternguard/changes/<change-type>.yaml` as a subagent — only the orchestrator writes PatternGuard index files
- Do not expand your scope beyond what the handoff note or envelope declares
- Do not skip the index freshness check before selecting a mode
- Do not spawn subagents without first surfacing the file count, scoped paths, execution path, likely impact, risk, and tests
- Do not report done for an indexed file unless it contains the required PatternGuard header
- Do not edit a file that contains a `// Code generated` or `// DO NOT EDIT` header — redirect to the template
- Do not spawn edit subagents when the pattern has both `template:` and `generator.command` — edit the template, run the generator, then go straight to review
