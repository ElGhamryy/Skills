# Agent Skills

A curated collection of reusable Agent Skills for extending Claude (and other compatible agentic tools) with domain-specific capabilities.

Each skill packages together the context, conventions, and occasionally scripts or reference files needed to perform a specific kind of task well — so it doesn't have to be re-explained every time.

## What's in here

- Self-contained skill folders, each with a `SKILL.md` describing what the skill does, when it should trigger, and how to use it
- Supporting scripts, templates, or reference files a skill depends on
- Version history as skills get refined over time

## Structure

skill-name/
├── SKILL.md # Required. Description, triggers, instructions
├── scripts/ # Optional. Helper scripts the skill uses
└── resources/ # Optional. Templates, reference docs, examples

## Skills

| Skill | Description |
|---|---|
| _(add rows as you go)_ | |

## Goals

- Keep skills consistent, well-documented, and easy to discover
- Make it easy to install/enable individual skills in different environments
- Iterate on skill quality using real usage feedback

## Usage

Point your agent's skills directory (or equivalent config) at the relevant skill folder(s) in this repo, or copy the folder into your local skills path.

## Contributing

This is primarily a personal skill library, but suggestions and improvements are welcome via issues or pull requests.

## License

_(add a license if you plan to share this publicly)_
