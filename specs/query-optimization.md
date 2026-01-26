# Query Optimization

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-26
**Code:** [MarkMpn.Sql4Cds.Engine/](../MarkMpn.Sql4Cds.Engine/)

---

## Overview

The query optimizer transforms execution plans to minimize data transfer between Dataverse and the client. It uses a rule-based query folding strategy that pushes operations down into FetchXML wherever possible, reducing intermediate data volumes and enabling server-side execution of filters, sorts, and aggregations.

### Goals

- **Minimize data transfer**: Push filters, sorts, and pagination into FetchXML to reduce network overhead
- **Eliminate redundant operations**: Remove unnecessary nodes when their effects can be achieved by sources
- **Leverage server-side processing**: Convert SQL operations to native Dataverse FetchXML equivalents

### Non-Goals

- Cost-based optimization with plan enumeration
- Join reordering optimization (join order follows query structure)
- Materialized view selection or index recommendations

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ExecutionPlanBuilder                              │
│  (builds initial plan, passes to optimizer)                         │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ExecutionPlanOptimizer                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │ FoldQuery() │─▶│AddRequired  │─▶│SortFetchXml │─▶│MarkComplete│ │
│  │   Phase     │  │  Columns    │  │  Elements   │  │  Callback  │ │
│  └──────┬──────┘  └─────────────┘  └─────────────┘  └────────────┘ │
└─────────┼───────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Node-Level Query Folding (Recursive)                    │
│                                                                      │
│  FilterNode ──▶ FoldFiltersToDataSources ──▶ FetchXmlScan           │
│  SortNode   ──▶ FoldSorts                 ──▶ FetchXml <order>      │
│  TopNode    ──▶ FoldTop                   ──▶ FetchXml.top          │
│  DistinctNode ──▶ FoldDistinct            ──▶ FetchXml.distinct     │
│  JoinNode   ──▶ FoldFetchXmlJoin          ──▶ FetchXml <link-entity>│
└─────────────────────────────────────────────────────────────────────┘
```

The optimizer orchestrates a multi-phase transformation where each execution plan node attempts to fold itself into its data source.

### Components

| Component | Responsibility |
|-----------|----------------|
| ExecutionPlanOptimizer | Orchestrates optimization phases and coordinates node callbacks |
| FoldQuery methods | Node-specific logic to push operations into sources |
| AddRequiredColumns | Ensures FetchXML includes all columns needed upstream |
| FetchXmlElementComparer | Canonicalizes FetchXML structure for readability |
| TranslateFetchXMLCriteria | Converts SQL predicates to FetchXML conditions |

### Dependencies

- Depends on: [execution-plan-nodes.md](./execution-plan-nodes.md) for node interfaces
- Depends on: [fetchxml-translation.md](./fetchxml-translation.md) for FetchXML primitives
- Uses patterns from: [architecture.md](./architecture.md)

---

## Specification

### Core Requirements

1. **Recursive folding**: Each node must fold its sources before attempting self-optimization
2. **Parent linkage**: After folding, nodes must update `Parent` references for tree integrity
3. **Hint observance**: All folding decisions must respect optimizer hints
4. **Non-destructive**: Original plan structure preserved until folding succeeds

### Optimization Phases

The optimizer ([`ExecutionPlanOptimizer.cs:45-75`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanOptimizer.cs#L45-L75)) executes these phases in sequence:

**Phase 1: Query Folding**

```csharp
var nodes = bypassOptimization
    ? new[] { node }
    : node.FoldQuery(context, hints);
```

Each node's `FoldQuery()` recursively transforms the plan tree, pushing operations toward data sources.

**Phase 2: Column Requirements**

```csharp
n.AddRequiredColumns(context, new List<string>());
```

Propagates column needs downward to ensure FetchXML retrieves all required attributes.

**Phase 3: Element Sorting**

```csharp
SortFetchXmlElements(n);
```

Arranges FetchXML elements in canonical order: allattributes → attributes → link-entities → filter → order.

**Phase 4: Completion Callbacks**

```csharp
MarkComplete(n, context);
```

Notifies nodes that folding is finished for final cleanup via `FinishedFolding()`.

### Primary Flows

**Filter Folding:**

1. **Normalize**: Convert `NOT (x IS NULL)` to `x IS NOT NULL` for translation compatibility
2. **Outer-to-inner**: Convert outer joins to inner when filter implies non-null
3. **Fold source**: Recursively fold the source node
4. **Extract filters**: Call `TranslateFetchXMLCriteria()` for conditions compatible with FetchXML
5. **Push down**: Move extractable filters to FetchXmlScan's filter element
6. **Propagate**: Re-fold source if filters were successfully pushed down

**Join Folding:**

1. **Validate**: Check 24 preconditions for FetchXML link-entity compatibility
2. **Create link-entity**: Build `FetchLinkEntityType` with join attributes and type
3. **Merge**: Insert link-entity into appropriate parent entity
4. **Cleanup**: Remove now-implicit NOT NULL conditions

**Sort Folding:**

1. **Skip redundant**: Remove sorts that override previous sorts
2. **Navigate layers**: Skip through FilterNode, ComputeScalarNode to reach data source
3. **Apply to FetchXML**: Add `<order>` elements to entity or link-entity

### Constraints

- FetchXML TOP limited to 5000 records maximum
- Link-entity count limited to 10 (or 15 for version 9.2.22043+)
- Elastic tables cannot participate in FetchXML joins
- Virtual entity providers may not reliably implement all folded operations

---

## Core Types

### ExecutionPlanOptimizer

Orchestrates the optimization process and provides context to folding methods.

```csharp
class ExecutionPlanOptimizer
{
    public IRootExecutionPlanNodeInternal[] Optimize(
        IRootExecutionPlanNodeInternal node,
        IList<OptimizerHint> hints);
}
```

The implementation ([`ExecutionPlanOptimizer.cs:18-107`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanOptimizer.cs#L18-L107)) instantiates a `NodeCompilationContext` and delegates to each node's `FoldQuery()` method.

### FoldQuery Contract

Every data node implements this method ([`IDataExecutionPlanNode.cs:44-49`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/IDataExecutionPlanNode.cs#L44-L49)):

```csharp
IDataExecutionPlanNodeInternal FoldQuery(
    NodeCompilationContext context,
    IList<OptimizerHint> hints);
```

Returns either `this` (no optimization possible), `Source` (node eliminated), or a new node (transformation applied).

### RowCountEstimate

Provides cardinality information for optimization decisions.

```csharp
class RowCountEstimate
{
    public int Value { get; }
}

class RowCountEstimateDefiniteRange : RowCountEstimate
{
    public int Minimum { get; }
    public int Maximum { get; }
    public static RowCountEstimateDefiniteRange ExactlyOne { get; }
    public static RowCountEstimateDefiniteRange ZeroOrOne { get; }
}
```

Used by nodes like DistinctNode to determine if DISTINCT can be eliminated (e.g., when primary key is in column set).

---

## Optimization Catalog

### Filter Pushdown

Moves WHERE conditions into FetchXML `<filter>` elements.

| SQL Predicate | FetchXML Operator | Notes |
|---------------|-------------------|-------|
| `=` | `eq` | Standard equality |
| `<>` | `ne` | Not equal |
| `<`, `>`, `<=`, `>=` | `lt`, `gt`, `le`, `ge` | Comparisons |
| `IS NULL` | `null` | Null check |
| `IS NOT NULL` | `not-null` | Not null check |
| `LIKE` | `like` | Pattern matching |
| `IN (...)` | `in` with values | Multiple values |
| `CONTAINS` | `like` | Full-text search approximation |

### Sort Folding

Converts ORDER BY into FetchXML `<order>` elements.

```xml
<order attribute="name" descending="false" />
```

Sorting can only be folded when:
- Sort column exists on the target entity or link-entity
- No TOP clause conflicts with sort requirements
- Virtual entity provider supports ordering

### Distinct Folding

Converts DISTINCT into `distinct="true"` attribute on FetchXML.

```xml
<fetch distinct="true">
```

**Elimination rules:**
- DISTINCT removed if primary key is in column list (guarantees uniqueness)
- DISTINCT removed if source returns at most 1 row
- DISTINCT converted to StreamAggregateNode if data already sorted by distinct columns

### Top/Pagination Folding

Converts TOP and OFFSET/FETCH into FetchXML attributes.

```xml
<fetch top="100" />
<fetch page="2" count="50" />
```

**Constraints:**
- TOP limited to 5000 (FetchXML maximum)
- OFFSET must align with page boundaries (`offset % count == 0`)
- Cannot fold with virtual entity providers due to unreliable support

### Join Folding

Converts JOINs into FetchXML `<link-entity>` elements.

```xml
<link-entity name="contact" from="contactid" to="primarycontactid" link-type="inner">
```

**Validation requirements (24 checks):**

| Check | Condition |
|-------|-----------|
| Join key | Must have single-column equality join |
| Data source | Both sides from same Dataverse instance |
| Archived data | Cannot join with archived/retained data |
| DISTINCT | Cannot have mismatched DISTINCT settings |
| TOP/Paging | Neither side can have TOP or paging applied |
| Aliases | No alias conflicts between entities |
| Collations | No explicit collations on join columns |
| Virtual entity | Both from same virtual entity provider |
| Elastic table | Neither entity can be elastic (partitioned) |
| Link count | Total link-entities ≤ 10 (or 15 in newer versions) |

### Index Spool Creation

Creates IndexSpoolNode for efficient nested loop joins.

```csharp
if (rowCount.Value >= 100)
{
    indexSpool = new IndexSpoolNode
    {
        Source = this,
        KeyColumn = alias + "." + attribute,
        SeekValue = variableValue
    };
}
```

Triggered when left side of nested loop returns ≥100 rows and right side has variable condition on join key.

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| InvalidHint | Unknown optimizer hint name | Throws NotSupportedQueryFragmentException |
| ConflictingHints | Multiple page size hints | Throws NotSupportedQueryFragmentException |
| FoldingFailure | Cannot translate to FetchXML | Falls back to client-side execution |

### Recovery Strategies

- **Translation failure**: Keep original node, execute filter/sort on client
- **Join folding failure**: Use HashJoin or MergeJoin with explicit sorts
- **Virtual entity unreliability**: Add redundant client-side validation

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Empty filter after folding | Return source node directly |
| All columns required | FetchXML uses `<all-attributes/>` |
| Parameterized conditions | Create IndexSpoolNode for caching |

---

## Design Decisions

### Why Rule-Based Folding?

**Context:** SQL4CDS needs to optimize queries against a non-relational data source (Dataverse FetchXML) that has specific capabilities and limitations.

**Decision:** Use recursive rule-based query folding where each node type encapsulates its own optimization logic.

**Alternatives considered:**
- Cost-based optimizer with plan enumeration: Rejected - FetchXML's fixed execution model makes cost comparison between plans less meaningful
- Global rewrite rules: Rejected - Node-local rules are easier to maintain and test

**Consequences:**
- Positive: Each node type owns its optimization logic, making extensions straightforward
- Positive: No complex cost model to maintain or calibrate
- Negative: Cannot optimize across node boundaries in some cases
- Negative: Join order follows query structure rather than optimal order

### Why 100-Row Spool Threshold?

**Context:** Nested loop joins with parameterized conditions can cause repeated FetchXML requests for the same values.

**Decision:** Create IndexSpoolNode when left side returns ≥100 rows.

**Test results:**
| Scenario | Result |
|----------|--------|
| 50 left rows, repeated lookups | 50 FetchXML requests |
| 150 left rows, with spool | Reduced requests via caching |

**Consequences:**
- Positive: Significant reduction in API calls for large joins
- Negative: Memory overhead for spool storage
- Negative: Threshold is heuristic, may not be optimal for all cases

### Why Stable Sort for FetchXML Elements?

**Context:** FetchXML element order affects readability but not semantics.

**Decision:** Use stable sort ([`FetchXmlElementComparer.cs:12-56`](../MarkMpn.Sql4Cds.Engine/FetchXmlElementComparer.cs#L12-L56)) to canonicalize element order while preserving relative order of same-type elements.

**Order applied:** allattributes → attributes → link-entities → filter → order

**Consequences:**
- Positive: Consistent, readable FetchXML output
- Positive: Matches documentation examples
- Negative: Minor CPU overhead for sorting

---

## Configuration

### Optimizer Hints

Hints are specified using `OPTION (USE HINT ('hint_name'))` syntax.

| Hint | Effect |
|------|--------|
| `DEBUG_BYPASS_OPTIMIZATION` | Disables all query folding |
| `FORCE_SQL4CDS` | Forces SQL4CDS engine instead of TDS Endpoint |
| `BYPASS_CUSTOM_PLUGIN_EXECUTION` | Skips plugin execution on DML |
| `RETRIEVE_TOTAL_RECORD_COUNT` | Uses cached count for COUNT(*) |
| `FETCHXML_PAGE_SIZE_n` | Sets FetchXML page size (1-5000) |
| `BATCH_SIZE_n` | Sets DML batch size |
| `CONTINUE_ON_ERROR` | Continues DML batch on individual failures |
| `MINIMAL_UPDATES` | Only updates changed attributes |
| `IGNORE_DUP_KEY` | Ignores duplicate keys on insert |
| `NO_DIRECT_DML` | Disables direct DML optimization |

The validation logic ([`OptimizerHintValidatingVisitor.cs:37-92`](../MarkMpn.Sql4Cds.Engine/Visitors/OptimizerHintValidatingVisitor.cs#L37-L92)) enforces valid hint names and numeric suffixes.

### IQueryExecutionOptions

| Setting | Type | Default | Effect on Optimization |
|---------|------|---------|------------------------|
| UseTDSEndpoint | bool | true | May bypass SQL4CDS optimization entirely |
| MaxDegreeOfParallelism | int | 10 | Affects parallel DML execution |
| BatchSize | int | 100 | Default batch size for DML |
| BypassCustomPlugins | bool | false | Default plugin bypass setting |

---

## Testing

### Acceptance Criteria

- [ ] Filter on indexed column folds into FetchXML
- [ ] Sort on entity attribute folds into FetchXML order
- [ ] DISTINCT folds to FetchXML distinct attribute
- [ ] TOP ≤5000 folds to FetchXML top attribute
- [ ] Inner join folds to link-entity when preconditions met
- [ ] DEBUG_BYPASS_OPTIMIZATION returns unoptimized plan
- [ ] Required columns propagate to FetchXML attributes

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| DISTINCT on primary key | `SELECT DISTINCT id FROM account` | DISTINCT eliminated |
| Sort already applied | `ORDER BY` after aggregate sort | Redundant sort removed |
| Outer join with non-null filter | `LEFT JOIN WHERE right.id IS NOT NULL` | Converted to INNER JOIN |
| TOP > 5000 | `SELECT TOP 6000` | TopNode remains, not folded |

### Test Examples

```csharp
[Fact]
public void FilterFoldsToFetchXml()
{
    var sql = "SELECT name FROM account WHERE accountid = '00000000-0000-0000-0000-000000000001'";
    var plan = BuildPlan(sql);

    var fetch = plan.GetDescendants<FetchXmlScan>().Single();
    Assert.Contains(fetch.Entity.Items.OfType<filter>(),
        f => f.Items.OfType<condition>()
            .Any(c => c.attribute == "accountid" && c.@operator == @operator.eq));
}

[Fact]
public void BypassHintSkipsOptimization()
{
    var sql = "SELECT name FROM account OPTION (USE HINT ('DEBUG_BYPASS_OPTIMIZATION'))";
    var plan = BuildPlan(sql);

    // Plan should have FilterNode on top of FetchXmlScan
    Assert.IsType<SelectNode>(plan);
}
```

---

## Related Specs

- [execution-plan-nodes.md](./execution-plan-nodes.md) - Node interfaces that define the FoldQuery contract
- [fetchxml-translation.md](./fetchxml-translation.md) - FetchXML structure and capabilities
- [query-compilation.md](./query-compilation.md) - How initial unoptimized plans are built
- [expression-evaluation.md](./expression-evaluation.md) - Expression compilation for filter conditions

---

## Roadmap

- Implement cost-based join ordering for complex multi-table queries
- Add statistics collection for better cardinality estimation
- Support parallel query execution for read operations
- Add query plan caching for repeated queries
