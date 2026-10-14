# Raw SQL and stored procedures in EF Core 3.1

## 1. The four methods

| Method | On | Returns | Parameters |
|---|---|---|---|
| `FromSqlRaw(string sql, params object[] parameters)` | `DbSet<T>` only | `IQueryable<T>` | `{0}` placeholders for plain values, or `DbParameter` instances referenced by name |
| `FromSqlInterpolated(FormattableString sql)` | `DbSet<T>` only | `IQueryable<T>` | `$"... {value}"` - every hole becomes a parameter |
| `Database.ExecuteSqlRaw(string sql, params object[] parameters)` | `DatabaseFacade` | `int` rows affected | same as FromSqlRaw |
| `Database.ExecuteSqlInterpolated(FormattableString sql)` | `DatabaseFacade` | `int` | same as FromSqlInterpolated |

Async versions: `ExecuteSqlRawAsync`, `ExecuteSqlInterpolatedAsync`, and `ToListAsync()` on the
FromSql query.

Obsolete 2.x names:

| 2.x | 3.1 status | Compiler |
|---|---|---|
| `FromSql(...)` | `[Obsolete(..., error: true)]` | CS0619 - build fails |
| `ExecuteSqlCommand(...)`, `ExecuteSqlCommandAsync(...)` | `[Obsolete]` | CS0618 warning only - easy to miss |
| `DbQuery<T>`, `context.Query<T>()`, `modelBuilder.Query<T>()` | `[Obsolete]` | CS0618 warning |

Rules:

- Never build SQL with `+` or `string.Format` from values. `FromSqlRaw($"... {x}")` (an
  interpolated string passed to the RAW method) is plain concatenation and is injectable; the
  analyzer in `Microsoft.EntityFrameworkCore.Analyzers` does not catch every case.
- `FromSqlInterpolated` requires a `FormattableString`. Assigning the interpolated string to a
  `string` variable first turns it into concatenation.
- `ExecuteSqlRaw` returns the driver's rows-affected count. A procedure with `SET NOCOUNT ON`
  returns -1.

## 2. Composability: what may follow FromSqlRaw

3.1 does not parse the SQL. When anything composes over it, EF generates
`SELECT ... FROM (<raw sql>) AS [x] WHERE ...` and first checks the raw SQL against
`^\s*?SELECT\b` (case-insensitive). Anything else throws:

```
System.InvalidOperationException: FromSqlRaw or FromSqlInterpolated was called with
non-composable SQL and with a query composing over it. Consider calling `AsEnumerable` after the
FromSqlRaw or FromSqlInterpolated method to perform the composition on the client side.
```

Non-composable in practice: `EXEC`/`EXECUTE`, a bare procedure name, `WITH cte AS (...)`,
`DECLARE`, `SET`, a leading `--` or `/* */` comment, a leading `(`.

What counts as composing: `Where`, `OrderBy`, `Select` (to anything other than the entity),
`First`/`FirstOrDefault`/`Single`/`Any`/`Count`/`Max`, `Skip`/`Take`, `Include`, `Join`.
What does not: `AsNoTracking()`, `AsTracking()`, `ToList()`, `AsEnumerable()`.

Global query filters compose too: if `T` has a `HasQueryFilter(...)`, the filter becomes a `WHERE`
around the raw SQL and `EXEC` throws. Add `.IgnoreQueryFilters()` (and apply the filter in the procedure) when
calling a procedure into a filtered type.

```csharp
// Throws
var first = db.OrderSummaries
    .FromSqlRaw("EXEC dbo.GetOrderSummaries {0}", customerId)
    .FirstOrDefault();

// Works
var first = db.OrderSummaries
    .FromSqlRaw("EXEC dbo.GetOrderSummaries {0}", customerId)
    .AsEnumerable()
    .FirstOrDefault();

// Works and composes on the server, because it starts with SELECT
var big = db.Orders
    .FromSqlRaw("SELECT * FROM dbo.Orders WITH (NOLOCK) WHERE Total > {0}", 100m)
    .Where(o => o.Status == "Open")
    .OrderBy(o => o.Id)
    .ToList();
```

To compose a CTE on the server, wrap it in a view or an inline table-valued function and select
from that: `FromSqlRaw("SELECT * FROM dbo.fn_OpenOrders({0})", customerId)`.

## 3. Column mapping rules

- The result set must contain a column for EVERY mapped property of `T`, matched by column name
  (case-insensitive), in any order. Extra columns are ignored.
- A missing column throws
  `InvalidOperationException: The required column 'Total' was not present in the results of a 'FromSql' operation.`
- The check runs immediately after `ExecuteReader` returns, BEFORE any row is read
  (`QueryingEnumerable.BuildIndexMap`, and `BufferedDataReader.InitializeFields` when buffering). An
  empty result still fails on a missing column. This is what makes "call every proc wrapper with
  dummy args" a strong test (see `testing-the-model.md`).
- Duplicate column names (case-insensitive) in a non-composed FromSql result: without a retrying
  strategy the index map is built with `ToDictionary` and throws `ArgumentException: An item with
  the same key has already been added`; with buffering the last duplicate silently wins. Alias
  every column uniquely.
- Column TYPES are not checked up front. Each value is read with the typed getter of the property's
  mapping (`GetInt32`, `GetString`, ...), so a mismatch throws only when a row arrives:
  `InvalidCastException: Specified cast is not valid.` Common sources: `ROW_NUMBER()`, `COUNT_BIG()`,
  `@@IDENTITY` and `SCOPE_IDENTITY()` (numeric/bigint) into `int`; `tinyint` into `int`; `decimal`
  into `double`. Fix the property type or `CAST` in the SQL.
- A NULL read into a property mapped as required (or a non-nullable value type) throws
  `System.Data.SqlTypes.SqlNullValueException: Data is Null. This method or property cannot be called on Null values.`
  3.1 emits an `IsDBNull` guard only for columns the model says are nullable.
- `Include` cannot follow `FromSqlRaw("EXEC ...")`. Load related data separately or return it from
  another query.

## 4. Keyless entity types (the usual target of procedures)

```csharp
public class OrderSummary
{
    public int CustomerId { get; set; }
    public string CustomerName { get; set; }
    public int OrderCount { get; set; }
    public decimal? LastTotal { get; set; }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<OrderSummary>().HasNoKey();                       // procedure result
    modelBuilder.Entity<CustomerBalance>().HasNoKey().ToView("v_CustomerBalance"); // view
}

public DbSet<OrderSummary> OrderSummaries { get; set; }
public DbSet<CustomerBalance> CustomerBalances { get; set; }
```

Facts:

- Keyless types are never tracked; `Add`/`Update`/`Remove`/`Find` throw
  (`KeylessTypeTracked`, `InvalidSetKeylessOperation`).
- Forgetting `HasNoKey()` on a type with no `Id`/`<Type>Id` property fails model validation:
  `The entity type 'OrderSummary' requires a primary key to be defined. If you intended to use a keyless entity type call 'HasNoKey()'.`
- `ToView(...)` maps the type for queries and makes migrations ignore it.
- Without `ToView`/`ToTable`, a keyless type still has a default table name (the DbSet name). A
  stray `db.OrderSummaries.ToList()` then queries a table that does not exist. Only ever query it
  through `FromSqlRaw`, and guard with a test.
- ALL-NULL ROW -> `null`. The materializer for a keyless type creates an instance only if at least
  one property value is non-null (an `OR` of not-null checks across every property). An outer-join
  style procedure that returns a row of NULLs yields a `null` element in the list. 2.x returned an
  object with null properties. Filter: `rows.Where(r => r != null)`. Documented as a 3.x issue
  (dotnet/efcore#19537).
- Entity types WITH a key in a no-tracking query return `null` when any key column is NULL.

## 5. Writing a stored-procedure wrapper correctly

```csharp
using Microsoft.Data.SqlClient;   // NOT System.Data.SqlClient
using System.Data;

public List<OrderSummary> GetOrderSummaries(int customerId, DateTime? since)
{
    var pCustomer = new SqlParameter("@CustomerId", SqlDbType.Int) { Value = customerId };
    var pSince = new SqlParameter("@Since", SqlDbType.DateTime2) { Value = (object)since ?? DBNull.Value };

    return _db.OrderSummaries
        .FromSqlRaw("EXEC dbo.GetOrderSummaries @CustomerId, @Since", pCustomer, pSince)
        .ToList()                          // materialize exactly once
        .Where(r => r != null)             // all-NULL rows
        .ToList();
}

// Equivalent without SqlParameter objects - parameters are created per execution
public List<OrderSummary> GetOrderSummaries2(int customerId, DateTime? since) =>
    _db.OrderSummaries
        .FromSqlInterpolated($"EXEC dbo.GetOrderSummaries {customerId}, {since}")
        .ToList();
```

Rules:

1. Return `List<T>` (or an array). Never `IQueryable<T>` (callers compose -> throws) and never a
   lazy `IEnumerable<T>` (re-enumeration -> re-executes, and with `DbParameter` instances throws).
2. Pass `null` as `DBNull.Value` inside a `SqlParameter`. Plain `null` values passed to
   `FromSqlRaw`/`FromSqlInterpolated` are converted to `DBNull.Value` by EF.
3. Set `SqlDbType` (and `Size` for strings) when the procedure parameter is `varchar`, `date`,
   `datetime`, or a decimal with specific precision. Parameters EF creates from plain values use
   EF's default mappings (`nvarchar`, `datetime2`); a `SqlParameter` without `SqlDbType` is
   inferred by SqlClient (`nvarchar`, `datetime`). Either can force implicit conversions.
4. Output parameters: create `new SqlParameter("@Total", SqlDbType.Decimal) { Direction = ParameterDirection.Output, Precision = 18, Scale = 2 }`,
   use `EXEC dbo.Proc @Total OUTPUT`, run with `ExecuteSqlRaw`, read `.Value` after the call. For
   FromSql, output values are only available after the reader is fully consumed (after `ToList()`).
5. Table-valued parameters: `new SqlParameter("@Ids", SqlDbType.Structured) { TypeName = "dbo.IntList", Value = dataTable }`.
   The `DataTable` is reusable; the `SqlParameter` is not (see section 6).

## 6. Re-enumeration hazard

EF adds the caller's `DbParameter` instance directly to each new `DbCommand`
(`RawRelationalParameter.AddDbParameter` -> `command.Parameters.Add(value)`). A `SqlParameter`
remembers its owning collection, so the second command rejects it:

```
System.ArgumentException: The SqlParameter is already contained by another SqlParameterCollection.
```

Triggers: enumerating a returned query twice, `Count()` then `foreach`, a retry by the execution
strategy after a transient error, a compiled query reused with the same parameter object, LINQ to
Objects operators that enumerate twice. Fix: materialize once, or let EF create parameters
(`FromSqlInterpolated`, or plain values with `{0}` placeholders in `FromSqlRaw`).

## 7. Wrong SqlParameter type

```
System.InvalidCastException: The SqlParameterCollection only accepts non-null
Microsoft.Data.SqlClient.SqlParameter type objects, not System.Data.SqlClient.SqlParameter objects.
```

EF treats any `DbParameter` as a raw parameter, so a `System.Data.SqlClient.SqlParameter` compiles
and fails at execution. Detection: `grep -rn "System.Data.SqlClient" --include=*.cs`. Also check
`catch (SqlException)` blocks: with the old namespace imported they never match.

## 8. Procedures that return no result set, or several

| Procedure behaviour | No retry strategy | `EnableRetryOnFailure` (buffered) |
|---|---|---|
| `RETURN` before any `SELECT` on some path | `The required column '<first>' was not present in the results of a 'FromSql' operation.` | `The underlying reader doesn't have as many fields as expected.` |
| First result set is fine, a second result set has different columns | Second set ignored | EVERY result set is buffered and validated against `T` -> throws as above |
| Only `INSERT/UPDATE` (no SELECT) called through FromSql | throws as row 1 | throws as row 1 |
| Rows-affected messages without `SET NOCOUNT ON` | Harmless | Harmless |

Fixes:

- Every code path of a procedure called via `FromSqlRaw` must end in a `SELECT` with the full
  column list, even when empty: `SELECT CAST(NULL AS int) AS CustomerId, ... WHERE 1 = 0`.
- Call procedures that return nothing with `ExecuteSqlRaw`, never with FromSql.
- A procedure that returns several result sets: EF reads only the first. For the others, use ADO.NET
  on `db.Database.GetDbConnection()` with `reader.NextResult()`. Under a retrying strategy make
  sure all result sets are compatible with `T` or split the procedure.

## 9. Raw ADO.NET on the context's connection

```csharp
var conn = db.Database.GetDbConnection();
var mustClose = conn.State != ConnectionState.Open;
if (mustClose)
{
    conn.Open();
}
try
{
    using (var cmd = conn.CreateCommand())
    {
        cmd.CommandText = "dbo.GetDashboard";
        cmd.CommandType = CommandType.StoredProcedure;
        cmd.Transaction = db.Database.CurrentTransaction?.GetDbTransaction();
        cmd.Parameters.Add(new SqlParameter("@CustomerId", SqlDbType.Int) { Value = customerId });
        using (var reader = cmd.ExecuteReader())
        {
            // read set 1, reader.NextResult(), read set 2 ...
        }
    }
}
finally
{
    if (mustClose)
    {
        conn.Close();
    }
}
```

`GetDbTransaction()` lives in `Microsoft.EntityFrameworkCore.Storage` (namespace import needed).
Do not dispose the connection; the context owns it.

## 10. Retries and raw SQL

With `EnableRetryOnFailure`:

- FromSql queries are retried: `QueryingEnumerable` opens the reader inside
  `ExecutionStrategy.Execute`. Combined with section 6, a retry re-adds the same `SqlParameter`
  instances and fails with the "already contained" error instead of recovering.
- `ExecuteSqlRaw`/`ExecuteSqlInterpolated` are NOT wrapped in the strategy in 3.1 - a transient
  error propagates on the first failure. Wrap them yourself:
  `db.Database.CreateExecutionStrategy().Execute(() => db.Database.ExecuteSqlRaw(...));`
  Only do that for idempotent SQL, or inside a transaction in the same `Execute` delegate.
- Any query or `SaveChanges` run while a user transaction (`BeginTransaction`, an enlisted
  transaction, or an ambient `TransactionScope`) is open throws
  `The configured execution strategy 'SqlServerRetryingExecutionStrategy' does not support user initiated transactions.`
  unless the whole unit runs inside `strategy.Execute(...)`.

## Sources

- `src/EFCore.Relational/Query/QuerySqlGenerator.cs` (`_composableSql` regex, `FromSqlNonComposable`)
- `src/EFCore.Relational/Query/Internal/QueryingEnumerable.cs` (`BuildIndexMap` called right after `ExecuteReader`)
- `src/EFCore.Relational/Query/Internal/BufferedDataReader.cs` (buffers every result set, `InitializeFields`, "doesn't have as many fields")
- `src/EFCore.Relational/Query/RelationalShapedQueryCompilingExpressionVisitor.RelationalProjectionBindingRemovingExpressionVisitor.cs` (typed getters, `IsDBNull` only when nullable)
- `src/EFCore/Query/ShapedQueryCompilingExpressionVisitor.cs` (keyless: OR of not-null checks)
- `src/EFCore.Relational/Storage/Internal/RawRelationalParameter.cs`, `Query/Internal/ParameterValueBasedSelectExpressionOptimizer.cs`
- `src/EFCore/Storage/ExecutionStrategy.cs` (`OnFirstExecution` transaction check), `src/EFCore.Relational/Extensions/RelationalDatabaseFacadeExtensions.cs` (`ExecuteSqlRaw` has no strategy wrapper)
- `src/EFCore.Relational/Extensions/RelationalQueryableExtensions.cs`, `RelationalDatabaseFacadeExtensions.cs` (Obsolete attributes)
- `src/EFCore.Relational/Properties/RelationalStrings.resx`, `src/EFCore/Properties/CoreStrings.resx`
- dotnet/SqlClient `src/Microsoft.Data.SqlClient/src/Microsoft/Data/Common/AdapterUtil.cs` and `Resources/Strings.resx` (`ADP_CollectionInvalidType`, `ADP_CollectionIsNotParent`)
- https://learn.microsoft.com/ef/core/what-is-new/ef-core-3.x/breaking-changes#fromsqlsproc
- https://learn.microsoft.com/ef/core/modeling/keyless-entity-types
- https://github.com/dotnet/efcore/issues/19537
