# PatternGuard — Claude Code Plugin

PatternGuard keeps AI-agent changes from diverging across repeated code patterns: plan the blast radius, execute within bounds, and review consistency before committing.

## Skills

| Skill | Invoke when... |
|---|---|
| `patternguard-planner` | Designing structure, adding routes, planning cross-cutting logic, receiving R3 signals |
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
  → reads/upgrades existing index (CODEBASE_INDEX.yaml)
  → classifies logic, chooses resolution (R1/R2/R3)
  → outputs index files + handoff note
      ↓
patternguard-executor (orchestrator mode)
  → reads handoff note, picks mode
  → builds task envelopes, spawns subagents
      ↓
patternguard-executor (subagent) × N
  → executes one file per subagent
  → reports status + R3 signals
      ↓
patternguard-reviewer (orchestrator mode)
  → generates diffs, builds review envelopes
  → spawns reviewer subagents
      ↓
patternguard-reviewer (subagent) × N
  → approved / changes-needed / escalate
      ↓
patternguard-executor (orchestrator)
  → all approved → updates index → done
  → changes-needed → returns to executor
  → escalate → notifies planner
```

## Files this plugin produces in your repo

| File | Purpose |
|---|---|
| `CODEBASE_INDEX.yaml` | Declares which files share logic (patterns) |
| `IMPACT_MAP.yaml` | Declares scope per change type |
| `patternguard-handoff.yaml` | Carries task scope, affected files, and retry state between skills |
| `generator/templates/*.tmpl` | Single source of truth for cross-cutting logic |
