# PatternGuard

> Guard repeated code patterns by mapping each change's blast radius, applying edits within declared bounds, and verifying sibling consistency before commit.

---

![PatternGuard workflow overview](docs/images/patternguard-overview.png)

The workflow has one entry point: `patternguard-executor`. It reads the index, chooses the correct execution path, bounds each edit, and hands the result to an independent reviewer before anything is considered done.

PatternGuard state lives under `.patternguard/`. On first run, or when legacy root `CODEBASE_INDEX.yaml` / `IMPACT_MAP.yaml` files are found, the planner creates `.patternguard/` and migrates the state there. Legacy `IMPACT_MAP.yaml` entries are split into `.patternguard/changes/index.yaml` plus focused files in `.patternguard/changes/`.

---

## Why PatternGuard?

AI agents are strongest when the task fits inside one clear context window. They get risky when a change has to propagate across sibling files, generated files, templates, tests, and index contracts. The hard part is not the edit itself; it is knowing the full blast radius before the first line changes.

PatternGuard turns that hidden blast radius into an explicit workflow:

- `patternguard-planner` maps the files and patterns that define the change.
- `patternguard-executor` chooses the right path: solo edit, bounded subagents, or template regeneration.
- `patternguard-reviewer` checks correctness, sibling consistency, index conformance, and generated-file integrity before the work is considered complete.

**The problem without pattern awareness:**

- An agent edits `routes/payments.js` to add a validation field.
- The same validation pattern also lives in `routes/refunds.js`, `routes/disputes.js`, and four other sibling routes.
- Those siblings are easy to miss because they are outside the agent's immediate context.
- The change looks finished, narrow tests can pass, and the bug ships in the first unedited flow a user hits.

That is the scope problem: **the codebase knows the change is cross-cutting, but the agent only sees the file it was asked to edit.**

**What PatternGuard gives you:**

| Without | With |
|---|---|
| Agent infers scope from nearby files | `.patternguard/CODEBASE_INDEX.yaml` declares affected patterns and files before execution |
| One agent edits everything from a bloated context | Executor keeps work bounded: solo for one file, subagents for siblings, generator path for templates |
| Generated files can be edited directly by mistake | Generated-file guard redirects changes to the source template |
| Review happens in the same context that made the change | Reviewer never edits; it can only approve, request fixes, or escalate |
| Pattern drift is discovered in production | Reviewer compares sibling implementations before commit |
| Shared abstractions are introduced too early | Extraction requires proven repetition, divergence, and stable blast radius signals |

**The concrete payoff:**

1. **Blast radius is declared up front** — `.patternguard/CODEBASE_INDEX.yaml`, `.patternguard/changes/index.yaml`, and focused files in `.patternguard/changes/` record which files share a pattern, which features may be affected, which tests matter, what could break, and when a template must be regenerated.
2. **Execution stays bounded** — one-file tasks use Solo mode; sibling changes get one envelope per file; template-backed patterns go through the generator only when a generator command exists.
3. **Pattern drift is treated as a stop sign** — if headers and the index disagree, code edits pause while planning updates the map, then execution continues with the repaired scope.
4. **Generated files stay generated** — agents are redirected away from `DO NOT EDIT` files and toward the template that owns them.
5. **Review is structurally independent** — `patternguard-reviewer` checks the diff against siblings and the index without touching code, then returns `approved`, `changes-needed`, or `ESCALATE`.
6. **Abstraction is earned, not guessed** — templates are suggested only after repetition, divergence, and stable blast radius prove the pattern needs a single source.

---

## How it works

You only ever invoke one skill. patternguard-executor orchestrates everything else internally.

```
User: /patternguard-executor "change fraud threshold from 0.85 to 0.80"
        │
        ├─ No index yet? → calls patternguard-planner internally
        │    patternguard-planner inspects relevant structure and likely pattern locations,
        │    produces or updates .patternguard/CODEBASE_INDEX.yaml + change-impact files + handoff note
        │    surfaces plan to user for visibility → returns to executor
        │
        ├─ Index exists, pattern has template + generator.command?
        │    → edits generator/templates/<name>.tmpl
        │    → runs the generator
        │    → verifies generated files updated
        │    → passes to patternguard-reviewer
        │
        ├─ Index exists, pattern is duplicated files?
        │    → spawns one patternguard-executor subagent per file
        │    → each subagent edits exactly its assigned file
        │    → passes results to patternguard-reviewer
        │
        └─ patternguard-reviewer checks each file
             approved       → updates index → done
             changes-needed → loops back (with extraction hint if siblings diverged)
             retry ≥ 3      → calls patternguard-planner to re-plan → re-enters execution
```

### The three mechanics

![PatternGuard core mechanics](docs/images/patternguard-mechanics.png)

**1. Index as blast radius map**

`.patternguard/CODEBASE_INDEX.yaml` is the single record of which files share logic. Before changing code, the executor reads it, checks whether the affected pattern is template-backed, a template candidate, or duplicated, counts the scope, and surfaces the scope summary for visibility. The summary includes the scoped paths, execution path, feature or behavior impact, risk if wrong, and relevant tests. Subagents never discover siblings on their own — the index tells them.

Files that implement a cross-cutting pattern carry a header comment:
```
// Patterns: payment-validation
// Index: .patternguard/CODEBASE_INDEX.yaml
```
The executor scans these headers as a second drift check. If a file declares a pattern but is absent from the index, the executor pauses code edits and re-runs planning to repair the map before continuing.

Headers are required, not optional. If the planner adds a file to `.patternguard/CODEBASE_INDEX.yaml`, it records any missing header as a `header_repairs` item in the handoff note. The executor repairs those headers during execution, subagents receive the required header in their task envelope, and the reviewer rejects indexed files that are missing or misnaming the `Patterns` / `Index` lines.

**2. Template layer**

Once cross-cutting logic is proven to need a single source of truth, PatternGuard can mark it as a template candidate. It only moves into template-backed execution when the project already has a working generator command. Until then, the pattern stays in duplicated mode and executor still updates each indexed sibling directly.

When template mode is available, the generator renders the template into flat copies, one per route. What agents actually touch are those flat copies.

Generated files carry:
```
// Code generated by generator/main.go. DO NOT EDIT.
// Patterns: payment-validation
// Index: .patternguard/CODEBASE_INDEX.yaml
// To change this logic, edit generator/templates/payment_handler.tmpl
// then run the generator. Changes to this file will be overwritten.
```

Any agent that tries to edit a generated file directly is halted by the executor's guard and redirected to the template. Template-backed patterns are evaluated before execution, so the executor can report that it will edit one template and regenerate outputs instead of incorrectly saying it will spawn one edit subagent per generated file.

**Scope summary includes feature risk**

When a change affects sibling files, PatternGuard does not rely on filenames alone. The executor reads `.patternguard/changes/index.yaml`, opens the matching detailed impact file, and shows what each file can affect before continuing:

```
Scope: 3 files across 1 pattern:

- auth-timeout-response:
  - execution: edit template at generator/templates/auth-timeout-response.tmpl and regenerate
  - routes/checkout.js — affects checkout expired-session response; risk: cart recovery can redirect incorrectly; tests: checkout expired-token scenario
  - routes/billing.js — affects billing expired-session response; risk: payment retry can surface the wrong auth error; tests: billing expired-token scenario
  - routes/profile.js — affects profile expired-session response; risk: account settings can show an inconsistent session error; tests: profile expired-token scenario

Why these files are included: all three routes implement the indexed expired-token response pattern.

Continuing without a confirmation gate.
```

If the changes index or detailed impact file is missing or incomplete, the executor infers the feature impact from the codebase index, handoff note, and surrounding code, and labels uncertain items as inferred.

**3. Earned abstraction**

Duplication is the default. Templates are only created when the cost of duplication is proven, not speculated. The executor watches for these signals after each successful change:

- Same files appear in the blast radius for 3+ different change types
- Instructions to each subagent were identical with no meaningful variation
- 3+ sibling files implement the same pattern with no template yet

When 2 of 3 are true, the executor surfaces a suggestion to mark the pattern as a template candidate. If sibling inconsistency keeps causing review failures, the reviewer itself flags the pattern as a candidate. Promotion to template-backed execution happens only after a generator command exists. Extraction never happens automatically — the user decides.

---

## Skills

| Skill | Invoked by | What it does |
|---|---|---|
| `patternguard-executor` | User | Entry point for all changes. Orchestrates planning, execution, and review internally. |
| `patternguard-planner` | patternguard-executor (or user directly) | Inspects relevant project structure, classifies patterns, decides when to extract to a template, produces `.patternguard/CODEBASE_INDEX.yaml` + change-impact files + handoff note. |
| `patternguard-reviewer` | patternguard-executor | Reads diffs only, never edits. Checks correctness, sibling consistency, and generated file integrity. Emits `approved`, `changes-needed`, or `ESCALATE`. |

Call `/patternguard-planner` directly only when you want to inspect or update the index without executing any changes.

---

## Files produced in your repo

| File | Purpose |
|---|---|
| `.patternguard/CODEBASE_INDEX.yaml` | Declares which files share each pattern, whether it is duplicated, a template candidate, or generator-backed template mode |
| `.patternguard/changes/index.yaml` | Indexes reusable change-impact categories by `name`, `impact`, and detailed impact file path |
| `.patternguard/changes/<change-type>.yaml` | Pre-declares blast radius for one change type — which files to touch, affected user-visible behavior, risk if wrong, tests to run, and which template to regenerate from |
| `.patternguard/patternguard-handoff.yaml` | Task context passed between skills — what to change, which files are in scope, retry count |
| `.patternguard/lock` | Fail-fast workflow lock for top-level planner/executor runs. Subagents do not create or clear it. |
| `generator/templates/*.tmpl` | Single source of truth for cross-cutting logic. Edit these, never the generated files. |

---

## Install — Claude Code

### Local dev / test without installing

```bash
# Skills reload immediately on file save — no restart needed for SKILL.md changes
claude --plugin-dir ./patternguard
```

> Changes to hooks, agents, or `.mcp.json` require `/reload-plugins` or a session restart.

### Manual — repo scope

```bash
mkdir -p ./plugins
cp -R patternguard ./plugins/patternguard
# Then add to your project's plugin config or drop in .claude/plugins/
```

### Manual — personal scope (all repos)

```bash
mkdir -p ~/.claude/plugins
cp -R patternguard ~/.claude/plugins/patternguard
```

---

## Install — Codex

### Repo scope

```bash
# Step 1: copy plugin into your repo
mkdir -p ./plugins
cp -R patternguard ./plugins/patternguard

# Step 2: create the repo marketplace file
mkdir -p .agents/plugins
cat > .agents/plugins/marketplace.json << 'EOF'
{
  "name": "local-repo",
  "interface": { "displayName": "Local Plugins" },
  "plugins": [
    {
      "name": "patternguard",
      "source": { "source": "local", "path": "./plugins/patternguard" },
      "policy": { "installation": "AVAILABLE", "authentication": "ON_INSTALL" },
      "category": "Productivity"
    }
  ]
}
EOF

# Step 3: restart Codex → Plugin Directory → "Local Plugins" → Install
```

### Personal scope (all repos)

```bash
# Step 1: copy to personal plugins folder
mkdir -p ~/.codex/plugins
cp -R patternguard ~/.codex/plugins/patternguard

# Step 2: create the personal marketplace file
mkdir -p ~/.agents/plugins
cat > ~/.agents/plugins/marketplace.json << 'EOF'
{
  "name": "personal-plugins",
  "interface": { "displayName": "My Plugins" },
  "plugins": [
    {
      "name": "patternguard",
      "source": { "source": "local", "path": "./.codex/plugins/patternguard" },
      "policy": { "installation": "AVAILABLE", "authentication": "ON_INSTALL" },
      "category": "Productivity"
    }
  ]
}
EOF

# Step 3: restart Codex
```

### Local dev / test without installing

```bash
codex --plugin-dir ./patternguard
```

---

## Known Limits

**Drift detection is bounded.**
The executor performs a preflight check before editing. It compares `.patternguard/CODEBASE_INDEX.yaml` with files in indexed or previously scanned areas, and scans PatternGuard headers for known patterns. If it finds a file that declares or matches a known pattern but is missing from the index, it pauses code edits and invokes `patternguard-planner` to repair the map before continuing. This is drift detection, not whole-repo discovery: new directories and unmarked files can remain outside PatternGuard coverage until a planning pass includes them. Large existing projects can be indexed incrementally around the requested change instead of deeply mapped on first use.

**The reviewer approves structure, not behavior.**
`patternguard-reviewer` checks sibling consistency and index conformance, then emits a targeted test checklist derived from the actual diff. It does not run tests, linters, or type checkers. Use the checklist as your manual verification guide before committing.

**Template mode requires an existing generator.**
PatternGuard can identify repeated logic as a template candidate, but it only uses template-backed execution when the project already has a working generator command. If no generator exists, the planner keeps the pattern in duplicated-file mode and flags a generator setup task instead of pretending regeneration is available. A project-specific generator such as `generator/main.go` must already exist and know how to render templates into the target files before PatternGuard edits templates and regenerates outputs.

---

## License

[MIT](LICENSE) (c) 2026 CalvinZ

Contact: yiwen.calvin@gmail.com
