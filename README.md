# Agent Skills

A collection of reusable `SKILL.md` files for GitHub Copilot agent customization. Each skill packages domain knowledge, workflows, and best practices that Copilot loads on demand to produce higher-quality, more consistent outputs.

## What is a Skill?

A skill is a `SKILL.md` file inside a named folder. It contains a YAML frontmatter block (with a `name` and `description`) followed by the instructions Copilot should follow when that skill is active. Skills are triggered automatically when a user request matches the skill's description.

## Skills

### [`first-principles`](./first-principles/SKILL.md)
Structured four-phase first-principles analysis modeled on Aristotle's method: surface assumptions → establish first principles → rebuild from the foundation → identify the high-leverage move. Triggers when a user wants to think through a problem more rigorously, challenge assumptions, or break something down from scratch.

### [`letter-grading-scale`](./letter-grading-scale/SKILL.md)
Converts any percentage score (0–100%) to a US college letter grade (A+ through F) with GPA equivalent, using the standard scale sourced from Wikipedia. Triggers when a percentage needs to be expressed as a letter grade.

## Usage

Reference these skills in your VS Code Copilot configuration (`.github/copilot-instructions.md` or a `.vscode/` instructions file) by pointing to the skill folder. Copilot will load the relevant `SKILL.md` when the trigger conditions are met.

## Contributing

Each skill lives in its own folder:

```
agent-skills/
└── skill-name/
    └── SKILL.md
```

The `SKILL.md` frontmatter `description` field determines when the skill is invoked — write it to match the natural language a user would use when they need that skill.
