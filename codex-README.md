# PatternGuard — Codex Plugin

PatternGuard keeps AI-agent changes from diverging across repeated code patterns: plan the blast radius, execute within bounds, and review consistency before committing.

## Skills

| Skill | Invoke when... |
|---|---|
| `patternguard-planner` | Designing structure, adding routes, planning cross-cutting logic, receiving R3 signals |
| `patternguard-executor` | Executing code changes, fixing bugs, being assigned a task by an orchestrator |
| `patternguard-reviewer` | Reviewing changes after execution, verifying correctness before committing |

## Install — Repo scope

Copy the plugin folder into your repo and wire it to a marketplace file:

```bash
# Step 1: Copy plugin into your repo
mkdir -p ./plugins
cp -R patternguard ./plugins/patternguard

# Step 2: Create (or append to) the repo marketplace file
mkdir -p .agents/plugins
cat > .agents/plugins/marketplace.json << 'EOF'
{
  "name": "local-repo",
  "interface": { "displayName": "Local Plugins" },
  "plugins": [
    {
      "name": "patternguard",
      "source": {
        "source": "local",
        "path": "./plugins/patternguard"
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
cp -R patternguard ~/.codex/plugins/patternguard

# Step 2: Create (or append to) the personal marketplace file
mkdir -p ~/.agents/plugins
cat > ~/.agents/plugins/marketplace.json << 'EOF'
{
  "name": "personal-plugins",
  "interface": { "displayName": "My Plugins" },
  "plugins": [
    {
      "name": "patternguard",
      "source": {
        "source": "local",
        "path": "./.codex/plugins/patternguard"
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
codex --plugin-dir ./patternguard
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
| `.patternguard/CODEBASE_INDEX.yaml` | Declares which files share logic and whether patterns are duplicated, candidates, or generator-backed |
| `.patternguard/changes/index.yaml` | Indexes reusable change-impact categories by name, impact, and detailed impact file path |
| `.patternguard/changes/<change-type>.yaml` | Declares scope for one change type, affected behavior, risk if wrong, tests, and regeneration source |
| `.patternguard/patternguard-handoff.yaml` | Carries task scope, affected files, and retry state between skills |
| `.patternguard/lock` | Fail-fast workflow lock for top-level planner/executor runs |
| `generator/templates/*.tmpl` | Single source of truth for cross-cutting logic |
