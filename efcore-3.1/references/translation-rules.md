# EF Core 3.1 query translation rules

What the 3.1 relational pipeline (SQL Server provider unless noted) translates, what it throws on,
and the rewrite that returns the same result. Every "throws" row was read from the `release/3.1`
source or proven by a failing test on a real 2.x -> 3.1 migration.

Example model used throughout:

```csharp
public class Customer { public int Id { get; set; } public string Name { get; set; } public ICollection<Order> Orders { get; set; } }
public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public Customer Customer { get; set; }
    public DateTime PlacedOn { get; set; }
    public DateTime? ShippedOn { get; set; }
    public int Quantity { get; set; }
    public decimal Total { get; set; }
    public string Status { get; set; }
    public ICollection<OrderLine> Lines { get; set; }
}
public class OrderLine { public int Id { get; set; } public int OrderId { get; set; } public Order Order { get; set; } public int ProductId { get; set; } }
```

## 1. The two exception messages

Untranslatable expression (most common):

```
System.InvalidOperationException: The LINQ expression 'DbSet<Order>
    .Where(o => Helper(o.Status))' could not be translated. Either rewrite the query in a form
that can be translated, or switch to client evaluation explicitly by inserting a call to either
AsEnumerable(), AsAsyncEnumerable(), ToList(), or ToListAsync(). See
https://go.microsoft.com/fwlink/?linkid=2101038 for more information.
```

Pipeline failure (an operator the navigation expander cannot handle, e.g. `DefaultIfEmpty(value)`
followed by an aggregate):

```
System.InvalidOperationException: Processing of the LINQ expression '...' by
'NavigationExpandingExpressionVisitor' failed. This may indicate either a bug or a limitation in
EF Core.
```

Both mean "rewrite the query". The second is not a bug report candidate for the patterns below.

## 2. Where client evaluation is allowed

Only in the top-level projection (the last `Select`). Inside it, any C# is fine as long as its
inputs are columns:

```csharp
// OK: Describe() runs on the client over columns EF fetched
db.Orders.Where(o => o.Total > 100)
  .Select(o => new { o.Id, Text = Describe(o.Status, o.Total) })
  .ToList();
```

Anywhere else (`Where`, `OrderBy`, `GroupBy`, `Join`, `Any`, aggregate selectors, a `Select`
followed by another operator) the expression must translate.

Do not capture a non-static instance method or an object constant in the final projection of a
hot query: 3.1 throws `Client projection contains reference to constant expression of '...'`
when the projection would keep an instance alive. Use a static method or a local variable.

## 3. Operator table

| Pattern | 3.1 behaviour | Identical-result rewrite |
|---|---|---|
| `Where(o => CSharpMethod(o))` | Throws TranslationFailed | Inline the logic as translatable expressions, or filter server-side as far as possible, then `.AsEnumerable().Where(...)` |
| `Select(...).DefaultIfEmpty(0).Max()` | Throws QueryFailed (NavigationExpandingExpressionVisitor) | `.Max(o => (int?)o.Quantity) ?? 0` |
| `Max`/`Min`/`Average` with non-nullable selector, zero rows | Throws `InvalidOperationException: Sequence contains no elements.` | Cast selector to nullable: `.Max(o => (decimal?)o.Total)` |
| `Sum` over zero rows | Returns 0 | none needed |
| `DefaultIfEmpty()` (no argument) | Translates (LEFT JOIN with an empty row) | - |
| `DefaultIfEmpty(value)` | Not translated | Nullable aggregate + `??`, or `FirstOrDefault()` + `??` |
| `GroupBy(k).Select(g => new { g.Key, n = g.Count() })` | Translates | - |
| `g.Count(predicate)` / `g.LongCount(predicate)` | Throws (explicit TODO in `TranslateCount`) | `g.Sum(x => predicate ? 1 : 0)` |
| `g.Where(p).Count()` | Throws | `g.Sum(x => p ? 1 : 0)` |
| `g.Sum(x => cond ? x.Total : 0)` | Translates (CASE inside SUM) | - |
| `g.Select(x => x.A).Distinct().Count()` | Throws | `.Select(x => new { x.Key, x.A }).Distinct().GroupBy(x => x.Key).Select(g => new { g.Key, n = g.Count() })` (Distinct before GroupBy is pushed into a subquery) |
| `g.First()`, `g.ToList()`, `foreach` over groups, `GroupBy(o => o.Customer)` (entity key), `GroupBy` as the final operator | Throws | Filter/project on the server, then `.AsEnumerable().GroupBy(...)`; or group by the key columns and join back |
| `OrderBy(...).Distinct()` | Ordering silently dropped (`ApplyDistinct` calls `ClearOrdering`) | `Distinct().OrderBy(...)` |
| `OrderBy(...).GroupBy(...)` | Ordering dropped (`ApplyGrouping` clears it) | Order after the aggregate projection |
| `Last()` / `LastOrDefault()` without `OrderBy` | Throws | Add `OrderBy`; or `OrderByDescending(...).First()` |
| `Last()` with `OrderBy` | Translates (orderings reversed, TOP 1) | - |
| `Skip(n)` without `OrderBy` | Translates to `ORDER BY (SELECT 1) OFFSET n` - non-deterministic | `OrderBy(key).ThenBy(o => o.Id).Skip(n)` |
| `Take(n)` without `Skip` | `TOP(n)` | - |
| `Skip/Take` (SQL Server) | `OFFSET ... FETCH` (SQL Server 2012+ / compat level 110+). `UseRowNumberForPaging` is ignored | Upgrade the server/compat level |
| `ElementAt`, `ElementAtOrDefault`, `Reverse`, `SkipWhile`, `TakeWhile` | Not translated | `OrderBy..Skip(n).First()`; `OrderByDescending`; rewrite as `Where` |
| `Aggregate(...)` | Not translated | Client-side after filtering |
| `localList.Contains(o.Id)` (List, HashSet, array, any `ICollection<T>`) | Translates to `IN (...)` with the values inlined as constants | For large lists see `performance.md` |
| `localObjects.Any(l => l.Id == o.Id)` / join to an in-memory list of objects | Throws | Project the keys: `var ids = localObjects.Select(l => l.Id).ToList(); ... ids.Contains(o.Id)` |
| `o.Lines.Any()`, `o.Lines.Count()`, `o.Lines.Sum(...)` | Translates (correlated subquery) | - |
| `Select(c => new { c.Id, Orders = c.Orders.Select(o => o.Id).ToList() })` | Translates: one query, LEFT JOIN, ordered by keys | Watch row explosion (`performance.md`) |
| `Include(c => c.Orders.Where(...))` | Throws `InvalidIncludeLambdaExpression` (filtered Include is 5.0+) | Project the filtered collection, or load it separately |
| `Include(...)` followed by a `Select` that does not return the entity | Include silently ignored (the 3.1 `IncludeIgnoredWarning` logger call is commented out, so nothing is logged) | Project the needed related data explicitly |
| `Concat`/`Union`/`Intersect`/`Except` | Translate when both sides have compatible shapes | - |
| Anything over `FromSqlRaw("EXEC ...")` | Throws `FromSqlNonComposable` | `AsEnumerable()` right after `FromSqlRaw` |

## 4. Strings (SQL Server)

| Expression | SQL | Notes |
|---|---|---|
| `a + b`, `string.Concat(a, b)` | `+` | 2-argument Concat only |
| `$"{a} {b}"`, `string.Format`, `string.Join` | Throws | Use `+` |
| `s.Length` | `CAST(LEN(s) AS int)` | LEN ignores trailing spaces |
| `s.ToUpper()`, `s.ToLower()` | `UPPER`, `LOWER` | Comparisons follow the column collation anyway |
| `s.Trim()`, `TrimStart()`, `TrimEnd()` | `LTRIM(RTRIM())` | Only with no arguments (or an empty char array) |
| `s.Substring(start, len)` | `SUBSTRING(s, start + 1, len)` | The one-argument overload is not translated |
| `s.Replace(a, b)` | `REPLACE` | - |
| `s.IndexOf(x)` | `CHARINDEX(x, s) - 1` | single string argument only |
| `s.Contains(x)` | `CHARINDEX(x, s) > 0` (plus `x = N''` check for a parameter) | Not sargable |
| `s.StartsWith(constant)` | `s LIKE N'abc%'` with wildcards escaped | Sargable |
| `s.StartsWith(variable)` | `LEFT(s, LEN(@p)) = @p` | Not sargable; for an index seek use `EF.Functions.Like(s, prefix + "%")` after escaping `%`, `_`, `[` |
| `s.EndsWith(...)` | `LIKE '%abc'` / `RIGHT(...)` | Never sargable |
| `string.IsNullOrEmpty(s)` | `s IS NULL OR s = N''` | - |
| `string.IsNullOrWhiteSpace(s)` | `s IS NULL OR LTRIM(RTRIM(s)) = N''` | - |
| `string.Equals(a, b)`, `a.Equals(b)`, `a == b` | `=` | Uses database collation (usually case-insensitive) |
| `string.Equals(a, b, StringComparison.X)`, `a.Equals(b, StringComparison.X)` | Throws | Use `==`; for case-sensitive compare use a case-sensitive column collation |
| `string.Compare(a, b) > 0`, `a.CompareTo(b) > 0` | `a > b` | Two-argument form only |
| `x.ToString()` | `CONVERT(VARCHAR(n), x)` for numerics, Guid, DateTime, etc. | Format overloads (`ToString("yyyy")`) are not translated |
| `Regex.IsMatch` | Throws | `EF.Functions.Like` or client-side after filtering |

## 5. Dates (SQL Server)

| Expression | SQL |
|---|---|
| `d.Year`, `Month`, `Day`, `DayOfYear`, `Hour`, `Minute`, `Second`, `Millisecond` | `DATEPART(...)` |
| `d.Date` | `CONVERT(date, d)` |
| `d.TimeOfDay` | `CONVERT(time, d)` |
| `d.AddYears/AddMonths(int)`, `AddDays/AddHours/AddMinutes/AddSeconds/AddMilliseconds(double)` | `DATEADD(...)` |
| `DateTime.Now`, `DateTime.UtcNow`, `DateTime.Today` | `GETDATE()`, `GETUTCDATE()`, `CONVERT(date, GETDATE())` - evaluated on the SERVER clock and time zone |
| `d.DayOfWeek` | Not translated |
| `(d2 - d1).Days`, `TotalDays`, any `TimeSpan` arithmetic | Not translated |
| `EF.Functions.DateDiffDay(d1, d2)` and Year/Month/Hour/Minute/Second/Millisecond/Microsecond/Nanosecond | `DATEDIFF(unit, d1, d2)` - counts boundaries crossed, not elapsed whole units |
| `EF.Functions.DateDiffWeek` | Does not exist in 3.1 (5.0+); use `DateDiffDay(...) / 7` |

To compare against "now" in the app's clock, capture it first: `var now = DateTime.Now;` and use
`now` in the query - it becomes a parameter.

`DateDiff*` methods also have a client implementation, so they work in the final projection and
in LINQ to Objects.

## 6. Math, conversions, misc

| Expression | 3.1 |
|---|---|
| `Math.Abs, Ceiling, Floor, Round(x), Round(x, digits), Truncate, Pow, Sqrt, Exp, Log, Sign, Sin, Cos, Tan, Asin, Acos, Atan` | Translated |
| `Math.Max(a, b)`, `Math.Min(a, b)` | Not translated - use `a > b ? a : b` |
| `Convert.ToInt32/ToInt64/ToInt16/ToByte/ToDecimal/ToDouble/ToString(x)` | `CONVERT(type, x)` |
| `(int)x`, `(decimal)x` casts | `CAST` when the types map |
| `e.HasFlag(Flags.A)` | `(e & 1) = 1` |
| `n.HasValue`, `n.Value`, `n.GetValueOrDefault()` | `IS NOT NULL`, column, `COALESCE` |
| `Guid.NewGuid()` | `NEWID()` |
| `EF.Property<T>(e, "Name")` | Column access (for shadow properties) |
| `EF.Functions.Like(s, pattern)` / `Like(s, pattern, escape)` | `LIKE` |
| `EF.Functions.Contains`, `FreeText` | Full-text predicates (needs a full-text index) |
| `EF.Functions.IsDate(s)` | `ISDATE(s)` |

## 7. Null semantics

3.1 applies C# null semantics by default:

- `o.Status == null` -> `o.Status IS NULL`.
- `a == b` with both sides nullable -> `(a = b) OR (a IS NULL AND b IS NULL)`.
- `a != b` with nullable sides adds the matching null terms, so `!=` returns rows where exactly one
  side is NULL, like C#.
- A nullable parameter compared with `==` is specialised on its runtime value (`IS NULL` when null);
  that query's SQL is regenerated per execution.

`optionsBuilder.UseSqlServer(cs, o => o.UseRelationalNulls())` switches to plain SQL three-valued
logic (smaller SQL, different results when NULLs are present). Do not flip it on an existing
codebase without re-testing every nullable comparison.

## 8. Subqueries and correlated collections

- Navigations inside predicates/projections become JOINs or correlated subqueries: translate.
- `FirstOrDefault()` on a navigation collection inside a projection translates (the exact SQL shape
  varies; check it with logging).
- A collection projected inside the final `Select` (`c.Orders.Select(...).ToList()`) is fetched in
  the SAME query, ordered by the parent key. Two such collections multiply rows (see
  `performance.md`).
- A local in-memory collection can only be used through `Contains` (or as constants); it cannot be
  joined, grouped or queried with `Any(predicate)`.

## 9. Finding the offending expression

The exception text prints the expression tree. Look for the first node that is not a column, a
parameter or a translatable member/method from the tables above. Then:

1. Can it be rewritten with translatable pieces? Do that.
2. Is the remaining work tiny after server-side filtering? Put `.AsEnumerable()` right after the
   last translatable operator.
3. Does it need SQL features LINQ cannot express? Use a keyless type over `FromSqlRaw` (a `SELECT`
   is composable) or a view.

## Sources

- `src/EFCore/Properties/CoreStrings.resx` (`TranslationFailed`, `QueryFailed`, `NoElements`,
  `ClientProjectionCapturingConstantInTree`, `InvalidIncludeLambdaExpression`)
- `src/EFCore.Relational/Query/RelationalSqlTranslatingExpressionVisitor.cs` (grouping aggregates,
  `TranslateCount` TODO for predicates)
- `src/EFCore.Relational/Query/RelationalQueryableMethodTranslatingExpressionVisitor.cs`
  (`TranslateDefaultIfEmpty`, `TranslateLastOrDefault`, `TranslateElementAtOrDefault`,
  `TranslateReverse`, `TranslateSkipWhile`, `TranslateGroupBy`, `AggregateResultShaper`)
- `src/EFCore.Relational/Query/SqlExpressions/SelectExpression.cs` (`ApplyDistinct`,
  `ApplyGrouping`, `PrepareForAggregate`)
- `src/EFCore.Relational/Query/QuerySqlGenerator.cs` (`ORDER BY (SELECT 1)`, composable SQL regex)
- `src/EFCore.Relational/Query/Internal/ContainsTranslator.cs`
- `src/EFCore.SqlServer/Query/Internal/SqlServerStringMethodTranslator.cs`,
  `SqlServerDateTimeMemberTranslator.cs`, `SqlServerDateTimeMethodTranslator.cs`,
  `SqlServerMathTranslator.cs`, `SqlServerConvertTranslator.cs`, `SqlServerObjectToStringTranslator.cs`
- `src/EFCore.SqlServer/Extensions/SqlServerDbFunctionsExtensions.cs`
- All paths are on branch `release/3.1` of https://github.com/dotnet/efcore
- https://learn.microsoft.com/ef/core/what-is-new/ef-core-3.x/breaking-changes#linq-queries-are-no-longer-evaluated-on-the-client
