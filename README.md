# agent-skills

Personal collection of [SKILL.md](https://opencode.ai/v2/docs/skills/) agent skills —
portable across OpenCode, Claude Code, Codex, and other tools that read the
`SKILL.md` standard.

Skills are distilled from technical books and real-world experience. Each
skill is a single `SKILL.md` holding the condensed rules, workflows, and
checklists.

## Layout

```
agent-skills/
├── rust/
│   └── SKILL.md            # idioms, pitfalls, decision rules
├── sre-architecture/
│   └── SKILL.md
├── system-design/
│   └── SKILL.md
└── proactive-advisor/
    └── SKILL.md
```

Each skill directory is one skill:

- `SKILL.md` — frontmatter (`name`, `description`) + the skill body, loaded when
  the agent invokes it. Keep under ~500 lines.

## Using with OpenCode

Global, all projects — add to `~/.config/opencode/opencode.jsonc`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "skills": ["~/Projects/agent-skills"]
}
```

Single project — add to that project's `opencode.jsonc` instead, or commit
skills under `.opencode/skills/`.

## Contents

- [x] `rust` — idiomatic Rust from *The Rust Programming Language* (official book)
- [x] `sre-architecture` — from *Site Reliability Engineering: How Google Runs Production Systems*
- [x] `system-design` — from *Designing Data-Intensive Applications* (2nd ed.)
- [x] `proactive-advisor` — volunteer unasked strategic ideas, risks, and trade-offs on planning, product, and career work
