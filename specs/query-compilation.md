# Query Compilation

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-25
**Code:** [MarkMpn.Sql4Cds.Engine/](../MarkMpn.Sql4Cds.Engine/)

---

## Overview

The query compilation system transforms T-SQL text into executable plan node trees. It uses Microsoft's ScriptDom parser for SQL parsing, applies 28 visitor classes for AST transformation and validation, and constructs an optimized execution plan suitable for Dataverse execution. The compiler handles all SQL statement types including SELECT, DML operations, control flow, cursors, and DDL for temporary tables.

### Goals

- **T-SQL Compatibility**: Parse and compile T-SQL syntax using SQL Server 2019/2022 compatible parser
- **Execution Plan Generation**: Build optimized node trees that can execute against Dataverse
- **TDS Endpoint Detection**: Identify queries executable via TDS Endpoint for maximum performance
- **CTE Support**: Handle both non-recursive and recursive Common Table Expressions

### Non-Goals

- Custom SQL dialect extensions (uses standard T-SQL only)
- Query plan caching (each Build() call compiles fresh)
- Parallel compilation (single-threaded)
- Deferred to: [query-optimization.md](./query-optimization.md) for FoldQuery optimization

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SQL Text Input                               │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    TSql160Parser (ScriptDom)                         │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │    Parse SQL → TSqlScript AST with Batches/Statements           ││
│  │    Handle QuotedIdentifiers option                               ││
│  │    Collect ParseErrors → QueryParseException                     ││
│  └─────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Validation Visitors                               │
│  ┌──────────────────────┐ ┌──────────────────────┐                  │
│  │OptimizerHintValidator│ │TDSEndpointCompatible │                  │
│  └──────────────────────┘ └──────────────────────┘                  │
│           │                        │                                 │
│           │        ┌───────────────┴─────────────────┐              │
│           │        │ Compatible?                      │              │
│           │        │ YES → Return SqlNode (fast path) │              │
│           │        │ NO  → Continue compilation       │              │
│           │        └─────────────────────────────────┘              │
└───────────┼─────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Transformation Visitors                           │
│  ┌──────────────────────────┐ ┌────────────────────────────────────┐│
│  │ReplacePrimaryFunctionsVis│ │ExplicitCollationVisitor            ││
│  │(COALESCE→CASE, IIF→CASE) │ │(Wrap collations in function calls) ││
│  └──────────────────────────┘ └────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                ExecutionPlanBuilder                                  │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ For each Batch/Statement:                                       ││
│  │   1. ConvertStatement() → Dispatch by statement type            ││
│  │   2. ConvertControlOfFlow / ConvertStatementInternal            ││
│  │   3. Build node tree with schema propagation                    ││
│  │   4. Apply ExecutionPlanOptimizer                               ││
│  └─────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Post-Compilation                                  │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 1. Validate GOTO labels (unique, no jumps into TRY/CATCH)      ││
│  │ 2. Validate THROW in CATCH blocks                              ││
│  │ 3. EstimateRowsOut() if EstimatedPlanOnly                      ││
│  └─────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  IRootExecutionPlanNode[]                            │
│                  (Compiled Execution Plan)                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Responsibility |
|-----------|----------------|
| **TSql160Parser** | Microsoft ScriptDom parser for T-SQL 2019/2022 syntax |
| **ExecutionPlanBuilder** | Central orchestrator for compilation (6000+ lines) |
| **Visitors (28)** | AST transformation, validation, and analysis |
| **ExecutionPlanOptimizer** | Query folding and optimization |
| **Context Objects** | NodeCompilationContext, ExpressionCompilationContext |

### Dependencies

- Depends on: [architecture.md](./architecture.md) (system patterns)
- Depends on: [type-system.md](./type-system.md) (type resolution)
- Depends on: [execution-plan-nodes.md](./execution-plan-nodes.md) (node contracts)
- Referenced by: [query-optimization.md](./query-optimization.md) (optimizer)

---

## Specification

### Core Requirements

1. Parse T-SQL using `TSql160Parser` from Microsoft.SqlServer.TransactSql.ScriptDom
2. Support all major SQL statement types (SELECT, INSERT, UPDATE, DELETE, control flow)
3. Build execution plan node trees that implement `IExecutionPlanNode` hierarchy
4. Detect TDS Endpoint compatible queries for fast-path execution
5. Validate semantic correctness during compilation (column references, types)
6. Support recursive and non-recursive CTEs with MAXRECURSION limits

### Compilation Pipeline

**Stage 1: Parsing**

1. **Parse SQL**: Create `TSql160Parser` with `QuotedIdentifiers` option
2. **Capture errors**: Collect `ParseError` list from parser
3. **Throw on error**: First parse error becomes `QueryParseException`

**Stage 2: Validation**

1. **Validate hints**: `OptimizerHintValidatingVisitor` checks hint names
2. **Check TDS compatibility**: `TDSEndpointCompatibilityVisitor` analyzes full query
3. **Fast path**: If TDS-compatible, return `SqlNode` immediately

**Stage 3: Transformation**

1. **Normalize functions**: `ReplacePrimaryFunctionsVisitor` converts COALESCE, IIF, NULLIF to CASE
2. **Explicit collations**: `ExplicitCollationVisitor` wraps collations in function calls

**Stage 4: Statement Conversion**

1. **Iterate batches**: Process each batch in script
2. **Iterate statements**: Convert each statement via `ConvertStatement()`
3. **Dispatch by type**: Route to specific conversion method
4. **Optimize**: Apply `ExecutionPlanOptimizer` to each plan

**Stage 5: Finalization**

1. **Validate GOTOs**: Check labels unique, valid targets
2. **Validate TRY/CATCH**: Ensure no GOTOs into blocks, THROWs in CATCH
3. **Estimate rows**: If `EstimatedPlanOnly`, compute row estimates

### Statement Dispatch

The [`ConvertStatementInternal()`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L303-L400) method routes statements:

| Statement Type | Method | Output Node |
|----------------|--------|-------------|
| SelectStatement | `ConvertSelectStatement()` | SelectNode |
| InsertStatement | `ConvertInsertStatement()` | InsertNode |
| UpdateStatement | `ConvertUpdateStatement()` | UpdateNode |
| DeleteStatement | `ConvertDeleteStatement()` | DeleteNode |
| ExecuteStatement | `ConvertExecuteStatement()` | ExecuteMessageNode/ExecuteAsNode |
| DeclareVariableStatement | `ConvertDeclareVariableStatement()` | DeclareVariablesNode |
| SetVariableStatement | `ConvertSetVariableStatement()` | AssignVariablesNode |
| IfStatement | `ConvertIfStatement()` | ConditionalNode |
| WhileStatement | `ConvertWhileStatement()` | ConditionalNode + GoToNode |
| TryCatchStatement | `ConvertTryCatchStatement()` | TryCatchNode |
| ThrowStatement | `ConvertThrowStatement()` | ThrowNode |
| PrintStatement | `ConvertPrintStatement()` | PrintNode |
| WaitForStatement | `ConvertWaitForStatement()` | WaitForNode |
| CreateTableStatement | `ConvertCreateTableStatement()` | CreateTableNode |
| DropTableStatement | `ConvertDropTableStatement()` | DropTableNode |

### Constraints

- Single-threaded compilation
- Session context cloned to isolate temporary table modifications
- Parameters must be declared before use
- CTE names must be unique within scope
- MAXRECURSION limit: 0-32,767 (default 100)

### Validation Rules

| Rule | Validated By | Error |
|------|--------------|-------|
| Unique CTE names | CteValidatorVisitor | CteDuplicateName |
| Valid GOTO targets | Post-compilation check | UnknownGotoLabel |
| No GOTO into TRY/CATCH | Post-compilation check | GotoIntoTryOrCatch |
| THROW in CATCH only | Post-compilation check | ThrowOutsideCatch |
| Duplicate table names | DuplicateTableNameValidatingVisitor | DuplicateTableName |
| GROUP BY restrictions | GroupByValidatingVisitor | GroupByError |
| Window function placement | WindowFunctionVisitor | WindowFunctionError |

---

## Core Types

### ExecutionPlanBuilder

The central compilation class ([`ExecutionPlanBuilder.cs:20-6059`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L20-L6059)):

```csharp
class ExecutionPlanBuilder
{
    public SessionContext Session { get; }
    public IQueryExecutionOptions Options { get; }
    public bool EstimatedPlanOnly { get; set; }
    public Action<Sql4CdsError> Log { get; set; }

    public IRootExecutionPlanNode[] Build(
        string sql,
        IDictionary<string, DataTypeReference> parameters,
        out bool useTDSEndpointDirectly);
}
```

Constructor clones session to allow temporary table modifications during compilation without affecting the original session ([`ExecutionPlanBuilder.cs:26-37`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L26-L37)).

### Build Method

The main entry point ([`ExecutionPlanBuilder.cs:68-181`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L68-L181)):

```csharp
public IRootExecutionPlanNode[] Build(string sql,
    IDictionary<string, DataTypeReference> parameters,
    out bool useTDSEndpointDirectly)
{
    // 1. Initialize contexts
    _staticContext = new ExpressionCompilationContext(...);
    _nodeContext = new NodeCompilationContext(...);

    // 2. Parse SQL
    var dom = new TSql160Parser(Options.QuotedIdentifiers);
    var fragment = dom.Parse(new StringReader(sql), out var errors);
    if (errors.Count > 0)
        throw new QueryParseException(errors[0]);

    // 3. Check TDS compatibility (fast path)
    // 4. Apply transformation visitors
    // 5. Convert each statement
    // 6. Validate and finalize
}
```

### NodeCompilationContext

Carries state through compilation ([`NodeContext.cs`](../MarkMpn.Sql4Cds.Engine/NodeContext.cs)):

```csharp
class NodeCompilationContext
{
    public SessionContext Session { get; }
    public IQueryExecutionOptions Options { get; }
    public IDictionary<string, DataTypeReference> ParameterTypes { get; }
    public DataSource PrimaryDataSource { get; }
    public Action<Sql4CdsError> Log { get; }

    public string GetExpressionName();  // Unique column name generator
    public NodeCompilationContext CreateChildContext(INodeSchema outerReferences);
}
```

### ExpressionCompilationContext

Context for expression compilation ([`NodeContext.cs`](../MarkMpn.Sql4Cds.Engine/NodeContext.cs)):

```csharp
class ExpressionCompilationContext
{
    public SessionContext Session { get; }
    public IQueryExecutionOptions Options { get; }
    public IDictionary<string, DataTypeReference> ParameterTypes { get; }
    public INodeSchema Schema { get; }
    public INodeSchema NonAggregateSchema { get; }
}
```

### QueryParseException

Thrown for SQL syntax errors ([`QueryParseException.cs`](../MarkMpn.Sql4Cds.Engine/QueryParseException.cs)):

```csharp
public class QueryParseException : NotSupportedException, ISql4CdsErrorException
{
    public QueryParseException(ParseError error);
    public IReadOnlyList<Sql4CdsError> Errors { get; }
}
```

---

## Visitor System

### Visitor Categories

**Validation Visitors** - Check semantic correctness:

| Visitor | Purpose | Key Methods |
|---------|---------|-------------|
| OptimizerHintValidatingVisitor | Validates hint names | `ExplicitVisit(UseHintList)` |
| TDSEndpointCompatibilityVisitor | Checks TDS compatibility | Multiple Visit overrides |
| CteValidatorVisitor | Validates CTE structure | `Visit(CommonTableExpression)` |
| DuplicateTableNameValidatingVisitor | Checks table uniqueness | `Visit(NamedTableReference)` |
| GroupByValidatingVisitor | Validates GROUP BY | `Visit(GroupByClause)` |
| UpdateableViewValidatingVisitor | Validates DML targets | Multiple Visit overrides |
| WindowFunctionVisitor | Validates window functions | `ExplicitVisit(FunctionCall)` |

**Transformation Visitors** - Modify AST:

| Visitor | Purpose | Base Class |
|---------|---------|------------|
| ReplacePrimaryFunctionsVisitor | COALESCE/IIF/NULLIF→CASE | RewriteVisitorBase |
| ExplicitCollationVisitor | Wrap collations | RewriteVisitorBase |
| ReplaceCtesWithSubqueriesVisitor | CTE→subquery for TDS | TSqlFragmentVisitor |
| RemoveRecursiveCTETableReferencesVisitor | Process recursive CTEs | TSqlConcreteFragmentVisitor |
| RefactorNotIsNullVisitor | NOT IS NULL→IS NOT NULL | BooleanRewriteVisitor |
| BooleanRewriteVisitor | Boolean expression rewriting | TSqlFragmentVisitor |

**Collection Visitors** - Extract information:

| Visitor | Collects | Output Property |
|---------|----------|-----------------|
| ColumnCollectingVisitor | Column references | `Columns` |
| AggregateCollectingVisitor | Aggregate functions | `Aggregates`, `SelectAggregates` |
| FunctionCollectingVisitor | All function calls | `Functions` |
| VariableCollectingVisitor | Variables and globals | `Variables`, `GlobalVariables` |
| ExistsSubqueryVisitor | EXISTS predicates | `ExistsSubqueries` |
| InSubqueryVisitor | IN subqueries | `InSubqueries` |
| ScalarSubqueryVisitor | Scalar subqueries | `Subqueries` |
| ParameterlessCollectingVisitor | Parameterless calls | `ParameterlessCalls` |

**Analysis Visitors** - Support compilation decisions:

| Visitor | Purpose | Output |
|---------|---------|--------|
| JoinConditionVisitor | Extract join keys | `LhsKey`, `RhsKey` |
| SimpleFilterVisitor | Flatten filters for FetchXML | `Conditions` |
| GroupValidationVisitor | Check FetchXML compatibility | `Valid` |
| UpdateTargetVisitor | Find UPDATE target | `Target`, `TargetDataSource` |
| NormalizeColNamesVisitor | Resolve column aliases | Modified AST |
| RewriteVisitor | Replace aggregates with columns | Modified AST |

### Visitor Inheritance Hierarchy

```
TSqlFragmentVisitor / TSqlConcreteFragmentVisitor (ScriptDom)
├── RewriteVisitorBase (abstract)
│   ├── ExplicitCollationVisitor
│   ├── NormalizeColNamesVisitor
│   ├── ReplacePrimaryFunctionsVisitor
│   └── RewriteVisitor
├── BooleanRewriteVisitor
│   └── RefactorNotIsNullVisitor
└── 22 other direct implementations
```

### Visitor Pattern Usage

```csharp
// Collect information
var columnVisitor = new ColumnCollectingVisitor();
querySpec.Accept(columnVisitor);
var usedColumns = columnVisitor.Columns;

// Transform AST
var normalizer = new NormalizeColNamesVisitor(schema);
expression.Accept(normalizer);

// Validate structure
var cteValidator = new CteValidatorVisitor();
cte.Accept(cteValidator);
if (cteValidator.IsRecursive) { ... }
```

---

## Query Conversion

### SELECT Statement Conversion

The [`ConvertSelectStatement()`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L2675-L2811) method handles SELECT:

1. **TDS Check**: Attempt TDS Endpoint if compatible
2. **CTE Processing**: Convert CTEs via `ConvertCTEs()`
3. **Query Dispatch**: Route to `ConvertSelectQuerySpec()` or `ConvertBinaryQuery()`
4. **Wrap in SelectNode**: Final projection and materialization

**Query Spec Conversion** ([`ConvertSelectQuerySpec()`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L2991)):

```
FROM clause     → ConvertFromClause()      → Data source nodes
WHERE clause    → FilterNode               → Predicate filter
Subqueries      → ConvertInSubqueries()    → Join transformations
                  ConvertExistsSubqueries()
                  ConvertScalarSubqueries()
GROUP BY        → ConvertGroupByAggregates() → HashMatchAggregateNode
HAVING          → ConvertHavingClause()    → FilterNode
Window funcs    → ConvertWindowFunctions() → Partitioned nodes
SELECT columns  → ComputeScalarNode        → Projections
ORDER BY        → ConvertOrderByClause()   → SortNode
TOP/OFFSET      → TopNode/OffsetFetchNode  → Row limiting
```

### DML Statement Conversion

**INSERT** ([`ConvertInsertStatement()`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L1516-L1784)):

1. Validate OUTPUT not used (unsupported)
2. Route to `ConvertInsertValuesSource()` or `ConvertInsertSelectSource()`
3. Build InsertNode with column mappings

**UPDATE** ([`ConvertUpdateStatement()`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L2189-L2672)):

1. Validate WHERE clause if `BlockUpdateWithoutWhere`
2. Use `UpdateTargetVisitor` to find target entity
3. Build inner SELECT for record identification
4. Build UpdateNode with column mappings

**DELETE** ([`ConvertDeleteStatement()`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L1786-L2187)):

1. Validate WHERE clause if `BlockDeleteWithoutWhere`
2. Use `UpdateTargetVisitor` to find target entity
3. Build inner SELECT for record identification
4. Build DeleteNode with primary key

### CTE Conversion

The [`ConvertCTEs()`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L402-L612) method handles CTEs:

**Non-Recursive CTEs:**
1. Validate with `CteValidatorVisitor`
2. Convert anchor query
3. Cache as `AliasNode` for reference

**Recursive CTEs:**
1. Validate recursion rules (no DISTINCT, GROUP BY, TOP, etc.)
2. Build complex node structure:
   - `ComputeScalarNode` for recursion depth
   - `ConcatenateNode` for anchor + recursive results
   - `IndexSpoolNode` for stack-based spool
   - `TableSpoolNode` for lazy intermediate results
   - `NestedLoopNode` for recursive iteration
   - `AssertNode` for MAXRECURSION enforcement

```
Anchor Query
    │
    ▼
ComputeScalar (add depth=0)
    │
    ▼
ConcatenateNode ◄────────────────────┐
    │                                │
    ├──► TableSpoolNode (lazy) ──►───┤
    │                                │
    ▼                                │
IndexSpoolNode (stack)               │
    │                                │
    ▼                                │
NestedLoopNode                       │
    │                                │
    ▼                                │
AssertNode (MAXRECURSION) ──► ComputeScalar (depth+1) ──► Recursive Query
```

### Subquery Conversion

**IN Subquery** ([`ConvertInSubquery()`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L3696)):

Transforms `col1 IN (SELECT col2 FROM t)` to LEFT OUTER JOIN:
```sql
-- Original
SELECT * FROM a WHERE x IN (SELECT y FROM b)

-- Transformed
SELECT * FROM a LEFT JOIN b ON a.x = b.y WHERE b.y IS NOT NULL
```

**EXISTS Subquery** ([`ConvertExistsSubquery()`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L3858)):

Transforms to outer semi-join with TOP 1:
```sql
-- Original
SELECT * FROM a WHERE EXISTS (SELECT 1 FROM b WHERE b.fk = a.pk)

-- Transformed to semi-join with:
-- - TOP 1 node on inner query
-- - TableSpoolNode for result caching
```

### Join Conversion

The [`ConvertTableReference()`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L5228) method handles joins:

1. Convert left and right operands recursively
2. Use `JoinConditionVisitor` to extract join keys
3. Select join algorithm:
   - **MergeJoin**: When joining on primary key of either side
   - **NestedLoop**: Default for other join conditions

---

## TDS Endpoint Detection

The [`TDSEndpointCompatibilityVisitor`](../MarkMpn.Sql4Cds.Engine/Visitors/TDSEndpointCompatibilityVisitor.cs) checks if a query can execute directly on the Dataverse TDS Endpoint.

### Compatible Features

- SELECT queries without modifications
- Standard T-SQL functions
- Simple WHERE clauses
- ORDER BY, GROUP BY, HAVING
- Joins between TDS-accessible tables

### Incompatible Features

| Feature | Reason |
|---------|--------|
| INSERT/UPDATE/DELETE | TDS is read-only |
| EXECUTE statements | Not supported |
| IF/WHILE/GOTO | Control flow not supported |
| Recursive CTEs | Requires client processing |
| JSON functions | Not available |
| XML methods | Not available |
| Global variables (@@IDENTITY, etc.) | Session state not available |
| EXECUTE AS / REVERT | Impersonation not supported |
| Cursors | Not supported |
| Temporary tables | CREATE TABLE/DROP TABLE not supported |
| EntityReference types | Special type handling needed |

### Fast Path

When TDS-compatible ([`ExecutionPlanBuilder.cs:97-124`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L97-L124)):

```csharp
if (tdsCompatibilityVisitor.IsCompatible && !tdsCompatibilityVisitor.RequiresCteRewrite)
{
    useTDSEndpointDirectly = true;
    return new IRootExecutionPlanNode[] { new SqlNode { Sql = sql } };
}
```

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| QueryParseException | SQL syntax error | Fix SQL syntax |
| NotSupportedQueryFragmentException | Unsupported SQL construct | Use alternative syntax |
| Sql4CdsError | Semantic/validation error | Check error message and code |

### Parse Error Handling

```csharp
var fragment = dom.Parse(new StringReader(sql), out var errors);
if (errors.Count > 0)
    throw new QueryParseException(errors[0]);
```

Parse errors include line number and column for precise location.

### Validation Errors

| Error Code | Description |
|------------|-------------|
| CteDuplicateName | Duplicate CTE name in WITH clause |
| UnknownGotoLabel | GOTO references non-existent label |
| DuplicateGotoLabel | Multiple labels with same name |
| GotoIntoTryOrCatch | GOTO jumps into TRY or CATCH block |
| ThrowOutsideCatch | THROW without error in CATCH block |
| ExceededMaxRecursion | MAXRECURSION hint > 32,767 |

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Empty SQL | Parse succeeds, returns empty plan array |
| SQL with only comments | Parse succeeds, returns empty plan array |
| Multiple batches (GO) | Each batch processed, results concatenated |
| CTE column count mismatch | Throws TableValueConstructorTooManyColumns/TooFewColumns |

---

## Design Decisions

### Why TSql160Parser?

**Context:** Need to parse T-SQL syntax compatible with SQL Server 2019/2022.

**Decision:** Use `TSql160Parser` from Microsoft.SqlServer.TransactSql.ScriptDom.

**Alternatives considered:**
- Custom parser: Rejected - enormous effort, compatibility issues
- TSql150Parser (SQL 2019): Rejected - missing newer syntax
- ANTLR T-SQL grammar: Rejected - less accurate than Microsoft's own parser

**Consequences:**
- Positive: 100% syntax compatibility with SQL Server
- Positive: Microsoft-maintained, well-tested
- Positive: Supports all T-SQL features
- Negative: Large dependency (~2MB)
- Negative: No control over parser behavior

### Why Visitor Pattern for AST Processing?

**Context:** Need to analyze and transform parsed SQL in multiple ways.

**Decision:** Use TSqlFragmentVisitor subclasses for each transformation/analysis.

**Test results:**
| Approach | Lines of Code | Maintainability |
|----------|---------------|-----------------|
| Manual tree walking | ~8000 | Low (scattered logic) |
| Visitor pattern | ~3000 | High (focused classes) |

**Alternatives considered:**
- Manual tree walking: Rejected - hard to maintain, scattered logic
- Expression rewriting library: Rejected - overkill for this use case

**Consequences:**
- Positive: Clean separation of concerns (28 focused visitors)
- Positive: Easy to add new transformations
- Positive: Testable in isolation
- Negative: Many small classes to navigate

### Why Clone Session During Compilation?

**Context:** CREATE TABLE during compilation modifies TempDb, which could affect subsequent queries.

**Decision:** Clone `SessionContext` at start of `Build()` ([`ExecutionPlanBuilder.cs:28`](../MarkMpn.Sql4Cds.Engine/ExecutionPlanBuilder.cs#L28)).

**Consequences:**
- Positive: Compilation is isolated from session state
- Positive: Failed compilation doesn't leave artifacts
- Negative: Small memory overhead for cloning

### Why Method-Based Statement Dispatch?

**Context:** Need to handle 20+ statement types with different conversion logic.

**Decision:** Use large switch in `ConvertStatementInternal()` dispatching to dedicated methods.

**Alternatives considered:**
- Strategy pattern with IStatementConverter: Rejected - over-engineering
- Dictionary lookup: Rejected - harder to debug, same complexity

**Consequences:**
- Positive: Clear, debuggable code flow
- Positive: Each statement type has dedicated method
- Negative: Large methods (~400 lines for dispatch)

### Why Transform IN/EXISTS to Joins?

**Context:** Dataverse FetchXML doesn't support correlated subqueries.

**Decision:** Transform `IN (subquery)` and `EXISTS (subquery)` to join operations.

**Consequences:**
- Positive: Enables subquery support via standard join nodes
- Positive: Allows optimizer to fold conditions into FetchXML
- Negative: More complex execution plans
- Negative: May require TableSpoolNode for correct semantics

---

## Configuration

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| QuotedIdentifiers | bool | true | SQL parser quoted identifier mode |
| EstimatedPlanOnly | bool | true | Skip full optimization, estimate rows only |
| BlockUpdateWithoutWhere | bool | true | Require WHERE on UPDATE |
| BlockDeleteWithoutWhere | bool | true | Require WHERE on DELETE |
| PrimaryDataSource | string | required | Default data source for queries |

### Optimizer Hints

SQL4CDS custom hints (in USE HINT clause):

| Hint | Effect |
|------|--------|
| FORCE_SQL4CDS | Skip TDS Endpoint, use SQL4CDS engine |
| BYPASS_CUSTOM_PLUGIN_EXECUTION | Skip Dataverse plugins |
| RETRIEVE_TOTAL_RECORD_COUNT | Include total count in results |
| DEBUG_BYPASS_OPTIMIZATION | Skip FoldQuery optimization |
| MAXRECURSION n | CTE recursion limit (default 100) |
| FETCHXML_PAGE_SIZE_n | FetchXML page size |
| BATCH_SIZE_n | DML batch size |

---

## Testing

### Acceptance Criteria

- [ ] Parse valid T-SQL without errors
- [ ] Report parse errors with line numbers
- [ ] Convert SELECT to SelectNode with correct schema
- [ ] Convert INSERT/UPDATE/DELETE to appropriate DML nodes
- [ ] Handle CTEs with correct scoping
- [ ] Detect TDS-compatible queries accurately
- [ ] Validate GOTO labels correctly
- [ ] Apply transformation visitors in correct order

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| Empty query | "" | Empty plan array |
| Comment only | "-- comment" | Empty plan array |
| Multiple statements | "SELECT 1; SELECT 2" | Two SelectNodes |
| CTE with recursion | Recursive CTE | Complex node tree with Assert |
| Subquery correlation | Correlated subquery | Join with outer references |

### Test Examples

```csharp
[Fact]
public void Build_SimpleSelect_ReturnsSelectNode()
{
    // Arrange
    var builder = new ExecutionPlanBuilder(session, options);

    // Act
    var plans = builder.Build("SELECT name FROM account", parameters, out var tds);

    // Assert
    Assert.Single(plans);
    Assert.IsType<SelectNode>(plans[0]);
}

[Fact]
public void Build_SyntaxError_ThrowsQueryParseException()
{
    // Arrange
    var builder = new ExecutionPlanBuilder(session, options);

    // Act & Assert
    var ex = Assert.Throws<QueryParseException>(
        () => builder.Build("SELECT FROM", parameters, out _));
    Assert.Contains("Incorrect syntax", ex.Message);
}

[Fact]
public void Build_TDSCompatible_SetsFlagTrue()
{
    // Arrange
    var builder = new ExecutionPlanBuilder(session, options);

    // Act
    builder.Build("SELECT name FROM account", parameters, out var useTds);

    // Assert
    Assert.True(useTds);
}
```

---

## Related Specs

- [architecture.md](./architecture.md) - System architecture and patterns
- [type-system.md](./type-system.md) - Type system and conversions
- [execution-plan-nodes.md](./execution-plan-nodes.md) - Node interfaces and contracts
- [query-optimization.md](./query-optimization.md) - FoldQuery and optimization (planned)
- [expression-evaluation.md](./expression-evaluation.md) - Expression compilation (planned)

---

## Appendix: Visitor Reference

### RewriteVisitorBase

Base class for expression rewriting ([`RewriteVisitorBase.cs:1-268`](../MarkMpn.Sql4Cds.Engine/Visitors/RewriteVisitorBase.cs#L1-L268)):

```csharp
abstract class RewriteVisitorBase : TSqlFragmentVisitor
{
    protected abstract ScalarExpression ReplaceExpression(
        ScalarExpression expression, out string name);
    protected abstract BooleanExpression ReplaceExpression(
        BooleanExpression expression);
}
```

Subclasses override `ReplaceExpression()` to transform expressions throughout the AST.

### CteValidatorVisitor

Validates CTE structure and identifies recursion ([`CteValidatorVisitor.cs:1-220`](../MarkMpn.Sql4Cds.Engine/Visitors/CteValidatorVisitor.cs#L1-L220)):

```csharp
class CteValidatorVisitor : TSqlConcreteFragmentVisitor
{
    public string Name { get; }
    public bool IsRecursive { get; }
    public QueryExpression AnchorQuery { get; }
    public List<QueryExpression> RecursiveQueries { get; }
}
```

Validates:
- No ORDER BY without TOP
- No GROUP BY, DISTINCT, TOP in recursive member
- UNION ALL required for recursive CTEs
- No outer joins, scalar aggregates, subqueries in recursive member

### TDSEndpointCompatibilityVisitor

Checks TDS Endpoint compatibility ([`TDSEndpointCompatibilityVisitor.cs:1-542`](../MarkMpn.Sql4Cds.Engine/Visitors/TDSEndpointCompatibilityVisitor.cs#L1-L542)):

```csharp
class TDSEndpointCompatibilityVisitor : TSqlFragmentVisitor
{
    public bool IsCompatible { get; }
    public bool RequiresCteRewrite { get; }
}
```

Queries TDS Endpoint for supported tables and validates all features used.
