# Execution Plan Nodes

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-25
**Code:** [MarkMpn.Sql4Cds.Engine/ExecutionPlan/](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/)

---

## Overview

The execution plan node system is the core query execution engine for SQL4CDS. Based on SQL Server's query execution model, it uses a tree of nodes that implement an iterator-based pull model where each node yields data to its parent. The system supports 80+ node types organized into a clean interface hierarchy.

### Goals

- **SQL Server Compatibility**: Mirror SQL Server's Init/GetNext/Close execution model
- **Streaming Execution**: Process data as an iterator stream to minimize memory usage
- **Query Optimization**: Support query folding to push operations into FetchXML
- **Extensibility**: Enable new node types via well-defined interfaces
- **Statistics Tracking**: Capture execution timing and row counts for plan analysis

### Non-Goals

- Parallel execution (all nodes execute single-threaded)
- Adaptive query processing (plan is fixed at compile time)
- Index selection (deferred to Dataverse FetchXML engine)

---

## Architecture

```
                    ┌──────────────────────────┐
                    │   SelectNode / DML Node   │  ← Root execution
                    │   (IRootExecutionPlanNode)│
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │        FilterNode         │  ← Single source
                    │  (ISingleSourceExecutionPlanNode)
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │      HashJoinNode         │  ← Multiple sources
                    │     (BaseJoinNode)        │
                    └──────┬─────────┬─────────┘
                           │         │
              ┌────────────▼───┐ ┌───▼────────────┐
              │  FetchXmlScan  │ │  FetchXmlScan  │  ← Data sources
              │  (IFetchXml    │ │                │
              │  ExecutionPlan │ │                │
              │  Node)         │ │                │
              └────────────────┘ └────────────────┘
```

Data flows **upward** through the tree: leaf nodes (FetchXmlScan) produce Entity records, intermediate nodes transform/combine them, and root nodes consume the final result.

### Components

| Component | Responsibility |
|-----------|----------------|
| **IExecutionPlanNode** | Base contract for all plan nodes |
| **IDataExecutionPlanNode** | Nodes that produce a data stream |
| **IDmlQueryExecutionPlanNode** | Nodes that modify data (INSERT/UPDATE/DELETE) |
| **BaseNode / BaseDataNode** | Common implementation with statistics tracking |
| **NodeSchema** | Column metadata flowing through the plan |

### Dependencies

- Depends on: [architecture.md](./architecture.md) (system context)
- Uses patterns from: [type-system.md](./type-system.md) (data type handling)
- Referenced by: query-compilation.md (plan construction)

---

## Specification

### Core Requirements

1. All data nodes must implement the iterator pattern via `Execute()` returning `IEnumerable<Entity>`
2. Nodes must track execution statistics (count, duration, rows) without explicit instrumentation
3. Schema must flow from source nodes to parent nodes during compilation
4. Query folding must be attempted to push operations into FetchXML where possible
5. All nodes must support cloning for plan serialization

### Execution Model

The execution model ([`IExecutionPlanNode.cs:11-20`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/IExecutionPlanNode.cs#L11-L20)) maps SQL Server's query processor:

| SQL Server | SQL4CDS | Description |
|------------|---------|-------------|
| Init() | Execute() | Initialize node and begin iteration |
| GetNext() | MoveNext() | Retrieve next row from iterator |
| Close() | Dispose() | Release resources |

**Execution Flow:**

1. **Compilation**: `GetSchema()` called to determine output columns
2. **Optimization**: `FoldQuery()` attempts to push operations into source
3. **Initialization**: `Execute()` called on root node
4. **Iteration**: Parent calls `MoveNext()` on child's enumerator
5. **Cleanup**: Enumerator disposed when complete or cancelled

### Node Lifecycle

```csharp
// Compilation phase
var schema = node.GetSchema(compilationContext);
node.AddRequiredColumns(context, requiredColumns);
var optimized = node.FoldQuery(context, hints);
optimized.FinishedFolding(context);

// Execution phase
foreach (var entity in node.Execute(executionContext))
{
    // Process entity
}
```

### Constraints

- Nodes must not modify their source data (immutable data flow)
- Schema objects are read-only after construction
- Execution count increments before yielding first row
- Duration includes time waiting for source nodes

### Validation Rules

| Aspect | Rule | Error Behavior |
|--------|------|----------------|
| Column reference | Must exist in source schema | Throws during compilation |
| Type compatibility | Join keys must have compatible types | Throws during compilation |
| Required columns | All referenced columns must be requested | Silent (column omitted) |

---

## Core Types

### IExecutionPlanNode

The root interface ([`IExecutionPlanNode.cs:21-43`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/IExecutionPlanNode.cs#L21-L43)) for all execution plan nodes:

```csharp
public interface IExecutionPlanNode
{
    IExecutionPlanNode Parent { get; }
    IEnumerable<IExecutionPlanNode> GetSources();
    int ExecutionCount { get; }
    TimeSpan Duration { get; }
}
```

Extended by internal interface `IExecutionPlanNodeInternal` ([`IExecutionPlanNode.cs:56-75`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/IExecutionPlanNode.cs#L56-L75)) which adds:
- Settable `Parent` property
- `AddRequiredColumns()` for column pruning
- `FinishedFolding()` notification
- `ICloneable` for plan serialization

### IDataExecutionPlanNode

Nodes that produce a data stream ([`IDataExecutionPlanNode.cs:14-25`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/IDataExecutionPlanNode.cs#L14-L25)):

```csharp
public interface IDataExecutionPlanNode : IExecutionPlanNode
{
    int EstimatedRowsOut { get; }
    int RowsOut { get; }
}
```

The internal interface adds the core execution methods ([`IDataExecutionPlanNode.cs:27-64`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/IDataExecutionPlanNode.cs#L27-L64)):

```csharp
internal interface IDataExecutionPlanNodeInternal : IDataExecutionPlanNode
{
    RowCountEstimate EstimateRowsOut(NodeCompilationContext context);
    IEnumerable<Entity> Execute(NodeExecutionContext context);
    IDataExecutionPlanNodeInternal FoldQuery(NodeCompilationContext context, IList<OptimizerHint> hints);
    INodeSchema GetSchema(NodeCompilationContext context);
    IEnumerable<string> GetVariables(bool recurse);
}
```

### BaseNode

Abstract base class ([`BaseNode.cs:13-79`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/BaseNode.cs#L13-L79)) providing:

```csharp
abstract class BaseNode : IExecutionPlanNode
{
    public IExecutionPlanNode Parent { get; set; }
    public abstract int ExecutionCount { get; }
    public abstract TimeSpan Duration { get; }
    public abstract IEnumerable<IExecutionPlanNode> GetSources();
    public abstract void AddRequiredColumns(NodeCompilationContext context, IList<string> requiredColumns);
}
```

The `ToString()` method converts class names to display names (e.g., `FilterNode` becomes `"Filter"`).

### BaseDataNode

Extends BaseNode for data-producing nodes ([`BaseDataNode.cs:24-1409`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/BaseDataNode.cs#L24-L1409)):

```csharp
abstract class BaseDataNode : BaseNode, IDataExecutionPlanNodeInternal
{
    private int _executionCount;
    private readonly Timer _timer = new Timer();
    private int _rowsOut;

    public virtual IEnumerable<Entity> Execute(NodeExecutionContext context)
    {
        _executionCount++;
        foreach (var entity in ExecuteInternal(context))
        {
            _rowsOut++;
            yield return entity;
        }
    }

    protected abstract IEnumerable<Entity> ExecuteInternal(NodeExecutionContext context);
    public abstract INodeSchema GetSchema(NodeCompilationContext context);
    public abstract IDataExecutionPlanNodeInternal FoldQuery(NodeCompilationContext context, IList<OptimizerHint> hints);
}
```

Key features:
- Automatic execution counting and timing via `Timer` wrapper
- Row counting happens automatically as entities are yielded
- Exception wrapping into `QueryExecutionException` with node reference
- Cancellation token support

### INodeSchema

Describes output columns ([`NodeSchema.cs:426-462`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/NodeSchema.cs#L426-L462)):

```csharp
public interface INodeSchema
{
    string PrimaryKey { get; }
    IReadOnlyDictionary<string, IColumnDefinition> Schema { get; }
    IReadOnlyDictionary<string, IReadOnlyList<string>> Aliases { get; }
    IReadOnlyList<string> SortOrder { get; }
    bool ContainsColumn(string column, out string normalized);
    bool IsSortedBy(ISet<string> requiredSorts);
}
```

The concrete `NodeSchema` class ([`NodeSchema.cs:15-123`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/NodeSchema.cs#L15-L123)) provides immutable schema storage with copy semantics.

### IColumnDefinition

Column metadata ([`NodeSchema.cs:467-518`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/NodeSchema.cs#L467-L518)):

```csharp
public interface IColumnDefinition
{
    string SourceServer { get; }
    string SourceTable { get; }
    string SourceColumn { get; }
    DataTypeReference Type { get; }
    bool IsNullable { get; }
    bool IsCalculated { get; }
    bool IsVisible { get; }
    bool IsWildcardable { get; }
}
```

Extension methods provide fluent modification: `col.NotNull().Calculated().Invisible()`.

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| `QueryExecutionException` | Runtime error during execution | Includes node reference for debugging |
| `NotSupportedQueryFragmentException` | Unsupported SQL syntax | Suggest alternative syntax |
| `Sql4CdsError` | Compilation or semantic error | Contains error code and message |

### Recovery Strategies

- **Cancellation**: Check `CancellationToken` before yielding each row
- **Timeout**: Handled at connection level, propagates via cancellation
- **API errors**: Wrapped in `QueryExecutionException` with original exception

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Empty source | Return empty enumerable, not null |
| NULL values | Preserved through data flow |
| Schema mismatch | Throws during compilation, not execution |
| Out of memory | Let exception propagate (no special handling) |

---

## Design Decisions

### Why Iterator-Based Pull Model?

**Context:** Query results can be very large (millions of rows). Need efficient memory usage.

**Decision:** Use C# iterator pattern (`yield return`) where parent nodes pull data from children.

**Alternatives considered:**
- Push model: Parent pushes data to children. Rejected - harder to implement cancellation
- Materialized buffers: Collect all results first. Rejected - excessive memory for large results

**Consequences:**
- Positive: Constant memory usage regardless of result size
- Positive: Natural cancellation points between rows
- Negative: Can't easily re-iterate without re-execution

### Why Automatic Statistics Tracking?

**Context:** Users need execution plan analysis without modifying node implementations.

**Decision:** BaseDataNode wraps execution and counts rows/time automatically.

**Test results:**
| Approach | Code Complexity | Accuracy |
|----------|-----------------|----------|
| Manual tracking | High (each node) | Inconsistent |
| Automatic wrapping | Low (base class) | Consistent |

**Consequences:**
- Positive: Every node gets statistics for free
- Positive: Impossible to forget tracking
- Negative: Small overhead from wrapping (negligible vs API calls)

### Why Immutable Schema Objects?

**Context:** Schema flows through many nodes. Need to prevent accidental modification.

**Decision:** Use `IReadOnlyDictionary` and create new schema objects for modifications.

**Alternatives considered:**
- Mutable schemas: Rejected - race conditions and hard-to-track bugs
- Deep copy everywhere: Rejected - performance overhead

**Consequences:**
- Positive: Thread-safe schema access
- Positive: Clear ownership semantics
- Negative: More object allocations during compilation (acceptable)

### Why Query Folding via FoldQuery()?

**Context:** FetchXML is more efficient than client-side filtering/sorting.

**Decision:** Each node attempts to push its operation into its source via `FoldQuery()`.

**Example:**
```
FilterNode(FetchXmlScan) → FetchXmlScan (filter moved into FetchXML)
```

**Consequences:**
- Positive: Significant performance improvement for foldable operations
- Positive: Reduces data transfer from Dataverse
- Negative: Complex folding logic in each node type

---

## Extension Points

### Adding a New Data Node

1. **Create class**: Extend `BaseDataNode` in `ExecutionPlan/` folder
2. **Implement required methods**:

```csharp
class MyNode : BaseDataNode, ISingleSourceExecutionPlanNode
{
    [Browsable(false)]
    public IDataExecutionPlanNodeInternal Source { get; set; }

    protected override IEnumerable<Entity> ExecuteInternal(NodeExecutionContext context)
    {
        foreach (var entity in Source.Execute(context))
        {
            // Transform entity
            yield return entity;
        }
    }

    public override IEnumerable<IExecutionPlanNode> GetSources()
    {
        yield return Source;
    }

    public override INodeSchema GetSchema(NodeCompilationContext context)
    {
        return Source.GetSchema(context); // Or modified schema
    }

    public override IDataExecutionPlanNodeInternal FoldQuery(NodeCompilationContext context, IList<OptimizerHint> hints)
    {
        Source = Source.FoldQuery(context, hints);
        // Attempt to fold into source
        return this;
    }

    protected override RowCountEstimate EstimateRowsOutInternal(NodeCompilationContext context)
    {
        return Source.EstimateRowsOut(context);
    }

    public override void AddRequiredColumns(NodeCompilationContext context, IList<string> requiredColumns)
    {
        Source.AddRequiredColumns(context, requiredColumns);
    }

    public override object Clone()
    {
        return new MyNode { Source = (IDataExecutionPlanNodeInternal)Source.Clone() };
    }
}
```

3. **Register**: Add construction logic in `ExecutionPlanBuilder`

### Node Categories

| Category | Base Class | Interface | Example |
|----------|------------|-----------|---------|
| Data source | BaseDataNode | IFetchXmlExecutionPlanNode | FetchXmlScan |
| Single input | BaseDataNode | ISingleSourceExecutionPlanNode | FilterNode, SortNode |
| Multiple inputs | BaseJoinNode | - | HashJoinNode, MergeJoinNode |
| DML | BaseDmlNode | IDmlQueryExecutionPlanNode | InsertNode, UpdateNode |
| Aggregate | BaseAggregateNode | - | HashMatchAggregateNode |
| Control flow | BaseNode | IGoToNode | ConditionalNode, GoToNode |

---

## Testing

### Acceptance Criteria

- [ ] All node types implement IExecutionPlanNode correctly
- [ ] ExecutionCount increments exactly once per Execute() call
- [ ] Duration includes source node time
- [ ] RowsOut matches actual yielded entities
- [ ] GetSchema() returns consistent schema across calls
- [ ] FoldQuery() returns valid node (may be self or replacement)
- [ ] Clone() produces deep copy with no shared mutable state

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| Empty source | FetchXmlScan returning 0 rows | FilterNode yields 0 rows, RowsOut = 0 |
| Cancellation mid-stream | Cancel after 5 rows | Enumeration stops, partial RowsOut |
| NULL column values | Entity with null attribute | NULL preserved in output |
| Schema with aliases | Ambiguous column name | ContainsColumn returns false |

### Test Examples

```csharp
[Fact]
public void FilterNode_CountsRowsCorrectly()
{
    // Arrange
    var source = new ConstantScanNode { Values = { entity1, entity2, entity3 } };
    var filter = new FilterNode
    {
        Source = source,
        Filter = new BooleanComparisonExpression { /* id > 1 */ }
    };

    // Act
    var results = filter.Execute(context).ToList();

    // Assert
    Assert.Equal(2, filter.RowsOut);
    Assert.Equal(1, filter.ExecutionCount);
}

[Fact]
public void BaseDataNode_TracksExecutionTime()
{
    // Arrange
    var node = new SlowNode { DelayMs = 100 };

    // Act
    node.Execute(context).ToList();

    // Assert
    Assert.True(node.Duration >= TimeSpan.FromMilliseconds(100));
}
```

---

## Related Specs

- [architecture.md](./architecture.md) - System overview and layer diagram
- [type-system.md](./type-system.md) - Data type handling and conversions
- query-compilation.md - How nodes are constructed from SQL (planned)
- query-optimization.md - Query folding and optimization (planned)

---

## Appendix: Node Type Catalog

### Data Source Nodes

| Node | Description |
|------|-------------|
| FetchXmlScan | Retrieves data via FetchXML from Dataverse |
| TableScanNode | Scans temporary table data |
| ConstantScanNode | Returns constant dataset |
| MetadataQueryNode | Queries entity/attribute metadata |
| GlobalOptionSetQueryNode | Queries global option sets |
| SystemFunctionNode | Executes system functions |

### Transform Nodes

| Node | Description |
|------|-------------|
| FilterNode | Applies WHERE clause filter |
| ComputeScalarNode | Calculates scalar expressions |
| SortNode | Sorts data stream |
| TopNode | Limits to TOP N rows |
| OffsetFetchNode | Pagination with OFFSET/FETCH |
| DistinctNode | Removes duplicates |
| AliasNode | Renames columns |
| AssertNode | Validates row conditions |

### Join Nodes

| Node | Description |
|------|-------------|
| NestedLoopNode | Nested loop join |
| HashJoinNode | Hash-based equijoin |
| MergeJoinNode | Merge join on sorted data |

### Aggregate Nodes

| Node | Description |
|------|-------------|
| HashMatchAggregateNode | Hash-based grouping |
| StreamAggregateNode | Pre-sorted grouping |
| PartitionedAggregateNode | Parallel partitioned aggregation |

### DML Nodes

| Node | Description |
|------|-------------|
| InsertNode | INSERT operation |
| UpdateNode | UPDATE operation |
| DeleteNode | DELETE operation |
| BulkDeleteJobNode | Bulk delete via async job |

### Control Flow Nodes

| Node | Description |
|------|-------------|
| ConditionalNode | IF-THEN branching |
| GoToNode | GOTO statement |
| TryCatchNode | Error handling |
| ContinueBreakNode | Loop control |

### Spool Nodes

| Node | Description |
|------|-------------|
| TableSpoolNode | Rewindable data cache |
| IndexSpoolNode | Hash table lookup cache |
| WindowSpoolNode | Window function frame support |
