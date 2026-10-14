# skills-public

Agent skills (Claude Code / Agent Skills format) for .NET work that isn't Sitefinity-specific,
with a focus on older stacks that general model knowledge covers poorly.

| Skill | What it covers |
|---|---|
| [efcore-3.1](efcore-3.1/) | Entity Framework Core 3.1: query translation rules, raw SQL / stored procedures, migrating from 2.x, running on .NET Framework 4.7.2/4.8, performance |

## Install

Copy a skill folder into `~/.claude/skills/` (personal) or `.claude/skills/` (per project):

```bash
git clone https://github.com/sitefinitysteve/skills-public.git
cp -r skills-public/efcore-3.1 ~/.claude/skills/
```
