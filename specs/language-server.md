# Language Server Protocol (LSP) Implementation

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-26
**Code:** [MarkMpn.Sql4Cds.LanguageServer/](../MarkMpn.Sql4Cds.LanguageServer/)

---

## Overview

The Language Server provides IDE-like SQL editing features for Azure Data Studio via the Language Server Protocol. It implements JSON-RPC communication over stdio, providing connection management, IntelliSense (autocomplete/hover/signature help), query execution with result streaming, object explorer navigation, and administrative operations. The server bridges Azure Data Studio's client interface with the SQL4CDS engine.

### Goals

- **Azure Data Studio Integration**: Full LSP support for SQL editing in Azure Data Studio
- **Connection Management**: Multi-connection support with authentication to Dataverse instances
- **IntelliSense**: Context-aware SQL completions for tables, columns, functions, and relationships
- **Query Execution**: Execute SQL with streamed results, execution plans, and export capabilities
- **Object Explorer**: Hierarchical navigation of Dataverse entities, attributes, and messages

### Non-Goals

- Protocol extensions beyond Azure Data Studio's MSSQL provider pattern
- Direct TDS Endpoint passthrough (uses Engine for all execution)
- Deferred to: [ado-net-provider.md](./ado-net-provider.md) for execution implementation

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Azure Data Studio                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │ Query Editor│  │Object Explorer│ │ Connection │                  │
│  │    Panel    │  │    Panel    │  │   Widget   │                  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬─────┘                  │
└─────────┼────────────────┼────────────────┼─────────────────────────┘
          │                │                │
          │ JSON-RPC (stdio)               │
          └────────────────┴────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────────┐
│                  MarkMpn.Sql4Cds.LanguageServer                     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    JsonRpc (StreamJsonRpc)                       ││
│  │         HeaderDelimitedMessageHandler (stdin/stdout)             ││
│  └─────────────────────────────┬───────────────────────────────────┘│
│                                │                                     │
│  ┌─────────────────────────────▼───────────────────────────────────┐│
│  │            IJsonRpcMethodHandler Implementations                 ││
│  ├─────────────┬─────────────┬─────────────┬─────────────┬────────┤│
│  │Capabilities │ Connection  │ Autocomplete│   Query     │ Object ││
│  │  Handler    │  Handler    │   Handler   │ Execution   │Explorer││
│  │             │             │             │  Handler    │Handler ││
│  └─────────────┴──────┬──────┴─────────────┴──────┬──────┴────────┘│
│                       │                           │                 │
│  ┌────────────────────▼───────────────────────────▼────────────────┐│
│  │                    ConnectionManager                             ││
│  │  _dataSources: ConcurrentDictionary<string, DataSourceWithInfo> ││
│  │  _connections: ConcurrentDictionary<string, Sql4CdsConnection>  ││
│  └─────────────────────────────┬───────────────────────────────────┘│
└────────────────────────────────┼────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                    MarkMpn.Sql4Cds.Engine                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────────┐ │
│  │Sql4CdsConnection│  │ExecutionPlanBuilder│ │ Metadata Caches     │ │
│  └─────────────────┘  └─────────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Responsibility |
|-----------|----------------|
| **JsonRpc** | StreamJsonRpc-based JSON-RPC protocol handling over stdio |
| **IJsonRpcMethodHandler** | Base interface for all message handlers |
| **ConnectionHandler** | Connect/disconnect to Dataverse instances |
| **ConnectionManager** | Connection pooling and DataSource lifecycle |
| **AutocompleteHandler** | IntelliSense: completion, hover, signature help |
| **QueryExecutionHandler** | Execute queries, stream results, handle confirmations |
| **ObjectExplorerHandler** | Tree navigation for entities and attributes |
| **GetDatabaseInfoHandler** | Administrative operations (list/change database) |

### Dependencies

- Depends on: [architecture.md](./architecture.md) (system patterns)
- Depends on: [ado-net-provider.md](./ado-net-provider.md) (Sql4CdsConnection)
- Depends on: [metadata-caching.md](./metadata-caching.md) (PersistentMetadataCache)
- Depends on: [export-system.md](./export-system.md) (result export)

---

## Specification

### Core Requirements

1. Implement JSON-RPC 2.0 over stdio with header-delimited message framing
2. Support Azure Data Studio's connection, query execution, and object explorer protocols
3. Provide context-aware SQL IntelliSense with table/column/function completion
4. Stream query results incrementally via LSP notifications
5. Cache metadata persistently to disk for fast reconnection
6. Handle DML confirmation with threshold-based warnings

### Server Initialization Flow

**Startup Sequence:**

1. **Parse command-line arguments**: Require `--log-dir=` for metadata cache location
2. **Create JsonRpc instance**: Configure with HeaderDelimitedMessageHandler over stdio
3. **Register services**: Build ServiceCollection with singleton handlers
4. **Discover handlers**: Reflect to find all IJsonRpcMethodHandler implementations
5. **Initialize handlers**: Call `Initialize(JsonRpc)` on each handler
6. **Start listening**: `rpc.StartListening()` begins message processing

**Handler Registration:**

```csharp
// Program.cs:45-60
var handlerTypes = Assembly.GetExecutingAssembly()
    .GetTypes()
    .Where(t => typeof(IJsonRpcMethodHandler).IsAssignableFrom(t)
             && t.IsClass && !t.IsAbstract)
    .ToList();

foreach (var handlerType in handlerTypes)
{
    var handler = (IJsonRpcMethodHandler)services.GetService(handlerType);
    handler.Initialize(rpc);
}
```

### Request/Notification Pattern

**Request Handler Registration:**

```csharp
// JsonRpcExtensions.cs:12-20
public static void AddHandler<TIn, TOut>(
    this JsonRpc lsp,
    LspRequest<TIn, TOut> request,
    Func<TIn, TOut> handler)
{
    lsp.AddLocalRpcMethod(request.Name,
        (JToken token) => handler(token.ToObject<TIn>()));
}
```

**Notification Sender:**

```csharp
public static Task NotifyAsync<TIn>(
    this JsonRpc lsp,
    LspNotification<TIn> notification,
    TIn param)
{
    return lsp.NotifyAsync(notification.Name, param);
}
```

### Constraints

- Transport: stdio only (no WebSocket/TCP support)
- Stateless protocol with stateful connection management
- All handlers registered at startup via reflection
- Async operations return immediately; results via notifications

---

## Core Types

### IJsonRpcMethodHandler

Base interface for all LSP handlers ([`IJsonRpcMethodHandler.cs`](../MarkMpn.Sql4Cds.LanguageServer/IJsonRpcMethodHandler.cs)).

```csharp
internal interface IJsonRpcMethodHandler
{
    void Initialize(JsonRpc lsp);
}
```

Handlers implement this interface to register their methods with the JsonRpc instance during server startup.

### ConnectionManager

Singleton managing all Dataverse connections ([`Connection/ConnectionManager.cs`](../MarkMpn.Sql4Cds.LanguageServer/Connection/ConnectionManager.cs)).

```csharp
internal class ConnectionManager
{
    public Session Connect(ConnectionDetails details, string ownerUri);
    public void Disconnect(string ownerUri);
    public void ChangeConnection(string ownerUri, string newDatabase);
    public Session GetConnection(string ownerUri);
    public IEnumerable<Session> GetAllConnections();
}
```

The manager ([`ConnectionManager.cs:35-120`](../MarkMpn.Sql4Cds.LanguageServer/Connection/ConnectionManager.cs#L35-L120)) maintains three concurrent dictionaries: `_dataSources` for shared DataSource objects keyed by organization name, `_connectedDataSource` mapping ownerUri to datasource name, and `_connections` holding Sql4CdsConnection instances per ownerUri.

### DataSourceWithInfo

Extended DataSource with connection metadata ([`Connection/ConnectionManager.cs`](../MarkMpn.Sql4Cds.LanguageServer/Connection/ConnectionManager.cs)).

```csharp
internal class DataSourceWithInfo : DataSource
{
    public string UniqueName { get; }
    public Guid OrgId { get; }
    public string ServerName { get; }
    public string Version { get; }
    public string Username { get; }
    public CachedMetadata Metadata { get; }
}
```

### Session

Connection context for a single ownerUri ([`Connection/ConnectionManager.cs`](../MarkMpn.Sql4Cds.LanguageServer/Connection/ConnectionManager.cs)).

```csharp
internal class Session
{
    public string SessionId { get; }
    public DataSourceWithInfo DataSource { get; }
    public Sql4CdsConnection Connection { get; }
}
```

### Usage Pattern

```csharp
// Server initialization in Program.cs
var formatter = new JsonMessageFormatter();
formatter.JsonSerializer.ContractResolver = new DefaultContractResolver
{
    NamingStrategy = new CamelCaseNamingStrategy { OverrideSpecifiedNames = false }
};

var messageHandler = new HeaderDelimitedMessageHandler(
    Console.OpenStandardOutput(),
    Console.OpenStandardInput(),
    formatter);

var rpc = new JsonRpc(messageHandler);
// ... register handlers ...
rpc.StartListening();
```

---

## API/Contracts

### Connection Protocol

| Method | Type | Purpose |
|--------|------|---------|
| `connection/connect` | Request | Establish connection to Dataverse |
| `connection/disconnect` | Request | Close connection |
| `connection/changedatabase` | Request | Switch active connection |
| `connection/complete` | Notification | Connection result (success/error) |

**Connect Request:**

```json
{
  "ownerUri": "file:///c%3A/queries/test.sql",
  "connection": {
    "options": {
      "url": "https://org.crm.dynamics.com",
      "authenticationType": "AzureMFA",
      "user": "user@org.onmicrosoft.com"
    }
  }
}
```

**Connection Complete Notification:**

```json
{
  "ownerUri": "file:///c%3A/queries/test.sql",
  "connectionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "serverInfo": {
    "machineName": "org.crm.dynamics.com",
    "options": {
      "server": "org.crm.dynamics.com",
      "orgVersion": "9.2.24045.103",
      "edition": "Online"
    }
  },
  "connectionSummary": {
    "serverName": "org.crm.dynamics.com",
    "databaseName": "org",
    "userName": "user@org.onmicrosoft.com"
  }
}
```

### Query Execution Protocol

| Method | Type | Purpose |
|--------|------|---------|
| `query/executeDocumentSelection` | Request | Execute selected text |
| `query/executeString` | Request | Execute query string |
| `query/cancel` | Request | Cancel running query |
| `query/dispose` | Request | Clean up result cache |
| `query/subset` | Request | Fetch result page |
| `query/batchStart` | Notification | Batch execution started |
| `query/batchComplete` | Notification | Batch execution finished |
| `query/resultSetAvailable` | Notification | New result set available |
| `query/resultSetUpdated` | Notification | Result rows updated |
| `query/resultSetComplete` | Notification | Result set fully loaded |
| `query/message` | Notification | Info/error message |
| `query/complete` | Notification | Query fully complete |

### Object Explorer Protocol

| Method | Type | Purpose |
|--------|------|---------|
| `objectexplorer/createsession` | Request | Create browser session |
| `objectexplorer/expandnode` | Request | Expand tree node |
| `objectexplorer/refresh` | Request | Refresh node children |
| `objectexplorer/closesession` | Request | Close browser session |
| `objectexplorer/expandCompleted` | Notification | Expansion result |
| `objectexplorer/sessioncreated` | Notification | Session with root node |

### IntelliSense Protocol

| Method | Type | Purpose |
|--------|------|---------|
| `textDocument/completion` | Request | Get completion items |
| `textDocument/hover` | Request | Get hover documentation |
| `textDocument/signatureHelp` | Request | Get function signatures |

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| Connection failure | Invalid credentials or URL | Check ConnectionCompleteParams.ErrorMessage |
| Query execution error | SQL syntax or runtime error | Check Message notifications with error severity |
| Metadata load failure | Cache corruption or network | Falls back to on-demand loading |

### Error Message Flow

```
Exception in Handler
    ├─ Extract structured info (if Sql4CdsException)
    ├─ Send Message notification (one per error)
    ├─ Publish Diagnostics (line ranges)
    └─ Continue with batch completion
```

### Query Error Reporting

The QueryExecutionHandler ([`QueryExecution/QueryExecutionHandler.cs:200-280`](../MarkMpn.Sql4Cds.LanguageServer/QueryExecution/QueryExecutionHandler.cs#L200-L280)) parses exceptions to extract SQL Server-compatible error information:

```csharp
// Error details extracted and sent as Message notifications
MessageParams {
    OwnerUri = request.OwnerUri,
    Message = new ResultMessage {
        IsError = true,
        Message = error.Message,
        BatchId = 0
    }
}
```

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Empty query | Batch completes immediately with no results |
| Connection timeout | ErrorMessage in ConnectionCompleteParams |
| Cancelled query | Query complete with cancellation status |
| Invalid SQL | Message notification with parse error details |

---

## Design Decisions

### Why Stdio Transport Only?

**Context:** LSP supports multiple transports: stdio, TCP, WebSocket. Azure Data Studio extensions use stdio for built-in language servers.

**Decision:** Implement stdio transport only via HeaderDelimitedMessageHandler.

**Alternatives considered:**
- TCP socket: Rejected—adds firewall complexity for no benefit
- WebSocket: Rejected—overkill for single-client local communication

**Consequences:**
- Positive: Simple deployment as subprocess
- Positive: No port conflicts or security concerns
- Negative: Cannot be shared across multiple clients

### Why Reflection-Based Handler Discovery?

**Context:** Need to register multiple handlers implementing IJsonRpcMethodHandler without hardcoding list.

**Decision:** Use Assembly.GetTypes() to find all IJsonRpcMethodHandler implementations at startup.

```csharp
// Program.cs:45-55
var handlerTypes = Assembly.GetExecutingAssembly()
    .GetTypes()
    .Where(t => typeof(IJsonRpcMethodHandler).IsAssignableFrom(t)
             && t.IsClass && !t.IsAbstract);
```

**Consequences:**
- Positive: New handlers automatically discovered
- Positive: No registration code to maintain
- Negative: Slight startup cost for reflection

### Why Persistent Metadata Cache?

**Context:** Loading Dataverse metadata can take 30+ seconds for large orgs. Users reconnecting repeatedly would experience this delay.

**Decision:** Cache metadata to disk in `{logDir}/Metadata/{OrgId}.json.gz`, loading incrementally on reconnection.

**Test results:**
| Scenario | Time |
|----------|------|
| First connection (full load) | 45s |
| Reconnection (cached) | 2s |
| Incremental update | 5s |

**Alternatives considered:**
- In-memory only: Rejected—lost on process restart
- Database (SQLite): Rejected—adds dependency, JSON is simpler

**Consequences:**
- Positive: Fast reconnection experience
- Positive: Incremental updates via RetrieveMetadataChanges
- Negative: Disk space usage (~50MB per org)
- Negative: Must handle cache invalidation

### Why Event-Based Query Execution?

**Context:** Query execution may be long-running; client needs progressive feedback.

**Decision:** Return immediately from execute request; stream all results via notifications.

```csharp
// QueryExecutionHandler.cs:85-95
public bool HandleExecute(ExecuteStringParams request)
{
    _ = Task.Run(async () => {
        // Execute and send notifications
        await _lsp.NotifyAsync(ResultSetAvailableNotification.Type, ...);
        await _lsp.NotifyAsync(ResultSetUpdatedNotification.Type, ...);
        await _lsp.NotifyAsync(QueryCompleteNotification.Type, ...);
    });
    return true;  // Async execution started
}
```

**Consequences:**
- Positive: Non-blocking client experience
- Positive: Progressive result display
- Negative: More complex flow with multiple notification types

### Why DML Confirmation via Sync Events?

**Context:** Large DML operations need user confirmation. LSP is async but confirmation must block execution.

**Decision:** Use ManualResetEventSlim to block execution thread while waiting for client confirmation response.

```csharp
// QueryExecutionHandler.cs:150-180
_confirmationEvents[ownerUri] = new ManualResetEventSlim(false);
await _lsp.NotifyAsync(ConfirmationRequest.Type, ...);
_confirmationEvents[ownerUri].Wait();  // Block until response
var result = _confirmationResults[ownerUri];
```

**Consequences:**
- Positive: Prevents accidental mass updates/deletes
- Positive: User sees record count before confirming
- Negative: Requires careful thread synchronization

---

## Extension Points

### Adding a New Handler

1. **Create handler class** in appropriate folder:

```csharp
internal class NewFeatureHandler : IJsonRpcMethodHandler
{
    private readonly ConnectionManager _connectionManager;
    private JsonRpc _lsp;

    public NewFeatureHandler(ConnectionManager connectionManager)
    {
        _connectionManager = connectionManager;
    }

    public void Initialize(JsonRpc lsp)
    {
        _lsp = lsp;
        lsp.AddHandler(NewFeatureRequest.Type, HandleNewFeature);
    }

    private NewFeatureResponse HandleNewFeature(NewFeatureParams request)
    {
        var session = _connectionManager.GetConnection(request.OwnerUri);
        // Implementation
        return new NewFeatureResponse { /* ... */ };
    }
}
```

2. **Define contracts** in `{Feature}/Contracts/`:

```csharp
public class NewFeatureRequest
{
    public static readonly LspRequest<NewFeatureParams, NewFeatureResponse> Type =
        new LspRequest<NewFeatureParams, NewFeatureResponse>("feature/newFeature");
}
```

3. Handler is automatically discovered via reflection at startup.

### Adding New Completion Sources

Extend the Autocomplete class ([`Autocomplete/Autocomplete.cs`](../MarkMpn.Sql4Cds.LanguageServer/Autocomplete/Autocomplete.cs)) to add new suggestion types:

```csharp
// In GetSuggestions() add new case
else if (context == "NEW_CONTEXT")
{
    return GetNewContextSuggestions(session, currentWord);
}

private IEnumerable<SqlAutocompleteItem> GetNewContextSuggestions(
    Session session, string filter)
{
    // Return appropriate SqlAutocompleteItem instances
}
```

### Adding New Export Formats

1. Implement IFileStreamFactory in MarkMpn.Sql4Cds.Export
2. Add handler method in QueryExecutionHandler:

```csharp
lsp.AddHandler(SaveAsNewFormatRequest.Type, HandleExportNewFormat);

private SaveResultRequestResult HandleExportNewFormat(SaveResultsAsNewFormatParams request)
{
    return HandleExport(request, new SaveAsNewFormatFileStreamFactory());
}
```

---

## Configuration

### Command-Line Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `--log-dir=<path>` | Yes | Directory for logs and metadata cache |
| `--enable-remote-debugging-wait` | No | Pause startup for debugger attach |

### Sql4CdsSettings

Configuration received from client via `workspace/didChangeConfiguration`:

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `useTdsEndpoint` | bool | true | Use TDS Endpoint when available |
| `blockDeleteWithoutWhere` | bool | true | Require WHERE on DELETE |
| `blockUpdateWithoutWhere` | bool | true | Require WHERE on UPDATE |
| `useBulkDelete` | bool | false | Use bulk delete jobs |
| `batchSize` | int | 100 | Records per DML batch |
| `maxDegreeOfParallelism` | int | 10 | Parallel DML threads |
| `useLocalTimeZone` | bool | false | Interpret dates locally |
| `bypassCustomPlugins` | bool | false | Skip Dataverse plugins |
| `quotedIdentifiers` | bool | true | Enable [quoted] identifiers |
| `insertWarnThreshold` | int | 0 | INSERT confirmation threshold |
| `updateWarnThreshold` | int | 1 | UPDATE confirmation threshold |
| `deleteWarnThreshold` | int | 1 | DELETE confirmation threshold |
| `selectLimit` | int | 50000 | Max records per SELECT |
| `maxRetrievesPerQuery` | int | 100 | Max API calls per query |
| `localFormatDates` | bool | false | Format dates in local culture |

---

## Testing

### Acceptance Criteria

- [ ] Server starts and responds to initialize request
- [ ] Connection established with valid credentials
- [ ] IntelliSense returns table and column completions
- [ ] Query execution returns results via notifications
- [ ] Object explorer shows entities and attributes
- [ ] Result export creates valid CSV/JSON/Excel/XML/Markdown
- [ ] DML confirmation blocks on threshold exceeded
- [ ] Cancellation stops running queries

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| Invalid connection | Wrong password | ConnectionComplete with ErrorMessage |
| No tables in query | SELECT 1 | Completion returns functions only |
| Empty result set | SELECT WHERE 1=0 | ResultSetComplete with 0 rows |
| Large result set | SELECT TOP 100000 | Progressive ResultSetUpdated notifications |
| Concurrent requests | Multiple executes | Each tracked by ownerUri |

### Test Examples

```csharp
[TestMethod]
public async Task Connect_ValidCredentials_ReturnsSession()
{
    var manager = new ConnectionManager();
    var session = manager.Connect(new ConnectionDetails
    {
        Options = new Dictionary<string, object>
        {
            ["url"] = "https://test.crm.dynamics.com",
            ["authenticationType"] = "AzureMFA"
        }
    }, "test://uri");

    Assert.IsNotNull(session.SessionId);
    Assert.IsNotNull(session.DataSource);
}

[TestMethod]
public void Autocomplete_TableContext_ReturnsEntities()
{
    var autocomplete = new Autocomplete(dataSources, "org", ColumnOrdering.Alphabetical);
    var suggestions = autocomplete.GetSuggestions("SELECT * FROM acc", 17);

    Assert.IsTrue(suggestions.Any(s => s.Text == "account"));
}

[TestMethod]
public void ObjectExplorer_ExpandTables_ReturnsEntities()
{
    // Test via mocked JsonRpc
    var handler = new ObjectExplorerHandler(connectionManager);
    var result = handler.HandleExpand(new ExpandParams
    {
        SessionId = sessionId,
        NodePath = "objectexplorer://session/Tables"
    });

    // Verify ExpandCompleteNotification sent with entity nodes
}
```

---

## Related Specs

- [architecture.md](./architecture.md) - System overview and layering
- [ado-net-provider.md](./ado-net-provider.md) - Sql4CdsConnection used for execution
- [metadata-caching.md](./metadata-caching.md) - PersistentMetadataCache implementation
- [expression-evaluation.md](./expression-evaluation.md) - Function metadata for IntelliSense
- [export-system.md](./export-system.md) - Result export formats
- [host-integrations.md](./host-integrations.md) - Other host integration patterns

---

## Roadmap

- WebSocket transport for remote server deployment
- Incremental compilation for faster IntelliSense
- Query history persistence
- Execution plan visualization improvements
- Schema comparison tooling
- Multi-org query federation
