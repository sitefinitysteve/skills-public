# EF Core 3.1 performance

Measure first: log the SQL (section 10), find the slow statement, then pick the fix. The usual
order of payoff: fewer rows and columns (projection, server-side filters) > fewer round trips (N+1)
> no row explosion (split collections) > less tracking > plan reuse > micro-tuning.

## 1. Read-only queries: AsNoTracking or projection

```csharp
// Tracking: every entity is snapshotted for change detection
var orders = db.Orders.Where(o => o.CustomerId == id).ToList();

// No tracking: cheaper materialization, no snapshot
var orders = db.Orders.AsNoTracking().Where(o => o.CustomerId == id).ToList();

// Best for read paths: only the columns needed, never tracked
var rows = db.Orders
    .Where(o => o.CustomerId == id)
    .Select(o => new OrderRow { Id = o.Id, PlacedOn = o.PlacedOn, Total = o.Total, CustomerName = o.Customer.Name })
    .ToList();
```

3.x no-tracking queries do NOT resolve identity: the same customer appearing on 1,000 orders is
materialized 1,000 times. For wide graphs that costs more memory than tracking. Either project to
DTOs (no entity instances at all) or use tracking when the graph has heavy duplication.

Context-wide default for read-heavy contexts:

```csharp
db.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
// opt back in per query with .AsTracking()
```

Lazy-loading proxies on a no-tracking entity throw (`DetachedLazyLoadingWarning`), so no-tracking
plus lazy loading is not an option.

## 2. N+1 queries

Symptom: one query per row, often from lazy loading or a query inside a loop.

```csharp
// N+1: one query for orders, then one per order
foreach (var o in db.Orders.Where(o => o.PlacedOn >= since).ToList())
{
    var name = db.Customers.Single(c => c.Id == o.CustomerId).Name;
}

// One query
var rows = db.Orders.Where(o => o.PlacedOn >= since)
    .Select(o => new { o.Id, CustomerName = o.Customer.Name })
    .ToList();
```

Detection: count commands per request with the interceptor in section 10; anything that scales with
the row count is N+1. Lazy loading makes N+1 invisible in code review - prefer explicit `Include`
or projection on hot paths.

## 3. Cartesian explosion (single-query Include)

3.0 loads `Include`d collections in the same SQL statement. Sibling collections multiply:

```csharp
// 1 blog x 1,000 posts x 50 tags = 50,000 rows, every blog/post column repeated
var blog = db.Blogs.Include(b => b.Posts).Include(b => b.Tags).Single(b => b.Id == id);
```

There is no `AsSplitQuery()` in 3.1. Split manually.

Tracking (navigation fix-up stitches the graph):

```csharp
var blogs = db.Blogs.Where(b => b.OwnerId == ownerId).ToList();
db.Posts.Where(p => p.Blog.OwnerId == ownerId).Load();
db.Tags.Where(t => t.Blog.OwnerId == ownerId).Load();
// blogs[i].Posts and blogs[i].Tags are now populated
```

Re-applying the parent's filter (instead of `ids.Contains(...)`) keeps each query parameterized
and plan-stable. Wrap the three queries in a snapshot/serializable transaction if consistency
between them matters.

No tracking (stitch by key):

```csharp
var blogs = db.Blogs.AsNoTracking().Where(b => b.OwnerId == ownerId)
    .Select(b => new BlogDto { Id = b.Id, Name = b.Name }).ToList();
var posts = db.Posts.AsNoTracking().Where(p => p.Blog.OwnerId == ownerId)
    .Select(p => new PostDto { Id = p.Id, BlogId = p.BlogId, Title = p.Title }).ToList()
    .ToLookup(p => p.BlogId);
foreach (var b in blogs)
{
    b.Posts = posts[b.Id].ToList();
}
```

One collection Include (or one projected collection) is usually fine; two or more siblings on a
non-trivial parent set is the trigger to split. Nested single-chain Includes
(`Include(o => o.Lines).ThenInclude(l => l.Product)`) multiply only by the reference, which is
cheap.

## 4. Contains with large lists

`ids.Contains(o.Id)` translates for any `ICollection<T>` (List, HashSet, array). In 3.1 the values
are written into the SQL as constants (`IN (1, 2, 3, ...)`) and EF marks that query as
non-cacheable, so:

- Every different list produces a different SQL text: a new SQL Server compile and plan-cache
  entry each time (plan cache pollution, CPU on the server).
- EF regenerates the SQL on every execution (CPU on the client).
- Very large lists hit SQL Server limits (query processor out of resources, batch size).

Options, best first:

| Approach | Plan reuse | Notes |
|---|---|---|
| Re-express as a join/filter on the server (`o.Customer.Region == region`) | yes | Best when the ids came from another query |
| OPENJSON parameter (SQL Server 2016+, compat level 130+) | yes | One `nvarchar(max)` parameter |
| Table-valued parameter | yes | Needs a user-defined table type |
| Chunk the list (e.g. 500) | partial | Still constants; limits damage, more round trips |

```csharp
// OPENJSON: composable because it starts with SELECT
var json = "[" + string.Join(",", ids) + "]";      // ids are ints: no escaping needed
var orders = db.Orders
    .FromSqlRaw("SELECT o.* FROM dbo.Orders AS o WHERE o.Id IN (SELECT CAST([value] AS int) FROM OPENJSON({0}))", json)
    .AsNoTracking()
    .ToList();

// Table-valued parameter (CREATE TYPE dbo.IntList AS TABLE (Id int PRIMARY KEY))
var table = new DataTable();
table.Columns.Add("Id", typeof(int));
foreach (var i in ids)
{
    table.Rows.Add(i);
}
var tvp = new SqlParameter("@ids", SqlDbType.Structured) { TypeName = "dbo.IntList", Value = table };
var orders2 = db.Orders
    .FromSqlRaw("SELECT o.* FROM dbo.Orders AS o JOIN @ids AS i ON i.Id = o.Id", tvp)
    .ToList();   // materialize once: the SqlParameter cannot be reused
```

Small fixed lists (a handful of status codes) are fine as `Contains`.

## 5. Compiled queries

Every LINQ query is compiled once per shape and cached; the cache lookup itself (hashing the
expression tree) costs measurable CPU on very hot paths. `EF.CompileQuery` skips it:

```csharp
private static readonly Func<ShopContext, int, Customer> CustomerById =
    EF.CompileQuery((ShopContext db, int id) => db.Customers.AsNoTracking().FirstOrDefault(c => c.Id == id));

private static readonly Func<ShopContext, int, IEnumerable<Order>> OrdersByCustomer =
    EF.CompileQuery((ShopContext db, int id) => db.Orders.AsNoTracking().Where(o => o.CustomerId == id));

var c = CustomerById(db, 42);
var list = OrdersByCustomer(db, 42).ToList();
```

Use for a few hot, simple queries only. `EF.CompileAsyncQuery` returns `IAsyncEnumerable<T>` for
sequence queries.

## 6. DbContext lifetime and pooling

- Keep a context short-lived: one per request/unit of work. A long-lived context accumulates
  tracked entities and every `DetectChanges` gets slower.
- `AddDbContextPool<TContext>(options => ..., poolSize: 128)` (128 is the default) reuses context
  instances under ASP.NET Core DI. The context must have a single constructor taking
  `DbContextOptions<TContext>` and must not hold per-request state in fields; `OnConfiguring` must
  not vary per instance.
- Classic ASP.NET / .NET Framework without Microsoft DI: there is no `PooledDbContextFactory` in
  3.1 (added in 6.0). Build `DbContextOptions` ONCE (static) and `new` the context per unit of
  work. Rebuilding options per instance with a different `ILoggerFactory` or interceptor instance
  creates new internal service providers and, at 20, throws `ManyServiceProvidersCreatedWarning`.

## 7. SaveChanges batching

SQL Server provider defaults in 3.1:

| Setting | Default | Meaning |
|---|---|---|
| `MinBatchSize` | 4 | Fewer pending commands than this are sent one by one |
| `MaxBatchSize` | effectively 1000 rows | Hard cap of 1000 commands per batch; also capped below 2100 parameters and a maximum script length |

```csharp
optionsBuilder.UseSqlServer(cs, sql => sql.MaxBatchSize(100).MinBatchSize(1));
```

Wide rows hit the 2100-parameter cap well before 1000 rows. For tens of thousands of inserts, EF
change tracking dominates:

- Use `SqlBulkCopy` (from `Microsoft.Data.SqlClient`) or a TVP-based procedure for bulk loads.
- If staying in EF, save in chunks and dispose/recreate the context every few thousand entities
  (`ChangeTracker.Clear()` does not exist until 5.0).
- Set `db.ChangeTracker.AutoDetectChangesEnabled = false` while adding many entities. `SaveChanges`
  then skips DetectChanges too, which is fine for entities in the Added state; call
  `db.ChangeTracker.DetectChanges()` first if tracked entities were also modified.

## 8. Paging

- SQL Server: `Skip/Take` -> `OFFSET/FETCH`. Always `OrderBy` a unique key (add `ThenBy(x => x.Id)`);
  without `OrderBy` 3.1 emits `ORDER BY (SELECT 1)` and page contents are undefined.
- Deep offsets scan and discard rows. For "next page" UIs use keyset paging:

```csharp
var page = db.Orders.AsNoTracking()
    .Where(o => o.PlacedOn < lastPlacedOn || (o.PlacedOn == lastPlacedOn && o.Id < lastId))
    .OrderByDescending(o => o.PlacedOn).ThenByDescending(o => o.Id)
    .Take(50)
    .ToList();
```

- `Count()` for a total re-runs the filter; cache it or show "more" instead of page numbers on big
  tables.

## 9. Indexes and sargability

EF does not add hints; index use depends on the SQL shape the LINQ produces.

| Pattern | Index-friendly? | Alternative |
|---|---|---|
| `o.Status == status` | yes | |
| `o.Name.StartsWith("abc")` (constant) | yes (`LIKE 'abc%'`) | |
| `o.Name.StartsWith(prefix)` (variable) | no (`LEFT(...) = @p`) | `EF.Functions.Like(o.Name, EscapeLike(prefix) + "%")` |
| `o.Name.Contains(x)`, `EndsWith` | no | Full-text (`EF.Functions.Contains`) or a search column |
| `o.PlacedOn.Date == day` | no (function on column) | `o.PlacedOn >= day && o.PlacedOn < day.AddDays(1)` |
| `o.PlacedOn.Year == 2024` | no | Range on `PlacedOn` |
| `o.Code.ToUpper() == code` | no | Rely on a case-insensitive collation; compare directly |
| Nullable comparisons `a == b` (both nullable) | weaker | Compare to non-null values where the domain allows |

String parameter types follow the column mapping. If a column is `varchar` but the model does not
say so, EF sends `nvarchar` parameters and SQL Server converts the column side
(`CONVERT_IMPLICIT`), which can turn a seek into a scan. Map it:

```csharp
b.Property(c => c.Code).IsUnicode(false).HasMaxLength(20);   // or .HasColumnType("varchar(20)")
```

`TagWith` labels a query in the SQL text so it can be found in Query Store / Extended Events:

```csharp
db.Orders.TagWith("Dashboard: open orders by customer").Where(o => o.Status == "Open").ToList();
```

## 10. Seeing the SQL and finding slow commands

3.1 has no `LogTo` and no `ToQueryString` (both 5.0). Options:

- `UseLoggerFactory(staticLoggerFactory)` with a Microsoft.Extensions.Logging provider; the
  command text is logged under `RelationalEventId.CommandExecuted` (Information). Create the
  factory ONCE (static) - a new factory per context instance creates new internal service
  providers. Add `EnableSensitiveDataLogging()` only in development (it logs parameter values).
- A command interceptor, which also measures duration:

```csharp
public sealed class SlowCommandInterceptor : DbCommandInterceptor
{
    private readonly TimeSpan _threshold;
    private readonly Action<string, TimeSpan> _report;

    public SlowCommandInterceptor(TimeSpan threshold, Action<string, TimeSpan> report)
    {
        _threshold = threshold;
        _report = report;
    }

    public override DbDataReader ReaderExecuted(DbCommand command, CommandExecutedEventData eventData, DbDataReader result)
    {
        Check(command, eventData.Duration);
        return result;
    }

    public override Task<DbDataReader> ReaderExecutedAsync(DbCommand command, CommandExecutedEventData eventData, DbDataReader result, CancellationToken cancellationToken = default(CancellationToken))
    {
        Check(command, eventData.Duration);
        return Task.FromResult(result);
    }

    public override int NonQueryExecuted(DbCommand command, CommandExecutedEventData eventData, int result)
    {
        Check(command, eventData.Duration);
        return result;
    }

    private void Check(DbCommand command, TimeSpan duration)
    {
        if (duration > _threshold)
        {
            _report(command.CommandText, duration);
        }
    }
}

// Register once, on the static options
optionsBuilder.AddInterceptors(new SlowCommandInterceptor(TimeSpan.FromSeconds(1), (sql, d) => Trace.TraceWarning("{0} ms: {1}", d.TotalMilliseconds, sql)));
```

`ReaderExecuted` fires as soon as `ExecuteReader` returns - before rows are read and before
`EnableRetryOnFailure` buffering - so the duration excludes row transfer and materialization. A
query that is fast here but slow end to end is moving too many rows or columns.

## 11. Client evaluation as a hidden cost

The only legal client evaluation is in the final projection. The expensive mistake is an early
`AsEnumerable()`/`ToList()` added to silence a translation error:

```csharp
// Reads the whole table, filters in memory
db.Orders.ToList().Where(o => o.Total > 100 && IsPriority(o)).ToList();

// Server filters first; only the leftover predicate runs in memory
db.Orders.Where(o => o.Total > 100).AsEnumerable().Where(IsPriority).ToList();
```

## 12. Async on .NET Framework

- Async improves scalability (threads freed while waiting on I/O), not single-request latency.
- SqlClient reads large `varbinary(max)`/`nvarchar(max)` values very slowly in async mode
  (dotnet/SqlClient#593, open). For queries returning big blobs/text, use the sync API or move the
  large column to a separate query.
- Never block on async in classic ASP.NET (`.Result`, `.Wait()`): request-context deadlocks.
- A `DbContext` is not thread-safe: never run two queries on the same context concurrently.
  `Task.WhenAll` over one context throws `A second operation started on this context before a
  previous operation completed.`

## 13. Retries cost memory

`EnableRetryOnFailure` buffers every result set fully before returning the first row. Streaming
exports over millions of rows will hold them all in memory. For those, use a separate options instance
without the retrying strategy (buffering is decided when the query is compiled, from whether the
configured strategy retries) and stream with `AsNoTracking()` + `foreach`.

## Sources

- dotnet/efcore `release/3.1`: `src/EFCore.Relational/Query/Internal/ParameterValueBasedSelectExpressionOptimizer.cs`
  (IN values expanded to constants, `canCache = false`), `src/EFCore.SqlServer/Update/Internal/SqlServerModificationCommandBatch.cs`
  (1000 rows, 2100 parameters), `src/EFCore.Relational/Update/Internal/CommandBatchPreparer.cs` (MinBatchSize 4),
  `src/EFCore/Extensions/EntityFrameworkServiceCollectionExtensions.cs` (poolSize 128),
  `src/EFCore.Relational/Diagnostics/DbCommandInterceptor.cs`, `CommandEndEventData.cs` (Duration),
  `src/EFCore/Extensions/EntityFrameworkQueryableExtensions.cs` (TagWith, AsNoTracking; no ToQueryString/AsSplitQuery),
  `src/EFCore/Internal/ServiceProviderCache.cs` (20 providers), `src/EFCore.Relational/Storage/RelationalCommand.cs` (interceptor before buffering), `src/EFCore.SqlServer/Query/Internal/SqlServerStringMethodTranslator.cs`
- https://learn.microsoft.com/ef/core/what-is-new/ef-core-3.x/breaking-changes#eager-loading-single-query
- https://learn.microsoft.com/ef/core/what-is-new/ef-core-3.x/breaking-changes#notrackingresolution
- https://github.com/dotnet/efcore/issues/18022
- https://github.com/dotnet/SqlClient/issues/593
