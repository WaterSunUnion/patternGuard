# Blast Radius Skills — Codex Plugin

Three coordinated skills for codebases operated by AI agents.

## Skills

| Skill | Invoke when... |
|---|---|
| `blast-radius-planner` | Designing structure, adding routes, planning cross-cutting logic, receiving R3 signals |
| `blast-radius-executor` | Executing code changes, fixing bugs, being assigned a task by an orchestrator |
| `blast-radius-reviewer` | Reviewing changes after execution, verifying correctness before committing |

## Install — Repo scope

Copy the plugin folder into your repo and wire it to a marketplace file:

```bash
# Step 1: Copy plugin into your repo
mkdir -p ./plugins
cp -R blast-radius-skills ./plugins/blast-radius-skills

# Step 2: Create (or append to) the repo marketplace file
mkdir -p .agents/plugins
cat > .agents/plugins/marketplace.json << 'EOF'
{
  "name": "local-repo",
  "interface": { "displayName": "Local Plugins" },
  "plugins": [
    {
      "name": "blast-radius-skills",
      "source": {
        "source": "local",
        "path": "./plugins/blast-radius-skills"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "Productivity"
    }
  ]
}
EOF

# Step 3: Restart Codex, open Plugin Directory, choose "Local Plugins", install
```

## Install — Personal scope (all repos)

```bash
# Step 1: Copy plugin to personal plugins folder
mkdir -p ~/.codex/plugins
cp -R blast-radius-skills ~/.codex/plugins/blast-radius-skills

# Step 2: Create (or append to) the personal marketplace file
mkdir -p ~/.agents/plugins
cat > ~/.agents/plugins/marketplace.json << 'EOF'
{
  "name": "personal-plugins",
  "interface": { "displayName": "My Plugins" },
  "plugins": [
    {
      "name": "blast-radius-skills",
      "source": {
        "source": "local",
        "path": "./.codex/plugins/blast-radius-skills"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "Productivity"
    }
  ]
}
EOF

# Step 3: Restart Codex
```

## Local dev / test without installing

```bash
# Run Codex pointed directly at the plugin folder — no marketplace needed
codex --plugin-dir ./blast-radius-skills
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
