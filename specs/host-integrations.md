# Host Integrations

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-26
**Code:** [MarkMpn.Sql4Cds.XTB/](../MarkMpn.Sql4Cds.XTB/), [MarkMpn.Sql4Cds.SSMS/](../MarkMpn.Sql4Cds.SSMS/), [AzureDataStudioExtension/](../AzureDataStudioExtension/)

---

## Overview

SQL4CDS integrates with three host environments to provide SQL query capabilities against Dataverse: XrmToolBox (standalone desktop application), SQL Server Management Studio (Visual Studio extension), and Azure Data Studio (LSP-based extension). Each integration adapts the core ADO.NET engine to the host's UI paradigm while sharing common execution plan visualization through the Controls library.

### Goals

- **XrmToolBox Integration**: Full-featured query editor with tabbed documents, autocomplete, and session persistence
- **SSMS Integration**: Intercept query execution for Dataverse connections and display results in native SSMS grids
- **Azure Data Studio Integration**: LSP-based connection provider with progress notifications and confirmation dialogs

### Non-Goals

- Unified code across all hosts (each requires host-specific patterns)
- Real-time collaboration or shared sessions
- Mobile or web-based host integrations

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              HOST LAYER                                       │
├─────────────────────┬─────────────────────┬─────────────────────────────────┤
│   XrmToolBox (XTB)  │    SSMS (21/22)     │    Azure Data Studio (ADS)      │
│                     │                     │                                  │
│  ┌───────────────┐  │  ┌───────────────┐  │  ┌───────────────┐              │
│  │ PluginControl │  │  │Sql4CdsPackage │  │  │  main.ts      │              │
│  │   (MEF)       │  │  │  (AsyncPkg)   │  │  │  (LSP Client) │              │
│  └───────┬───────┘  │  └───────┬───────┘  │  └───────┬───────┘              │
│          │          │          │          │          │                       │
│  ┌───────▼───────┐  │  ┌───────▼───────┐  │          │                       │
│  │SqlQueryControl│  │  │  DmlExecute   │  │          ▼                       │
│  │   (WinForms)  │  │  │  (Reflection) │  │  ┌───────────────┐              │
│  └───────┬───────┘  │  └───────┬───────┘  │  │Language Server│              │
│          │          │          │          │  │   (stdio)     │              │
└──────────┼──────────┴──────────┼──────────┴──┴───────┬───────┴──────────────┘
           │                     │                     │
           ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SHARED COMPONENTS                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │ExecutionPlanView│  │Sql4CdsConnection│  │     QueryExecutionOptions   │  │
│  │   (Controls)    │  │   (ADO.NET)     │  │     (Host-Specific)         │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Responsibility |
|-----------|----------------|
| **PluginControl (XTB)** | MEF plugin entry point, docked window management, settings |
| **SqlQueryControl (XTB)** | Query editor with Scintilla, results grid, execution plan |
| **Sql4CdsPackage (SSMS)** | Visual Studio AsyncPackage, command registration |
| **DmlExecute (SSMS)** | Query interception via DTE CommandEvents, reflection wrappers |
| **main.ts (ADS)** | TypeScript LSP client, connection provider registration |
| **ExecutionPlanView** | Shared WinForms control for execution plan visualization |
| **QueryExecutionOptions** | Host-specific implementation of IQueryExecutionOptions |

### Dependencies

- Depends on: [ado-net-provider.md](./ado-net-provider.md)
- Depends on: [language-server.md](./language-server.md) (for ADS)
- Uses patterns from: [architecture.md](./architecture.md)

---

## Specification

### Core Requirements

1. Each host must provide connection management to Dataverse instances
2. Query execution must flow through Sql4CdsConnection/Command/Reader
3. Execution plans must be visualized using ExecutionPlanView control
4. DML operations must display confirmation dialogs with affected row counts
5. Progress must be reported during long-running queries
6. Errors must display with source line highlighting where possible

### Primary Flows

**XrmToolBox Query Execution:**

1. **User Action**: User clicks Execute button in SqlQueryControl
2. **Preparation**: Extract SQL from editor, create ExecuteParams
3. **Background Execution**: BackgroundWorker calls Sql4CdsCommand.ExecuteReader()
4. **Results Display**: DataTable bound to DataGridView, execution plan to ExecutionPlanView
5. **Completion**: Status bar updated, error indicators shown if applicable

**SSMS Query Interception:**

1. **Event Hook**: Query.Execute command fires BeforeExecute event
2. **Connection Check**: Verify connection string ends with `.dynamics.com`
3. **Query Analysis**: Sql4CdsCommand.Prepare() determines if DML execution needed
4. **Interception**: Set CancelDefault=true to prevent native SSMS execution
5. **Execution**: Execute via Sql4CdsCommand, wrap results in QEResultSetWrapper
6. **Display**: Inject results into SSMS grid via reflection

**Azure Data Studio Query:**

1. **LSP Request**: Editor sends query via LSP protocol
2. **Server Processing**: LanguageServer parses and executes query
3. **Notifications**: Progress updates via `sql4cds/progress` notification
4. **Confirmation**: DML warnings via `sql4cds/confirmation` notification
5. **Results**: Standard LSP result protocol to ADS

### Constraints

- XTB requires .NET Framework 4.8 (XrmToolBox limitation)
- SSMS requires .NET Framework 4.7.2 with AsyncPackage pattern
- ADS extension must use TypeScript with vscode-languageclient
- All hosts must handle Dataverse API throttling gracefully

---

## Core Types

### IDocumentWindow (XTB)

Interface for document windows in the XrmToolBox plugin ([`IDocumentWindow.cs:11-39`](../MarkMpn.Sql4Cds.XTB/IDocumentWindow.cs#L11-L39)).

```csharp
interface IDocumentWindow
{
    TabContent GetSessionDetails();
    void RestoreSessionDetails(TabContent tab);
    void SettingsChanged();
}

interface ISaveableDocumentWindow : IDocumentWindow
{
    string Filter { get; }
    void Save(string filename);
    void Open(string filename);
}
```

Used for SQL query tabs, FetchXML views, and M query tabs with session persistence.

### ITextBoxWrapper (XTB)

Adapter interface for text editor controls ([`Autocomplete/ITextBoxWrapper.cs:12-26`](../MarkMpn.Sql4Cds.XTB/Autocomplete/ITextBoxWrapper.cs#L12-L26)).

```csharp
public interface ITextBoxWrapper
{
    Control TargetControl { get; }
    string Text { get; }
    string SelectedText { get; set; }
    int SelectionStart { get; set; }
    Point GetPositionFromCharIndex(int pos);
}
```

Implementations include `ScintillaWrapper` for the SQL editor and `TextBoxWrapper` for generic text controls.

### ReflectionObjectBase (SSMS)

Base class for reflection-based access to SSMS internals ([`Reflection/ReflectionObjectBase.cs:11-106`](../MarkMpn.Sql4Cds.SSMS/Reflection/ReflectionObjectBase.cs#L11-L106)).

```csharp
protected object GetField(object target, string fieldName);
protected void SetField(object target, string fieldName, object value);
protected object InvokeMethod(object target, string methodName, params object[] args);
```

Required because SSMS internal APIs are not publicly exposed.

### ExecutionPlanView (Controls)

Shared WinForms control for execution plan visualization ([`ExecutionPlanView.cs`](../MarkMpn.Sql4Cds.Controls/ExecutionPlanView.cs)).

```csharp
public class ExecutionPlanView : ScrollableControl
{
    public IExecutionPlanNode Plan { get; set; }
    public bool Executed { get; set; }
    public IDictionary<string, DataSource> DataSources { get; set; }
    public event EventHandler<IExecutionPlanNode> NodeSelected;
}
```

Renders execution plan nodes as directed graph with icons, cost percentages, and tooltips.

---

## XrmToolBox Integration

### Plugin Architecture

The XTB plugin uses a bootstrapper pattern with dynamic assembly loading ([`Plugin.cs`](../MarkMpn.Sql4Cds/Plugin.cs)).

**Entry Point:**
```csharp
[Export(typeof(IXrmToolBoxPlugin)),
    ExportMetadata("Name", "SQL 4 CDS"),
    ExportMetadata("Description", "Query Dataverse using SQL")]
public class Plugin : PluginBase, IPayPalPlugin
{
    public override IXrmToolBoxPluginControl GetControl()
    {
        var controlType = _primaryAssembly.GetType("MarkMpn.Sql4Cds.XTB.PluginControl");
        return (IXrmToolBoxPluginControl)Activator.CreateInstance(controlType, this);
    }
}
```

**Implemented XTB Interfaces:**
- `IMessageBusHost` - Inter-plugin communication
- `IGitHubPlugin` - Version checking
- `IHelpPlugin` - Help documentation
- `ISettingsPlugin` - Settings dialog integration
- `IPayPalPlugin` - Donation support

### Document Window Management

Uses WeifenLuo.WinFormsUI.Docking for tabbed interface ([`PluginControl.cs:264-300`](../MarkMpn.Sql4Cds.XTB/PluginControl.cs#L264-L300)).

**Document Types:**
| Type | Class | Purpose |
|------|-------|---------|
| SQL Query | SqlQueryControl | SQL editor with results |
| FetchXML | FetchXmlControl | FetchXML viewer |
| M Query | MQueryControl | Power BI M output |

**Session Persistence:**
```csharp
// Save session
Settings.Instance.Session = dockPanel.Contents
    .OfType<IDocumentWindow>()
    .Select(query => query.GetSessionDetails())
    .ToArray();

// Restore session
dockPanel.LoadFromXml(stream, (name) => {
    var query = CreateQuery(connection, null);
    query.RestoreSessionDetails(savedTab);
    return query;
});
```

### Query Execution Flow

Execution handled in background worker ([`SqlQueryControl.cs:1107-1224`](../MarkMpn.Sql4Cds.XTB/SqlQueryControl.cs#L1107-L1224)).

```csharp
private void backgroundWorker_DoWork(object sender, DoWorkEventArgs e)
{
    using (var cmd = _connection.CreateCommand())
    using (var options = new QueryExecutionOptions(this, backgroundWorker, _connection, cmd))
    {
        options.ApplySettings(args.Execute);
        cmd.CommandText = sql;

        if (args.Execute)
        {
            using (var reader = cmd.ExecuteReader())
            {
                while (!reader.IsClosed)
                {
                    var dataTable = new DataTable();
                    dataTable.Load(reader);
                    ShowResult(null, args, dataTable, null, null);
                }
            }
        }
        else
        {
            var plan = cmd.GeneratePlan(false);
            foreach (var query in plan)
                ShowResult(query, args, null, null, null);
        }
    }
}
```

### Autocomplete System

Custom autocomplete with Scintilla integration ([`Autocomplete/`](../MarkMpn.Sql4Cds.XTB/Autocomplete/)).

**Components:**
- `AutocompleteMenu` - Orchestrates popup display and item selection
- `AutocompleteListView` - Custom UserControl rendering suggestions
- `ScintillaWrapper` - Adapts Scintilla control to ITextBoxWrapper
- `AutocompleteItem` - Individual suggestion with icon and tooltip

**Configuration:**
- `AppearInterval` - Milliseconds before popup shows
- `MaximumSize` - Popup dimensions
- `ImageList` - Icons for different suggestion types

---

## SSMS Integration

### Package Architecture

Visual Studio extension using AsyncPackage pattern ([`Sql4CdsPackage.cs:14-40`](../MarkMpn.Sql4Cds.SSMS/Sql4CdsPackage.cs#L14-L40)).

```csharp
[PackageRegistration(UseManagedResourcesOnly = true, AllowsBackgroundLoading = true)]
[InstalledProductRegistration("#110", "#112", "1.0", IconResourceID = 400)]
[ProvideMenuResource("Menus.ctmenu", 1)]
[ProvideAutoLoad(UIContextGuids.NoSolution, PackageAutoLoadFlags.BackgroundLoad)]
[ProvideOptionPage(typeof(OptionsPage), "SQL 4 CDS", "General", 0, 0, true, 0)]
public sealed class Sql4CdsPackage : AsyncPackage
```

**VSCT Commands:**
| Command ID | Name | Purpose |
|------------|------|---------|
| 0x0100 | Sql2FetchXmlCommand | Convert SQL to FetchXML |
| 0x0200 | FetchXml2SqlCommand | Convert FetchXML to SQL |
| 0x0300 | Sql2MCommand | Convert SQL to Power BI M |

### Query Interception

Hooks into SSMS command events via DTE ([`DmlExecute.cs:59-76`](../MarkMpn.Sql4Cds.SSMS/DmlExecute.cs#L59-L76)).

```csharp
var execute = dte.Commands.Item("Query.Execute");
QueryExecuteEvent = dte.Events.CommandEvents[execute.Guid, execute.ID];
QueryExecuteEvent.BeforeExecute += OnExecuteQuery;

var cancel = dte.Commands.Item("Query.CancelExecutingQuery");
QueryCancelEvent = dte.Events.CommandEvents[cancel.Guid, cancel.ID];
QueryCancelEvent.BeforeExecute += OnCancelQuery;
```

**Interception Logic:**
1. Check if connection targets `.dynamics.com` domain
2. Parse query with Sql4CdsCommand.Prepare()
3. If `UseTDSEndpointDirectly`, let SSMS handle natively
4. Otherwise, set `CancelDefault = true` and execute via SQL4CDS

### Reflection Wrappers

Access SSMS internal types that lack public APIs.

| Wrapper Class | SSMS Type | Purpose |
|---------------|-----------|---------|
| SqlScriptEditorControlWrapper | SqlScriptEditorControl | Access SQL editor |
| ScriptFactoryWrapper | IScriptFactory | Get current editor |
| DisplaySQLResultsControlWrapper | DisplaySQLResultsControl | Manage results panel |
| QEResultSetWrapper | QEResultSet | Wrap data reader |
| ResultSetAndGridContainerWrapper | ResultSetAndGridContainer | Create result grid |

**Example ([`SqlScriptEditorControlWrapper.cs:13-82`](../MarkMpn.Sql4Cds.SSMS/Reflection/SqlScriptEditorControlWrapper.cs#L13-L82)):**
```csharp
public SqlScriptEditorControlWrapper(object obj)
{
    Results = new DisplaySQLResultsControlWrapper(GetField(obj, "m_sqlResultsControl"));
    ServiceProvider = new ServiceProvider(GetField(obj, "m_serviceProvider"));
}

public string ConnectionString =>
    ((IDbConnection)GetField(Target, "m_connection"))?.ConnectionString;
```

### Authentication Bridge

Reuses SSMS's SqlAuthenticationProvider for OAuth tokens ([`AuthOverrideHook.cs:14-44`](../MarkMpn.Sql4Cds.SSMS/AuthOverrideHook.cs#L14-L44)).

```csharp
public string GetAuthToken(Uri connectedUri)
{
    // Check token cache
    if (_tokenCache.TryGetValue(connectedUri.ToString(), out var existingToken))
    {
        var parsedToken = new JwtSecurityToken(existingToken);
        if (parsedToken.ValidTo > DateTime.Now)
            return existingToken;
    }

    // Acquire new token via SSMS auth context
    var authParams = _authParams[uri.ToString()];
    var authProv = SqlAuthenticationProvider.GetProvider(authParams.AuthenticationMethod);
    var token = authProv.AcquireTokenAsync(authParams).Result;

    _tokenCache[connectedUri.ToString()] = token.AccessToken;
    return token.AccessToken;
}
```

---

## Azure Data Studio Integration

### Extension Structure

TypeScript LSP client extension ([`AzureDataStudioExtension/`](../AzureDataStudioExtension/)).

**Manifest ([`package.json`](../AzureDataStudioExtension/package.json)):**
- Provider ID: `SQL4CDS`
- Document selector: `['sql']`
- Activation: `*` (all events)
- Engine: Azure Data Studio >= 1.30.0

**Connection Options:**
| Option | Type | Purpose |
|--------|------|---------|
| authType | enum | AzureMFA, SqlLogin, Integrated, None |
| url | string | Dataverse instance URL |
| username | string | User credentials |
| clientid | string | OAuth client ID (S2S) |
| clientsecret | string | OAuth client secret (S2S) |

### LSP Client Setup

Activates and connects to .NET language server ([`main.ts:26-120`](../AzureDataStudioExtension/src/main.ts#L26-L120)).

```typescript
// Server executable configuration
const serverOptions: ServerOptions = {
    command: "dotnet",
    args: [serverPath],
    transport: TransportKind.stdio
};

// Client options
const clientOptions: ClientOptions = {
    providerId: Constants.providerId,
    documentSelector: ['sql'],
    synchronize: { configurationSection: "SQL4CDS" }
};

// Create and start client
const languageClient = new SqlOpsDataClient(
    Constants.serviceName,
    serverOptions,
    clientOptions
);
await languageClient.start();
```

### Notification Handlers

Custom notifications for SQL4CDS-specific features.

**Progress Notifications:**
```typescript
languageClient.onNotification("sql4cds/progress", (params) => {
    statusBarItem.text = params.message;
    statusBarItem.show();
});
```

**Confirmation Dialogs:**
```typescript
languageClient.onNotification("sql4cds/confirmation", async (params) => {
    const result = await vscode.window.showInformationMessage(
        params.message,
        "Yes", "All", "No"
    );
    languageClient.sendNotification("sql4cds/confirm", { confirmed: result === "Yes" });
});
```

**Diagnostics:**
```typescript
languageClient.onNotification("textDocument/publishDiagnostics", (params) => {
    const diagnostics = params.diagnostics.map(d => new vscode.Diagnostic(
        new vscode.Range(d.range.start.line, d.range.start.character,
                        d.range.end.line, d.range.end.character),
        d.message,
        d.severity
    ));
    diagnosticCollection.set(vscode.Uri.parse(params.uri), diagnostics);
});
```

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| QueryExecutionException | Runtime execution failure | Display error, highlight source line |
| Connection timeout | Dataverse unreachable | Retry with exponential backoff |
| Throttling (429) | Rate limit exceeded | Wait for Retry-After header |
| Authentication failure | Invalid credentials | Prompt for re-authentication |

### Recovery Strategies

- **XTB**: Errors displayed in Messages tab with clickable line numbers
- **SSMS**: Error injected into SSMS Messages window via reflection
- **ADS**: Errors published as LSP diagnostics with source location

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Query cancelled mid-execution | Clean up resources, show "Query cancelled" message |
| Connection lost during query | Retry once, then show connection error |
| Large result set (>10,000 rows) | Progress updates, memory-efficient streaming |
| Empty result set | Show "(0 rows affected)" message |

---

## Design Decisions

### Why Reflection for SSMS?

**Context:** SSMS internal APIs (SQLEditors.dll) are not publicly documented or exposed.

**Decision:** Create reflection wrappers around SSMS internal types.

**Alternatives considered:**
- Request public API from Microsoft: Rejected - No timeline, blocks development
- Create custom results window: Rejected - Breaks SSMS UX expectations
- Output to text only: Rejected - Loses grid functionality users expect

**Consequences:**
- Positive: Native SSMS grid experience for users
- Negative: Fragile to SSMS internal changes
- Negative: Must test against each SSMS version

### Why Shared Project for SSMS Versions?

**Context:** SSMS 21 and 22 require different assembly references but share 95% of code.

**Decision:** Use `.shproj` shared project with version-specific wrapper projects.

**Consequences:**
- Positive: No code duplication
- Positive: Easy to add SSMS 23+ support
- Negative: Conditional compilation for version differences

### Why LSP for Azure Data Studio?

**Context:** ADS is built on VS Code architecture and expects LSP-based extensions.

**Decision:** Implement full Language Server Protocol server with custom notifications.

**Alternatives considered:**
- Direct extension code: Rejected - ADS doesn't support direct Dataverse connectivity
- Embedded .NET: Rejected - Complexity of bridging TypeScript to .NET

**Consequences:**
- Positive: Clean separation between UI (TypeScript) and logic (.NET)
- Positive: Reuses LanguageServer for potential VS Code support
- Negative: Two-process architecture adds complexity

### Why Bootstrap Pattern for XTB?

**Context:** XrmToolBox scans plugin folders for MEF exports, but we need many dependencies.

**Decision:** Small bootstrap assembly (Plugin.cs) that loads main assembly from subfolder.

**Consequences:**
- Positive: Clean plugin folder structure
- Positive: Can update dependencies without conflicts
- Negative: Extra indirection in loading

---

## Extension Points

### Adding a New Host Integration

1. **Reference Engine**: Add reference to `MarkMpn.Sql4Cds.Engine`
2. **Create Connection**: Instantiate `Sql4CdsConnection` with `DataSource` dictionary
3. **Implement Options**: Create class implementing `IQueryExecutionOptions`
4. **Handle Events**: Wire up `InfoMessage`, `PreInsert/Update/Delete`, `Progress`
5. **Display Results**: Use `ExecutionPlanView` for plan visualization

**Example skeleton:**
```csharp
public class CustomHostIntegration
{
    public void ExecuteQuery(string sql, IOrganizationService org)
    {
        var dataSource = new DataSource
        {
            Name = "custom",
            Connection = org,
            Metadata = new AttributeMetadataCache(org),
            TableSizeCache = new TableSizeCache(org, new AttributeMetadataCache(org))
        };

        using var connection = new Sql4CdsConnection(
            new Dictionary<string, DataSource> { ["custom"] = dataSource });

        connection.InfoMessage += (s, e) => DisplayMessage(e.Message);
        connection.PreUpdate += (s, e) => ConfirmDml(e);

        using var cmd = connection.CreateCommand();
        cmd.CommandText = sql;

        using var reader = cmd.ExecuteReader();
        DisplayResults(reader);
    }
}
```

### Adding a New Document Type (XTB)

1. **Create Control**: Inherit from `DocumentWindowBase`
2. **Implement Interface**: Implement `IDocumentWindow` or `ISaveableDocumentWindow`
3. **Register Factory**: Add creation method to `PluginControl`
4. **Wire Session**: Implement `GetSessionDetails`/`RestoreSessionDetails`

---

## Configuration

### XTB Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| SelectLimit | int | 0 | Max rows to retrieve (0=unlimited) |
| InsertWarnThreshold | int | 1 | Warn when inserting more rows |
| UpdateWarnThreshold | int | 1 | Warn when updating more rows |
| DeleteWarnThreshold | int | 1 | Warn when deleting more rows |
| BlockUpdateWithoutWhere | bool | true | Block UPDATE without WHERE |
| BlockDeleteWithoutWhere | bool | true | Block DELETE without WHERE |
| BatchSize | int | 100 | Records per DML batch |
| MaxDegreeOfParallelism | int | 10 | Parallel DML threads |
| UseTDSEndpoint | bool | true | Use TDS endpoint when available |
| RememberSession | bool | true | Persist document layout |

### SSMS Options

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| BlockDeleteWithoutWhere | bool | true | Safety check |
| BlockUpdateWithoutWhere | bool | true | Safety check |
| BatchSize | int | 100 | API request batching |
| MaxDegreeOfParallelism | int | 10 | Thread count |
| BypassCustomPlugins | bool | false | Skip plugin execution |
| UseNativeFetchXmlToSql | bool | false | Use Dataverse conversion |
| ShowFetchXmlInPlans | bool | true | Display mode |

### ADS Configuration

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| blockDeleteWithoutWhere | bool | true | Prevent dangerous deletes |
| blockUpdateWithoutWhere | bool | true | Prevent dangerous updates |
| useBulkDelete | bool | false | Use bulk delete API |
| batchSize | int | 100 | Records per batch |
| maxDegreeOfParallelism | int | 10 | Parallel requests |
| selectLimit | int | 0 | Max results (0=unlimited) |
| bypassCustomPlugins | bool | false | Skip plugins for DML |

---

## Testing

### Acceptance Criteria

- [ ] XTB: Execute SELECT query and display results in grid
- [ ] XTB: Execute UPDATE with confirmation dialog showing affected rows
- [ ] XTB: Autocomplete shows table and column names
- [ ] XTB: Session persists documents across plugin restarts
- [ ] SSMS: Query interception works for Dataverse connections
- [ ] SSMS: Results display in native SSMS grid
- [ ] SSMS: SQL to FetchXML conversion produces valid FetchXML
- [ ] ADS: Connection provider appears in connection dialog
- [ ] ADS: Progress notifications display during query execution
- [ ] ADS: Confirmation dialogs appear for DML operations

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| Empty query | "" | No action, no error |
| Syntax error | "SELEC * FROM account" | Parser error with line number |
| Large result (100k rows) | SELECT with no WHERE | Progress updates, streaming results |
| Cancelled query | Click cancel during execution | Clean termination, partial results discarded |
| Multiple connections | Two Dataverse instances | Queries route to correct instance |

### Test Examples

```csharp
[TestMethod]
public void XTB_ExecuteSelectQuery_DisplaysResults()
{
    // Arrange
    var context = new XrmFakedContext();
    context.Initialize(new Entity("account") { ["name"] = "Test" });
    var queryControl = CreateSqlQueryControl(context);

    // Act
    queryControl.SetQuery("SELECT name FROM account");
    queryControl.Execute(true, false);
    WaitForBackgroundWorker();

    // Assert
    Assert.AreEqual(1, queryControl.ResultsGrid.Rows.Count);
    Assert.AreEqual("Test", queryControl.ResultsGrid.Rows[0]["name"]);
}

[TestMethod]
public void SSMS_DataverseConnection_InterceptsQuery()
{
    // Arrange
    var connectionString = "Server=org.crm.dynamics.com,5558";
    var dmlExecute = new DmlExecute();

    // Act
    var shouldIntercept = dmlExecute.ShouldIntercept(connectionString, "UPDATE account SET name = 'x'");

    // Assert
    Assert.IsTrue(shouldIntercept);
}
```

---

## Related Specs

- [architecture.md](./architecture.md) - Overall system architecture
- [ado-net-provider.md](./ado-net-provider.md) - Sql4CdsConnection/Command/Reader
- [execution-plan-nodes.md](./execution-plan-nodes.md) - Plan node types
- [language-server.md](./language-server.md) - LSP server for ADS
- [export-system.md](./export-system.md) - Export functionality used by hosts

---

## Roadmap

- VS Code extension (reuse LanguageServer with LSP client)
- Power Platform CLI integration
- Improved SSMS 23+ support as versions release
- Custom Object Explorer for XTB with drag-drop query building
