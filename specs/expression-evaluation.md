# Expression Evaluation

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-26
**Code:** [MarkMpn.Sql4Cds.Engine/ExecutionPlan/ExpressionExtensions.cs](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/ExpressionExtensions.cs)

---

## Overview

The expression evaluation system compiles T-SQL expressions from the ScriptDom AST into optimized .NET LINQ Expression trees that execute at runtime. This enables SQL4CDS to evaluate WHERE clauses, computed columns, CASE expressions, function calls, and arithmetic operations with SQL Server-compatible semantics while achieving near-native performance through JIT compilation and aggressive caching.

### Goals

- **SQL Server Compatibility**: Evaluate expressions with correct type coercion, null propagation, and collation behavior
- **Performance**: Compile expressions once and cache for reuse, eliminating interpretation overhead
- **Extensibility**: Support 50+ SQL functions and enable easy addition of new functions

### Non-Goals

- Full T-SQL expression parity (some window functions handled by dedicated nodes)
- Expression optimization (delegated to query optimizer)
- Parallel expression evaluation (single-threaded execution)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     EXPRESSION COMPILATION PIPELINE                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────┐                                                   │
│  │ TSqlFragment AST │  ← From ScriptDom parser                          │
│  │ (ScriptDom)      │                                                   │
│  └────────┬─────────┘                                                   │
│           │                                                             │
│           ▼                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    ExpressionExtensions                           │  │
│  │                                                                   │  │
│  │  ┌────────────┐    ┌────────────┐    ┌────────────────────────┐  │  │
│  │  │ GetType()  │───▶│ToExpression│───▶│ LINQ Expression Tree   │  │  │
│  │  │ (analyze)  │    │ (generate) │    │ (System.Linq.Expressions)│  │  │
│  │  └────────────┘    └────────────┘    └───────────┬────────────┘  │  │
│  │                                                   │               │  │
│  │                                                   ▼               │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │                    FoldLambdas()                            │  │  │
│  │  │         (inline nested lambda invocations)                  │  │  │
│  │  └───────────────────────┬────────────────────────────────────┘  │  │
│  │                          │                                        │  │
│  │                          ▼                                        │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │                Expression.Lambda.Compile()                  │  │  │
│  │  │              (JIT compile to native code)                   │  │  │
│  │  └───────────────────────┬────────────────────────────────────┘  │  │
│  └──────────────────────────┼────────────────────────────────────────┘  │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │               ConcurrentDictionary Cache                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────────┐  │  │
│  │  │ _cache      │  │ _boolCache  │  │ _intermediateCache       │  │  │
│  │  │ (objects)   │  │ (booleans)  │  │ (type analysis)          │  │  │
│  │  └─────────────┘  └─────────────┘  └──────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     RUNTIME EXECUTION                                    │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              Func<ExpressionExecutionContext, T>                  │  │
│  │                                                                   │  │
│  │  ExpressionExecutionContext provides:                            │  │
│  │  - Entity (current row data)                                     │  │
│  │  - ParameterValues (SQL parameters)                              │  │
│  │  - PrimaryDataSource (metadata access)                           │  │
│  │  - Error (for error handling functions)                          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    ExpressionFunctions                            │  │
│  │  (static methods implementing 50+ SQL functions)                  │  │
│  │                                                                   │  │
│  │  String: Left, Right, Substring, Replace, Upper, Lower, Trim...  │  │
│  │  Date: DateAdd, DateDiff, DatePart, GetDate, DateTrunc...        │  │
│  │  JSON: Json_Value, Json_Query, Json_Path_Exists, IsJson...       │  │
│  │  XML: Query, Value                                                │  │
│  │  Misc: NewId, Format, FormatMessage, IsNull...                   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Responsibility |
|-----------|----------------|
| **ExpressionExtensions** | Compiles TSqlFragment AST to LINQ expressions with caching |
| **ExpressionFunctions** | Static method implementations for 50+ SQL functions |
| **ExpressionExecutionContext** | Runtime context providing entity data and parameters |
| **ExpressionCompilationContext** | Compilation-time context with schema and data source info |
| **CompiledExpression<T>** | Cached compilation result with original AST and delegate |
| **IntermediateExpression** | Cached type analysis result for reuse |

### Dependencies

- Depends on: [type-system.md](./type-system.md) - Type conversion and collation handling
- Depends on: [architecture.md](./architecture.md) - Context pattern and execution model
- Uses: `Microsoft.SqlServer.TransactSql.ScriptDom` for TSqlFragment AST types
- Uses: `System.Linq.Expressions` for expression tree generation

---

## Specification

### Core Requirements

1. Compile all supported TSqlFragment expression types to executable functions
2. Cache compiled expressions for reuse across executions
3. Propagate SQL NULL through all operations correctly
4. Apply collation settings to string comparisons and functions
5. Support SQL Server type coercion rules for binary operations

### Supported Expression Types

| Category | Expression Types |
|----------|-----------------|
| **Literals** | IntegerLiteral, NumericLiteral, RealLiteral, StringLiteral, BinaryLiteral, NullLiteral, MoneyLiteral, IdentifierLiteral, OdbcLiteral |
| **Column Access** | ColumnReferenceExpression |
| **Comparisons** | BooleanComparisonExpression (=, <>, <, >, <=, >=) |
| **Boolean Logic** | BooleanBinaryExpression (AND, OR), BooleanNotExpression, BooleanParenthesisExpression |
| **Predicates** | BooleanIsNullExpression, InPredicate, LikePredicate, FullTextPredicate, DistinctPredicate |
| **Arithmetic** | BinaryExpression (+, -, *, /, %, &, \|, ^, <<, >>), UnaryExpression (+, -, ~) |
| **Conditionals** | SimpleCaseExpression, SearchedCaseExpression |
| **Type Conversion** | CastCall, ConvertCall |
| **Functions** | FunctionCall, ParameterlessCall |
| **Variables** | VariableReference, GlobalVariableExpression |
| **Grouping** | ParenthesisExpression |

### Compilation Flow

**Expression Compilation:**

1. **Type Analysis Phase**: Call `ToExpression()` with `createExpression=false` to determine result type and cache key
2. **Cache Lookup**: Check if compiled delegate exists in cache using type-based key
3. **Expression Generation**: If not cached, call `ToExpression()` with `createExpression=true` to build LINQ expression tree
4. **Lambda Folding**: Apply `FoldLambdas()` to inline nested lambda invocations for performance
5. **JIT Compilation**: Call `Expression.Lambda.Compile()` to produce native delegate
6. **Cache Storage**: Store compiled result in appropriate cache (`_cache` or `_boolCache`)

### Type Resolution Rules

Result types are determined bottom-up through the expression tree ([`ExpressionExtensions.cs:122-192`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/ExpressionExtensions.cs#L122-L192)):

| Expression Type | Type Determination |
|-----------------|-------------------|
| IntegerLiteral | `int` if in range, else `bigint` |
| NumericLiteral | `decimal(p,s)` from parsed value |
| StringLiteral | `nvarchar(n)` if IsNational, else `varchar(n)` |
| BinaryExpression | Common type via `CanMakeConsistentTypes()` |
| CaseExpression | Common type of all THEN/ELSE branches |
| FunctionCall | Return type of resolved function |

### Constraints

- All expressions must be compilable at query preparation time
- Expression execution must be deterministic (except for NewId, GetDate, etc.)
- NULL inputs propagate to NULL outputs (except IS NULL, ISNULL)
- Collation must be resolved before string comparison

---

## Core Types

### ExpressionExtensions

Static class providing expression compilation ([`ExpressionExtensions.cs:31`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/ExpressionExtensions.cs#L31)):

```csharp
static class ExpressionExtensions
{
    public static Type GetType(this TSqlFragment expr,
        ExpressionCompilationContext context, out DataTypeReference sqlType);

    public static Func<ExpressionExecutionContext, object> Compile(
        this TSqlFragment expr, ExpressionCompilationContext context);

    public static Func<ExpressionExecutionContext, bool> Compile(
        this BooleanExpression b, ExpressionCompilationContext context);
}
```

The `Compile` methods ([`ExpressionExtensions.cs:89-120`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/ExpressionExtensions.cs#L89-L120)) return delegates that accept runtime context and return evaluated results. Boolean expressions return `bool` directly; other expressions return boxed objects.

### ExpressionFunctions

Static class implementing SQL functions ([`ExpressionFunctions.cs:34`](../MarkMpn.Sql4Cds.Engine/ExpressionFunctions.cs#L34)):

```csharp
class ExpressionFunctions
{
    [SqlFunction(IsDeterministic = true)]
    public static SqlString Left(SqlString s, SqlInt32 length);

    [SqlFunction(IsDeterministic = false)]
    public static SqlDateTime GetDate();

    [CollationSensitive]
    public static SqlString Json_Value(SqlString json, SqlString jpath);
}
```

Functions are discovered via reflection and invoked through generated expression trees. Attributes provide metadata:
- `[SqlFunction(IsDeterministic = true/false)]` - Marks deterministic vs non-deterministic
- `[MaxLength]` - Specifies maximum output length
- `[CollationSensitive]` - Indicates collation-dependent behavior

### ExpressionExecutionContext

Runtime context for expression evaluation ([`NodeContext.cs`](../MarkMpn.Sql4Cds.Engine/NodeContext.cs)):

```csharp
class ExpressionExecutionContext : NodeExecutionContext
{
    public Entity Entity { get; }  // Current row being processed
}
```

Extends `NodeExecutionContext` which provides:
- `ParameterValues` - SQL parameter values dictionary
- `PrimaryDataSource` - Access to metadata and default collation
- `Error` - Current error for error-handling functions

### CompiledExpression<T>

Cached compilation result ([`ExpressionExtensions.cs:33-44`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/ExpressionExtensions.cs#L33-L44)):

```csharp
class CompiledExpression<T>
{
    public TSqlFragment Expression { get; }
    public Expression Converted { get; }
    public Func<ExpressionExecutionContext, TSqlFragment, T> Compiled { get; }
}
```

Stores original AST, intermediate LINQ expression, and compiled delegate for debugging and reuse.

---

## SQL Functions

### String Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `LEFT` | `Left(string, length)` | First N characters |
| `RIGHT` | `Right(string, length)` | Last N characters |
| `SUBSTRING` | `Substring(string, start, length)` | Extract substring (1-based) |
| `REPLACE` | `Replace(string, find, replace)` | Replace all occurrences |
| `LEN` | `Len(string)` | Length excluding trailing spaces |
| `DATALENGTH` | `DataLength(value)` | Byte length |
| `UPPER` | `Upper(string)` | Uppercase (culture-aware) |
| `LOWER` | `Lower(string)` | Lowercase (culture-aware) |
| `LTRIM` | `LTrim(string [, chars])` | Remove leading whitespace/chars |
| `RTRIM` | `RTrim(string [, chars])` | Remove trailing whitespace/chars |
| `TRIM` | `Trim([type] [chars FROM] string)` | Remove leading/trailing chars |
| `CHARINDEX` | `CharIndex(find, string [, start])` | Find substring position |
| `PATINDEX` | `PatIndex(pattern, string)` | Pattern match position |
| `STUFF` | `Stuff(string, start, length, insert)` | Delete and insert |
| `CHAR` | `Char(code)` | ASCII character |
| `NCHAR` | `NChar(code)` | Unicode character |
| `ASCII` | `Ascii(string)` | ASCII code of first char |
| `UNICODE` | `Unicode(string)` | Unicode value of first char |

### Date/Time Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `DATEADD` | `DateAdd(interval, count, date)` | Add interval to date |
| `DATEDIFF` | `DateDiff(interval, start, end)` | Difference in interval units |
| `DATEPART` | `DatePart(part, date)` | Extract date component |
| `DATETRUNC` | `DateTrunc(part, date)` | Truncate to precision |
| `GETDATE` | `GetDate()` | Current local datetime |
| `GETUTCDATE` | `GetUtcDate()` | Current UTC datetime |
| `SYSDATETIME` | `SysDateTime()` | Current local datetime |
| `SYSUTCDATETIME` | `SysUtcDateTime()` | Current UTC datetime |
| `SYSDATETIMEOFFSET` | `SysDateTimeOffset()` | Current datetime with offset |
| `DAY` | `Day(date)` | Day of month |
| `MONTH` | `Month(date)` | Month number |
| `YEAR` | `Year(date)` | Year number |

**Supported Date Intervals:** year, quarter, month, day, week, hour, minute, second, millisecond, microsecond, nanosecond, iso_week

### JSON Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `JSON_VALUE` | `Json_Value(json, path)` | Extract scalar value (max 4000 chars) |
| `JSON_QUERY` | `Json_Query(json, path)` | Extract object or array |
| `JSON_PATH_EXISTS` | `Json_Path_Exists(json, path)` | Test if path exists |
| `ISJSON` | `IsJson(string [, type])` | Validate JSON format |

JSON path supports lax/strict mode (e.g., `$.store.book[0].title` or `strict $.name`).

### XML Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `query()` | `xml.query(xquery)` | Execute XQuery, return XML |
| `value()` | `xml.value(xquery, type)` | Execute XQuery, return scalar |

### Entity Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `CREATELOOKUP` | `CreateLookup(table, id)` | Create entity reference |
| `CREATEELASTICLOOKUP` | `CreateElasticLookup(table, id, pid)` | Create elastic table reference |

### Miscellaneous Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `NEWID` | `NewId()` | Generate new GUID |
| `ISNULL` | `IsNull(check, replacement)` | Null substitution |
| `FORMAT` | `Format(value, format [, culture])` | Format with pattern |
| `FORMATMESSAGE` | `FormatMessage(format, args...)` | Printf-style formatting |
| `USER_NAME` | `User_Name()` | Current user |
| `SERVERPROPERTY` | `ServerProperty(property)` | Server information |
| `COLLATIONPROPERTY` | `CollationProperty(collation, property)` | Collation information |
| `SQL_VARIANT_PROPERTY` | `Sql_Variant_Property(variant, property)` | Variant type info |

### Error Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `ERROR_NUMBER` | `Error_Number()` | Current error number |
| `ERROR_MESSAGE` | `Error_Message()` | Current error message |
| `ERROR_SEVERITY` | `Error_Severity()` | Error severity level |
| `ERROR_STATE` | `Error_State()` | Error state |
| `ERROR_LINE` | `Error_Line()` | Line where error occurred |
| `ERROR_PROCEDURE` | `Error_Procedure()` | Procedure name |

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| `NotSupportedQueryFragmentException` | Unsupported expression type | Rewrite query |
| `QueryExecutionException` | Runtime evaluation failure | Check input data |
| `Sql4CdsError.InvalidCollation` | Unknown collation name | Use valid collation |
| `Sql4CdsError.TypeMismatch` | Incompatible types in operation | Add explicit CAST |

### Recovery Strategies

- **Type mismatch**: Add explicit CAST/CONVERT to resolve
- **Collation conflict**: Use explicit COLLATE clause
- **NULL in unexpected place**: Use ISNULL or COALESCE
- **JSON/XML path not found**: Use lax mode or check path

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| NULL + any value | Returns NULL (null propagation) |
| Division by zero | Throws QueryExecutionException |
| String overflow | Depends on function (truncate or error) |
| JSON strict mode, path not found | Throws QueryExecutionException |
| LIKE with NULL pattern | Returns NULL (not false) |

---

## Design Decisions

### Why LINQ Expression Trees?

**Context:** Expressions execute millions of times during query processing. Interpretation would be too slow.

**Decision:** Compile TSqlFragment AST to .NET LINQ Expression trees, then JIT compile to native delegates.

**Test results:**
| Approach | Performance |
|----------|-------------|
| AST interpretation | ~50,000 ops/sec |
| LINQ Expression compilation | ~5,000,000 ops/sec |

**Alternatives considered:**
- Direct interpretation: Rejected - 100x slower than compilation
- Code generation to IL: Rejected - More complex, similar performance to Expression trees
- Roslyn compilation: Rejected - High startup cost, overkill for expressions

**Consequences:**
- Positive: Near-native execution speed after one-time compilation
- Positive: Leverages .NET JIT optimizations
- Negative: Initial compilation overhead (amortized through caching)

### Why Three-Level Caching?

**Context:** Same expressions appear repeatedly across query executions. Compilation is expensive.

**Decision:** Use three `ConcurrentDictionary` caches:
1. `_cache` - Compiled object-returning expressions
2. `_boolCache` - Compiled boolean expressions
3. `_intermediateCache` - Type analysis results

**Cache Key Strategy:**
- Key includes expression type and SQL types of inputs
- Example: `"(int)<ColumnReference> > <IntLiteral>"`
- Collation included for string operations

**Consequences:**
- Positive: Compilation happens once per unique expression signature
- Positive: Thread-safe via ConcurrentDictionary
- Negative: Memory usage grows with expression variety (bounded by query complexity)

### Why Lambda Folding?

**Context:** Sub-expressions compile to lambdas that get invoked at runtime. Nested invocations add overhead.

**Decision:** `FoldLambdas()` pass ([`ExpressionExtensions.cs`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/ExpressionExtensions.cs)) inlines lambda bodies directly into parent expression.

**Example:**
```
Before: Expression.Invoke(childLambda, context, expr)
After:  [childLambda body with parameters substituted]
```

**Consequences:**
- Positive: Eliminates function call overhead for sub-expressions
- Positive: Enables further JIT optimizations (inlining, register allocation)
- Negative: Slightly larger expression trees (acceptable tradeoff)

### Why Static ExpressionFunctions Class?

**Context:** SQL functions need to be discovered and invoked from compiled expressions.

**Decision:** Implement functions as static methods with attributes for metadata. Use reflection to discover and generate call expressions.

**Implementation Pattern:**
```csharp
[SqlFunction(IsDeterministic = true)]
[CollationSensitive]
public static SqlString Left(SqlString s, SqlInt32 length)
{
    if (s.IsNull || length.IsNull)
        return SqlString.Null;
    // ...
}
```

**Consequences:**
- Positive: Easy to add new functions (just add static method)
- Positive: Attributes provide metadata without modifying signature
- Positive: Natural NULL handling via SqlTypes
- Negative: All function parameters must use SqlTypes (not raw .NET types)

### Why INullable for All Parameters?

**Context:** SQL has three-valued logic. Functions must handle NULL inputs correctly.

**Decision:** All function parameters use `System.Data.SqlTypes` (SqlString, SqlInt32, etc.) which implement INullable.

**NULL Handling Pattern:**
```csharp
if (s.IsNull || length.IsNull)
    return SqlString.Null;  // Propagate NULL
```

**Consequences:**
- Positive: Consistent NULL propagation across all functions
- Positive: No separate null-checking wrappers needed
- Negative: More verbose than using nullable value types

---

## Extension Points

### Adding a New SQL Function

1. **Add static method** to `ExpressionFunctions.cs`:
```csharp
[SqlFunction(IsDeterministic = true)]
public static SqlInt32 MyFunction(SqlString input, SqlInt32 param)
{
    if (input.IsNull || param.IsNull)
        return SqlInt32.Null;

    // Implementation
    return new SqlInt32(result);
}
```

2. **Add attributes** as needed:
   - `[SqlFunction(IsDeterministic = true/false)]` - Required
   - `[CollationSensitive]` - If result depends on string collation
   - `[MaxLength(n)]` - If string output has fixed max length

3. **Functions are auto-discovered** via reflection when matching FunctionCall names

### Adding a New Expression Type

1. **Add dispatch case** in `ToExpression()` ([`ExpressionExtensions.cs:122-192`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/ExpressionExtensions.cs#L122-L192)):
```csharp
else if (expr is MyNewExpression myExpr)
    expression = ToExpression(myExpr, context, contextParam, exprParam, createExpression, out sqlType, out cacheKey);
```

2. **Implement overload** for the expression type:
```csharp
private static Expression ToExpression(MyNewExpression expr,
    ExpressionCompilationContext context,
    ParameterExpression contextParam,
    ParameterExpression exprParam,
    bool createExpression,
    out DataTypeReference sqlType,
    out string cacheKey)
{
    // Determine sqlType and generate cacheKey
    // If createExpression, build and return LINQ expression
}
```

3. **Handle sub-expressions** using `InvokeSubExpression()` helper for proper caching

---

## Testing

### Acceptance Criteria

- [ ] All literal types compile and evaluate correctly
- [ ] Boolean comparisons return SqlBoolean with proper null handling
- [ ] CASE expressions return common type of all branches
- [ ] String functions respect collation settings
- [ ] Date functions handle all interval types
- [ ] JSON functions support lax and strict modes
- [ ] Cache hit rate > 95% for repeated queries
- [ ] NULL propagates through all arithmetic operations

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| NULL comparison | `NULL = NULL` | `NULL` (not true) |
| NULL IS NULL | `NULL IS NULL` | `true` |
| Integer overflow | `2147483647 + 1` | Overflow error |
| Decimal precision | `1.0 / 3.0` | Proper scale preservation |
| Collation compare | `'a' = 'A'` (CI collation) | `true` |
| LIKE with escape | `'10%' LIKE '10[%]'` | `true` |
| JSON lax mode | `JSON_VALUE('{}', '$.missing')` | `NULL` |
| JSON strict mode | `JSON_VALUE('{}', 'strict $.missing')` | Error |

### Test Examples

```csharp
[TestMethod]
public void Compile_IntegerLiteral_ReturnsCorrectValue()
{
    // Arrange
    var literal = new IntegerLiteral { Value = "42" };
    var context = CreateCompilationContext();

    // Act
    var compiled = literal.Compile(context);
    var result = compiled(CreateExecutionContext());

    // Assert
    Assert.AreEqual(new SqlInt32(42), result);
}

[TestMethod]
public void Compile_NullPropagation_ReturnsNull()
{
    // Arrange - Expression: NULL + 5
    var expr = new BinaryExpression
    {
        BinaryExpressionType = BinaryExpressionType.Add,
        FirstExpression = new NullLiteral(),
        SecondExpression = new IntegerLiteral { Value = "5" }
    };
    var context = CreateCompilationContext();

    // Act
    var compiled = expr.Compile(context);
    var result = compiled(CreateExecutionContext());

    // Assert
    Assert.IsTrue(((INullable)result).IsNull);
}

[TestMethod]
public void Left_WithCollation_PreservesCollation()
{
    // Arrange
    var collation = Collation.USEnglish;
    var input = collation.ToSqlString("Hello World");

    // Act
    var result = ExpressionFunctions.Left(input, new SqlInt32(5));

    // Assert
    Assert.AreEqual("Hello", result.Value);
    Assert.AreEqual(input.LCID, result.LCID);
    Assert.AreEqual(input.SqlCompareOptions, result.SqlCompareOptions);
}
```

---

## Related Specs

- [type-system.md](./type-system.md) - Type conversion rules and collation handling
- [execution-plan-nodes.md](./execution-plan-nodes.md) - How nodes use compiled expressions
- [query-compilation.md](./query-compilation.md) - Expression extraction during plan building
- [architecture.md](./architecture.md) - Context pattern and overall execution model

---

## Roadmap

- Additional SQL Server function compatibility (STRING_AGG, STRING_SPLIT)
- Expression optimization pass (constant folding, dead code elimination)
- Improved error messages with expression location
- Support for additional JSON path features
