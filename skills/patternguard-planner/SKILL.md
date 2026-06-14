---
name: patternguard-planner
description: Use when the user wants to inspect or update the codebase index without making any code changes, or when explicitly asked to plan a change before executing it. Do NOT use when the user wants to apply changes — use patternguard-executor instead, which calls patternguard-planner internally whenever planning is needed.
---

# patternguard-planner

You are the planning phase of a three-skill system for agent-operated codebases. Your job is to map every file a change touches and produce the artifacts that make safe, scoped execution possible.

## When you are invoked

- patternguard-executor calls you when no `.patternguard/CODEBASE_INDEX.yaml` exists and execution is about to start
- patternguard-executor calls you when `retry_count` reaches 3 or when patternguard-reviewer returns `ESCALATE`
- patternguard-executor calls you when index drift is detected and the index needs updating
- User calls you directly to inspect or update the index without executing any changes

In all cases, surface your plan to the user for visibility before returning control to patternguard-executor. Do not wait for confirmation unless the user explicitly asked for a planning-only stop or told you to pause before execution.

## Workflow lock

Before writing PatternGuard state, respect `.patternguard/lock`.

- If patternguard-executor invoked you, the executor owns the top-level workflow lock. Do not create or clear another lock; keep operating under that workflow.
- If the user invoked you directly, ensure `.patternguard/` exists, then create `.patternguard/lock` only if it does not already exist. Use an atomic create operation when available; never overwrite an existing lock.
- A direct planner lock records at least: `owner`, `task`, `started_at`, and `state: planning`.
- If `.patternguard/lock` already exists for another workflow, fail fast. Report the lock owner, task, and start time from the file if present. Do not wait, retry, or write PatternGuard state.
- If the lock appears stale, report that it may be stale and ask the user before clearing it. Do not clear it automatically.
- If you created the lock for a direct planner run, remove only your own lock when planning is complete or blocked and you are returning control to the user.

## Step 1 — Read the current state

1. Ensure `.patternguard/` exists. If it does not exist, create it before reading or writing PatternGuard state. Ensure `.patternguard/changes/` exists before reading or writing change-impact files.
2. Check for legacy root state:
   - If root `CODEBASE_INDEX.yaml` exists and `.patternguard/CODEBASE_INDEX.yaml` does not, migrate the root index into `.patternguard/CODEBASE_INDEX.yaml`.
   - If root `IMPACT_MAP.yaml` exists, split it into `.patternguard/changes/index.yaml` plus one `.patternguard/changes/<change-type>.yaml` file per reusable change type. Preserve each change's `touch`, `test`, `regenerate-from`, `user-visible-impact`, `risk-if-wrong`, and `rationale` fields in the detailed file.
   - If both legacy root files and `.patternguard/` files exist, prefer `.patternguard/` as authoritative and report that the root files are legacy. Do not write new PatternGuard state to root files.
   - Do not delete legacy root files unless the user explicitly asks; leave them as stale migration artifacts after the `.patternguard/` files are written.
3. Check if `.patternguard/CODEBASE_INDEX.yaml` exists. If it does, read it. You are upgrading an existing index, not starting from scratch.
4. Scan only application files. Always skip:
   - Directories: `node_modules/`, `.git/`, `dist/`, `build/`, `out/`, `.next/`, `.nuxt/`, `__pycache__/`, `coverage/`, `.cache/`
   - Files: `*.min.js`, `*.min.css`, `*.map`, `*.lock`, `*.log`, `*.snap`
   If the repo is very large (1000+ non-skipped files), focus first on the directories the user mentioned and expand outward only as needed.
5. Read the files relevant to the user's request.
6. Identify every file that shares logic with the change target — siblings are files that implement the same pattern, even if the user didn't mention them.

## Step 2 — Classify the logic

For each group of related files:

- **Route-specific** — logic that belongs to one file only, no siblings
- **Cross-cutting** — logic that must be consistent across multiple files
- **Pure utility** — stateless, no business rules (formatters, sanitizers, parsers). Allowed as shared runtime. Must be flagged `utility: pure` in the index.
- **Business utility** — contains any domain logic or business rule. Must NOT be a shared import. Treat it as cross-cutting: duplicate it into each caller and declare those copies in the index.

## Step 3 — Decide how cross-cutting logic is managed

**Default: duplicate.** When you declare a new cross-cutting pattern, the files are duplicated copies. No template, no shared function. The index is the only centralization — it records which files implement the pattern so the executor knows the blast radius before spawning subagents.

### When to mark a template candidate

Only mark a pattern as a template candidate when at least **2 of these 3 conditions** are true:

1. **Divergence** — the same logic was changed in one file but not its siblings, causing a bug or requiring a manual audit to catch
2. **Mechanical repetition** — the last cross-cutting change required sending identical instructions to every sibling with no meaningful variation per file
3. **Stable blast radius** — the same set of files appears in the `touch` list every time this kind of change happens

If 0 or 1 conditions are true, keep duplicating. Do not mark the pattern as a template candidate.

### How to promote a candidate to template mode

Template mode requires a working project-specific generator. Before adding a `template:` field to `.patternguard/CODEBASE_INDEX.yaml`, verify that a generator entrypoint and command already exist, such as `generator/main.go` plus `go run ./generator`, or the project's equivalent.

- If a generator exists, promote the pattern to template mode: add `template:`, add `generator.command`, and require generated outputs to carry the generated header shown below.
- If no generator exists, keep the pattern in duplicated mode. Set `template_candidate: true` and `generator_status: missing`, and include a setup recommendation in the plan. Do not create a template path, do not mark files as generated, and do not tell the executor to regenerate.
- Do not create the generator yourself. Generator setup is project-specific and must be done outside PatternGuard before template-backed execution is available.

```
Is the logic identical across all files (just inlined at generation time)?
├── YES, and a generator exists?
│   ├── YES → Add template mode with generator.command
│         Parameters handle per-file variation (e.g. MaxAmount, HandlerName)
│         Run the generator to produce flat copies in each route file
│         Generated files get a DO NOT EDIT header — agents never touch them directly
│   └── NO  → Keep duplicated mode; mark template_candidate: true and generator_status: missing
│
└── NO → Logic varies by file
         ├── Variation is data-driven (only config values differ)?
         │   → Same as above: template mode only if a generator exists; otherwise candidate only
         │
         └── Variation is structural (different control flow per file)?
             → Extract to a shared function (last resort only)
               Declare the shared function file as a named pattern in the index
               along with all files that call it — so the executor knows
               a change there affects all callers
```

The template path is always preferred once a generator exists. A shared runtime function is only warranted when template parameters cannot express the structural differences.

### What a template produces

Each generated file must have this header so any agent reading it knows not to edit it directly:

```
// Code generated by generator/main.go. DO NOT EDIT.
// Patterns: <pattern-name>
// Index: .patternguard/CODEBASE_INDEX.yaml
// To change cross-cutting logic, edit generator/templates/<name>.tmpl
// then run the generator. Changes to this file will be overwritten.
```

For duplicated (non-generated) files that implement a cross-cutting pattern, add:

```
// Patterns: <pattern-name>
// Index: .patternguard/CODEBASE_INDEX.yaml
```

These comments let the executor scan for drift: if a file declares a pattern but is absent from the index, the executor pauses code edits and re-runs planning before touching anything.

Pattern headers are required for every file listed in `.patternguard/CODEBASE_INDEX.yaml`.

- If a file is generated or template-backed, the generated output must include the full generated header shown above.
- If a file is duplicated and non-generated, it must include the two-line `Patterns` / `Index` header shown above.
- If you find an indexed file without the required header, do not ignore it. Add a `header_repairs` entry to `.patternguard/patternguard-handoff.yaml` so patternguard-executor repairs the header before or during execution.
- If you add a file to `.patternguard/CODEBASE_INDEX.yaml`, the corresponding header repair is required unless the file already contains the correct header.

## Step 4 — Write output files

Before writing any output, confirm again that `.patternguard/` and `.patternguard/changes/` exist. Never write `CODEBASE_INDEX.yaml`, `IMPACT_MAP.yaml`, or `patternguard-handoff.yaml` at the repo root.

### .patternguard/CODEBASE_INDEX.yaml

```yaml
version: 1
patterns:
  - name: <pattern-name>
    description: "<what this pattern enforces — sent as context to each subagent>"
    mode: duplicated                            # duplicated | template
    template_candidate: true                    # only when extraction conditions fired but no generator exists
    generator_status: missing                   # only for template candidates blocked on generator setup
    template: generator/templates/<name>.tmpl   # only in template mode
    generator:
      command: go run ./generator               # only in template mode
    utility: pure                               # only for pure stateless utilities
    files:
      - <path-to-file>
      - <path-to-sibling>
```

Presence of both `template:` and `generator.command` means the files are generated — the executor will edit the template and run the generator instead of editing these files directly. Absence of either field means the files remain duplicated copies that subagents edit individually, even when `template_candidate: true` is present.

Only include files you have confirmed exist or will be created. Do not speculate.

### .patternguard/changes/index.yaml

Pre-declare a searchable index of known change-impact categories. This lets the executor choose the smallest relevant impact file before reading detailed blast-radius data. This must be the only index file inside `.patternguard/changes/`; all other files in that folder are detailed impact files.

```yaml
version: 1
changes:
  - name: <change-type>          # e.g. fraud-threshold, add-currency, checkout-order-flow
    impact: "<short concrete description of the feature, workflow, API behavior, UI state, or background job affected>"
    filepath: .patternguard/changes/<change-type>.yaml
```

`impact` must be concrete enough for retrieval. Prefer feature names, routes, workflows, API responses, data writes, permissions, payment/auth behavior, or background jobs over generic labels like "business logic".

Do not add a new indexed change for every ordinary user edit. Add or update a change entry only when there is a reusable known impact category, a new blast radius, or a meaningful change to an existing impact category.

If you migrated a legacy root `IMPACT_MAP.yaml`, each old `changes[]` item becomes one detailed impact file. The new `changes/index.yaml` contains only `name`, `impact`, and `filepath` for those files.

### .patternguard/changes/<change-type>.yaml

Each detailed impact file declares the blast radius for one known type of change. This lets the executor know exactly which files to touch before reading application code.

```yaml
version: 1
name: <change-type>
impact: "<same short concrete impact description used in changes/index.yaml>"
entries:
  - pattern: <pattern-name>
    touch:
      - <path>
    test:
      - <path-to-test-file>
    regenerate-from: generator/templates/<name>.tmpl   # only if template mode has template + generator.command; null otherwise
    user-visible-impact:
      - "<feature, route, workflow, API behavior, UI state, or background job this file can affect>"
    risk-if-wrong:
      - "<what user-facing or operational behavior could break if this file is changed incorrectly>"
    rationale: "<why these files are affected>"
```

A change type with a single file in `touch` tells the executor that subagent has full authority over that file and nothing else is affected.

`user-visible-impact` and `risk-if-wrong` must be concrete enough for a user to understand the blast radius before execution continues. Prefer feature names, routes, workflows, API responses, data writes, permissions, payment/auth behavior, or background jobs over generic labels like "business logic".

### .patternguard/patternguard-handoff.yaml

```yaml
task: "<what needs to change, in one sentence>"
template: generator/templates/<name>.tmpl   # include only if template mode has template + generator.command
retry_count: 0
affected_patterns:
  - name: <pattern-name>
    files:
      - <path>
header_repairs:
  - file: <path>
    pattern: <pattern-name>
    generated: false
    required-header:
      - "// Patterns: <pattern-name>"
      - "// Index: .patternguard/CODEBASE_INDEX.yaml"
```

Always initialize `retry_count: 0`. The executor and reviewer increment this — never reset it yourself unless responding to a fresh ESCALATE signal that represents a genuinely new planning cycle.

Omit `header_repairs` only when every indexed file already contains the correct header.

## What you must not do

- Do not write application code
- Do not write PatternGuard state while `.patternguard/lock` is owned by another workflow — fail fast instead
- Do not clear a `.patternguard/lock` you did not create unless the user explicitly confirms it is stale
- Do not make assumptions about files you have not read
- Do not create a template, template candidate, or shared function for a pattern with fewer than 3 confirmed instances
- Do not mark a template candidate unless at least 2 of the 3 conditions above have fired
- Do not add `template:` unless a working generator command already exists
- Do not clear `retry_count` unless this is a confirmed new planning cycle
