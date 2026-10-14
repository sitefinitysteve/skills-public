# Migrating from EF Core 2.x to 3.1

Ordered by how often each change breaks real code. Each entry: what changed, how to find it, how to
fix it (keeping 2.x behaviour where that is what the code relied on).

Items marked "observed" were proven by failing tests during a real 2.1 -> 3.1 migration; the rest
come from the official breaking-changes document and the 3.1 source.

## Before changing any package: harvest the 2.x warnings

2.x logs every client-evaluated expression. Turn the warning into an exception on the 2.x build
and run the test suite and the app:

```csharp
optionsBuilder.ConfigureWarnings(w => w.Throw(RelationalEventId.QueryClientEvaluationWarning));
```

Every exception is a query that 3.1 will reject. Fix them on 2.x, where the old behaviour still
works as a comparison baseline.

## Upgrade procedure

1. Update all `Microsoft.EntityFrameworkCore.*` packages to the same 3.1.x (3.1.32 is the last).
2. Add `Microsoft.Data.SqlClient` explicitly if using packages.config (see
   `sqlclient-and-dotnet-framework.md`), fix binding redirects.
3. Build. Fix every CS0619 (hard errors), then every CS0618 that mentions EF.
4. Replace `System.Data.SqlClient` with `Microsoft.Data.SqlClient` in every file that touches EF.
5. Add the model tests from `testing-the-model.md` and run them against a real database.
6. Run every query path. Translation errors only appear at execution.

---

## 1. LINQ is no longer evaluated on the client (very high)

**Change:** only the final projection may run on the client. Everything else that cannot be
translated throws `The LINQ expression '...' could not be translated`.

**Detect:** the 2.x warning harvest above; any query calling application methods, `string.Format`,
interpolation, `StringComparison` overloads, `DayOfWeek`, `TimeSpan` arithmetic, `Math.Max`,
`GroupBy` that enumerates groups, `Last()` without `OrderBy`, `DefaultIfEmpty(value)`.

```
grep -rnE "\.(Where|OrderBy|OrderByDescending|ThenBy|Any|All|Count|First|FirstOrDefault|Single|SingleOrDefault)\(.*\b[A-Z][A-Za-z]+\(" --include=*.cs
grep -rnE "GroupBy\(" --include=*.cs
grep -rnE "StringComparison\.|\.DayOfWeek|DefaultIfEmpty\([^)]" --include=*.cs
```

**Fix:** `translation-rules.md`. Keep 2.x results with `.AsEnumerable()` placed after the last
operator that translates, never at the start.

## 2. FromSql / ExecuteSqlCommand renamed (very high)

| 2.x | 3.1 | Compiler |
|---|---|---|
| `FromSql(string, params object[])` | `FromSqlRaw` | CS0619 error |
| `FromSql(FormattableString)` / `FromSql($"...")` | `FromSqlInterpolated` | CS0619 error |
| `ExecuteSqlCommand(...)` | `ExecuteSqlRaw` / `ExecuteSqlInterpolated` | CS0618 warning only |
| `ExecuteSqlCommandAsync(...)` | `ExecuteSqlRawAsync` / `ExecuteSqlInterpolatedAsync` | CS0618 warning only |

```
grep -rnE "\.FromSql\(|ExecuteSqlCommand(Async)?\(" --include=*.cs
```

Mechanical, but check each call: `FromSql($"...")` must become `FromSqlInterpolated`, not
`FromSqlRaw` (which would turn it into string concatenation). `FromSqlRaw` must be called directly
on the `DbSet` (it no longer exists on `IQueryable<T>`).

## 3. FromSql over a stored procedure can no longer be composed (high, observed)

**Change:** 2.x detected non-composable SQL and silently evaluated `FirstOrDefault`/`Where`/`Count`
on the client. 3.x composes in SQL and throws `FromSqlNonComposable` for anything not starting with
`SELECT`.

```
grep -rnE "FromSql(Raw|Interpolated)?\([^)]*(EXEC|exec|Exec)" --include=*.cs
```

Then inspect what follows each call, including in callers that receive the returned `IQueryable`.

**Fix:** `.AsEnumerable()` or `.ToList()` immediately after the FromSql call; change wrapper return
types from `IQueryable<T>` to `List<T>`.

## 4. Query types became keyless entity types (high)

| 2.x | 3.1 |
|---|---|
| `modelBuilder.Query<T>()` | `modelBuilder.Entity<T>().HasNoKey()` |
| `modelBuilder.Query<T>().ToView("v")` | `modelBuilder.Entity<T>().HasNoKey().ToView("v")` |
| `public DbQuery<T> Items { get; set; }` | `public DbSet<T> Items { get; set; }` |
| `context.Query<T>()` | `context.Set<T>()` |
| `IQueryTypeConfiguration<T>` | `IEntityTypeConfiguration<T>` |

All old forms compile with CS0618 warnings and still work, so they survive until someone removes
them. Converting `DbQuery<T>` to `DbSet<T>` without adding `HasNoKey()` fails model validation for a
type without a key-like property.

```
grep -rnE "DbQuery<|\.Query<|IQueryTypeConfiguration" --include=*.cs
```

**Behaviour change (observed):** a keyless row whose columns are all NULL now materializes as
`null` (dotnet/efcore#19537). Code doing `result.First().Name` gets a `NullReferenceException`.
Filter nulls or make the SQL return at least one non-null column.

## 5. System.Data.SqlClient -> Microsoft.Data.SqlClient (high, observed)

**Change:** the SQL Server provider depends on Microsoft.Data.SqlClient (1.1.x for EF 3.1).

**Breaks:**
- `new System.Data.SqlClient.SqlParameter(...)` passed to FromSql/ExecuteSql compiles and throws
  `InvalidCastException: The SqlParameterCollection only accepts non-null Microsoft.Data.SqlClient.SqlParameter type objects, not System.Data.SqlClient.SqlParameter objects.`
- `catch (System.Data.SqlClient.SqlException)` no longer matches; errors escape or hit a generic
  handler. Retry and deadlock logic keyed on `SqlException.Number` silently stops working.
- `(SqlConnection)db.Database.GetDbConnection()` casts with the old type throw `InvalidCastException`.

```
grep -rn "System.Data.SqlClient" --include=*.cs --include=*.cshtml --include=*.config
```

**Fix:** switch the namespace. Code that must talk to both stacks (EF and older ADO.NET helpers)
should catch `DbException` and read the number via the concrete type it actually received.

## 6. Required properties and exact CLR types are trusted (high, observed)

**Change:** the 3.x shaper reads a property with the typed getter of its mapping and adds an
`IsDBNull` check only when the column is nullable in the model.

**Breaks:**
- `.IsRequired()` (or `[Required]`, or a non-nullable value type) over a NULLable column throws
  `SqlNullValueException: Data is Null. This method or property cannot be called on Null values.`
  when a NULL is read.
- A `bigint` value read into an `int` property throws `InvalidCastException: Specified cast is not valid.`
  (`ROW_NUMBER()`, `COUNT_BIG()`, `SCOPE_IDENTITY()` results in procedures are the usual sources).

Neither exception names the property; 3.1's detailed per-property error wrapping in the relational
shaper is commented out, so `EnableDetailedErrors()` does not help here.

**Detect:** compare the model to INFORMATION_SCHEMA (`testing-the-model.md`, section 4). For
procedure results, read the procedure's final SELECT.

**Fix:** make the model honest: drop `IsRequired()`, use `int?`/`long`, or `CAST`/`ISNULL` in SQL.

## 7. FK convention no longer matches a property named like the principal key (high, observed)

**Change:** 2.x used a dependent property with the same name as the principal key (`RunId` ->
principal key `RunId`) as the FK. 3.x only matches `<NavigationName><KeyName>` and
`<PrincipalTypeName><KeyName>`. With no match it creates a shadow FK named after the navigation,
e.g. navigation `Log` + key `RunId` -> shadow property `LogRunId`, and queries fail with
`SqlException: Invalid column name 'LogRunId'.`

**Detect:**

```csharp
var shadowFks = db.Model.GetEntityTypes()
    .SelectMany(t => t.GetProperties())
    .Where(p => p.IsShadowProperty() && p.IsForeignKey())
    .Select(p => p.DeclaringEntityType.DisplayName() + "." + p.Name)
    .ToList();   // should be empty unless a shadow FK is intended
```

**Fix:** configure the relationship explicitly:

```csharp
modelBuilder.Entity<RunEntry>()
    .HasOne(e => e.Log).WithMany(l => l.Entries)
    .HasForeignKey(e => e.RunId);
```

## 8. Cascade deletes happen immediately (medium)

**Change:** `Remove(principal)` marks tracked required dependents `Deleted` at once (and orphans
are deleted when a required relationship is severed), instead of at `SaveChanges`.

**Breaks:** code that inspects `ChangeTracker.Entries()`, re-parents children, or counts deletions
between `Remove` and `SaveChanges`.

**Fix (restore 2.x):**

```csharp
db.ChangeTracker.CascadeDeleteTiming = CascadeTiming.OnSaveChanges;
db.ChangeTracker.DeleteOrphansTiming = CascadeTiming.OnSaveChanges;
```

Set them in the context constructor to apply everywhere.

## 9. Lazy loading on a disposed context or no-tracking entity throws (medium, observed)

**Change:** 3.x proxies track which navigations are loaded. Touching an unloaded navigation after
the context is disposed raises `LazyLoadOnDisposedContextWarning`, and on an entity from an
`AsNoTracking` query raises `DetachedLazyLoadingWarning`. Both default to Throw.

```
An attempt was made to lazy-load navigation property 'Customer' on entity type 'OrderProxy'
after the associated DbContext was disposed.
```

**Fix:** load what the caller needs (`Include`, projection) before the context goes away. To
restore 2.x "returns null/empty" behaviour deliberately:

```csharp
optionsBuilder.ConfigureWarnings(w => w
    .Ignore(CoreEventId.LazyLoadOnDisposedContextWarning)
    .Ignore(CoreEventId.DetachedLazyLoadingWarning));
```

Ignoring hides real N+1 and stale-data bugs. Prefer fixing the call site.

## 10. No-tracking queries do not resolve identity (medium)

**Change:** `AsNoTracking().Include(o => o.Customer)` creates a separate `Customer` instance per
order, even for the same key.

**Breaks:** reference equality, in-memory `Distinct()` on entities, mutating one instance and
expecting all to change, memory use on large graphs.

**Fix:** tracking query where identity matters; DTO projection otherwise; or group by key in
memory.

## 11. Eager loading runs as one query (medium)

**Change:** each collection `Include` used to be a separate query. Now everything is JOINed into
one statement. Multiple collection Includes multiply rows (cartesian explosion); timeouts and
memory spikes appear on queries that were fine in 2.x.

**Fix:** split manually (`performance.md`, section 3). `AsSplitQuery()` does not exist until 5.0.

## 12. Temporary key values are not set on entities (medium)

**Change:** after `Add`, a store-generated key property keeps its CLR default (0) until
`SaveChanges`. 2.x wrote a large negative temporary value into it.

**Breaks:** `child.ParentId = parent.Id` before saving (writes 0, FK violation or wrong parent);
dictionaries keyed by `Id` of unsaved entities.

**Fix:** set navigations (`child.Parent = parent`) or read the temporary value from the tracker:
`db.Entry(parent).Property(p => p.Id).CurrentValue`.

## 13. DetectChanges honours store-generated keys (medium)

**Change:** an untracked entity discovered through a navigation is tracked as `Modified` (UPDATE)
if its generated key is set, and `Added` (INSERT) only if the key is default.

**Breaks:** graphs where new children carry an explicit key value now try to UPDATE rows that do
not exist (`DbUpdateConcurrencyException: ... expected to affect 1 row(s) but actually affected 0`).

**Fix:** leave generated keys at default for new entities, call `Add` explicitly, or mark the key
`ValueGeneratedNever()` if the application assigns keys.

## 14. Metadata API flattened (medium, compile errors)

| 2.x | 3.1 |
|---|---|
| `entityType.Relational().TableName` | `entityType.GetTableName()` |
| `entityType.Relational().Schema` | `entityType.GetSchema()` (null means default) |
| `model.Relational().DefaultSchema` | `model.GetDefaultSchema()` |
| `property.Relational().ColumnName` | `property.GetColumnName()` |
| `property.Relational().ColumnType` | `property.GetColumnType()` |
| `property.Relational().DefaultValueSql` | `property.GetDefaultValueSql()` |
| `entityType.SqlServer().IsMemoryOptimized` | `entityType.IsMemoryOptimized()` |
| `entityType.QueryFilter` | `entityType.GetQueryFilter()` |
| `entityType.DefiningQuery` | `entityType.GetDefiningQuery()` |
| `property.IsShadowProperty` | `property.IsShadowProperty()` |
| `property.BeforeSaveBehavior` / `AfterSaveBehavior` | `GetBeforeSaveBehavior()` / `GetAfterSaveBehavior()` |

The `Relational()`/`SqlServer()` accessors are gone (compile error). Column nullability as the
database sees it: `property.IsColumnNullable()`.

```
grep -rnE "\.Relational\(\)|\.SqlServer\(\)" --include=*.cs
```

## 15. SQL Server builder renames (low, warnings)

| 2.x | 3.1 |
|---|---|
| `ForSqlServerIsClustered()` | `IsClustered()` |
| `ForSqlServerInclude(...)` | `IncludeProperties(...)` |
| `ForSqlServerIsCreatedOnline()` | `IsCreatedOnline()` |
| `ForSqlServerUseSequenceHiLo()` | `UseHiLo()` |
| `ForSqlServerHasHiLoSequence()` | `HasHiLoSequence()` |
| `UseSqlServerIdentityColumn()` | `UseIdentityColumn()` |
| `ForSqlServerUseIdentityColumns()` (model) | `UseIdentityColumns()` |
| `ForSqlServerHasIndex(...)` | `HasIndex(...)` |

The old names are `[Obsolete]` (CS0618) and still work in 3.1.

```
grep -rnE "ForSqlServer|UseSqlServerIdentityColumn" --include=*.cs
```

## 16. UseRowNumberForPaging is ignored (low, silent)

`UseSqlServer(cs, o => o.UseRowNumberForPaging())` still compiles (CS0618) but the 3.1 SQL
generator always emits `OFFSET ... FETCH`. On SQL Server 2008/2008 R2 or a database at
compatibility level below 110, every `Skip` fails with a syntax error. There is no 3.1 workaround
other than upgrading the server or compatibility level, or rewriting paged queries as raw SQL.

## 17. DbSet<T> implements IAsyncEnumerable<T> (low, compile errors, observed)

**Change:** `DbSet<T>` implements `IAsyncEnumerable<T>` (from Microsoft.Bcl.AsyncInterfaces on
.NET Standard 2.0 / .NET Framework).

**Breaks:**
- With System.Linq.Async referenced, `db.Orders.Where(...)`, `.Select(...)`, `.FirstOrDefault()`
  become ambiguous: CS0121 between `Queryable.X` and `AsyncEnumerable.X`.
- 2.x projects that referenced System.Interactive.Async 3.x (which EF 2.x pulled in and which
  defined its own `IAsyncEnumerable<T>`) can hit type conflicts. Remove that reference.

**Fix:** remove System.Linq.Async from projects that query EF. If another dependency needs it at
runtime in an ASP.NET (System.Web) app, keep the DLL deployed but stop Razor runtime compilation
from referencing it:

```xml
<system.web>
  <compilation>
    <assemblies>
      <remove assembly="System.Linq.Async" />
    </assemblies>
  </compilation>
</system.web>
```

Local workaround: `db.Orders.AsQueryable().Where(...)`.

## 18. Excessive internal service providers is an error (low)

Creating 20 or more internal service providers (usually by building new `DbContextOptions` with
a different `ILoggerFactory`, interceptor instance or `ReplaceService` per context instance) now
throws `ManyServiceProvidersCreatedWarning`. Fix: build options once and reuse them; keep one
static `ILoggerFactory`. Downgrade only if understood:
`w.Log(CoreEventId.ManyServiceProvidersCreatedWarning)`.

## 19. Smaller changes that still bite

| Change | Symptom | Fix / restore |
|---|---|---|
| Backing fields used by default when known | Setter logic (validation, change notification) no longer runs during materialization or save | `modelBuilder.UsePropertyAccessMode(PropertyAccessMode.PreferFieldDuringConstruction)` |
| String/byte[] keys not client-generated | `Unable to track an entity of type 'X' because primary key property 'Id' is null.` on `Add` | `.Property(e => e.Id).ValueGeneratedOnAdd()` |
| `DeleteBehavior.Restrict` no longer changes fix-up | Dependents now keep FK values differently on delete | `DeleteBehavior.ClientNoAction` for old semantics |
| `FindAsync`/`AddAsync` return `ValueTask<T>` | Code passing them to `Task.WhenAll` fails to compile | `.AsTask()` |
| `AddDbContext` no longer calls `AddLogging`/`AddMemoryCache` | Missing service exceptions in apps that relied on it | Register them explicitly |
| Dependent sharing a table with the principal is optional | Owned/table-split dependents can come back null | Handle null; configure as required if that is the intent |
| Owned types cannot be queried without the owner in a tracking query | Exception on `db.Set<Address>()` | Query through the owner |
| `DbContext.Entry` runs a local DetectChanges | Small perf change, different state timing | Usually nothing |
| Multiple ambiguous self-references must be configured | Model build error | Configure each relationship |
| `Microsoft.EntityFrameworkCore.Design` is a development dependency | Design-time types missing at runtime in projects that used them | Reference explicitly with `PrivateAssets` cleared |
| `dotnet ef` no longer in the .NET Core SDK | `dotnet ef` not found | `dotnet tool install --global dotnet-ef --version 3.1.32` |

## 20. Things that are NOT broken (do not "fix" them)

- `HashSet<T>.Contains(x.Id)` translates; any `ICollection<T>` does (`ContainsTranslator`).
  Switching to `List<T>` changes nothing.
- `AsNoTracking()` directly after `FromSqlRaw("EXEC ...")` is fine; it does not compose.
- `DateTime.Now` in a query translates (to `GETDATE()` - server clock, which may be the real
  problem).

## Sources

- https://learn.microsoft.com/ef/core/what-is-new/ef-core-3.x/breaking-changes (source:
  `entity-framework/core/what-is-new/ef-core-3.x/breaking-changes.md` in dotnet/EntityFramework.Docs)
- dotnet/efcore `release/3.1`: `src/EFCore.Relational/Extensions/RelationalQueryableExtensions.cs`,
  `RelationalDatabaseFacadeExtensions.cs`, `src/EFCore/DbContext.cs`, `src/EFCore/ModelBuilder.cs`,
  `src/EFCore/DbQuery\`1.cs`, `src/EFCore/DbSet.cs`, `src/EFCore/Infrastructure/CoreOptionsExtension.cs`
  (default Throw warnings), `src/EFCore/Internal/LazyLoader.cs`, `src/EFCore/ChangeTracking/ChangeTracker.cs`,
  `src/EFCore.SqlServer/Infrastructure/SqlServerDbContextOptionsBuilder.cs` (UseRowNumberForPaging obsolete),
  `src/EFCore.SqlServer/Query/Internal/SqlServerQuerySqlGenerator.cs` (OFFSET/FETCH only),
  `src/EFCore.SqlServer/Extensions/SqlServerIndexBuilderExtensions.cs`, `SqlServerPropertyBuilderExtensions.cs`,
  `src/EFCore.Relational/Extensions/RelationalPropertyExtensions.cs`, `RelationalEntityTypeExtensions.cs`,
  `RelationalModelExtensions.cs`
- https://github.com/dotnet/efcore/issues/19537 (all-null keyless rows)
