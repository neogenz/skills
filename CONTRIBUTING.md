# Contributing

## Skill structure

```
skills/
└── {skill-name}/
    ├── SKILL.md              # Frontmatter + content
    └── references/
        └── *.md              # Advanced patterns, detailed examples
```

## SKILL.md frontmatter

YAML block between `---` delimiters:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Kebab-case skill identifier |
| `description` | yes | Keywords the agent matches to activate the skill |
| `argument-hint` | no | Placeholder shown to the user (e.g. `<question>`) |
| `metadata.author` | no | GitHub username |
| `metadata.version` | no | Semver version |

Example:

```yaml
---
name: angular-architecture
description: Angular enterprise architecture advisor for placement decisions, dependency rules, and isolation patterns. Use when user asks "where should I put this", "can X import from Y", "Angular folder structure"…
argument-hint: <question-or-placement-decision>
metadata:
  author: maximedesogus
  version: 2.0.0
---
```

The `description` is what triggers activation. Pack it with real keywords and phrasings users would type — generic descriptions won't match.

## SKILL.md content

Recommended sections (adapt to the domain):

1. **Role** — Who the agent becomes, what it covers
2. **Defaults** — Modern APIs and patterns to prefer
3. **Critical Rules** — Hard constraints (3-5 max)
4. **Workflow / Decision trees** — How to handle requests
5. **Examples** — Working code blocks

Keep it practical. If a section doesn't help the agent make a decision or write code, cut it.

## `references/` directory

One Markdown file per sub-topic that's too large to fit in SKILL.md.

- **Naming**: kebab-case, descriptive (`architecture-rules.md`, `state-management.md`)
- **Self-contained**: each file readable on its own, no assumed reading order

## Conventions

- Kebab-case for all file and directory names
- Code examples must use current APIs — no deprecated patterns
- One skill = one bounded expertise domain
- Prefer slight duplication over coupling between skills

## Checklist

1. Create `skills/{skill-name}/SKILL.md` with frontmatter
2. Add `skills/{skill-name}/references/` with at least one file
3. Write a `description` loaded with trigger keywords
4. Manually test code examples
5. Add the skill to the **Available Skills** table in the [README](README.md)
