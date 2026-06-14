# PatternGuard — Claude Code Plugin

PatternGuard keeps AI-agent changes from diverging across repeated code patterns: plan the blast radius, execute within bounds, and review consistency before committing.

## Skills

| Skill | Invoke when... |
|---|---|
| `patternguard-planner` | Inspecting or updating the index, repairing drift, or re-planning after `ESCALATE` / `retry_count >= 3` |
| `patternguard-executor` | Executing code changes, fixing bugs, being assigned a task by an orchestrator |
| `patternguard-reviewer` | Reviewing changes after execution, verifying correctness before committing |

## Install — Marketplace

```bash
claude plugin install patternguard
```

## Install — Local dev / test without installing

```bash
# Run Claude Code pointed directly at the plugin folder
claude --plugin-dir ./patternguard
```

Changes to SKILL.md files take effect immediately in the current session.
Changes to other plugin components (hooks, agents, .mcp.json) require `/reload-plugins` or a restart.

## Install — Manual (repo scope)

```bash
# Copy plugin into your repo
mkdir -p ./plugins
cp -R patternguard ./plugins/patternguard

# Tell Claude Code to load it
# Add to your project's claude plugin config or drop in .claude/plugins/
```

## Install — Manual (personal scope, all repos)

```bash
mkdir -p ~/.claude/plugins
cp -R patternguard ~/.claude/plugins/patternguard
```

## Typical workflow

```
patternguard-planner
  → acquires .patternguard/lock for direct planner runs, or uses executor's lock
  → reads/upgrades existing index (.patternguard/CODEBASE_INDEX.yaml)
  → classifies logic, chooses duplicated, template-candidate, or template-backed handling
  → outputs .patternguard index files + handoff note
      ↓
patternguard-executor (orchestrator mode)
  → acquires .patternguard/lock or fails fast if another workflow owns it
  → reads handoff note and index
  → checks template-backed, template-candidate, and duplicated patterns
  → surfaces scoped paths, feature impact, risk, and tests
  → edits template + regenerates only when generator.command exists, or builds task envelopes for duplicated files
      ↓
patternguard-executor (subagent) × N, only for duplicated files
  → executes one scoped file per subagent
  → reports done / blocked / ESCALATE
      ↓
patternguard-reviewer (orchestrator mode)
  → builds review envelopes from the changed files
  → spawns reviewer subagents
      ↓
patternguard-reviewer (subagent) × N
  → approved / changes-needed / ESCALATE
      ↓
patternguard-executor (orchestrator)
  → all approved → updates index → done
  → changes-needed → returns to executor
  → ESCALATE or retry_count >= 3 → calls planner to re-plan
```

## Files this plugin produces in your repo

| File | Purpose |
|---|---|
| `.patternguard/CODEBASE_INDEX.yaml` | Declares which files share logic and whether patterns are duplicated, candidates, or generator-backed |
| `.patternguard/changes/index.yaml` | Indexes reusable change-impact categories by name, impact, and detailed impact file path |
| `.patternguard/changes/<change-type>.yaml` | Declares scope for one change type, affected behavior, risk if wrong, tests, and regeneration source |
| `.patternguard/patternguard-handoff.yaml` | Carries task scope, affected files, template path when generator-backed, `retry_count`, and header repairs between skills |
| `.patternguard/lock` | Fail-fast workflow lock for top-level planner/executor runs |
| `generator/templates/*.tmpl` | Project-provided template source used only when a working `generator.command` exists |
