# Skills

A collection of skills for AI-assisted development. These skills provide coding agents such as Claude, Gemini, OpenCode, etc with up-to-date patterns, best practices, and code examples.

## Installation

Install all skills from this repository:

```bash
npx skills add maximedesogus/skills
```

Or install individual skills:

```bash
npx skills add maximedesogus/skills/skills/angular-architecture
```

## Available Skills

| Skill | Description |
|-------|-------------|
| [angular-architecture](skills/angular-architecture) | Enterprise architecture advisor for placement decisions, dependency rules, isolation patterns, and lazy loading optimization |

## Skill Structure

Each skill follows the standard structure:

```
skills/
└── {skill-name}/
    ├── SKILL.md              # Main skill file with patterns and examples
    └── references/
        └── *.md              # Advanced patterns and additional examples
```

## Contributing

Contributions are welcome! Please ensure any additions:

1. Follow the existing skill structure
2. Include practical, working code examples
3. Provide reference files for advanced patterns

## License

MIT
