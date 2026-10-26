# agent-skills

Personal collection of [SKILL.md](https://opencode.ai/v2/docs/skills/) agent skills —
portable across OpenCode, Claude Code, Codex, and other tools that read the
`SKILL.md` standard.

Skills are distilled from technical books and real-world experience, organized
as progressive disclosure: `SKILL.md` holds the condensed rules and workflows,
`references/` holds the deeper material loaded on demand.

## Layout

```
agent-skills/
├── rust/
│   ├── SKILL.md            # idioms, pitfalls, decision rules
│   └── references/         # book chapters distilled by topic
├── sre-architecture/
│   ├── SKILL.md
│   └── references/
└── system-design/
    ├── SKILL.md
    └── references/
```

Each skill directory is one skill:

- `SKILL.md` — frontmatter (`name`, `description`) + the skill body, loaded when
  the agent invokes it. Keep under ~500 lines.
- `references/` — supporting files, read only when the task needs them.
- `scripts/` — optional, for helper scripts the skill can run.

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

## Roadmap

- [ ] `rust` — from the Rust book(s)
- [ ] `sre-architecture` — from SRE / distributed systems books
- [ ] `system-design` — from system design books
