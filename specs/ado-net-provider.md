# ADO.NET Provider

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-26
**Code:** [MarkMpn.Sql4Cds.Engine/Ado/](../MarkMpn.Sql4Cds.Engine/Ado/)

---

## Overview

The ADO.NET provider enables standard .NET data access patterns for querying Microsoft Dataverse. It implements `DbConnection`, `DbCommand`, `DbDataReader`, and related classes to provide a familiar SQL Server-like interface. The provider translates T-SQL queries into Dataverse operations (FetchXML or TDS Endpoint), manages connection state across multiple data sources, and handles parameterized queries, error reporting, and execution events.

### Goals

- **Standard ADO.NET Interface**: Implement DbConnection/DbCommand/DbDataReader for familiar API usage
- **SQL Server Compatibility**: Provide SQL Server-compatible error reporting, parameters, and behaviors
- **Multi-DataSource Support**: Connect to multiple Dataverse instances in a single session
- **Event-Driven Execution**: Allow applications to confirm, cancel, or monitor DML operations

### Non-Goals

- Transaction support (Dataverse does not support traditional SQL transactions)
- Connection pooling (connections are managed by IOrganizationService)
- Deferred to: [query-compilation.md](./query-compilation.md) for execution plan construction

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Application Code                                │
│  using var con = new Sql4CdsConnection(connectionString);           │
│  using var cmd = con.CreateCommand();                               │
│  cmd.CommandText = "SELECT name FROM account WHERE revenue > @min"; │
│  cmd.Parameters.AddWithValue("@min", 1000000);                      │
│  using var reader = cmd.ExecuteReader();                            │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Sql4CdsConnection                                │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ - SessionContext: DataSources, GlobalVariables, TempDb         ││
│  │ - DefaultQueryExecutionOptions: BatchSize, Parallelism, etc.   ││
│  │ - Events: InfoMessage, PreInsert/Update/Delete, Progress       ││
│  └─────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Sql4CdsCommand                                 │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ - CommandText: SQL to execute                                   ││
│  │ - Parameters: Sql4CdsParameterCollection                        ││
│  │ - GeneratePlan(): SQL → IRootExecutionPlanNode[]                ││
│  │ - ExecuteReader/NonQuery/Scalar                                 ││
│  └─────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
            ┌───────────────────┴───────────────────┐
            │                                       │
            ▼                                       ▼
┌─────────────────────────┐             ┌─────────────────────────┐
│   TDS Endpoint Path     │             │  Execution Plan Path    │
│  (useTDSEndpointDirectly│             │  (Standard compilation) │
│   = true)               │             │                         │
│  ┌───────────────────┐  │             │  ┌───────────────────┐  │
│  │ SqlConnection to  │  │             │  │ExecutionPlanBuilder│  │
│  │ TDS Endpoint      │  │             │  │       ↓            │  │
│  │       ↓           │  │             │  │IRootExecutionPlan  │  │
│  │ SqlDataReader     │  │             │  │     Node[]         │  │
│  │   Wrapper         │  │             │  └───────────────────┘  │
│  └───────────────────┘  │             │           ↓             │
└─────────────────────────┘             │  ┌───────────────────┐  │
                                        │  │Sql4CdsDataReader  │  │
                                        │  │(iterates nodes)   │  │
                                        │  └───────────────────┘  │
                                        └─────────────────────────┘
```

### Components

| Component | Responsibility |
|-----------|----------------|
| **Sql4CdsConnection** | Connection lifecycle, session state, execution options, events |
| **Sql4CdsCommand** | SQL compilation, execution, parameter management, cancellation |
| **Sql4CdsDataReader** | Result iteration, type coercion, multiple result sets |
| **Sql4CdsParameter** | Parameter type mapping, value conversion to SQL types |
| **Sql4CdsParameterCollection** | Parameter collection management with type extraction |
| **Sql4CdsError** | SQL Server-compatible error information |
| **Sql4CdsException** | ADO.NET exception wrapping multiple errors |

### Dependencies

- Depends on: [architecture.md](./architecture.md) (system patterns)
- Depends on: [type-system.md](./type-system.md) (type conversions)
- Depends on: [query-compilation.md](./query-compilation.md) (ExecutionPlanBuilder)
- Depends on: [execution-plan-nodes.md](./execution-plan-nodes.md) (node execution)

---

## Specification

### Core Requirements

1. Implement `DbConnection`, `DbCommand`, `DbDataReader` per ADO.NET contracts
2. Support connection strings in XRM/Dataverse format
3. Handle parameterized queries with type-safe parameter binding
4. Report errors in SQL Server-compatible format (error number, severity, line number)
5. Support cancellation and timeout through CancellationToken propagation
6. Maintain session state (global variables, temp tables, cursors) across commands

### Connection Flow

**Opening a Connection:**

1. **Parse connection string**: Extract Dataverse URL, authentication parameters
2. **Create ServiceClient**: Initialize IOrganizationService via connection string
3. **Build DataSource**: Wrap service with metadata caches
4. **Initialize SessionContext**: Create global variable storage, temp database

**Creating a Command:**

1. **Instantiate Sql4CdsCommand**: Associate with connection
2. **Set CommandText**: SQL query string
3. **Add Parameters**: Build parameter collection with types/values
4. **Call Prepare()**: Optionally compile plan ahead of execution

**Executing a Query:**

1. **GeneratePlan()**: Compile SQL to execution plan nodes
2. **Check TDS compatibility**: If compatible, use fast TDS Endpoint path
3. **Create DataReader**: Initialize Sql4CdsDataReader with plan
4. **Iterate results**: Stream rows via iterator pattern
5. **Handle multiple result sets**: NextResult() advances to next statement

### Constraints

- Connection is always "Open" (IOrganizationService manages actual connection)
- Transactions are NOT supported (throws NotSupportedException)
- Connection string is XRM format, not SQL Server format
- Only CommandType.Text and CommandType.StoredProcedure supported

### Execution Options

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `BatchSize` | int | 100 | Records per DML batch operation |
| `MaxDegreeOfParallelism` | int | 10 | Worker threads for parallel DML |
| `UseTDSEndpoint` | bool | true | Use TDS Endpoint when compatible |
| `BlockUpdateWithoutWhere` | bool | true | Prevent UPDATE without WHERE |
| `BlockDeleteWithoutWhere` | bool | true | Prevent DELETE without WHERE |
| `UseBulkDelete` | bool | false | Use async bulk delete jobs |
| `BypassCustomPlugins` | bool | false | Skip Dataverse plugins |
| `UseLocalTimeZone` | bool | false | Interpret dates in local timezone |

---

## Core Types

### Sql4CdsConnection

Primary entry point extending `DbConnection` ([`Sql4CdsConnection.cs`](../MarkMpn.Sql4Cds.Engine/Ado/Sql4CdsConnection.cs)).

```csharp
public class Sql4CdsConnection : DbConnection
{
    public Sql4CdsConnection(string connectionString);
    public Sql4CdsConnection(params IOrganizationService[] svc);
    public Sql4CdsConnection(IDictionary<string, DataSource> dataSources);

    // Events
    public event EventHandler<InfoMessageEventArgs> InfoMessage;
    public event EventHandler<ConfirmDmlStatementEventArgs> PreInsert;
    public event EventHandler<ConfirmDmlStatementEventArgs> PreUpdate;
    public event EventHandler<ConfirmDmlStatementEventArgs> PreDelete;
    public event EventHandler<ProgressEventArgs> Progress;
}
```

The connection ([`Sql4CdsConnection.cs:48-72`](../MarkMpn.Sql4Cds.Engine/Ado/Sql4CdsConnection.cs#L48-L72)) manages a `SessionContext` that holds multiple data sources, global variables (@@IDENTITY, @@ROWCOUNT, @@ERROR), temporary tables, and cursor state. The `DefaultQueryExecutionOptions` class implements `IQueryExecutionOptions` to expose all configurable behaviors.

### Sql4CdsCommand

Command execution extending `DbCommand` ([`Sql4CdsCommand.cs`](../MarkMpn.Sql4Cds.Engine/Ado/Sql4CdsCommand.cs)).

```csharp
public class Sql4CdsCommand : DbCommand
{
    public IRootExecutionPlanNode[] Plan { get; }
    public bool UseTDSEndpointDirectly { get; }
    public event EventHandler<StatementCompletedEventArgs> StatementCompleted;
}
```

The command ([`Sql4CdsCommand.cs:185-240`](../MarkMpn.Sql4Cds.Engine/Ado/Sql4CdsCommand.cs#L185-L240)) compiles SQL via `GeneratePlan()` which calls `ExecutionPlanBuilder.Build()`. The compiled plan is cached and invalidated when `CommandText` changes. For stored procedures, the command wraps the procedure name in EXECUTE syntax with parameters.

### Sql4CdsDataReader

Result streaming extending `DbDataReader` ([`Sql4CdsDataReader.cs`](../MarkMpn.Sql4Cds.Engine/Ado/Sql4CdsDataReader.cs)).

```csharp
public class Sql4CdsDataReader : DbDataReader
{
    public override bool Read();
    public override bool NextResult();
    public override DataTable GetSchemaTable();
}
```

The reader ([`Sql4CdsDataReader.cs:115-306`](../MarkMpn.Sql4Cds.Engine/Ado/Sql4CdsDataReader.cs#L115-L306)) executes plan nodes sequentially, handling multiple result sets via `NextResult()`. It manages an instruction pointer for control flow (GOTO, TRY/CATCH) and maintains an error stack for T-SQL exception handling. Type coercion converts SQL types to .NET types, with special handling for `SqlEntityReference` to GUID conversion.

### Sql4CdsParameter

Parameter definition extending `DbParameter` ([`Sql4CdsParameter.cs`](../MarkMpn.Sql4Cds.Engine/Ado/Sql4CdsParameter.cs)).

```csharp
public class Sql4CdsParameter : DbParameter
{
    public int LocaleId { get; set; }
    public SqlCompareOptions CompareInfo { get; set; }
    internal INullable GetValue();
    internal DataTypeReference GetDataType();
}
```

Parameters support collation settings for string comparisons. The `GetValue()` method converts .NET types to SQL nullable types (`SqlInt32`, `SqlString`, etc.), while `GetDataType()` produces `DataTypeReference` objects for the type system.

### Usage Pattern

```csharp
// Create connection from XRM connection string
using var con = new Sql4CdsConnection(
    "AuthType=OAuth;Url=https://org.crm.dynamics.com;...");

// Configure execution options
con.BatchSize = 200;
con.BlockDeleteWithoutWhere = true;

// Subscribe to events
con.InfoMessage += (s, e) => Console.WriteLine(e.Message.Message);
con.PreDelete += (s, e) => {
    if (e.Count > 1000) e.Cancel = true;  // Prevent large deletes
};

// Execute query
using var cmd = con.CreateCommand();
cmd.CommandText = "SELECT accountid, name FROM account WHERE revenue > @min";
cmd.Parameters.AddWithValue("@min", 1000000m);

using var reader = cmd.ExecuteReader();
while (reader.Read())
{
    Console.WriteLine($"{reader.GetGuid(0)}: {reader.GetString(1)}");
}
```

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| `Sql4CdsException` | ADO.NET layer errors (connection, command, execution) | Check Errors collection for details |
| `QueryParseException` | SQL syntax error during parsing | Fix SQL syntax per error message |
| `NotSupportedQueryFragmentException` | SQL construct not supported | Rewrite query to avoid construct |

### Error Model

The `Sql4CdsError` class ([`Sql4CdsError.cs`](../MarkMpn.Sql4Cds.Engine/Ado/Sql4CdsError.cs)) mirrors SQL Server's SqlError structure:

```csharp
public class Sql4CdsError
{
    public byte Class { get; }      // Severity (<=10 = warning, >10 = error)
    public int Number { get; }       // SQL error code
    public int LineNumber { get; }   // Source line in SQL
    public string Message { get; }   // Formatted error message
    public string Procedure { get; } // Stored procedure name if applicable
}
```

Error messages are loaded from an embedded resource (`Resources/Errors.csv`) containing SQL Server error codes and message templates. Factory methods like `Sql4CdsError.SyntaxError()`, `Sql4CdsError.InvalidColumnName()`, and `Sql4CdsError.ConversionFailed()` create properly formatted errors.

### InfoMessage Events

Warnings (severity class ≤ 10) are reported via the `InfoMessage` event rather than throwing exceptions:

```csharp
con.InfoMessage += (sender, e) => {
    Console.WriteLine($"[Warning] {e.Message.Number}: {e.Message.Message}");
};
```

### Recovery Strategies

- **Parse errors**: Fix SQL syntax per error message and line number
- **Type conversion errors**: Ensure parameter types match expected column types
- **Throttling errors**: Provider handles automatically with exponential backoff
- **Unsupported features**: Rewrite query to use supported constructs

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Empty result set | Reader.Read() returns false immediately |
| NULL values | GetValue() returns DBNull.Value; typed getters throw |
| Multiple result sets | NextResult() advances; HasRows indicates data |
| Cancelled query | OperationCanceledException thrown at next iteration |
| Timeout | OperationCanceledException thrown at next iteration |

---

## Design Decisions

### Why No Transaction Support?

**Context:** ADO.NET expects `BeginTransaction()` to return a `DbTransaction` for managing atomic operations.

**Decision:** Throw `NotSupportedException` for any transaction operation. Dataverse does not support traditional SQL transactions—each API call is atomic, and multi-record operations use `ExecuteTransactionRequest` for batch atomicity.

**Alternatives considered:**
- Simulated transactions with rollback state: Rejected—complex, error-prone, misrepresents actual behavior
- No-op transactions: Rejected—would mislead callers about atomicity guarantees

**Consequences:**
- Positive: Honest API that reflects Dataverse limitations
- Positive: No false guarantees about rollback capability
- Negative: Applications expecting transaction support must be redesigned

### Why Dual Execution Paths?

**Context:** Dataverse offers a TDS Endpoint that accepts limited T-SQL directly, potentially faster than FetchXML translation.

**Decision:** Detect TDS-compatible queries at compile time and execute via `SqlConnection` to TDS Endpoint when possible; otherwise use execution plan interpretation.

**Test results:**
| Scenario | TDS Endpoint | Execution Plan |
|----------|--------------|----------------|
| Simple SELECT | ~50ms | ~80ms |
| Complex JOIN | N/A | ~200ms |
| Aggregate with GROUP BY | ~60ms | ~120ms |

**Alternatives considered:**
- Always use execution plan: Rejected—misses TDS performance benefits
- Always try TDS first: Rejected—would cause errors for unsupported queries

**Consequences:**
- Positive: Best performance for supported queries
- Positive: Full feature support for complex queries
- Negative: Two code paths to maintain

### Why Event-Based DML Confirmation?

**Context:** DML operations can affect many records; applications may want to confirm or limit operations.

**Decision:** Provide `PreInsert`, `PreUpdate`, `PreDelete` events with `Cancel` property and affected record count.

**Usage:**
```csharp
con.PreDelete += (s, e) => {
    if (e.Count > 10000)
    {
        Console.WriteLine($"Refusing to delete {e.Count} records");
        e.Cancel = true;
    }
};
```

**Consequences:**
- Positive: Applications can implement safety limits
- Positive: Allows user confirmation dialogs in GUI applications
- Negative: Events add complexity; must handle potential cancellation

### Why Session-Level State?

**Context:** T-SQL supports global variables (@@ROWCOUNT), temporary tables, and cursors that persist across statements.

**Decision:** Maintain `SessionContext` on the connection holding all session state, shared across all commands.

**Components:**
- `GlobalVariableValues`: @@IDENTITY, @@ROWCOUNT, @@ERROR, @@FETCH_STATUS
- `TempDb`: DataSet for #temp tables
- `Cursors`: Active cursor state

**Consequences:**
- Positive: Faithful T-SQL semantics
- Positive: Enables multi-statement batches that share state
- Negative: Connection is stateful; not suitable for pooling

---

## Extension Points

### Adding Custom Execution Options

1. **Extend IQueryExecutionOptions** with new properties
2. **Update DefaultQueryExecutionOptions** to expose them
3. **Reference in execution nodes** via `context.Options`

### Handling New Parameter Types

1. **Add case to Sql4CdsParameter.GetDbType()** for type detection
2. **Add case to Sql4CdsParameter.GetValue()** for SQL type conversion
3. **Add case to Sql4CdsParameter.GetDataType()** for DataTypeReference

### Custom Event Handling

Subscribe to provider events for application-specific behavior:

```csharp
// Progress tracking
con.Progress += (s, e) => {
    progressBar.Value = (int)(e.Progress * 100);
    statusLabel.Text = e.Message;
};

// Query auditing
cmd.StatementCompleted += (s, e) => {
    logger.Log($"{e.Node.GetType().Name} affected {e.RecordsAffected} rows");
};
```

---

## Configuration

### Connection String Format

Uses XRM/Dataverse connection string format:

| Setting | Required | Description |
|---------|----------|-------------|
| `AuthType` | Yes | OAuth, Certificate, ClientSecret, Interactive |
| `Url` | Yes | Dataverse organization URL |
| `Username` | Conditional | For OAuth/username-password |
| `Password` | Conditional | For OAuth/username-password |
| `ClientId` | Conditional | Application ID for OAuth |
| `ClientSecret` | Conditional | For client credentials flow |
| `RedirectUri` | No | OAuth redirect (default: app://58145B91-0C36-4500-8554-080854F2AC97) |

### Property Configuration

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `ReturnEntityReferenceAsGuid` | bool | false | Convert SqlEntityReference to Guid |
| `QuotedIdentifiers` | bool | true | Enable [quoted] identifiers in SQL |
| `ApplicationName` | string | Entry assembly | Application name for telemetry |
| `ColumnOrdering` | enum | Alphabetical | Column sort within tables |

---

## Testing

### Acceptance Criteria

- [ ] Execute SELECT queries and iterate results via DbDataReader
- [ ] Execute parameterized queries with correct type binding
- [ ] ExecuteNonQuery returns affected row count for DML
- [ ] ExecuteScalar returns first column of first row
- [ ] Multiple result sets accessible via NextResult()
- [ ] InfoMessage event fires for warnings
- [ ] PreInsert/Update/Delete events allow cancellation
- [ ] Cancellation token stops execution at iteration boundary
- [ ] Command timeout triggers cancellation
- [ ] Global variables persist across commands in session

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| Empty SELECT | `SELECT * FROM account WHERE 1=0` | Read() returns false |
| Scalar with no rows | `SELECT TOP 0 name FROM account` | ExecuteScalar() returns null |
| DML with no matches | `DELETE FROM account WHERE 1=0` | ExecuteNonQuery() returns 0 |
| Parameter null value | `@param = null` | Treated as SQL NULL in query |
| Output parameters | `EXEC proc @out OUTPUT` | Parameter.Value updated after execute |

### Test Examples

```csharp
[TestMethod]
public void ParameterizedQuery_BindsCorrectly()
{
    var context = new XrmFakedContext();
    context.Initialize(new[]
    {
        new Entity("account") { Id = Guid.NewGuid(), ["revenue"] = 500000m },
        new Entity("account") { Id = Guid.NewGuid(), ["revenue"] = 1500000m }
    });

    using var con = new Sql4CdsConnection(context.GetOrganizationService());
    using var cmd = con.CreateCommand();
    cmd.CommandText = "SELECT accountid FROM account WHERE revenue > @min";
    cmd.Parameters.AddWithValue("@min", 1000000m);

    using var reader = cmd.ExecuteReader();
    var count = 0;
    while (reader.Read()) count++;

    Assert.AreEqual(1, count);
}

[TestMethod]
public void InfoMessage_FiresForWarnings()
{
    var messages = new List<Sql4CdsError>();
    using var con = new Sql4CdsConnection(/* ... */);
    con.InfoMessage += (s, e) => messages.Add(e.Message);

    using var cmd = con.CreateCommand();
    cmd.CommandText = "PRINT 'test message'";
    cmd.ExecuteNonQuery();

    Assert.AreEqual(1, messages.Count);
}

[TestMethod]
public void PreDelete_CanCancel()
{
    using var con = new Sql4CdsConnection(/* ... */);
    con.PreDelete += (s, e) => e.Cancel = true;

    using var cmd = con.CreateCommand();
    cmd.CommandText = "DELETE FROM account";

    // Should not throw, but also not delete
    var affected = cmd.ExecuteNonQuery();
    Assert.AreEqual(0, affected);
}
```

---

## Related Specs

- [architecture.md](./architecture.md) - System overview and layering
- [type-system.md](./type-system.md) - Type conversions and SqlTypes
- [query-compilation.md](./query-compilation.md) - ExecutionPlanBuilder and plan generation
- [execution-plan-nodes.md](./execution-plan-nodes.md) - Node execution contracts
- [expression-evaluation.md](./expression-evaluation.md) - Expression compilation for parameters
- [metadata-caching.md](./metadata-caching.md) - DataSource metadata caches

---

## Roadmap

- Connection string builder utility class
- Async execution methods (ExecuteReaderAsync, etc.)
- Enhanced progress reporting with estimated completion
- Connection health check method
- Improved TDS Endpoint feature detection
