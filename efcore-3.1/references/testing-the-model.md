# Proving an EF Core 3.1 model is right (without migrations)

Database-first and hand-mapped models drift from the schema silently. 3.1 reports most mistakes
only when the bad row or code path is hit. These tests find them up front. Five layers, cheapest
first:

| # | Test | Needs DB | Catches |
|---|---|---|---|
| 1 | Build `context.Model` | no | Missing keys, unmapped property types, ambiguous relationships, shadow FKs, keyless types not marked |
| 2 | Wrapper signatures | no | `IQueryable` / lazy returns over raw SQL |
| 3 | Query every DbSet once | yes | Wrong table/view/column names, missing columns (even on empty tables) |
| 4 | Model vs INFORMATION_SCHEMA | yes | Required-over-NULL (`Data is Null`), `int` over `bigint`, missing columns |
| 5 | Call every read-only proc wrapper | yes | Missing/renamed result columns (even with zero rows), non-composable use, re-enumeration |

Examples use MSTest and C# 7.3; the logic ports to any framework. `ShopContext` is the context
under test with a constructor taking `DbContextOptions<ShopContext>`.

## Layer 1: hermetic model build

Building the model never opens a connection, so a fake connection string is fine.

```csharp
[TestClass]
public class ModelShapeTests
{
    private static ShopContext NewContext()
    {
        var options = new DbContextOptionsBuilder<ShopContext>()
            .UseSqlServer("Server=model-only;Database=model-only;Integrated Security=true")
            .Options;
        return new ShopContext(options);
    }

    [TestMethod]
    public void Model_builds_and_validates()
    {
        using (var db = NewContext())
        {
            // Throws on validation errors, e.g.
            // "The entity type 'X' requires a primary key to be defined. If you intended to use a keyless entity type call 'HasNoKey()'."
            Assert.IsNotNull(db.Model);
        }
    }

    [TestMethod]
    public void No_unintended_shadow_foreign_keys()
    {
        var allowed = new HashSet<string> { /* "AuditEntry.TenantId" */ };
        using (var db = NewContext())
        {
            var shadowFks = db.Model.GetEntityTypes()
                .SelectMany(t => t.GetProperties())
                .Where(p => p.IsShadowProperty() && p.IsForeignKey())
                .Select(p => p.DeclaringEntityType.DisplayName() + "." + p.Name)
                .Where(n => !allowed.Contains(n))
                .ToList();

            Assert.AreEqual(0, shadowFks.Count, "Shadow FKs (3.x FK convention change?): " + string.Join(", ", shadowFks));
        }
    }

    [TestMethod]
    public void Procedure_result_types_are_keyless()
    {
        // Types only ever filled by FromSqlRaw("EXEC ..."): list them, or find them by convention
        var procResultTypes = new[] { typeof(OrderSummary), typeof(CustomerBalanceRow) };
        using (var db = NewContext())
        {
            foreach (var clr in procResultTypes)
            {
                var et = db.Model.FindEntityType(clr);
                Assert.IsNotNull(et, clr.Name + " is not in the model");
                Assert.IsNull(et.FindPrimaryKey(), clr.Name + " must be HasNoKey()");
            }
        }
    }
}
```

`IsShadowProperty()`, `IsForeignKey()`, `DisplayName()` and `FindPrimaryKey()` are the 3.x
metadata APIs (`Microsoft.EntityFrameworkCore` namespace extension methods).

## Layer 2: wrapper signatures (no DB)

A wrapper returning `IQueryable<T>` over raw SQL invites callers to compose (FromSqlNonComposable),
and one returning a lazy `IEnumerable<T>` re-executes and, with `SqlParameter` instances, throws on
the second enumeration. Ban both by reflection over the classes that hold data access code:

```csharp
[TestMethod]
public void Data_access_methods_do_not_return_IQueryable_of_keyless_types()
{
    using (var db = NewContext())
    {
        var keyless = new HashSet<Type>(db.Model.GetEntityTypes()
            .Where(t => t.FindPrimaryKey() == null)
            .Select(t => t.ClrType));

        var offenders = typeof(OrderRepository).Assembly.GetTypes()
            .Where(t => t.Name.EndsWith("Repository", StringComparison.Ordinal))
            .SelectMany(t => t.GetMethods(BindingFlags.Public | BindingFlags.Instance | BindingFlags.Static | BindingFlags.DeclaredOnly))
            .Where(m => m.ReturnType.IsGenericType
                && (m.ReturnType.GetGenericTypeDefinition() == typeof(IQueryable<>)
                    || m.ReturnType.GetGenericTypeDefinition() == typeof(IEnumerable<>))
                && keyless.Contains(m.ReturnType.GetGenericArguments()[0]))
            .Select(m => m.DeclaringType.Name + "." + m.Name)
            .ToList();

        Assert.AreEqual(0, offenders.Count, "Return List<T> instead: " + string.Join(", ", offenders));
    }
}
```

Adapt the class filter to the codebase. `IEnumerable<T>` is banned for keyless results because the
signature cannot show whether it is materialized; `List<T>`/`IReadOnlyList<T>`/arrays can.

## Layer 3: query every DbSet once (DB required)

A SELECT with an explicit column list is compiled and name-bound by SQL Server before it runs, so
a misspelled table or column fails with `Invalid column name` / `Invalid object name` even when the
table is empty. EF always lists columns explicitly. `Take(1)` keeps the cost negligible.

```csharp
// Inside [TestClass] public class ModelProbeTests
[TestMethod]
public void Every_mapped_table_and_view_can_be_queried()
{
    using (var db = new ShopContext(IntegrationOptions()))
    {
        var existing = ExistingTablesAndViews(db);   // schema.name, from INFORMATION_SCHEMA.TABLES
        var probe = typeof(ModelProbeTests).GetMethod(nameof(Probe), BindingFlags.NonPublic | BindingFlags.Static);
        var failures = new List<string>();

        foreach (var et in db.Model.GetEntityTypes())
        {
            if (et.IsOwned() || et.GetDefiningQuery() != null || et.BaseType != null)
            {
                continue;   // owned types and derived types are covered by their owner/root
            }

            var schema = et.GetSchema() ?? db.Model.GetDefaultSchema() ?? "dbo";
            var name = schema + "." + et.GetTableName();
            if (et.FindPrimaryKey() == null && !existing.Contains(name))
            {
                continue;   // keyless and no table/view: a procedure result type (layer 5)
            }

            try
            {
                probe.MakeGenericMethod(et.ClrType).Invoke(null, new object[] { db });
            }
            catch (TargetInvocationException ex)
            {
                failures.Add(et.DisplayName() + " -> " + name + ": " + ex.InnerException.Message);
            }
        }

        Assert.AreEqual(0, failures.Count, string.Join(Environment.NewLine, failures));
    }
}

private static int Probe<T>(DbContext db) where T : class
{
    return db.Set<T>().AsNoTracking().Take(1).ToList().Count;
}

private static HashSet<string> ExistingTablesAndViews(DbContext db)
{
    var result = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
    var conn = db.Database.GetDbConnection();
    conn.Open();
    try
    {
        using (var cmd = conn.CreateCommand())
        {
            cmd.CommandText = "SELECT TABLE_SCHEMA + '.' + TABLE_NAME FROM INFORMATION_SCHEMA.TABLES";
            using (var r = cmd.ExecuteReader())
            {
                while (r.Read())
                {
                    result.Add(r.GetString(0));
                }
            }
        }
    }
    finally
    {
        conn.Close();
    }
    return result;
}
```

`IntegrationOptions()` points at a real (non-production) database. Using `Take(1)` also reads one
real row when the table has data, which catches `Data is Null` and cast errors for that row - but
an empty table proves only names. Layer 4 proves nullability and types without depending on data.

New DbSets are covered automatically; there is nothing to forget to add.

## Layer 4: model nullability and types vs INFORMATION_SCHEMA (DB required)

3.1 skips the `IsDBNull` check for properties the model says are required and reads with the typed
getter of the mapped CLR type (`RelationalProjectionBindingRemovingExpressionVisitor.CreateGetValueExpression`).
So these are bugs waiting for the first NULL or the first large value:

- property not nullable in the model, column `IS_NULLABLE = 'YES'` -> `SqlNullValueException: Data is Null`
- CLR `int` over `bigint`, `short` over `int`, `bool` over `int`, `double` over `decimal` -> `InvalidCastException`

```csharp
[TestMethod]
public void Model_matches_column_nullability_and_integer_widths()
{
    using (var db = new ShopContext(IntegrationOptions()))
    {
        var columns = LoadColumns(db);   // key: "schema.table.column" -> (isNullable, dataType)
        var problems = new List<string>();

        foreach (var et in db.Model.GetEntityTypes().Where(t => !t.IsOwned() && t.GetDefiningQuery() == null))
        {
            var table = (et.GetSchema() ?? db.Model.GetDefaultSchema() ?? "dbo") + "." + et.GetTableName();
            foreach (var p in et.GetProperties())
            {
                var key = table + "." + p.GetColumnName();
                Tuple<bool, string> col;
                if (!columns.TryGetValue(key, out col))
                {
                    if (columns.Keys.Any(k => k.StartsWith(table + ".", StringComparison.OrdinalIgnoreCase)))
                    {
                        problems.Add("MISSING COLUMN  " + key);
                    }
                    continue;   // table absent: layer 3 reports it; keyless proc types land here
                }

                if (!p.IsNullable && col.Item1)
                {
                    problems.Add("REQUIRED/NULL   " + key + " (model required, column NULLable)");
                }

                var clr = Nullable.GetUnderlyingType(p.ClrType) ?? p.ClrType;
                string expected;
                if (IntegerWidths.TryGetValue(clr, out expected) && !string.Equals(expected, col.Item2, StringComparison.OrdinalIgnoreCase))
                {
                    problems.Add("TYPE            " + key + " (CLR " + clr.Name + ", column " + col.Item2 + ")");
                }
            }
        }

        Assert.AreEqual(0, problems.Count, string.Join(Environment.NewLine, problems));
    }
}

private static readonly Dictionary<Type, string> IntegerWidths = new Dictionary<Type, string>
{
    { typeof(long), "bigint" },
    { typeof(int), "int" },
    { typeof(short), "smallint" },
    { typeof(byte), "tinyint" },
    { typeof(bool), "bit" },
};

private static Dictionary<string, Tuple<bool, string>> LoadColumns(DbContext db)
{
    var result = new Dictionary<string, Tuple<bool, string>>(StringComparer.OrdinalIgnoreCase);
    var conn = db.Database.GetDbConnection();
    conn.Open();
    try
    {
        using (var cmd = conn.CreateCommand())
        {
            cmd.CommandText =
                "SELECT TABLE_SCHEMA + '.' + TABLE_NAME + '.' + COLUMN_NAME, IS_NULLABLE, DATA_TYPE " +
                "FROM INFORMATION_SCHEMA.COLUMNS";
            using (var r = cmd.ExecuteReader())
            {
                while (r.Read())
                {
                    result[r.GetString(0)] = Tuple.Create(r.GetString(1) == "YES", r.GetString(2));
                }
            }
        }
    }
    finally
    {
        conn.Close();
    }
    return result;
}
```

Notes:

- Views report nullability as SQL Server infers it, which is often `YES` for computed expressions.
  Fix the view (`ISNULL(x, 0)`) or the model; do not blanket-ignore views.
- A value converter or an enum changes the CLR/provider type pairing; skip properties with
  `p.GetValueConverter() != null` in the type check.
- The reverse direction (model nullable, column NOT NULL) is harmless for reads.

## Layer 5: call every read-only stored-procedure wrapper (DB required)

Verified in the 3.1 source (`QueryingEnumerable.Enumerator.InitializeReader`): for a
non-composed FromSql query, EF resolves every expected column name against the reader
(`BuildIndexMap`) immediately after `ExecuteReader` returns, before reading the first row. With
`EnableRetryOnFailure`, `BufferedDataReader.InitializeFields` performs the same check per result
set. Therefore a wrapper called with arguments that match nothing still fails if the procedure's
result set lacks a column the keyless type maps:

```
InvalidOperationException: The required column 'LastTotal' was not present in the results of a 'FromSql' operation.
```

What zero rows do NOT prove: column types and nullability (those fail only when a row is read -
layer 4 covers tables, read the procedure's SELECT for procedures).

```csharp
[TestMethod]
public void Read_only_procedure_wrappers_return_every_mapped_column()
{
    using (var db = new ShopContext(IntegrationOptionsWithoutRetry()))
    {
        var repo = new OrderRepository(db);
        var calls = new Dictionary<string, Func<object>>
        {
            // Dummy arguments: ids that do not exist, wide date ranges, empty strings
            { "GetOrderSummaries", () => repo.GetOrderSummaries(-1, null) },
            { "GetCustomerBalances", () => repo.GetCustomerBalances(DateTime.Today.AddYears(-1), DateTime.Today) },
        };

        var failures = new List<string>();
        using (var tx = db.Database.BeginTransaction())
        {
            foreach (var call in calls)
            {
                try
                {
                    var result = call.Value();
                    var list = result as System.Collections.IEnumerable;
                    if (list != null)
                    {
                        // Re-enumeration: must not re-execute or throw "already contained by another SqlParameterCollection"
                        var first = list.Cast<object>().Count();
                        var second = list.Cast<object>().Count();
                        Assert.AreEqual(first, second, call.Key + " changed between enumerations");
                        Assert.IsInstanceOfType(result, typeof(System.Collections.ICollection), call.Key + " must return a materialized collection");
                    }
                }
                catch (Exception ex)
                {
                    failures.Add(call.Key + ": " + ex.GetBaseException().Message);
                }
            }
            tx.Rollback();   // belt and braces in case a "read-only" procedure writes
        }

        Assert.AreEqual(0, failures.Count, string.Join(Environment.NewLine, failures));
    }
}
```

Rules for this layer:

- Only read-only procedures. Never call a procedure that sends mail, calls external systems or
  writes outside the transaction (linked servers, `xp_`, CLR).
- Use options WITHOUT `EnableRetryOnFailure` here, or run the block inside
  `db.Database.CreateExecutionStrategy().Execute(...)`; a retrying strategy rejects the user
  transaction (`does not support user initiated transactions`).
- A failure with `The required column '<first column>' was not present` for every column usually
  means the procedure took a path that `RETURN`s without a SELECT for these arguments. That is a
  real bug for FromSql callers too (and under a retrying strategy it surfaces as `The underlying
  reader doesn't have as many fields as expected.`): make every path return the full column list.
- To find wrappers nobody listed, grep for `FromSqlRaw("EXEC` / `FromSqlInterpolated($"EXEC` and
  compare the count with the dictionary size in a test.

## Keeping the layers honest

Layers 1-2 are hermetic and belong in the fast suite. Layers 3-5 need a database; tag them so the
fast suite stays green offline, and run them before every release and after every schema change.
A hermetic test that quietly starts touching the database (a probe added to layer 1) is the usual
way that split decays - time the fast suite; nothing in it should take more than tens of
milliseconds.

## Sources

- dotnet/efcore `release/3.1`:
  - `src/EFCore.Relational/Query/Internal/QueryingEnumerable.cs` - `InitializeReader` calls
    `relationalCommand.ExecuteReader(...)` then `BuildIndexMap(_columnNames, _dataReader.DbDataReader)`,
    which throws `FromSqlMissingColumn` for any absent name; no row has been read at that point.
  - `src/EFCore.Relational/Query/Internal/BufferedDataReader.cs` - `InitializeFields` (same check, per result set, plus "doesn't have as many fields")
  - `src/EFCore.Relational/Query/RelationalShapedQueryCompilingExpressionVisitor.RelationalProjectionBindingRemovingExpressionVisitor.cs` - `IsDBNull` only for nullable projections; typed getter from the mapping
  - `src/EFCore.Relational/Extensions/RelationalEntityTypeExtensions.cs`, `RelationalPropertyExtensions.cs`, `RelationalModelExtensions.cs` (`GetTableName`, `GetSchema`, `GetColumnName`, `GetDefaultSchema`)
  - `src/EFCore/Extensions/PropertyExtensions.cs`, `PropertyBaseExtensions.cs`, `EntityTypeExtensions.cs` (`IsForeignKey`, `IsShadowProperty`, `IsOwned`, `DisplayName`, `GetDefiningQuery`)
  - `src/EFCore/Storage/ExecutionStrategy.cs` (`OnFirstExecution` rejects user transactions)
  - `src/EFCore/Properties/CoreStrings.resx` (`EntityRequiresKey`), `src/EFCore.Relational/Properties/RelationalStrings.resx` (`FromSqlMissingColumn`)
