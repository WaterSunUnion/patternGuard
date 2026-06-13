# Blast Radius Skills — Claude Code Plugin

Three coordinated skills for codebases operated by AI agents.

## Skills

| Skill | Invoke when... |
|---|---|
| `blast-radius-planner` | Designing structure, adding routes, planning cross-cutting logic, receiving R3 signals |
| `blast-radius-executor` | Executing code changes, fixing bugs, being assigned a task by an orchestrator |
| `blast-radius-reviewer` | Reviewing changes after execution, verifying correctness before committing |

## Install — Marketplace

```bash
claude plugin install blast-radius-skills
```

## Install — Local dev / test without installing

```bash
# Run Claude Code pointed directly at the plugin folder
claude --plugin-dir ./blast-radius-skills
```

Changes to SKILL.md files take effect immediately in the current session.
Changes to other plugin components (hooks, agents, .mcp.json) require `/reload-plugins` or a restart.

## Install — Manual (repo scope)

```bash
# Copy plugin into your repo
mkdir -p ./plugins
cp -R blast-radius-skills ./plugins/blast-radius-skills

# Tell Claude Code to load it
# Add to your project's claude plugin config or drop in .claude/plugins/
```

## Install — Manual (personal scope, all repos)

```bash
mkdir -p ~/.claude/plugins
cp -R blast-radius-skills ~/.claude/plugins/blast-radius-skills
```

## Typical workflow

```
blast-radius-planner
  → reads/upgrades existing index (CODEBASE_INDEX.yaml)
  → classifies logic, chooses resolution (R1/R2/R3)
  → outputs index files + handoff note
      ↓
blast-radius-executor (orchestrator mode)
  → reads handoff note, picks mode
  → builds task envelopes, spawns subagents
      ↓
blast-radius-executor (subagent) × N
  → executes one file per subagent
  → reports status + R3 signals
      ↓
blast-radius-reviewer (orchestrator mode)
  → generates diffs, builds review envelopes
  → spawns reviewer subagents
      ↓
blast-radius-reviewer (subagent) × N
  → approved / changes-needed / escalate
      ↓
blast-radius-executor (orchestrator)
  → all approved → updates index → done
  → changes-needed → returns to executor
  → escalate → notifies planner
```

## Files this plugin produces in your repo

| File | Purpose |
|---|---|
| `CODEBASE_INDEX.yaml` | Declares which files share logic (patterns) |
| `IMPACT_MAP.yaml` | Declares blast radius per change type |
| `generator/templates/*.tmpl` | Single source of truth for cross-cutting logic |
