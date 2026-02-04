# VemagEBillProcessor - Improvement Suggestions

## Architecture Overview

```
VemagEBillProcessor
├── VemagMailLib (shared library)
│   ├── Database utilities
│   ├── Configuration parsing
│   ├── Mailbox configuration
│   └── Static SQL strings
└── VemagOauth (OAuth2 library)
    └── Token acquisition for Microsoft 365
```

---

## Critical Issues

### 1. Duplicate DbService Classes

**Location**: `VemagEBillProcessor/DbService.cs` vs `VemagMailerService/DbService.cs`

`VemagEBillProcessor/DbService.cs` is nearly identical to `VemagMailerService/DbService.cs` (~95% code duplication).

**Recommendation**: Extract to `VemagMailLib` as a shared base class:

```csharp
// VemagMailLib/DbServiceBase.cs - expand existing
public abstract class DbServiceBase<TConfig> where TConfig : IMandatConfiguration
{
    protected abstract Dictionary<string, TConfig> Configs { get; }
    public bool Register(TConfig config, string application) { ... }
    public bool Refresh(string name, string application) { ... }
}
```

---

### 2. HttpClient Anti-Pattern

**Location**: `VemagEBillProcessor/EmailProcessor.cs:399-411`

```csharp
private string DownloadHtml(string url)
{
    using (var httpClient = new HttpClient())  // Creates new instance each call
    {
        string result = httpClient.GetStringAsync(url).GetAwaiter().GetResult();
        return result;
    }
}
```

**Issues**:
- Creates new `HttpClient` per request (socket exhaustion)
- Blocking on async code (`.GetAwaiter().GetResult()`)

**Fix**: Use `IHttpClientFactory` or a static/singleton `HttpClient`:

```csharp
private static readonly HttpClient _httpClient = new HttpClient();

private async Task<string> DownloadHtmlAsync(string url)
{
    return await _httpClient.GetStringAsync(url);
}
```

---

### 3. Thread.Abort() Usage

**Location**: `VemagEBillProcessor/EmailProcessor.cs:586`

```csharp
_worker.Abort();  // Deprecated and dangerous
```

**Fix**: Use `CancellationToken` properly (already partially implemented):

```csharp
_cancelTokenSource.Cancel();
_worker.Join(TimeSpan.FromSeconds(30));
```

---

## High Priority

### 4. Inline SQL in StaticStrings

**Location**: `VemagMailLib/StaticStrings.cs`

Contains ~600 lines of raw SQL strings.

**Issues**:
- Hard to maintain
- No IntelliSense
- Prone to SQL injection if misused

**Recommendation**:
- Move to embedded `.sql` resource files
- Or use stored procedures
- Consider Dapper for cleaner data access

---

### 5. Configuration Parsing Fragility

**Location**: `VemagEBillProcessor/Configuration.cs:36-163`

Parses text config files using regex:

```csharp
foreach (var line in lines)
{
    if (StaticStrings.EmailBillRgx.IsMatch(line))
        workdir = StaticStrings.EmailBillRgx.Match(line).Groups[1].Value;
    // ... 10+ more regex checks
}
```

**Recommendation**: Migrate to JSON/YAML configuration:

```json
{
  "EmailBill": {
    "WorkDir": "C:\\path",
    "Mandates": ["01", "02"],
    "ServiceInterval": "00:05:00"
  }
}
```

---

### 6. Exception Handling

**Location**: `VemagEBillProcessor/DbService.cs` (7 occurrences)

Example at line 396:

```csharp
catch (Exception ex)
{
    log.Error($"RefreshInternal: Refresh failed...", ex);
    return false;
}
```

**Issues**:
- Catches all exceptions including `OutOfMemoryException`
- Silent failures - caller just sees `false`

**Fix**: Catch specific exceptions and consider rethrowing or result types:

```csharp
catch (SqlException ex) when (ex.Number == 4060)
{
    Logger.Warn($"Database not found: {ex.Message}");
    return RefreshResult.DatabaseNotFound;
}
catch (SqlException ex)
{
    Logger.Error($"SQL error: {ex.Message}", ex);
    throw;
}
```

---

## Medium Priority

### 7. Package Version Inconsistencies

| Package | VemagMailLib | VemagEBillProcessor |
|---------|--------------|---------------------|
| log4net | 2.0.15 | 3.1.0 |
| MailKit | 4.2.0 | 4.13.0 |

**Fix**: Use a `Directory.Build.props` for consistent versions:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="log4net" Version="3.1.0" />
    <PackageReference Include="MailKit" Version="4.13.0" />
  </ItemGroup>
</Project>
```

---

### 8. Missing Async/Await

**Location**: `VemagEBillProcessor/IMAPProcessor.cs`

Uses async IMAP operations but blocks with `.GetAwaiter().GetResult()`:

```csharp
// Line 383-384
.ExecuteAsync()
.GetAwaiter().GetResult();
```

**Fix**: Propagate async throughout:

```csharp
public override async Task<EBillResult[]> ProcessMessagesAsync()
{
    // ...
    _authResult = await app.AcquireTokenForClient(scopes).ExecuteAsync();
}
```

---

### 9. IMAP Client Not Pooled

**Location**: `VemagEBillProcessor/IMAPProcessor.cs:113`

```csharp
using (_client = GetClient())  // Creates new connection each process loop
```

IMAP connections are expensive. Consider:
- Connection pooling
- Keep-alive with IDLE command
- Reuse client across iterations

---

### 10. Hardcoded SQL Table Names

**Location**: `VemagMailLib/StaticStrings.cs`

Contains hardcoded schema references:

```sql
select ... from VFinanz.MPStamm
select ... from VFinanz.KRStamm
update VFinanz.VFinanz.MailboxTable ...  -- Note: VFinanz.VFinanz (duplicate schema)
```

**Fix**: Make schema configurable or use constants.

---

## Quick Wins

| Issue | File | Fix |
|-------|------|-----|
| Unused `Processors` field | `EBillProcessor.cs:29` | Remove or use |
| SSL validation bypasses errors | `IMAPProcessor.cs:74-105` | Log bypassed errors |
| Magic numbers | `EmailProcessor.cs:138-139` | Extract `MaxRetryCount = 3` |
| Commented code blocks | Multiple files | Remove |
| `Thread.Sleep(30)` hardcoded | `Configuration.cs:395` | Make configurable |

---

## Dependency Graph Improvements

```
Current:
┌─────────────────────┐
│ VemagEBillProcessor │
├──────────┬──────────┤
│VemagMailLib│VemagOauth│
└──────────┴──────────┘

Suggested:
┌─────────────────────┐
│ VemagEBillProcessor │
├─────────────────────┤
│  VemagMailLib.Core  │  <- Shared abstractions (interfaces, DTOs)
├──────────┬──────────┤
│  DbAccess │  OAuth   │  <- Data access and auth as separate concerns
└──────────┴──────────┘
```

---

## Testing Strategy

Since there are no tests, start with:

1. **Unit tests for DbService**
   - Mock `SqlConnection` with interface
   - Test register/refresh logic

2. **Integration tests for EmailProcessor**
   - Mock IMAP client
   - Test message processing logic

3. **Filter parsing tests**
   - `EbillFilters` JSON deserialization
   - Subject/attachment filtering rules

---

## No Unit Tests

**Priority**: Critical

The codebase has zero unit tests. This is a significant risk for a production system.

**Recommendation**:
- Add xUnit or NUnit test project
- Start with testing critical paths: DbService, EmailProcessor, filter logic
- Consider using Moq for mocking dependencies

---

## Legacy .NET Framework (4.8)

All projects target .NET Framework 4.8. Consider migrating to .NET 8+ for:
- Better performance
- Cross-platform support
- Modern async patterns
- Continued security updates

---

## Files Referenced

- `VemagEBillProcessor/EBillProcessor.cs`
- `VemagEBillProcessor/DbService.cs`
- `VemagEBillProcessor/EmailProcessor.cs`
- `VemagEBillProcessor/IMAPProcessor.cs`
- `VemagEBillProcessor/EWSProcessor.cs`
- `VemagEBillProcessor/Configuration.cs`
- `VemagMailLib/StaticStrings.cs`
- `VemagMailLib/DbServiceBase.cs`
- `VemagOauth/OAuth2Client.cs`
