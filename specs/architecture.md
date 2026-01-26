# SQL4CDS Architecture

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-25
**Code:** [MarkMpn.Sql4Cds.Engine/](../MarkMpn.Sql4Cds.Engine/)

---

## Overview

SQL4CDS is a SQL-to-FetchXML translation engine that enables T-SQL query execution against Microsoft Dataverse. It provides an ADO.NET provider interface, execution plan compilation, query optimization, and multi-host integration for XrmToolBox, SSMS, Azure Data Studio, and programmatic access.

### Goals

- **SQL Compatibility**: Execute T-SQL queries against Dataverse with maximum SQL Server compatibility
- **Multi-Host Support**: Integrate with XrmToolBox, SSMS, Azure Data Studio, and custom applications
- **Performance**: Optimize queries by folding operations into FetchXML where possible
- **Extensibility**: Support 80+ execution plan node types via interface hierarchy

### Non-Goals

- Full SQL Server parity (some T-SQL features not supported by Dataverse)
- Direct database writes (all mutations go through Dataverse API)
- Real-time replication or synchronization

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           HOST LAYER                                 │
├──────────────┬──────────────┬───────────────┬──────────────────────┤
│   XrmToolBox │     SSMS     │ Azure Data    │   Custom Apps        │
│   Plugin     │  Extension   │ Studio (LSP)  │   (ADO.NET)          │
│   (XTB)      │  (21/22)     │               │                      │
└──────┬───────┴──────┬───────┴───────┬───────┴──────────┬───────────┘
       │              │               │                  │
       ▼              ▼               ▼                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         ADO.NET LAYER                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │
│  │ Sql4CdsConnection│  │ Sql4CdsCommand  │  │ Sql4CdsDataReader   │  │
│  │                 │  │                 │  │                     │  │
│  │ - DataSources   │  │ - GeneratePlan()│  │ - Iterates nodes    │  │
│  │ - SessionContext│  │ - ExecuteReader │  │ - Returns Entity    │  │
│  └────────┬────────┘  └────────┬────────┘  └─────────────────────┘  │
│           │                    │                                     │
└───────────┼────────────────────┼─────────────────────────────────────┘
            │                    │
            ▼                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       COMPILATION LAYER                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    ExecutionPlanBuilder                         ││
│  │  TSql160Parser → Visitors → Node Construction → Optimization    ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                │                                     │
│                                ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                   IRootExecutionPlanNode[]                      ││
│  │                   (Compiled Execution Plan)                     ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       EXECUTION LAYER                                │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                 IDataExecutionPlanNode Tree                     ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐            ││
│  │  │FetchXml │  │ Filter  │  │  Join   │  │Aggregate│  ...       ││
│  │  │  Scan   │→ │  Node   │→ │  Node   │→ │  Node   │            ││
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘            ││
│  └─────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       DATA SOURCE LAYER                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                         DataSource                              ││
│  │  - IOrganizationService (Dataverse API)                         ││
│  │  - IAttributeMetadataCache                                      ││
│  │  - ITableSizeCache                                              ││
│  │  - IMessageCache                                                ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Responsibility |
|-----------|----------------|
| **Host Layer** | UI integration for XrmToolBox, SSMS, Azure Data Studio, programmatic access |
| **ADO.NET Layer** | Standard DbConnection/DbCommand/DbDataReader interface |
| **Compilation Layer** | SQL parsing, AST transformation, execution plan construction |
| **Execution Layer** | Node-based query execution with streaming data flow |
| **Data Source Layer** | Dataverse API abstraction with metadata caching |

### Dependencies

- **Microsoft.SqlServer.TransactSql.ScriptDom** - T-SQL parsing (TSql160Parser)
- **Microsoft.PowerPlatform.Dataverse.Client** - Dataverse connectivity (.NET 8)
- **Microsoft.CrmSdk.CoreAssemblies** - Dataverse connectivity (.NET Framework)
- **Microsoft.ApplicationInsights** - Telemetry

---

## Module Structure

### Project Organization

```
MarkMpn.Sql4Cds.sln
├── Core Engine
│   ├── MarkMpn.Sql4Cds.Engine/          # Core engine (net8.0, net462)
│   │   ├── Ado/                          # ADO.NET provider
│   │   ├── ExecutionPlan/                # 100+ execution nodes
│   │   └── Visitors/                     # 28 AST visitors
│   ├── MarkMpn.Sql4Cds.Export/          # Export formats (net8.0, net48)
│   └── MarkMpn.Sql4Cds.Controls/        # UI controls (net472)
│
├── Host Integrations
│   ├── MarkMpn.Sql4Cds.XTB/             # XrmToolBox plugin (net48)
│   ├── MarkMpn.Sql4Cds.SSMS.21/         # SSMS 21 extension (net472)
│   ├── MarkMpn.Sql4Cds.SSMS.22/         # SSMS 22 extension (net472)
│   ├── MarkMpn.Sql4Cds.SSMS/            # Shared SSMS code (shproj)
│   └── MarkMpn.Sql4Cds.LanguageServer/  # LSP server (net8.0)
│
├── Developer Tools
│   ├── MarkMpn.Sql4Cds.DebugVisualizer.DebugeeSide/
│   └── MarkMpn.Sql4Cds.DebugVisualizer.DebuggerSide/
│
└── Tests
    ├── MarkMpn.Sql4Cds.Engine.Tests/
    ├── MarkMpn.Sql4Cds.Engine.FetchXml.Tests/
    └── MarkMpn.Sql4Cds.Tests/
```

### Dependency Graph

```
MarkMpn.Sql4Cds.Engine  ◄──────────────────────────────────────────┐
    │                                                               │
    ├──► MarkMpn.Sql4Cds.Controls ──► MarkMpn.Sql4Cds.XTB          │
    │         │                              │                      │
    │         ├────────────────────► MarkMpn.Sql4Cds.SSMS.21       │
    │         │                              │                      │
    │         └────────────────────► MarkMpn.Sql4Cds.SSMS.22       │
    │                                        │                      │
    ├──► MarkMpn.Sql4Cds.Export ◄────────────┤                      │
    │         │                              │                      │
    │         └──────────────────► MarkMpn.Sql4Cds.LanguageServer  │
    │                                                               │
    ├──► MarkMpn.Sql4Cds.DebugVisualizer.DebugeeSide               │
    │         │                                                     │
    │         └──► MarkMpn.Sql4Cds.DebugVisualizer.DebuggerSide    │
    │                                                               │
    └──► Test Projects ─────────────────────────────────────────────┘
```

### Target Framework Strategy

| Project | Framework(s) | Rationale |
|---------|--------------|-----------|
| Engine | net8.0, net462 | Core library supports modern .NET and legacy Windows apps |
| Export | net8.0, net48 | Export features need modern .NET and XTB compatibility |
| Controls | net472 | WinForms controls for SSMS integration |
| XTB | net48 | XrmToolBox plugin requirement |
| SSMS.21/22 | net472 | Visual Studio extension requirement |
| LanguageServer | net8.0 | Modern console app for LSP |

---

## Layering

### Layer Responsibilities

```
┌─────────────────────────────────────────────────────────────────┐
│ PRESENTATION LAYER (Host-Specific)                               │
│ - XTB: PluginControl, SettingsForm, QueryExecutionOptions       │
│ - SSMS: Sql4CdsPackage, CommandBase, OptionsPage                │
│ - LSP: IJsonRpcMethodHandler implementations                    │
├─────────────────────────────────────────────────────────────────┤
│ ADO.NET LAYER                                                    │
│ - Sql4CdsConnection: Session management, DataSources            │
│ - Sql4CdsCommand: Query preparation, plan generation            │
│ - Sql4CdsDataReader: Result streaming from execution nodes      │
├─────────────────────────────────────────────────────────────────┤
│ COMPILATION LAYER                                                │
│ - ExecutionPlanBuilder: SQL → Execution Plan                    │
│ - TSqlFragmentVisitor subclasses: AST transformations           │
│ - ExecutionPlanOptimizer: Query folding, predicate pushdown     │
├─────────────────────────────────────────────────────────────────┤
│ EXECUTION LAYER                                                  │
│ - IExecutionPlanNode hierarchy (80+ node types)                 │
│ - NodeExecutionContext: Runtime state                           │
│ - INodeSchema: Column metadata flow                             │
├─────────────────────────────────────────────────────────────────┤
│ DATA ACCESS LAYER                                                │
│ - DataSource: Connection + caches per Dataverse instance        │
│ - IAttributeMetadataCache: Entity/attribute metadata            │
│ - ITableSizeCache: Row count estimation                         │
│ - IMessageCache: Available Dataverse messages                   │
└─────────────────────────────────────────────────────────────────┘
```

### Layer Communication

| From | To | Mechanism |
|------|-----|-----------|
| Host → ADO | Sql4CdsConnection/Command/Reader | Standard ADO.NET API |
| ADO → Compilation | ExecutionPlanBuilder.Build() | Returns IRootExecutionPlanNode[] |
| Compilation → Execution | Node.Execute(context) | Iterator pattern (IEnumerable<Entity>) |
| Execution → Data | DataSource.Execute(request) | IOrganizationService calls |

---

## Design Patterns

### Visitor Pattern

**Purpose:** Transform and analyze T-SQL AST without modifying parser classes.

**Location:** [`MarkMpn.Sql4Cds.Engine/Visitors/`](../MarkMpn.Sql4Cds.Engine/Visitors/)

**Implementation:**
```csharp
internal abstract class RewriteVisitorBase : TSqlFragmentVisitor
{
    protected TSqlFragment ReplaceExpression(TSqlFragment original,
                                             TSqlFragment replacement);
}
```

**Key Visitors (28 total):**
- `RewriteVisitor` - Rewrites expressions to column references
- `BooleanRewriteVisitor` - Boolean logic transformation
- `ColumnCollectingVisitor` - Extracts referenced columns
- `AggregateCollectingVisitor` - Identifies aggregate functions
- `CteValidatorVisitor` - Validates Common Table Expressions

### Builder Pattern

**Purpose:** Construct complex execution plans from SQL text.

**Location:** [`ExecutionPlanBuilder.cs`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs)

**Implementation:**
```csharp
internal class ExecutionPlanBuilder
{
    public IRootExecutionPlanNode[] Build(
        string sql,
        IDictionary<string, DataTypeReference> parameters,
        out bool useTDSEndpointDirectly);
}
```

**Build Pipeline:**
1. Parse SQL using TSql160Parser
2. Apply validation visitors
3. Convert each statement to execution nodes
4. Apply query folding optimizations
5. Return compiled execution plan

### Template Method Pattern

**Purpose:** Define execution node contracts with customizable steps.

**Location:** [`ExecutionPlan/BaseNode.cs`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/BaseNode.cs), [`BaseDataNode.cs`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/BaseDataNode.cs)

**Implementation:**
```csharp
abstract class BaseDataNode : BaseNode, IDataExecutionPlanNodeInternal
{
    protected abstract RowCountEstimate EstimateRowsOutInternal(
        NodeCompilationContext context);

    protected abstract IEnumerable<Entity> ExecuteInternal(
        NodeExecutionContext context);
}
```

**Node Hierarchy:**
- `BaseNode` → `BaseDataNode` → `FetchXmlScan`, `FilterNode`, etc.
- `BaseNode` → `BaseDmlNode` → `InsertNode`, `UpdateNode`, `DeleteNode`
- `BaseNode` → `BaseJoinNode` → `NestedLoopNode`, `HashJoinNode`, `MergeJoinNode`

### Strategy Pattern

**Purpose:** Allow pluggable implementations for caching and execution behavior.

**Interfaces:**
- [`IAttributeMetadataCache`](../MarkMpn.Sql4Cds.Engine/IAttributeMetadataCache.cs) - Entity metadata retrieval
- [`ITableSizeCache`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/ITableSizeCache.cs) - Row count estimation
- [`IQueryExecutionOptions`](../MarkMpn.Sql4Cds.Engine/IQueryExecutionOptions.cs) - Execution behavior control

**Usage:**
```csharp
var dataSource = new DataSource(
    org,
    customMetadataCache,    // Pluggable
    customTableSizeCache,   // Pluggable
    customMessageCache);    // Pluggable
```

### Context Pattern

**Purpose:** Carry state through compilation and execution phases.

**Location:** [`NodeContext.cs`](../MarkMpn.Sql4Cds.Engine/NodeContext.cs)

**Hierarchy:**
```
NodeCompilationContext
├── Session: SessionContext
├── Options: IQueryExecutionOptions
├── ParameterTypes: IDictionary<string, DataTypeReference>
└── PrimaryDataSource: DataSource

    └── NodeExecutionContext (extends)
        ├── ParameterValues: IDictionary<string, INullable>
        └── Error: Sql4CdsError

        └── ExpressionExecutionContext (extends)
            └── Entity: Entity (current row)
```

### Iterator Pattern

**Purpose:** Stream query results without loading entire result set.

**Implementation:**
```csharp
internal interface IDataExecutionPlanNodeInternal
{
    IEnumerable<Entity> Execute(NodeExecutionContext context);
}
```

Nodes yield results lazily, enabling memory-efficient processing of large datasets.

---

## Extension Points

### Adding a New Execution Plan Node

1. **Create node class** in `ExecutionPlan/`:
```csharp
class CustomNode : BaseDataNode
{
    protected override RowCountEstimate EstimateRowsOutInternal(
        NodeCompilationContext context) => new RowCountEstimate(100);

    protected override IEnumerable<Entity> ExecuteInternal(
        NodeExecutionContext context)
    {
        // Yield results
    }

    public override IEnumerable<IExecutionPlanNode> GetSources()
        => new[] { Source };
}
```

2. **Implement required methods:**
   - `GetSchema()` - Return column definitions
   - `AddRequiredColumns()` - Declare needed columns
   - `FoldQuery()` - Optimize with hints
   - `Clone()` - Deep copy for optimization

3. **Wire into ExecutionPlanBuilder** for appropriate SQL constructs.

### Adding a New SQL Function

1. **Add static method** to [`ExpressionFunctions.cs`](../MarkMpn.Sql4Cds.Engine/ExpressionFunctions.cs):
```csharp
[SqlFunction(IsDeterministic = true)]
public static SqlString CustomFunction(SqlString input)
{
    return input.IsNull ? SqlString.Null : new SqlString(input.Value.ToUpper());
}
```

2. Functions are discovered via reflection and mapped to T-SQL function calls.

### Adding a New Host Integration

1. **Reference** `MarkMpn.Sql4Cds.Engine`
2. **Create** `Sql4CdsConnection` with data sources
3. **Implement** `IQueryExecutionOptions` for host-specific behavior
4. **Handle** events: `InfoMessage`, `ConfirmInsert/Update/Delete`

### Extending Metadata Caching

Implement `IAttributeMetadataCache`:
```csharp
public class CustomMetadataCache : IAttributeMetadataCache
{
    public EntityMetadata this[string name] => LoadMetadata(name);
    public bool TryGetValue(string name, out EntityMetadata metadata);
    public bool TryGetMinimalData(string name, out EntityMetadata metadata);
}
```

---

## Cross-Cutting Concerns

### Error Handling

**Exception Hierarchy:**
- `Sql4CdsException` ([`Ado/Sql4CdsException.cs:11`](../MarkMpn.Sql4Cds.Engine/Ado/Sql4CdsException.cs#L11)) - ADO.NET layer errors
- `QueryExecutionException` ([`ExecutionPlan/QueryExecutionException.cs:16`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/QueryExecutionException.cs#L16)) - Runtime errors
- `NotSupportedQueryFragmentException` ([`NotSupportedQueryFragmentException.cs:13`](../MarkMpn.Sql4Cds.Engine/NotSupportedQueryFragmentException.cs#L13)) - Unsupported SQL

**Error Model:**
All exceptions implement `ISql4CdsErrorException` exposing `IReadOnlyList<Sql4CdsError>`:
- Mirrors SQL Server's SqlError structure
- Properties: Class, LineNumber, Number, Procedure, Server, State, Message

**Dataverse Fault Mapping:**
`QueryExecutionException.FaultCodeToSqlError()` maps Dataverse error codes to SQL error numbers for familiar error handling.

### Logging & Telemetry

**Application Insights:**
```csharp
// Sql4CdsConnection.cs:65
_telemetry = new TelemetryClient();

// Sql4CdsCommand.cs:106 - Tracks each statement execution
_connection.TelemetryClient.TrackEvent("Execute", new Dictionary<string, string>
{
    ["QueryType"] = node.GetType().Name,
    ["Source"] = _connection.ApplicationName
});
```

**Progress Reporting:**
```csharp
interface IQueryExecutionOptions
{
    void Progress(double? progress, string message);
}
```

**Info Messages:**
```csharp
connection.InfoMessage += (sender, e) =>
    Console.WriteLine($"[{e.Error.Class}] {e.Error.Message}");
```

### Configuration

**IQueryExecutionOptions** controls all execution behavior:

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| BatchSize | int | 100 | Records per DML batch |
| MaxDegreeOfParallelism | int | 10 | Parallel DML threads |
| UseTDSEndpoint | bool | true | Use TDS when available |
| BlockUpdateWithoutWhere | bool | true | Safety check |
| BlockDeleteWithoutWhere | bool | true | Safety check |
| UseBulkDelete | bool | false | Bulk delete jobs |
| BypassCustomPlugins | bool | false | Skip Dataverse plugins |
| UseLocalTimeZone | bool | false | Date interpretation |

### Cancellation & Timeout

**CancellationToken Propagation:**
- `IQueryExecutionOptions.CancellationToken` passed to all execution contexts
- Checked at iteration boundaries in data nodes
- DML operations check before each batch

**Throttling Handling:**
```csharp
// ExceptionExtensions.cs:13-39
if (exception.IsThrottlingException(out TimeSpan retryDelay))
{
    await Task.Delay(retryDelay, cancellationToken);
    // Retry operation
}
```

Error codes detected: 429, -2147015902, -2147015903, -2147015898
Default retry: 2 seconds, max 5 minutes, respects Retry-After header.

### Connection Management

**Multi-DataSource Support:**
```csharp
var connection = new Sql4CdsConnection(new Dictionary<string, DataSource>
{
    ["prod"] = prodDataSource,
    ["dev"] = devDataSource
});
```

**Session State:**
`SessionContext` maintains cross-query state:
- Global variables (@@IDENTITY, @@ROWCOUNT, @@ERROR)
- Temporary tables (TempDb)
- Cursor state
- Date format settings

---

## Specification

### Core Requirements

1. Parse T-SQL using Microsoft.SqlServer.TransactSql.ScriptDom (TSql160Parser)
2. Convert supported SQL constructs to FetchXML for Dataverse execution
3. Execute unsupported operations client-side when possible
4. Support multiple Dataverse instances in single session
5. Provide ADO.NET interface for standard data access patterns

### Query Execution Flow

1. **Parse**: `TSql160Parser.Parse(sql)` → TSqlScript AST
2. **Transform**: Apply visitor transformations (28 visitors)
3. **Build**: `ExecutionPlanBuilder.ConvertStatement()` → execution nodes
4. **Optimize**: `FoldQuery()` pushes operations to Dataverse
5. **Execute**: `IDataExecutionPlanNode.Execute()` → IEnumerable<Entity>
6. **Stream**: `Sql4CdsDataReader` iterates results

### TDS Endpoint Detection

If all operations can be expressed in pure T-SQL compatible with Dataverse TDS Endpoint:
```csharp
if (useTDSEndpointDirectly && options.UseTDSEndpoint)
{
    // Execute via SqlConnection to TDS endpoint
}
else
{
    // Execute via FetchXML + client-side processing
}
```

---

## Core Types

### Sql4CdsConnection

Primary entry point for SQL4CDS queries ([`Ado/Sql4CdsConnection.cs`](../MarkMpn.Sql4Cds.Engine/Ado/Sql4CdsConnection.cs)).

```csharp
public class Sql4CdsConnection : DbConnection
{
    public Sql4CdsConnection(string connectionString);
    public Sql4CdsConnection(params IOrganizationService[] svc);
    public Sql4CdsConnection(IDictionary<string, DataSource> dataSources);

    public event EventHandler<InfoMessageEventArgs> InfoMessage;
    public event EventHandler<ConfirmDmlStatementEventArgs> PreInsert;
    public event EventHandler<ConfirmDmlStatementEventArgs> PreUpdate;
    public event EventHandler<ConfirmDmlStatementEventArgs> PreDelete;
}
```

### IExecutionPlanNode

Base interface for all execution nodes ([`ExecutionPlan/IExecutionPlanNode.cs`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/IExecutionPlanNode.cs)).

```csharp
public interface IExecutionPlanNode
{
    IExecutionPlanNode Parent { get; }
    IEnumerable<IExecutionPlanNode> GetSources();
    int ExecutionCount { get; }
    TimeSpan Duration { get; }
}
```

### IDataExecutionPlanNode

Interface for data-producing nodes ([`ExecutionPlan/IDataExecutionPlanNode.cs`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/IDataExecutionPlanNode.cs)).

```csharp
public interface IDataExecutionPlanNode : IExecutionPlanNode
{
    int EstimatedRowsOut { get; }
    int RowsOut { get; }
}

internal interface IDataExecutionPlanNodeInternal : IDataExecutionPlanNode
{
    IEnumerable<Entity> Execute(NodeExecutionContext context);
    IDataExecutionPlanNodeInternal FoldQuery(NodeCompilationContext context,
                                              IList<OptimizerHint> hints);
    INodeSchema GetSchema(NodeCompilationContext context);
}
```

### DataSource

Encapsulates a Dataverse connection with caches ([`DataSource.cs`](../MarkMpn.Sql4Cds.Engine/DataSource.cs)).

```csharp
public class DataSource
{
    public string Name { get; set; }
    public IOrganizationService Connection { get; set; }
    public IAttributeMetadataCache Metadata { get; set; }
    public ITableSizeCache TableSizeCache { get; set; }
    public IMessageCache MessageCache { get; set; }
}
```

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| Sql4CdsException | Connection/command errors | Check inner errors, retry if transient |
| QueryExecutionException | Runtime execution failure | Check node, examine Dataverse fault |
| NotSupportedQueryFragmentException | SQL construct not supported | Rewrite query |
| QueryParseException | SQL syntax error | Fix SQL syntax |

### Recovery Strategies

- **Throttling**: Automatic retry with exponential backoff
- **Transient faults**: Retry with fresh connection
- **Validation errors**: Report to user with line number

---

## Design Decisions

### Why Execution Plan Model?

**Context:** Need to translate SQL to FetchXML while supporting operations FetchXML doesn't provide.

**Decision:** Build tree of execution plan nodes that can fold operations into FetchXML or execute client-side.

**Alternatives considered:**
- Direct SQL-to-FetchXML translation: Rejected - Can't handle joins, aggregations, or expressions not in FetchXML
- Full client-side execution: Rejected - Inefficient, doesn't leverage Dataverse indexes

**Consequences:**
- Positive: Maximum query support with optimal performance
- Positive: Reuses SQL Server execution plan concepts developers know
- Negative: Complex codebase (80+ node types, 6000+ line builder)

### Why Multi-Target Frameworks?

**Context:** Engine must run in XrmToolBox (.NET Framework), SSMS extensions (.NET Framework), and modern apps (.NET 8).

**Decision:** Multi-target Engine and Export projects to net8.0 and net462/net48.

**Test results:**
| Scenario | Result |
|----------|--------|
| XrmToolBox (net48) | Works with net462 target |
| SSMS 21/22 (net472) | Works with net462 target |
| Azure Data Studio (net8.0) | Works with net8.0 target |
| .NET 8 console app | Works with net8.0 target |

**Consequences:**
- Positive: Single codebase serves all hosts
- Negative: Must maintain compatibility with older .NET features
- Negative: Conditional compilation for Dataverse client differences

### Why Shared Projects for SSMS?

**Context:** SSMS 21 and 22 require different assembly references but share most code.

**Decision:** Use shared project (`.shproj`) with version-specific wrapper projects.

**Consequences:**
- Positive: No code duplication between SSMS versions
- Positive: Easy to add SSMS 23+ support
- Negative: Slightly more complex project structure

---

## Testing

### Acceptance Criteria

- [ ] Parse and execute standard SELECT/INSERT/UPDATE/DELETE
- [ ] Translate supported constructs to FetchXML
- [ ] Fall back to client-side execution for unsupported operations
- [ ] Report SQL Server-compatible errors
- [ ] Support cancellation at reasonable checkpoints

### Test Coverage

| Area | Project | Approach |
|------|---------|----------|
| Engine unit tests | MarkMpn.Sql4Cds.Engine.Tests | FakeXrmEasy mocking |
| FetchXML conversion | MarkMpn.Sql4Cds.Engine.FetchXml.Tests | Round-trip validation |
| Integration tests | MarkMpn.Sql4Cds.Tests | End-to-end with XTB plugin |

### Test Examples

```csharp
[TestMethod]
public void SelectSimple()
{
    var context = new XrmFakedContext();
    context.Initialize(new Entity("account") { Id = Guid.NewGuid(), ["name"] = "Test" });

    using var con = new Sql4CdsConnection(context.GetOrganizationService());
    using var cmd = con.CreateCommand();
    cmd.CommandText = "SELECT name FROM account";

    using var reader = cmd.ExecuteReader();
    Assert.IsTrue(reader.Read());
    Assert.AreEqual("Test", reader.GetString(0));
}
```

---

## Related Specs

- [type-system.md](./type-system.md) - SQL type system implementation
- [execution-plan-nodes.md](./execution-plan-nodes.md) - Node interface hierarchy
- [query-compilation.md](./query-compilation.md) - ExecutionPlanBuilder details
- [ado-net-provider.md](./ado-net-provider.md) - ADO.NET layer specifics
- [metadata-caching.md](./metadata-caching.md) - Caching strategies
- [language-server.md](./language-server.md) - LSP implementation
- [host-integrations.md](./host-integrations.md) - XTB/SSMS details

---

## Roadmap

- Extended TDS Endpoint support as Dataverse adds features
- Improved query optimization for complex joins
- Additional SQL Server function compatibility
- Performance improvements for large result sets
