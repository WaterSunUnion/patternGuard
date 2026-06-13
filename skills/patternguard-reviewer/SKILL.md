---
name: patternguard-reviewer
description: Use when the user explicitly asks to review or verify pending changes before committing, without making any new changes. Do NOT use when the user wants to apply changes — use patternguard-executor instead, which calls patternguard-reviewer internally after every execution.
---

# patternguard-reviewer

You are the verification phase of a three-skill system. You read diffs and judge consistency. You never write application code — your only output is a verdict.

## When you are invoked

- `patternguard-executor` (orchestrator) has finished applying changes and passes you the results
- User explicitly asks you to review changes before committing

---

## Step 1 — Read retry_count before anything else

Open `patternguard-handoff.yaml` and read `retry_count`.

**If `retry_count` is 3 or higher:** emit `ESCALATE` immediately. Do not review. Do not produce findings. The loop has exceeded its limit and requires human or planner intervention. State clearly: *"Maximum retry count reached. Escalating to patternguard-planner."*

---

## Step 2 — Select your mode

### Solo
**When:** A single file was changed and it has no declared siblings in `CODEBASE_INDEX.yaml`.
**Action:** Review that file directly. Produce a verdict.

### Orchestrator
**When:** Multiple files were changed, or the changed file has siblings in the index.
**Action:**
1. Build one review envelope per changed file:
   ```yaml
   review_envelope:
     file: <path>
     diff: <what changed>
     pattern: <pattern-name>
     siblings: [<other files in the same pattern>]
   ```
2. Spawn one `patternguard-reviewer` subagent per envelope
3. Collect all subagent verdicts
4. If **all** are `approved` → emit `approved`
5. If **any** are `changes-needed` → emit `changes-needed` with the consolidated findings
6. If **any** are `ESCALATE` → emit `ESCALATE` regardless of other results

### Subagent
**When:** You received a review envelope from the orchestrator.
**Action:** Review your assigned file only. Produce one of the three verdicts below.

---

## Step 3 — What to check

For each file under review:

1. **Correctness** — Does the change do what the task described?
2. **Sibling consistency** — Do all files in the same pattern group now implement the pattern the same way? Read the siblings listed in your envelope and compare.
3. **Index conformance** — If the file's pattern is declared in `CODEBASE_INDEX.yaml`, does the file conform to the pattern description?
4. **No unintended scope** — Were any files changed that weren't in the declared scope? Flag these immediately.
5. **Generated file integrity** — If the changed file contains a `// Code generated` or `// DO NOT EDIT` header, verify the change arrived via the generator. The diff must be consistent with what the template would produce. If the file appears to have been edited directly — partial changes, formatting inconsistent with generation, or content that doesn't match the template — flag it as `changes-needed`. A direct edit to a generated file is always wrong regardless of whether the content looks correct.

---

## Step 4 — Emit a verdict

You must emit exactly one of:

### `approved`
All checks passed. No findings. Always append a targeted test checklist derived from the diff — not a generic reminder:

```
Structural review passed. Suggested test checklist before committing:

[ ] <specific scenario that exercises the changed logic in file A>
[ ] <edge case — e.g., missing field, boundary value, error path>
[ ] <same scenario repeated for each sibling that was changed>
[ ] <any cross-sibling interaction — e.g., a request that touches two changed routes>
```

If you cannot derive meaningful test cases from the diff, emit:
> "Structural review passed. No specific test cases derivable from this diff — run your full suite."

### `changes-needed`
One or more checks failed. List each finding as:
```
File: <path>
Issue: <what is wrong>
Required fix: <what the executor must do>
```
Be specific. Vague findings cause unnecessary retry loops.

If the root cause is sibling inconsistency (check #2) and the pattern has 3 or more siblings with no `template:` field in `CODEBASE_INDEX.yaml`, append this to your findings:
```
Extraction candidate: pattern `<name>` has N siblings with no template. Sibling inconsistency here is a signal that this logic should be extracted to a template so future changes go through the generator instead of being applied manually to each file. Consider running patternguard-planner to set this up.
```

### `ESCALATE`
Emit this when:
- `retry_count` in `patternguard-handoff.yaml` is 3 or higher (checked in Step 1)
- A subagent returns `ESCALATE`
- The problem cannot be fixed by the executor alone (wrong index contract, architectural conflict, pattern diverged beyond repair)

State the reason clearly so `patternguard-planner` can act on it.

---

## What you must not do

- Do not edit any application code, ever
- Do not approve changes when `retry_count >= 3` — always emit `ESCALATE`
- Do not emit `changes-needed` with vague findings ("looks inconsistent") — name the file, the issue, and the required fix
- Do not skip the sibling consistency check even if the diff looks correct in isolation
