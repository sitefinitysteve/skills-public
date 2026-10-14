---
name: efcore-3.1
description: Use when writing, reviewing or debugging Entity Framework Core 3.1 code - LINQ queries that throw "could not be translated", model mappings (keyless types, nullability, foreign keys), raw SQL and stored-procedure calls (FromSqlRaw, ExecuteSqlRaw), migrating a codebase from EF Core 2.x to 3.x, running EF Core 3.1 on .NET Framework 4.7.2/4.8 (Microsoft.Data.SqlClient, native SNI, binding redirects, IIS), and tuning EF Core 3.1 performance (tracking, cartesian explosion, Contains lists, batching, retries). Also use when an exception mentions NavigationExpandingExpressionVisitor, FromSqlNonComposable, SqlNullValueException "Data is Null", "Invalid column name" on a shadow FK, or "The underlying reader doesn't have as many fields as expected".
---

# EF Core 3.1

EF Core 3.0 rewrote the query pipeline. 3.1 is the last EF Core that runs on .NET Framework
(5.0 targets .NET Standard 2.1). It is out of support (December 13, 2022) but still runs a lot of
production code. Most bugs in a 3.1 codebase come from code written for 2.x semantics, where EF
silently evaluated on the client, tolerated mismatched mappings and resolved identities in
no-tracking queries.

Every rule below was checked against the `release/3.1` source of dotnet/efcore, the official
breaking-changes doc, or a failing test on a real 2.x -> 3.1 migration. Depth, exact exception text and sources live in `references/`:

| Need | Read |
|---|---|
| A query throws, or "what does 3.1 translate?" | `references/translation-rules.md` |
| FromSqlRaw / stored procedures / keyless types | `references/raw-sql-and-stored-procedures.md` |
| Upgrading 2.x code | `references/migrating-from-2x.md` |
| .NET Framework host, SqlClient, IIS, binding redirects, MySQL | `references/sqlclient-and-dotnet-framework.md` |
| Slow queries, memory, round trips | `references/performance.md` |
| Proving a model is right without migrations | `references/testing-the-model.md` |

Snippets are C# 7.3 compatible (the .NET Framework default): no `using var`, no switch expressions.

---

## The rules

### 1. Client evaluation is allowed only in the final Select

Anything else EF cannot turn into SQL throws `InvalidOperationException: The LINQ expression '...'
could not be translated.` 2.x pulled the rows and filtered in memory, logging only a warning.

```csharp
// Throws: FormatName is a C# method
db.Customers.Where(c => FormatName(c) == "Ann Lee").ToList();

// Translates
db.Customers.Where(c => c.FirstName + " " + c.LastName == "Ann Lee").ToList();

// Or move the boundary on purpose - AFTER every filter the server can do
db.Customers.Where(c => c.LastName == "Lee").AsEnumerable()
    .Where(c => FormatName(c) == "Ann Lee").ToList();
```

Never "fix" a translation error by putting `AsEnumerable()`/`ToList()` at the start of the query.
That turns a seek into a full table read.

### 2. Never compose over SQL that does not start with SELECT

3.1 wraps composed raw SQL in a subquery and refuses unless it matches `^\s*?SELECT\b`. `EXEC`,
`WITH` (CTE), `DECLARE` and a leading `--comment` are all non-composable. `FirstOrDefault`,
`Where`, `OrderBy`, `Count`, `Any`, `Include` all compose.

```csharp
// Throws FromSqlNonComposable
db.OrderSummaries.FromSqlRaw("EXEC dbo.GetOrderSummaries {0}", customerId).FirstOrDefault();

// Works: materialize first, compose in memory
db.OrderSummaries.FromSqlRaw("EXEC dbo.GetOrderSummaries {0}", customerId)
    .AsEnumerable().FirstOrDefault();
```

### 3. Raw SQL wrappers return materialized lists, never IQueryable or lazy IEnumerable

A caller composing over a returned `IQueryable` hits rule 2. A lazy `IEnumerable` that was given
`SqlParameter` instances throws on the second enumeration: `ArgumentException: The SqlParameter is
already contained by another SqlParameterCollection` (EF adds the same instance to every command).

```csharp
// Bad
public IEnumerable<OrderSummary> GetSummaries(int id) =>
    db.OrderSummaries.FromSqlRaw("EXEC dbo.GetOrderSummaries @id", new SqlParameter("@id", id));

// Good
public List<OrderSummary> GetSummaries(int id) =>
    db.OrderSummaries.FromSqlRaw("EXEC dbo.GetOrderSummaries @id", new SqlParameter("@id", id)).ToList();
```

### 4. Use Microsoft.Data.SqlClient types, never System.Data.SqlClient

The SQL Server provider uses `Microsoft.Data.SqlClient`. A `System.Data.SqlClient.SqlParameter`
compiles (it is a `DbParameter`) and fails at runtime with `InvalidCastException: The
SqlParameterCollection only accepts non-null Microsoft.Data.SqlClient.SqlParameter type objects`.
`catch (System.Data.SqlClient.SqlException)` silently stops matching anything.

```csharp
using System.Data.SqlClient;      // bad
using Microsoft.Data.SqlClient;   // good
```

Grep: `System\.Data\.SqlClient` must have zero hits in code that talks to EF.

### 5. Use the 3.x raw SQL methods; never concatenate values into SQL

`FromSql` is `[Obsolete(error: true)]` (CS0619 - will not compile). `ExecuteSqlCommand` is only
`[Obsolete]` (CS0618 warning), so it survives upgrades unnoticed - replace it anyway.

```csharp
db.Database.ExecuteSqlCommand("UPDATE dbo.Orders SET Status = 2 WHERE Id = " + id);  // bad
db.Database.ExecuteSqlInterpolated($"UPDATE dbo.Orders SET Status = 2 WHERE Id = {id}"); // good: parameterized
db.Database.ExecuteSqlRaw("UPDATE dbo.Orders SET Status = 2 WHERE Id = {0}", id);     // good
```

`FromSqlRaw`/`FromSqlInterpolated` exist only on `DbSet<T>` (the query root).

### 6. Keyless results: HasNoKey + DbSet, and expect null rows

`DbQuery<T>`, `modelBuilder.Query<T>()` and `context.Query<T>()` are obsolete (warnings). A keyless
row whose columns are ALL NULL materializes as `null`, not as an object with null properties.

```csharp
modelBuilder.Entity<OrderSummary>().HasNoKey();          // was: modelBuilder.Query<OrderSummary>()
public DbSet<OrderSummary> OrderSummaries { get; set; }  // was: DbQuery<OrderSummary>

var rows = db.OrderSummaries.FromSqlRaw("EXEC dbo.GetOrderSummaries {0}", id).ToList();
var safe = rows.Where(r => r != null).ToList();          // all-NULL rows come back as null
```

### 7. Model nullability and CLR types must match the column exactly

3.x emits no `IsDBNull` check for a property mapped as required, and reads with the typed getter
for the mapped type. A NULL in a required column throws `SqlNullValueException: Data is Null`; a
`bigint` read into an `int` throws `InvalidCastException: Specified cast is not valid`.

```csharp
// Bad: column is NULLable, model says required
b.Property(x => x.Notes).IsRequired();
// Good
b.Property(x => x.Notes);                       // string is nullable by default

// Bad: ROW_NUMBER() / COUNT_BIG() are bigint
public int RowNum { get; set; }
// Good
public long RowNum { get; set; }
```

`int` properties over NULLable `int` columns need `int?`. The exception does not name the
property; find it by comparing the model with INFORMATION_SCHEMA (`references/testing-the-model.md`).

### 8. Configure relationships whose FK name equals the principal key name

3.0 dropped the convention that matched a dependent property named exactly like the principal's
key. EF then invents a shadow FK (navigation name + key name) and SQL fails with
`Invalid column name 'BuyerCustomerId'`.

```csharp
// Order.CustomerId, Customer key = CustomerId, navigation Order.Buyer -> shadow "BuyerCustomerId"
modelBuilder.Entity<Order>()
    .HasOne(o => o.Buyer).WithMany(c => c.Orders)
    .HasForeignKey(o => o.CustomerId);
```

Detect: any property where `IsShadowProperty() && IsForeignKey()` is a bug unless intended.

### 9. Aggregates over possibly-empty sets: cast to nullable

`Max`/`Min`/`Average` on a non-nullable selector over zero rows throws `Sequence contains no
elements`. `DefaultIfEmpty(value)` is not translated at all.

```csharp
// Throws (QueryFailed in NavigationExpandingExpressionVisitor)
var top = db.Orders.Where(o => o.CustomerId == id).Select(o => o.Quantity).DefaultIfEmpty(0).Max();
// Translates
var top = db.Orders.Where(o => o.CustomerId == id).Max(o => (int?)o.Quantity) ?? 0;
```

`Sum` returns 0 on empty, no cast needed.

### 10. GroupBy translates only to "key + aggregates of the group"

Allowed on a group: `g.Key`, `g.Count()`, `g.LongCount()`, `g.Sum(sel)`, `g.Min(sel)`,
`g.Max(sel)`, `g.Average(sel)`. Not allowed: `g.Count(predicate)`, `g.Where(...)`, `g.Select(...)
.Distinct()`, `g.First()`, iterating the group, grouping by an entity.

```csharp
// Throws
.GroupBy(o => o.CustomerId).Select(g => new { g.Key, Late = g.Count(o => o.IsLate) })
// Translates
.GroupBy(o => o.CustomerId).Select(g => new { g.Key, Late = g.Sum(o => o.IsLate ? 1 : 0) })

// Distinct count: distinct the pairs first, then group
db.OrderLines.Select(l => new { l.Order.CustomerId, l.ProductId }).Distinct()
    .GroupBy(x => x.CustomerId).Select(g => new { g.Key, Products = g.Count() });
```

If the groups themselves are needed, filter and project on the server, then `.AsEnumerable()`
immediately before `GroupBy`.

### 11. Ordering: Distinct drops it, paging needs a unique one, Last needs one

`OrderBy(...).Distinct()` silently discards the ordering (no exception). `GroupBy` also clears it.
`Skip` with no `OrderBy` emits `ORDER BY (SELECT 1)` - pages are non-deterministic. `Last`/
`LastOrDefault` without `OrderBy` throws.

```csharp
q.OrderBy(o => o.PlacedOn).Distinct().Take(10);                 // bad: order lost before Take
q.Distinct().OrderBy(o => o.PlacedOn).ThenBy(o => o.Id).Take(10); // good

q.OrderBy(o => o.PlacedOn).Skip(40).Take(20);                    // bad: ties shuffle between pages
q.OrderBy(o => o.PlacedOn).ThenBy(o => o.Id).Skip(40).Take(20);  // good: unique tiebreaker
```

### 12. Large Contains lists are inlined as constants

`ids.Contains(x.Id)` translates (any `ICollection<T>` - `List`, `HashSet`, arrays), but 3.1 writes
the values into the SQL text and disables its own SQL cache for that query. Every distinct list is
a new SQL Server plan; thousands of values hit the batch-size limit.

```csharp
db.Orders.Where(o => hugeIdList.Contains(o.Id)).ToList();   // bad for > a few hundred values
// good: one parameter, reusable plan (SQL Server 2016+)
var json = "[" + string.Join(",", hugeIdList) + "]";
db.Orders.FromSqlRaw(
    "SELECT o.* FROM dbo.Orders o WHERE o.Id IN (SELECT CAST([value] AS int) FROM OPENJSON({0}))", json)
  .ToList();
```

Do NOT tell anyone to switch `HashSet` to `List` for translation - both translate.

### 13. Include of collections is one query: watch the cartesian product

3.0 replaced "one query per Include" with one JOINed query. Two sibling collection Includes return
`parents x childrenA x childrenB` rows. There is no `AsSplitQuery()` in 3.1 (added in 5.0); split
by hand:

```csharp
var blogs = db.Blogs.Where(b => b.OwnerId == ownerId).ToList();   // tracking
db.Posts.Where(p => p.Blog.OwnerId == ownerId).Load();             // fix-up fills blog.Posts
db.Tags.Where(t => t.Blog.OwnerId == ownerId).Load();              // fix-up fills blog.Tags
```

Filtered Include (`Include(b => b.Posts.Where(...))`) does not exist in 3.1 either.

### 14. AsNoTracking no longer resolves identity

No-tracking queries now create one instance per row occurrence: 100 orders of the same customer
give 100 `Customer` objects. Reference-equality checks, `Distinct()` on entities in memory and
graph mutations behave differently from 2.x. Project to DTOs for read paths; use tracking where
shared instances matter.

### 15. Cascades run immediately; lazy loading after dispose throws

```csharp
db.Remove(customer);   // 3.x: tracked required dependents are marked Deleted NOW, not at SaveChanges
// Restore 2.x timing if code inspects state between Remove and SaveChanges:
db.ChangeTracker.CascadeDeleteTiming = CascadeTiming.OnSaveChanges;
db.ChangeTracker.DeleteOrphansTiming = CascadeTiming.OnSaveChanges;
```

Lazy loading on a disposed context (`LazyLoadOnDisposedContextWarning`) or on an entity from an
`AsNoTracking` query (`DetachedLazyLoadingWarning`) throws by default. Fix with `Include`/projection.
Downgrade only knowingly: `o.ConfigureWarnings(w => w.Ignore(CoreEventId.LazyLoadOnDisposedContextWarning))`.

### 16. EnableRetryOnFailure buffers everything and forbids bare transactions

With a retrying execution strategy every query result is fully buffered before the first row is
returned, every result set of a stored procedure is read and checked against the entity's columns,
and any query or `SaveChanges` inside a user transaction (`BeginTransaction` or a `TransactionScope`)
throws unless the whole unit runs inside the strategy. `ExecuteSqlRaw` is not retried at all.

```csharp
var strategy = db.Database.CreateExecutionStrategy();
strategy.Execute(() =>
{
    using (var tx = db.Database.BeginTransaction())
    {
        db.SaveChanges();
        tx.Commit();
    }
});
```

A procedure that `RETURN`s before any `SELECT` throws `The underlying reader doesn't have as many
fields as expected` under buffering. Give every code path a result set with the full column list.

### 17. DbSet<T> is IAsyncEnumerable<T>: System.Linq.Async makes Where ambiguous

`db.Orders.Where(...)` fails with CS0121 when System.Linq.Async (or old System.Interactive.Async)
is referenced, because both `Queryable.Where` and `AsyncEnumerable.Where` apply.

```csharp
db.Orders.AsQueryable().Where(o => o.Total > 100);   // local workaround
```

Real fix: remove the reference from projects that use EF. If it must stay deployed for something
else in an ASP.NET app, also exclude it from runtime view compilation:
`<compilation><assemblies><remove assembly="System.Linq.Async" /></assemblies></compilation>`.

### 18. Temporary keys are not written to the entity

After `Add`, a store-generated `Id` stays `0` until `SaveChanges`. Code that copies `parent.Id`
into `child.ParentId` before saving now writes 0. Set the navigation instead:

```csharp
child.ParentId = parent.Id;   // bad before SaveChanges: 0
child.Parent = parent;        // good: EF fixes the FK at save
```

### 19. On .NET Framework, 3.1 is the end of the line - pin the stack deliberately

- EF Core 3.1.32 + Microsoft.Data.SqlClient: EF pulls SqlClient 1.1.x; newer (up to 6.x) works with
  binding redirects. SqlClient 4.0+ defaults `Encrypt=true`: add a trusted certificate or set
  `Encrypt` / `TrustServerCertificate` explicitly.
- SqlClient needs its native SNI DLL in `bin` for the process architecture (arm64 for .NET
  Framework only from SNI 5.1).
- Every assembly version mismatch needs a binding redirect; verify the bin folder, not the csproj.

Details: `references/sqlclient-and-dotnet-framework.md`.

### 20. Prove the model with tests, not with production traffic

3.1 surfaces mapping mistakes only when the bad row or code path is hit. Cheap, decisive tests:
build `context.Model` hermetically (keyless types, no shadow FKs, keys present); query every
`DbSet` once against a real database (`Take(1)` - SQL Server rejects unknown columns even with zero
rows); compare model nullability with INFORMATION_SCHEMA; call every read-only stored-procedure
wrapper with dummy arguments (3.1 resolves FromSql column names immediately after `ExecuteReader`,
so a missing column fails even with zero rows); enumerate wrapper results twice. See
`references/testing-the-model.md`.

---

## Triage: exception -> rule

| Exception text (start) | Rule |
|---|---|
| `The LINQ expression '...' could not be translated` | 1, 9, 10, 11 |
| `Processing of the LINQ expression '...' by 'NavigationExpandingExpressionVisitor' failed` | 9, 10 |
| `FromSqlRaw or FromSqlInterpolated was called with non-composable SQL` | 2, 3 |
| `The SqlParameter is already contained by another SqlParameterCollection` | 3 |
| `The SqlParameterCollection only accepts non-null ...SqlParameter type objects` | 4 |
| `Data is Null. This method or property cannot be called on Null values.` | 7 |
| `Specified cast is not valid.` (during materialization) | 7 |
| `Invalid column name '<Nav><Key>'` | 8 |
| `Sequence contains no elements.` from Max/Min/Average | 9 |
| `The required column '...' was not present in the results of a 'FromSql' operation` | 16, raw SQL ref |
| `The underlying reader doesn't have as many fields as expected.` | 16 |
| `An attempt was made to lazy-load navigation property ...` | 15 |
| `The configured execution strategy '...' does not support user initiated transactions` | 16 |
| CS0121 ambiguous `Queryable.Where` / `AsyncEnumerable.Where` | 17 |
| CS0619 `FromSql` is obsolete | 5 |
| `The entity type '...' requires a primary key to be defined` | 6 |

## APIs that do NOT exist in 3.1 (do not suggest them)

`AsSplitQuery`, filtered `Include`, `ToQueryString`, `LogTo`, `IDbContextFactory` /
`AddDbContextFactory` / `AddPooledDbContextFactory`, `EF.Functions.DateDiffWeek`,
`DateFromParts`, `Math.Max`/`Math.Min` translation, `ExecuteUpdate`/`ExecuteDelete`,
`SqlQuery<T>` for unmapped types, `HasPrecision`, `UseRowNumberForPaging` (compiles with a
warning but the generator always emits OFFSET/FETCH).

## Review checklist

- [ ] No `AsEnumerable`/`ToList` before server-side filters; client logic only in the last `Select`.
- [ ] Every `FromSqlRaw("EXEC ...")` is followed directly by `ToList()`/`AsEnumerable()`.
- [ ] Raw SQL wrappers return `List<T>`/arrays; no `IQueryable` over keyless or EXEC results.
- [ ] No `System.Data.SqlClient` anywhere near EF.
- [ ] Keyless results tolerate `null` rows.
- [ ] Required/nullable and CLR types match the columns (`bigint` -> `long`, NULL -> nullable).
- [ ] No unintended shadow FKs.
- [ ] Aggregates over maybe-empty sets use `(T?)` casts.
- [ ] `Distinct` before `OrderBy`; paging has a unique tiebreaker.
- [ ] No sibling collection Includes on large parents; no unbounded `Contains` lists.
- [ ] Transactions run inside the execution strategy when retries are on.
