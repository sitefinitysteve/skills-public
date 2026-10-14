# efcore-3.1 - a Claude Code skill for Entity Framework Core 3.1

Rules, traps and fixes for code that runs on EF Core 3.1.x: query translation limits, raw SQL and
stored procedures, keyless types, the EF Core 2.x -> 3.x breaking changes that actually bite, running
on .NET Framework with Microsoft.Data.SqlClient, and performance.

Targets EF Core 3.1.x (3.1.32 is the final release) on .NET Framework 4.7.2/4.8 and .NET Core 3.1.
Most examples use the SQL Server provider; a section covers MySQL via Pomelo 3.2.x.

Claims were checked against the `release/3.1` branch of dotnet/efcore, the official EF Core docs,
Microsoft.Data.SqlClient release notes and package contents. Each reference file ends with its
sources. Items that could not be verified are labelled as such in the text.

## Layout

```
efcore-3.1/
  SKILL.md                                  core rules, triage table, review checklist
  references/
    translation-rules.md                    what 3.1 translates vs throws, with rewrites
    raw-sql-and-stored-procedures.md        FromSqlRaw/ExecuteSqlRaw, keyless types, procedure pitfalls
    migrating-from-2x.md                    breaking changes ordered by impact, detection, fixes
    sqlclient-and-dotnet-framework.md       SqlClient, native SNI, IIS, binding redirects, Pomelo
    performance.md                          tracking, N+1, cartesian explosion, Contains, batching, logging
    testing-the-model.md                    tests that prove a model matches the database
```

## Install

Copy the `efcore-3.1` folder into either:

- `~/.claude/skills/` - available in every project, or
- `<your-repo>/.claude/skills/` - available to everyone working in that repository.

Claude Code loads `SKILL.md` when a task matches its description and opens the reference files on
demand.

## License

MIT
