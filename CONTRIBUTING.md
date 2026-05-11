# Contributing to Radicalbit Claude Code Plugins

## Repository Structure

```
radicalbit-skills/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    └── radicalbit-ai-gateway/
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
            └── your-skill/
                ├── SKILL.md    ← required
                ├── examples/   ← optional
                └── references/ ← optional
```

---

## Adding or Updating Skills

Each skill lives in `plugins/radicalbit-ai-gateway/skills/<skill-name>/` and requires a `SKILL.md` with YAML frontmatter:

```markdown
---
name: your-skill-name
description: >
  When to invoke this skill. Include trigger phrases.
---

# Skill Title

Instructions for Claude...
```

**Required frontmatter fields:**

| Field | Description |
|-------|-------------|
| `name` | Kebab-case. Determines the `/radicalbit-ai-gateway:<name>` command. |
| `description` | When to invoke the skill. Include example trigger phrases. |

**Optional frontmatter fields:**

| Field | Description |
|-------|-------------|
| `disable-model-invocation` | `true` for skills that write files or take actions (not just advise). |

### Adding Reference Files

```
skills/your-skill/
├── SKILL.md
├── examples/
│   └── example.yaml
└── references/
    └── schema.md
```

Reference via relative paths in `SKILL.md`:

```markdown
See [examples/example.yaml](examples/example.yaml) for a complete example.
```

When adding a skill, update the `keywords` array in [`plugins/radicalbit-ai-gateway/.claude-plugin/plugin.json`](plugins/radicalbit-ai-gateway/.claude-plugin/plugin.json).

---

## Pull Request Process

1. Fork the repository and create a branch from `main`.
2. Add or update skill files following the structure above.
3. Test locally (see below).
4. Update `README.md` to document the new skill in the skills table.
5. Open a pull request describing what the skill does and when it triggers.

---

## Local Testing

Add the plugin from a local path in your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "radicalbit-local": {
      "source": {
        "source": "local",
        "path": "/path/to/radicalbit-skills"
      }
    }
  }
}
```

Then install and invoke:

```
/plugin install radicalbit-ai-gateway@radicalbit-local
/radicalbit-ai-gateway:your-skill-name
```
