# elite-skills

Production-grade [Agent Skills](https://docs.claude.com/en/docs/claude-code/skills) for
Claude Code, the Claude Agent SDK, and any agent runtime that speaks the
[skills.sh](https://skills.sh) format (Claude, OpenAI, Cursor, Cline, and other
SKILL.md-aware coding agents).

Each skill is self-contained, language-agnostic, and gated on real toolchain
output — no theater, no invented numbers, no silent edits to your repo.

## Skills in this pack

| Skill | What it does | Trigger phrases |
| ----- | ------------ | --------------- |
| [`elite-lean-audit`](./elite-lean-audit/SKILL.md) | Repo-wide "trim the fat" audit. Finds and safely deletes dead code, demo scaffolding, junk files, and oversized modules, then proves nothing broke with the project's own type-check, tests, and build. | `audit the repo`, `trim the fat`, `lean audit`, `remove dead code`, `what can we delete`, `shrink the codebase` |
| [`elite-security-agent`](./elite-security-agent/SKILL.md) | System-level security audit. Builds a threat model, runs SAST/SCA/IaC/secret scanners, walks every audit domain (DDoS, auth, injection, data, secrets, supply chain, infra, containers, CI/CD, AI/LLM, observability), scores findings, and ships a prioritized hardening roadmap. Advisory only by default. | `security audit`, `threat model`, `harden my system`, `pen test the architecture`, `protect against DDoS` |

## Install

### From skills.sh (recommended)

```bash
# All skills in this pack
skills add kbpsystem777/elite-skills

# Or a single skill
skills add kbpsystem777/elite-skills/elite-lean-audit
skills add kbpsystem777/elite-skills/elite-security-agent
```

### Claude Code (project-scoped)

Drop the skill folder into `.claude/skills/` at the root of your repo:

```bash
git clone https://github.com/KBPsystem777/elite-skills.git /tmp/elite-skills
mkdir -p .claude/skills
cp -r /tmp/elite-skills/elite-lean-audit .claude/skills/
cp -r /tmp/elite-skills/elite-security-agent .claude/skills/
```

### Claude Code (user-scoped, available in every project)

```bash
mkdir -p ~/.claude/skills
cp -r elite-lean-audit ~/.claude/skills/
cp -r elite-security-agent ~/.claude/skills/
```

### Claude Agent SDK / Claude API

Point your agent's skills loader at this repo or its individual skill folders.
Each `SKILL.md` is a standalone, self-describing module — the YAML frontmatter
(`name`, `description`) is the contract every compliant runtime reads.

## Skill format

Every skill in this pack follows the [Anthropic Agent Skills](https://docs.claude.com/en/docs/claude-code/skills)
spec, which is the de facto standard adopted by skills.sh and other coding
agents:

```
<skill-name>/
├── SKILL.md              # required: YAML frontmatter + instructions
├── references/           # optional: deeper docs loaded on demand
│   └── *.md
└── scripts/              # optional: helper scripts the skill may run
```

The frontmatter is the only field the runtime needs:

```yaml
---
name: <slug, kebab-case, matches the folder>
description: >-
  One paragraph. Says what the skill does AND when to invoke it
  (trigger phrases, keywords, situations). Max 1024 chars.
---
```

The agent reads `description` upfront for routing, and `SKILL.md`'s body only
when the skill activates. Reference files under `references/` are loaded lazily
when the skill instructions point to them — keeping the activation cost low.

## Design principles

These skills share a strict house style:

1. **Detect the toolchain first.** No assumption about language or framework.
2. **Gate every claim on the toolchain's own output.** Type-check, tests,
   build, scanners — not vibes.
3. **Never auto-commit, never silently rewrite.** The diff is for the human.
4. **Respect work-in-progress.** Files dirty in `git status` are off-limits.
5. **Report real numbers.** Before/after, exit codes, pass counts. No theater.
6. **Advisory by default for risky work.** The security skill audits; it only
   patches when you explicitly say "apply" or "fix".

## Contributing

Issues and PRs welcome. Keep new skills:

- Self-contained (one folder, one `SKILL.md`, references in `references/`).
- Language-agnostic where possible; if not, say so in the description.
- Verifiable — every action gated on a toolchain command, not narration.

## License

[MIT](./LICENSE) © 2026 Koleen Baes Paunon koleen.bp@outlook.com
