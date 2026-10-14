# EF Core 3.1 on .NET Framework: SqlClient, native SNI, IIS, binding redirects

## 1. Platform facts

- EF Core 3.1 targets .NET Standard 2.0, so it runs on .NET Framework. Use 4.7.2 or 4.8: earlier
  versions need extra .NET Standard facade assemblies and are a common source of binding failures.
- EF Core 3.1 is the LAST EF Core for .NET Framework. 5.0 targets .NET Standard 2.1, which .NET
  Framework does not implement. The upgrade path off 3.1 is moving the host to modern .NET, or
  EF6.x.
- 3.1.32 is the final patch. It has been out of support since December 13, 2022: no security fixes.
  Keep SqlClient current independently (section 4) because that is where network-facing fixes land.
- Async: EF's async APIs work on .NET Framework, but in classic ASP.NET never block on them
  (`.Result`, `.Wait()`) - the request `SynchronizationContext` deadlocks. Go async end to end or
  stay sync.

## 2. Package set (SQL Server)

`Microsoft.EntityFrameworkCore.SqlServer 3.1.32` (all EF packages must share the same 3.1.x) brings:

| Package | Version EF asks for | Notes |
|---|---|---|
| Microsoft.EntityFrameworkCore, .Relational, .Abstractions, .Analyzers | 3.1.32 | |
| Microsoft.Data.SqlClient | 1.1.3 (minimum) | Can be raised; see section 4 |
| Microsoft.Data.SqlClient.SNI | [1.1.0, 1.2.0) for SqlClient 1.1.x | .NET Framework native layer |
| Microsoft.Bcl.AsyncInterfaces | 1.1.1 | Defines `IAsyncEnumerable<T>` on .NET Framework |
| Microsoft.Bcl.HashCode | 1.1.1 | |
| Microsoft.Extensions.Caching.Memory, DependencyInjection, Logging | 3.1.32 | Pull in Options, Primitives, *.Abstractions |
| System.Collections.Immutable | 1.7.1 | |
| System.ComponentModel.Annotations | 4.7.0 | |
| System.Diagnostics.DiagnosticSource | 4.7.1 | |

Optional: `Microsoft.EntityFrameworkCore.Proxies 3.1.32` (lazy loading, pulls Castle.Core),
`Microsoft.EntityFrameworkCore.Design`/`.Tools 3.1.32` (development only).

## 3. Microsoft.Data.SqlClient is a different driver

The EF 3.x SQL Server provider uses `Microsoft.Data.SqlClient`. `System.Data.SqlClient` (the
in-box .NET Framework driver) is a different assembly with identically named types.

| Mistake | Result |
|---|---|
| `new System.Data.SqlClient.SqlParameter(...)` passed to FromSqlRaw/ExecuteSqlRaw | Compiles (it is a `DbParameter`); at execution `InvalidCastException: The SqlParameterCollection only accepts non-null Microsoft.Data.SqlClient.SqlParameter type objects, not System.Data.SqlClient.SqlParameter objects.` |
| `catch (System.Data.SqlClient.SqlException ex) when (ex.Number == 1205)` | Never matches - deadlock/retry handling silently disappears |
| `(System.Data.SqlClient.SqlConnection)db.Database.GetDbConnection()` | `InvalidCastException` |
| `SqlDependency`, `SqlBulkCopy` from the old namespace on EF's connection | `InvalidCastException`; use the Microsoft.Data.SqlClient versions |

Rule: in any file that touches EF, `using Microsoft.Data.SqlClient;` and never
`System.Data.SqlClient`. Grep before every release:

```
grep -rn "System.Data.SqlClient" --include=*.cs --include=*.cshtml
```

Other code in the process (an older library, a reporting tool) may keep using
System.Data.SqlClient on its own connections; the two drivers coexist.

## 4. Choosing a SqlClient version newer than 1.1.x

EF 3.1 compiles against SqlClient 1.x and only uses stable APIs (`SqlConnection`,
`SqlConnectionStringBuilder.InitialCatalog`/`MultipleActiveResultSets`, `SqlException`), so a newer
SqlClient works when the binding redirect points at it. Observed working in production: EF Core
3.1.32 with Microsoft.Data.SqlClient 6.0.2 on .NET Framework 4.8.

Target frameworks per SqlClient line (from the nuspecs):

| SqlClient | .NET Framework asset | Usable from a .NET Core 3.1 host |
|---|---|---|
| 1.1.x | net46 | yes |
| 2.1.x, 3.1.x | net46 / net461 | yes |
| 4.0.x | net461 | yes |
| 5.1.x, 5.2.x | net462 | yes (netstandard2.0 asset) |
| 6.0.x | net462 | NO - .NET Standard support was removed |
| 6.1.x | net462 | netstandard2.0 asset is back (not verified on a 3.1 runtime) |

Breaking changes along the way that matter here:

- **4.0: `Encrypt` defaults to `true`.** A server without a certificate the client trusts fails
  at login: `A connection was successfully established with the server, but then an error
  occurred during the login process. (provider: SSL Provider, error: 0 - The certificate chain was
  issued by an authority that is not trusted.)` See section 5.
- **5.0: `SqlConnectionStringBuilder.Encrypt` changed from `bool` to `SqlConnectionEncryptOption`**
  (binary breaking; implicit conversions keep source compatible). Recompile application code against
  the new version. EF 3.1 does not touch `Encrypt`, so it is unaffected. New values:
  `Optional` (= false), `Mandatory` (= true), `Strict` (TDS 8, TrustServerCertificate ignored).
- **5.0: .NET Framework 4.6.1 dropped** (net462 minimum).
- **5.1: Windows ARM64 support for .NET Framework** (`Microsoft.Data.SqlClient.SNI.arm64.dll`);
  ARM32 binaries removed.
- **6.0: .NET Standard removed**; dependency closure grows (Azure.Identity, Microsoft.Identity.Client,
  Microsoft.IdentityModel.*, System.Text.Json, System.Memory and friends, Microsoft.Extensions.Caching.Memory 8.x on net462).

Upgrading SqlClient on .NET Framework therefore means:

1. Add the new `Microsoft.Data.SqlClient` and the matching `Microsoft.Data.SqlClient.SNI` package
   explicitly.
2. Add every new transitive dependency (packages.config does not resolve them automatically on update in
   all tooling; check the nuspec).
3. Redirect `Microsoft.Data.SqlClient` (and every dependency whose assembly version changed) to the
   version of the DLL actually in bin. Read assembly versions from the DLL, not the package version:
   `[Reflection.AssemblyName]::GetAssemblyName("bin\Microsoft.Data.SqlClient.dll").Version`.
4. Verify the bin closure (section 8) and smoke-test a query, a FromSql call and a SaveChanges.

## 5. Encryption: keep old behaviour, or do it properly

Pick one explicitly in the connection string; do not rely on the default of whichever SqlClient
happens to be in bin.

| Goal | Connection string | Risk |
|---|---|---|
| Correct | `Encrypt=True;` with a server certificate whose chain the client trusts and whose name matches `Server=` (or `HostNameInCertificate=` on 5.0+) | none |
| Encrypted, certificate not validated | `Encrypt=True;TrustServerCertificate=True;` | Man-in-the-middle possible; acceptable only on a trusted network |
| 2.x / SqlClient 1.x behaviour | `Encrypt=False;` (5.0+: `Encrypt=Optional;`) | Traffic in clear unless the server forces encryption |

If the server has Force Encryption on, the connection is encrypted regardless of `Encrypt=False`,
and certificate validation rules then depend on `TrustServerCertificate`.

## 6. Native SNI on .NET Framework

Microsoft.Data.SqlClient on .NET Framework P/Invokes a native SNI library chosen by process
architecture (`RuntimeInformation.ProcessArchitecture`): x86, x64, and arm64 (5.1+).

| SNI package | Files placed in the output folder |
|---|---|
| 1.1.x | `x86\SNI.dll`, `x64\SNI.dll` (subfolders) |
| 2.0.x - 5.0.x | `Microsoft.Data.SqlClient.SNI.x86.dll`, `Microsoft.Data.SqlClient.SNI.x64.dll` |
| 5.1.x and later | the above plus `Microsoft.Data.SqlClient.SNI.arm64.dll` |

The files are copied by an MSBuild `.targets` file inside the SNI package
(`build/net46/` or `build/net462/Microsoft.Data.SqlClient.SNI.targets`, target `CopySNIFiles`), not by a normal
reference. Consequences:

- With packages.config the csproj must contain the `<Import Project="..\packages\Microsoft.Data.SqlClient.SNI.x.y.z\build\net462\Microsoft.Data.SqlClient.SNI.targets" ... />`
  that NuGet adds on install. Hand-edited csproj files or a version bump done by editing
  packages.config lose it, and bin silently has no SNI.
- Deployment scripts that copy only changed or managed files must copy the SNI DLLs too. They are
  native: no assembly version, no binding redirect, and they change only on a package upgrade.
- An IIS app pool with "Enable 32-Bit Applications" runs x86; on ARM64 Windows an x64 pool runs
  under emulation and loads the x64 SNI. Ship every architecture.
- Symptom when missing or wrong: `DllNotFoundException: Unable to load DLL
  'Microsoft.Data.SqlClient.SNI.x64.dll'` (or `BadImageFormatException`) on the first connection.

ASP.NET shadow copying copies managed assemblies to "Temporary ASP.NET Files" but not native
DLLs. Observed working: Microsoft.Data.SqlClient 6.0.2 on .NET Framework 4.8 under IIS, default
shadow copying ON, native ARM64 app pool, with the three SNI DLLs sitting only in `bin` - the SNI
loaded with no preload code. If a host still reports `Unable to load DLL` with the file present,
test with shadow copying disabled (`<hostingEnvironment shadowCopyBinAssemblies="false" />`), or
`LoadLibrary` the SNI by full path (resolved from the assembly's `CodeBase`) before the first
connection.

## 7. packages.config vs PackageReference

| Concern | packages.config | PackageReference |
|---|---|---|
| Transitive dependencies | Written into packages.config at install time; every one must stay listed. Removing a "not directly used" entry removes the DLL from bin | Resolved at restore; not listed |
| Build `.targets` (SNI natives, analyzers) | Imported by an explicit `<Import>` in the csproj; fragile | Imported automatically from `build/` and `buildTransitive/` |
| Version bump | Edit every dependent package manually; NuGet UI update is safer | One line, but transitive versions can float up |
| Flow to consumers | None - each project lists everything | A library's EF/SqlClient references flow to every consumer |

PackageReference leakage: if a class library uses EF internally and should not force its packages
onto consumers, mark them `PrivateAssets="all"`. The consumer then needs its own references for
anything it uses directly, and the runtime still needs the DLLs in the host's bin.

Mixed solutions (a PackageReference library consumed by a packages.config web app): the web app
does NOT get the library's transitive packages. List EF, SqlClient, SNI and all dependencies in
the web app's packages.config explicitly, at versions at least as high as the library's.

## 8. Binding redirect hygiene

Every managed assembly in bin whose version differs from what some other assembly was compiled
against needs a `bindingRedirect` whose `newVersion` equals the version of the DLL actually in bin.
Two failure modes:

- Missing redirect: `FileLoadException: Could not load file or assembly 'X, Version=a.b.c.d' ...
  The located assembly's manifest definition does not match the assembly reference.`
- Stale redirect (`newVersion` older or newer than the DLL in bin): the same exception, usually after
  a package update that changed the DLL but not web.config/app.config.

`<AutoGenerateBindingRedirects>` writes redirects for exe/test projects into `*.dll.config`; web
projects need them in Web.config (Visual Studio offers to add them; build warnings MSB3247/MSB3277
list conflicts).

Verify the bin folder, not the project file. This PowerShell 7 script reads every managed DLL's
references with System.Reflection.Metadata and compares them with bin and the config's redirects:

```powershell
# Test-BinClosure.ps1 - run with pwsh (PowerShell 7+)
param(
    [Parameter(Mandatory = $true)] [string] $Bin,
    [Parameter(Mandatory = $true)] [string] $Config
)

$inBin = @{}
foreach ($file in Get-ChildItem -Path $Bin -Filter *.dll) {
    try {
        $n = [System.Reflection.AssemblyName]::GetAssemblyName($file.FullName)
        $inBin[$n.Name] = $n.Version
    }
    catch {
        # Native DLL (SNI etc.) - no managed identity
    }
}

[xml] $xml = Get-Content -Raw -Path $Config
$ns = @{ b = 'urn:schemas-microsoft-com:asm.v1' }
$redirects = @{}
foreach ($node in (Select-Xml -Xml $xml -XPath '//b:dependentAssembly' -Namespace $ns)) {
    $redirects[$node.Node.assemblyIdentity.name] = [version] $node.Node.bindingRedirect.newVersion
}

$problems = New-Object System.Collections.Generic.List[string]
foreach ($name in $redirects.Keys) {
    if ($inBin.ContainsKey($name) -and $inBin[$name] -ne $redirects[$name]) {
        $problems.Add("STALE REDIRECT  $name -> $($redirects[$name]) but bin has $($inBin[$name])")
    }
}

foreach ($file in Get-ChildItem -Path $Bin -Filter *.dll) {
    $stream = [System.IO.File]::OpenRead($file.FullName)
    try {
        $pe = New-Object System.Reflection.PortableExecutable.PEReader($stream)
        if (-not $pe.HasMetadata) {
            continue
        }
        $md = [System.Reflection.Metadata.PEReaderExtensions]::GetMetadataReader($pe)
        foreach ($h in $md.AssemblyReferences) {
            $ref = $md.GetAssemblyReference($h)
            $refName = $md.GetString($ref.Name)
            if (-not $inBin.ContainsKey($refName)) {
                continue   # framework/GAC, or genuinely missing
            }
            $actual = $inBin[$refName]
            if ($actual -eq $ref.Version) {
                continue
            }
            if (-not $redirects.ContainsKey($refName)) {
                $problems.Add("NO REDIRECT     $($file.Name) wants $refName $($ref.Version), bin has $actual")
            }
            elseif ($actual -lt $ref.Version) {
                $problems.Add("DOWNGRADE       $($file.Name) wants $refName $($ref.Version), bin has only $actual")
            }
        }
    }
    finally {
        $stream.Dispose()
    }
}

$problems | Sort-Object -Unique
if ($problems.Count -eq 0) {
    'Bin closure OK'
}
```

Usage: `pwsh -File Test-BinClosure.ps1 -Bin .\MyWeb\bin -Config .\MyWeb\Web.config`.

Read the output with judgment:

- STALE REDIRECT: always fix.
- DOWNGRADE: the referencing assembly may call members that do not exist in the older DLL; upgrade
  the package in bin.
- NO REDIRECT for `System.*` facades such as `System.Runtime.InteropServices.RuntimeInformation`
  or `System.Net.Http`: on .NET Framework 4.7.1+ many of these unify to the framework's own copy and
  never load from bin. Treat as a warning; confirm with a Fusion log (`fuslogvw`) or by exercising
  the code path.
- Test assemblies and build-only tools in bin produce noise; exclude them or run against the
  deployed folder.

## 9. Microsoft.Extensions.* alignment

EF 3.1 brings Microsoft.Extensions.* 3.1.x. A newer SqlClient (6.x on net462 pulls
Microsoft.Extensions.Caching.Memory 8.x), logging libraries, or other packages pull newer ones.
Rules:

- One version of each `Microsoft.Extensions.*` assembly in bin, with a redirect to it.
- Keep each implementation package at the same major version as its `.Abstractions` package
  (`DependencyInjection` with `DependencyInjection.Abstractions`, `Logging` with
  `Logging.Abstractions`, `Caching.Memory` with `Caching.Abstractions`, `Options`, `Primitives`).
  Mixing majors (an 8.x abstraction with a 3.1 implementation) is where `MissingMethodException`
  and `TypeLoadException` come from.
- Observed working in production on .NET Framework 4.8: EF Core 3.1.32 with the whole
  Microsoft.Extensions.* family raised together to 10.x. This combination is not supported by
  Microsoft; smoke-test context creation, a query, a SaveChanges and logging after every bump.
- Microsoft.Bcl.AsyncInterfaces: one version, redirected. EF 3.1 needs >= 1.1.1; System.Linq.Async
  5.x and newer Microsoft.Extensions.* need higher, which is fine.

## 10. MySQL / MariaDB with Pomelo 3.2.x

`Pomelo.EntityFrameworkCore.MySql 3.2.x` is the EF Core 3.1 provider (it depends on
`Microsoft.EntityFrameworkCore.Relational [3.1.19, 5.0.0)` and `MySqlConnector [0.69.10, 1.0.0)`,
targets .NET Standard 2.0, so it runs on .NET Framework).

```csharp
optionsBuilder.UseMySql(connectionString, mySql => mySql
    .ServerVersion(new Version(8, 0, 34), ServerType.MySql)   // or ServerType.MariaDb
    .EnableRetryOnFailure());
```

- Always set `ServerVersion`. When omitted, 3.2.x assumes MySQL 8.0.21 and generates SQL for it,
  which can fail on 5.7 or MariaDB. `ServerVersion.AutoDetect(connectionString)` exists but opens a
  connection during configuration.
- MySqlConnector 0.69.x is the newest the 3.2.x provider accepts; 1.x+ is outside the dependency
  range.
- Pomelo has its own `EnableRetryOnFailure`; the buffering behaviour in section 11 applies to it as
  well because buffering lives in EF's relational layer.
- `ServerVersion(...)` in 3.2.x is a method on the options builder; in Pomelo 5.0+ it became a
  required argument of `UseMySql`. Do not copy 5.0+ samples.

## 11. EnableRetryOnFailure consequences

```csharp
optionsBuilder.UseSqlServer(cs, sql => sql.EnableRetryOnFailure(
    maxRetryCount: 6, maxRetryDelay: TimeSpan.FromSeconds(30), errorNumbersToAdd: null));
```

(6 retries and 30 seconds are also the defaults.)

Turning it on changes more than retry behaviour:

| Effect | Detail |
|---|---|
| Every query is buffered | `RelationalCommand` wraps the reader in `BufferedDataReader`, which reads ALL rows of ALL result sets into memory before the first row is returned. Streaming large results (`foreach` over millions of rows) is no longer streaming. |
| Every result set is validated | A procedure whose second result set has fewer/different columns, or that returns no result set, throws `The underlying reader doesn't have as many fields as expected.` or `FromSqlMissingColumn`, even though EF would read only the first set. |
| User transactions throw | Any query or SaveChanges while `BeginTransaction`, an enlisted transaction or a `TransactionScope` is active throws `The configured execution strategy 'SqlServerRetryingExecutionStrategy' does not support user initiated transactions. Use the execution strategy returned by 'DbContext.Database.CreateExecutionStrategy()' ...` |
| `ExecuteSqlRaw` is not retried | Only queries and SaveChanges go through the strategy in 3.1 |
| Retried FromSql with `SqlParameter` objects fails | The retry re-adds the same parameter instances: `The SqlParameter is already contained by another SqlParameterCollection.` |
| Non-idempotent work may run twice | A commit that succeeded on the server but whose acknowledgement was lost is retried |

Correct transaction pattern:

```csharp
var strategy = db.Database.CreateExecutionStrategy();
strategy.Execute(() =>
{
    using (var tx = db.Database.BeginTransaction())
    {
        db.Orders.Add(order);
        db.SaveChanges();
        db.Database.ExecuteSqlInterpolated($"EXEC dbo.RecalculateCustomer {order.CustomerId}");
        tx.Commit();
    }
});
```

The delegate may run more than once: create new entities inside it or make it idempotent, and do
not capture state that a failed attempt leaves half-modified.

## Sources

- https://learn.microsoft.com/ef/core/what-is-new/ (release table: EF Core 3.1 .NET Standard 2.0, support ended December 13, 2022)
- https://learn.microsoft.com/ef/core/what-is-new/ef-core-5.0/breaking-changes#netstandard21
- https://learn.microsoft.com/ef/core/what-is-new/ef-core-3.x/breaking-changes#SqlClient
- NuGet nuspecs: `microsoft.entityframeworkcore.sqlserver/3.1.32`, `microsoft.entityframeworkcore/3.1.32`,
  `microsoft.data.sqlclient/{1.1.3,1.1.4,2.1.7,3.1.7,4.0.6,5.1.6,5.2.2,6.0.2,6.1.1}`,
  `microsoft.data.sqlclient.sni/{1.1.0,2.0.0,2.1.0,3.0.0,5.1.1}` (file layout and `.targets`),
  `pomelo.entityframeworkcore.mysql/3.2.7`
- dotnet/SqlClient release notes: `release-notes/4.0/4.0.0.md` (Encrypt default true),
  `release-notes/5.0/5.0.0.md` (SqlConnectionEncryptOption, TDS 8, net462),
  `release-notes/5.1/5.1.0.md` (Windows ARM64 for .NET Framework), `release-notes/6.0/6.0.0.md`
  (.NET Standard removed)
- dotnet/SqlClient `src/Microsoft.Data.SqlClient/src/Interop/Windows/Sni/SniNativeWrapper.cs` (architecture switch)
- dotnet/efcore `release/3.1`: `src/EFCore.SqlServer/Storage/Internal/SqlServerConnection.cs`,
  `src/EFCore/Storage/ExecutionStrategy.cs`, `src/EFCore.Relational/Storage/RelationalCommand.cs`,
  `src/EFCore.Relational/Query/Internal/BufferedDataReader.cs`, `src/EFCore/Query/QueryCompilationContext.cs`
- PomeloFoundation/Pomelo.EntityFrameworkCore.MySql tag `3.2.7`: `src/EFCore.MySql/Infrastructure/MySqlDbContextOptionsBuilder.cs`, `src/EFCore.MySql/Storage/ServerVersion.cs`
