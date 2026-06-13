# Blast Radius Skills

> Three coordinated skills for codebases operated by AI agents — plan safe file structures, execute changes across bounded context, and verify correctness before committing.

---

## Skills

| Skill | Trigger phrase | What it does |
|---|---|---|
| `blast-radius-planner` | "design structure", "add a route", "plan a feature", "R3 signal received" | Classifies logic, chooses R1/R2/R3 resolution, outputs `CODEBASE_INDEX.yaml` + `IMPACT_MAP.yaml` + handoff note |
| `blast-radius-executor` | "execute", "apply the change", "fix the bug", "you've been assigned a task" | Self-selects Bootstrap / Solo / Orchestrator / Subagent mode and makes changes with blast-radius awareness |
| `blast-radius-reviewer` | "review changes", "verify before committing", "check correctness" | Reads diffs only, never edits — produces `approved`, `changes-needed`, or `escalate` verdicts |

---

## How it works

```
blast-radius-planner
  → reads/upgrades CODEBASE_INDEX.yaml
  → classifies logic (route-specific, cross-cutting, utility)
  → chooses resolution: R1 (template), R2 (index contract), R3 (earned abstraction)
  → outputs CODEBASE_INDEX.yaml + IMPACT_MAP.yaml + handoff note
          ↓
blast-radius-executor  [orchestrator]
  → reads handoff note, picks mode
  → builds task envelopes, spawns subagents
          ↓
blast-radius-executor  [subagent × N]
  → executes one file per subagent
  → reports status + any R3 signals
          ↓
blast-radius-reviewer  [orchestrator]
  → builds review envelopes, spawns reviewer subagents
          ↓
blast-radius-reviewer  [subagent × N]
  → approved / changes-needed / escalate
          ↓
blast-radius-executor  [orchestrator]
  → all approved  → updates index → done
  → changes-needed → loops back to executor
  → escalate      → notifies planner
```

---

## Install — Claude Code

### Marketplace (easiest)

```bash
claude plugin install blast-radius-skills
```

### Local dev / test without installing

```bash
# Skills reload immediately on file save — no restart needed for SKILL.md changes
claude --plugin-dir ./blast-radius-skills
```

> Changes to hooks, agents, or `.mcp.json` require `/reload-plugins` or a session restart.

### Manual — repo scope

```bash
mkdir -p ./plugins
cp -R blast-radius-skills ./plugins/blast-radius-skills
# Then add to your project's plugin config or drop in .claude/plugins/
```

### Manual — personal scope (all repos)

```bash
mkdir -p ~/.claude/plugins
cp -R blast-radius-skills ~/.claude/plugins/blast-radius-skills
```

---

## Install — Codex

### Repo scope

```bash
# Step 1: copy plugin into your repo
mkdir -p ./plugins
cp -R blast-radius-skills ./plugins/blast-radius-skills

# Step 2: create the repo marketplace file
mkdir -p .agents/plugins
cat > .agents/plugins/marketplace.json << 'EOF'
{
  "name": "local-repo",
  "interface": { "displayName": "Local Plugins" },
  "plugins": [
    {
      "name": "blast-radius-skills",
      "source": { "source": "local", "path": "./plugins/blast-radius-skills" },
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
cp -R blast-radius-skills ~/.codex/plugins/blast-radius-skills

# Step 2: create the personal marketplace file
mkdir -p ~/.agents/plugins
cat > ~/.agents/plugins/marketplace.json << 'EOF'
{
  "name": "personal-plugins",
  "interface": { "displayName": "My Plugins" },
  "plugins": [
    {
      "name": "blast-radius-skills",
      "source": { "source": "local", "path": "./.codex/plugins/blast-radius-skills" },
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
codex --plugin-dir ./blast-radius-skills
```

---

## Files produced in your repo

| File | Purpose |
|---|---|
| `CODEBASE_INDEX.yaml` | Declares which files share logic (patterns) |
| `IMPACT_MAP.yaml` | Declares blast radius per change type |
| `generator/templates/*.tmpl` | Single source of truth for cross-cutting logic |

---

## Why blast radius?

When an AI agent edits a file in isolation it can't know how many other files implement the same pattern. One change → silently diverged siblings. Blast Radius Skills solve this by:

1. **Indexing** shared patterns before any change is made
2. **Bounding** each subagent to a declared scope (no silent sprawl)
3. **Reviewing** sibling consistency as a first-class step before commit

---

## License

MIT © Calvin
