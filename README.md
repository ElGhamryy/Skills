# Agent Skills

A curated collection of reusable Agent Skills for extending Claude (and other compatible agentic tools) with domain-specific capabilities.

Each skill packages together the context, conventions, and occasionally scripts or reference files needed to perform a specific kind of task well — so it doesn't have to be re-explained every time.

## What's in here

- Self-contained skill folders, each with a `SKILL.md` describing what the skill does, when it should trigger, and how to use it
- Supporting scripts, templates, or reference files a skill depends on
- Version history as skills get refined over time

## Structure

```
skills/
└── skill-name/
    ├── SKILL.md    # Required. Description, triggers, instructions
    ├── scripts/    # Optional. Helper scripts the skill uses
    └── resources/  # Optional. Templates, reference docs, examples
rules/              # Shared coding rules the skills expect to find in a project
```

## Skills

| Skill | Description |
|---|---|
| [implement-feature](skills/implement-feature/SKILL.md) | Plan and build a React feature step by step: clarify, pseudocode plan, confirm, implement, verify, changelog. |
| [implement-feature-typescript](skills/implement-feature-typescript/SKILL.md) | Same as `implement-feature`, but new files are TypeScript in a partly-JS codebase. Legacy JS is never converted or typed. |
| [test-feature](skills/test-feature/SKILL.md) | Write focused unit tests for a React feature with Vitest/Jest + React Testing Library. |
| [pr-review](skills/pr-review/SKILL.md) | Advisory review of a PR's changed lines on GitHub, GitLab, or Bitbucket; posts inline comments and a summary. |

## Rules

| Rule | Description |
|---|---|
| [frontend-conventions](rules/frontend-conventions.md) | React code style, styling, accessibility, list keys, and paginated-list edge cases. Copy into a project's `.claude/rules/`. |

## Goals

- Keep skills consistent, well-documented, and easy to discover
- Make it easy to install/enable individual skills in different environments
- Iterate on skill quality using real usage feedback

## Usage

Copy a skill folder into your agent's skills path — for Claude Code, `~/.claude/skills/` (all projects) or `<project>/.claude/skills/` (one project). Copy rule files into `<project>/.claude/rules/`.

## Contributing

This is primarily a personal skill library, but suggestions and improvements are welcome via issues or pull requests.

## License

_(add a license if you plan to share this publicly)_
