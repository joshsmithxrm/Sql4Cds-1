# Type System

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-25
**Code:** [MarkMpn.Sql4Cds.Engine/](../MarkMpn.Sql4Cds.Engine/)

---

## Overview

The SQL4CDS type system provides SQL Server-compatible data type handling for query execution against Microsoft Dataverse. It implements type conversion, collation support, custom date/time types, entity references, and the sql_variant dynamic type—all while maintaining SQL Server's three-valued logic and null propagation semantics.

### Goals

- **SQL Server Compatibility**: Mirror SQL Server's type precedence, implicit/explicit conversion rules, and collation behavior
- **Dataverse Integration**: Bridge Dataverse-specific types (EntityReference, OptionSetValue, Money) to SQL types
- **Null Semantics**: Implement proper SQL NULL propagation through all type operations
- **Collation Support**: Full collation-aware string comparisons and case/accent sensitivity

### Non-Goals

- CLR type serialization (handled by Dataverse SDK)
- Custom user-defined types beyond EntityReference
- Full SQL Server type system (some types like hierarchyid not supported)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         TYPE SYSTEM OVERVIEW                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────┐     ┌──────────────────┐     ┌─────────────────┐ │
│  │  DataTypeHelpers │     │  SqlTypeConverter │     │    Collation    │ │
│  │                  │     │                   │     │                 │ │
│  │  - Factory       │────▶│  - Precedence     │◀───▶│  - LCID         │ │
│  │  - Classification│     │  - Implicit rules │     │  - Compare opts │ │
│  │  - Size/Precision│     │  - Explicit rules │     │  - ToSqlString  │ │
│  └────────┬─────────┘     │  - Conversions    │     └─────────────────┘ │
│           │               └─────────┬─────────┘                          │
│           │                         │                                    │
│           ▼                         ▼                                    │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                      SQL Type Implementations                       │ │
│  │  ┌─────────────┐  ┌────────────┐  ┌────────────┐  ┌─────────────┐ │ │
│  │  │ SqlVariant  │  │ SqlEntity  │  │  SqlDate   │  │  SqlString  │ │ │
│  │  │             │  │ Reference  │  │   Types    │  │ (w/collation)│ │ │
│  │  │ - BaseType  │  │            │  │            │  │             │ │ │
│  │  │ - Value     │  │ - Logical  │  │ - SqlDate  │  │ - LCID      │ │ │
│  │  │ - INullable │  │ - Id       │  │ - SqlTime  │  │ - Options   │ │ │
│  │  │ - ICompare  │  │ - Partition│  │ - DateTime2│  │             │ │ │
│  │  └─────────────┘  └────────────┘  │ - Offset   │  └─────────────┘ │ │
│  │                                   └────────────┘                   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                   System.Data.SqlTypes (BCL)                        │ │
│  │  SqlBoolean, SqlByte, SqlInt16, SqlInt32, SqlInt64, SqlDecimal,    │ │
│  │  SqlDouble, SqlSingle, SqlMoney, SqlGuid, SqlBinary, SqlDateTime   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Responsibility |
|-----------|----------------|
| **DataTypeHelpers** | Factory methods, type classification, size/precision extraction |
| **SqlTypeConverter** | Type conversion rules, precedence, expression generation |
| **Collation** | LCID mapping, comparison options, SqlString wrapping |
| **SqlVariant** | Dynamic type container with family-based comparison |
| **SqlEntityReference** | Dataverse entity reference with optional partition ID |
| **SqlDateTypes** | SQL Server date/time types missing from BCL |

### Dependencies

- Depends on: [architecture.md](./architecture.md) - Uses Context Pattern, integrates with execution nodes
- Uses: `Microsoft.SqlServer.TransactSql.ScriptDom` for `DataTypeReference` AST types
- Uses: `System.Data.SqlTypes` for base SQL type implementations

---

## Specification

### Core Requirements

1. Implement SQL Server type precedence for implicit type coercion
2. Support both implicit and explicit type conversions with proper validation
3. Maintain collation context through string operations
4. Provide null-safe arithmetic and comparison operators
5. Convert between .NET CLR types and SQL types bidirectionally

### Type Precedence Hierarchy

The type precedence determines which type wins when two different types are combined ([`SqlTypeConverter.cs:22-53`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/SqlTypeConverter.cs#L22-L53)):

| Rank | Type | Notes |
|------|------|-------|
| 1 | sql_variant | Highest precedence |
| 2 | datetimeoffset | |
| 3 | datetime2 | |
| 4 | datetime | |
| 5 | smalldatetime | |
| 6 | date | |
| 7 | time | |
| 8 | float | |
| 9 | real | |
| 10 | decimal | |
| 11 | money | |
| 12 | smallmoney | |
| 13 | bigint | |
| 14 | int | |
| 15 | smallint | |
| 16 | tinyint | |
| 17 | bit | |
| 18 | ntext | |
| 19 | text | |
| 20 | image | |
| 21 | rowversion | |
| 22 | uniqueidentifier | |
| 23 | nvarchar | |
| 24 | nchar | |
| 25 | varchar | |
| 26 | char | |
| 27 | varbinary | |
| 28 | binary | Lowest precedence |

### Conversion Rules

**Implicit Conversions** (no CAST required) ([`SqlTypeConverter.cs:376-479`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/SqlTypeConverter.cs#L376-L479)):

| From | To | Allowed |
|------|-----|---------|
| Any numeric | Any numeric | Yes |
| Any type | String types | Yes |
| String types | Any type (except EntityReference) | Yes |
| Binary | Most types (except date-only types) | Yes |
| Numeric | DateTime | Yes |
| DateTime family | Other DateTime types | Yes |
| GUID | Typed EntityReference | Yes |
| EntityReference | GUID | Yes |

**Explicit Conversions** (CAST/CONVERT required) ([`SqlTypeConverter.cs:523-561`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/SqlTypeConverter.cs#L523-L561)):

| From | To | Notes |
|------|-----|-------|
| datetime/smalldatetime | Numeric | Explicit only |
| xml | String/Binary | Explicit only |
| sql_variant | Most types | Explicit (except timestamp, image, text, ntext) |
| String/DateTime | Binary | Explicit only |
| Binary | date/time/datetime2/datetimeoffset | Explicit only |

### Collation Precedence

When combining string types with different collations ([`DataTypeHelpers.cs:742-823`](../MarkMpn.Sql4Cds.Engine/DataTypeHelpers.cs#L742-L823)):

| Label | Precedence | Description |
|-------|------------|-------------|
| Explicit | Highest | COLLATE clause specified |
| Implicit | High | From column definition |
| CoercibleDefault | Low | From database/system default |
| NoCollation | Lowest | No collation applies |

**Resolution Rules:**
1. Two explicit collations must match or error
2. Explicit wins over implicit/coercible
3. Two different implicit collations → NoCollation
4. Implicit wins over coercible default

### Constraints

- All comparison operators return `SqlBoolean` (three-valued: true/false/null)
- Null comparisons propagate null (except IS NULL)
- Type conversions at runtime generate `QueryExecutionException` on failure
- Truncation behavior configurable per-conversion

---

## Core Types

### SqlTypeConverter

Central type conversion engine that generates LINQ expressions for runtime conversion ([`ExecutionPlan/SqlTypeConverter.cs`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/SqlTypeConverter.cs)).

```csharp
class SqlTypeConverter
{
    public static bool CanMakeConsistentTypes(
        DataTypeReference lhs, DataTypeReference rhs,
        DataSource primaryDataSource, TSqlFragment fragment,
        string operation, out DataTypeReference consistent);

    public static bool CanChangeTypeImplicit(DataTypeReference from, DataTypeReference to);
    public static bool CanChangeTypeExplicit(DataTypeReference from, DataTypeReference to);

    public static Expression Convert(Expression expr, Expression context,
        DataTypeReference from, DataTypeReference to,
        Expression style = null, ...);
}
```

The `Convert` method ([`SqlTypeConverter.cs:1301-1567`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/SqlTypeConverter.cs#L1301-L1567)) generates complete conversion expressions including:
- Type validation at compile time
- Null checking at runtime
- Error handling with context-specific messages
- Truncation handling per target type

### DataTypeHelpers

Factory and utility methods for SQL data types ([`DataTypeHelpers.cs`](../MarkMpn.Sql4Cds.Engine/DataTypeHelpers.cs)).

```csharp
static class DataTypeHelpers
{
    // Factory methods
    public static SqlDataTypeReference VarChar(int length, Collation collation, CollationLabel label);
    public static SqlDataTypeReference DateTime2(short scale);
    public static SqlDataTypeReference Decimal(short precision, short scale);
    public static UserDataTypeReference TypedEntityReference(string logicalName);

    // Classification
    public static DataTypeFamily GetDataTypeFamily(DataTypeReference type);
    public static bool IsNumeric(SqlDataTypeOption type);
    public static bool IsStringType(SqlDataTypeOption type);
    public static bool IsDateTimeType(SqlDataTypeOption type);

    // Metadata
    public static int GetSize(DataTypeReference type);
    public static short GetPrecision(DataTypeReference type);
    public static short GetScale(DataTypeReference type);
}
```

### SqlVariant

Dynamic type container implementing SQL Server's sql_variant ([`SqlVariant.cs`](../MarkMpn.Sql4Cds.Engine/SqlVariant.cs)).

```csharp
struct SqlVariant : INullable, IComparable
{
    public DataTypeReference BaseType { get; }
    public INullable Value { get; }
    public bool IsNull => Value == null || Value.IsNull;

    public static readonly SqlVariant Null;
}
```

**Comparison Semantics** ([`SqlVariant.cs:38-91`](../MarkMpn.Sql4Cds.Engine/SqlVariant.cs#L38-L91)):
1. Nulls sort before all values
2. Different type families compared by family precedence
3. Same family: convert to consistent type, then compare values
4. String types: compare collation (LCID, then options) before values

### SqlEntityReference

Dataverse entity reference with partition support ([`SqlEntityReference.cs`](../MarkMpn.Sql4Cds.Engine/SqlEntityReference.cs)).

```csharp
struct SqlEntityReference : INullable, IComparable
{
    public string DataSource { get; }
    public string LogicalName { get; }
    public Guid Id { get; }
    public string PartitionId { get; }  // For elastic tables

    public bool IsNull => ((INullable)_guid).IsNull;
}
```

Supports typed and untyped variants:
- **Untyped**: `EntityReference` - accepts any entity logical name
- **Typed**: `EntityReference(account)` - validates logical name at runtime

### Collation

Encapsulates locale and comparison settings ([`Collation.cs`](../MarkMpn.Sql4Cds.Engine/Collation.cs)).

```csharp
class Collation
{
    internal int LCID { get; }
    internal SqlCompareOptions CompareOptions { get; }
    public string Name { get; }

    internal SqlString ToSqlString(string value);
    public static bool TryParse(string name, out Collation collation);
}
```

The `ToSqlString` method ([`Collation.cs:331-334`](../MarkMpn.Sql4Cds.Engine/Collation.cs#L331-L334)) wraps .NET strings with collation information so that SQL Server's `SqlString` respects the collation during comparisons.

### SqlDateTypes

Custom date/time types not provided by BCL ([`SqlDateTypes.cs`](../MarkMpn.Sql4Cds.Engine/SqlDateTypes.cs)).

```csharp
struct SqlSmallDateTime : INullable, IComparable  // Lines 13-183
struct SqlDate : INullable, IComparable           // Lines 185-294
struct SqlTime : INullable, IComparable           // Lines 296-454
struct SqlDateTime2 : INullable, IComparable      // Lines 456-645
struct SqlDateTimeOffset : INullable, IComparable // Lines 647-825
```

| Type | Storage | Scale | Range |
|------|---------|-------|-------|
| SqlSmallDateTime | DateTime | N/A | 1900-01-01 to 2079-06-06 |
| SqlDate | DateTime | N/A | .NET DateTime range |
| SqlTime | TimeSpan | 0-7 | 00:00:00 to 23:59:59.9999999 |
| SqlDateTime2 | DateTime | 0-7 | .NET DateTime range |
| SqlDateTimeOffset | DateTimeOffset | 0-7 | With timezone |

All types implement:
- Null singleton (`Null` static field)
- Comparison operators with null propagation
- Implicit conversions between date/time types
- Scale conversion methods where applicable

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| NotSupportedQueryFragmentException | Invalid conversion at compile time | Rewrite query with explicit CAST |
| QueryExecutionException | Conversion failure at runtime | Check input data, use TRY_CONVERT |
| FormatException (wrapped) | String parsing failure | Validate input format |
| OverflowException (wrapped) | Numeric overflow | Use larger target type |
| SqlTruncateException (wrapped) | Precision/scale violation | Increase precision |

### Recovery Strategies

- **Compile-time errors**: Caught during query parsing, report with SQL line number
- **Runtime conversion errors**: Wrapped in QueryExecutionException with context-specific message
- **Collation conflicts**: Resolve using explicit COLLATE clause

### Error Message Patterns

The type converter generates context-specific error messages ([`SqlTypeConverter.cs:1385-1396`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/SqlTypeConverter.cs#L1385-L1396)):

| Source Type | Target Type | Error Factory |
|-------------|-------------|---------------|
| String | UniqueIdentifier/EntityReference | `Sql4CdsError.GuidParseError()` |
| String | DateTime types | `Sql4CdsError.DateTimeParseError()` |
| String | Money | `Sql4CdsError.MoneyParseError()` |
| Any | Any (overflow) | `Sql4CdsError.ArithmeticOverflow()` |

---

## Design Decisions

### Why Custom Date/Time Types?

**Context:** .NET's `System.Data.SqlTypes` namespace doesn't include SqlDate, SqlTime, SqlDateTime2, or SqlDateTimeOffset—types added to SQL Server after the original BCL implementation.

**Decision:** Implement custom structs that mirror SQL Server semantics, including scale support and proper null handling.

**Test results:**
| Scenario | Result |
|----------|--------|
| datetime2(3) precision | Correctly rounds to milliseconds |
| time(0) precision | Correctly rounds to seconds |
| smalldatetime rounding | Rounds to nearest minute per SQL Server spec |

**Alternatives considered:**
- Use .NET DateTime directly: Rejected - No scale support, wrong null semantics
- External library: Rejected - No suitable library found with full SQL Server compatibility

**Consequences:**
- Positive: Full SQL Server compatibility for date/time operations
- Positive: Proper scale handling for datetime2/time/datetimeoffset
- Negative: Additional code to maintain (500+ lines)

### Why INullable Interface for All Types?

**Context:** SQL has three-valued logic (true/false/null) while .NET has two-valued (true/false).

**Decision:** All SQL type implementations implement `INullable` from `System.Data.SqlTypes`, ensuring consistent null handling.

**Alternatives considered:**
- Nullable<T> wrappers: Rejected - Doesn't provide SQL comparison semantics
- Custom null sentinel values: Rejected - Error-prone, inconsistent

**Consequences:**
- Positive: Consistent null propagation through all operations
- Positive: Comparison operators naturally return SqlBoolean
- Negative: More verbose code (must check IsNull before accessing Value)

### Why Collation-Aware SqlString?

**Context:** String comparisons in SQL Server respect collation settings (case sensitivity, accent sensitivity, etc.).

**Decision:** Wrap all string values using `Collation.ToSqlString()` which creates `SqlString` instances with proper LCID and CompareOptions.

**Implementation:**
```csharp
// SqlTypeConverter.cs:114 - String conversion with collation
AddNullableTypeConversion<SqlString, string>(
    (ds, v, dt) => ((SqlDataTypeReferenceWithCollation)dt).Collation.ToSqlString(v),
    v => v.Value);
```

**Consequences:**
- Positive: String comparisons respect collation automatically
- Positive: `SqlString.CompareTo()` uses proper locale-aware comparison
- Negative: Slightly more complex string handling

### Why SqlEntityReference as Custom Type?

**Context:** Dataverse entity references contain more than just a GUID—they include logical name and optional partition ID for elastic tables.

**Decision:** Implement `SqlEntityReference` struct that wraps `SqlGuid` but adds logical name validation and partition support.

**Type System Integration:**
- Typed: `EntityReference(account)` validates logical name at runtime
- Untyped: `EntityReference` accepts any logical name
- Implicit conversion to/from `SqlGuid` for GUID operations

**Consequences:**
- Positive: Type-safe entity references with validation
- Positive: Seamless integration with Dataverse API (EntityReference class)
- Positive: Partition ID support for elastic tables
- Negative: Custom type not in standard SQL Server

### Why Expression-Based Conversion?

**Context:** Type conversions happen millions of times during query execution—they must be fast.

**Decision:** `SqlTypeConverter.Convert()` generates LINQ Expression trees that are compiled to IL once, then cached and reused.

**Implementation:**
```csharp
// SqlTypeConverter.cs:2224-2303 - Cached conversion compilation
public static Func<object, ExpressionExecutionContext, object> GetConversion(
    Type sourceType, Type destType)
{
    var key = sourceType.FullName + " -> " + destType.FullName;
    return _conversions.GetOrAdd(key, _ => CompileConversion(sourceType, destType));
}
```

**Consequences:**
- Positive: Near-native performance for repeated conversions
- Positive: Complex conversion logic compiled once
- Negative: Initial compilation overhead (amortized over query execution)

---

## Extension Points

### Adding a New SQL Type

1. **Create struct** implementing `INullable` and `IComparable`:
```csharp
struct SqlCustomType : INullable, IComparable
{
    public bool IsNull => _value == null;
    public static readonly SqlCustomType Null = new SqlCustomType(null);

    public int CompareTo(object obj) { /* null-safe comparison */ }
}
```

2. **Register null value** in `SqlTypeConverter._nullValues` ([`SqlTypeConverter.cs:66-89`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/SqlTypeConverter.cs#L66-L89))

3. **Add type mappings** using `AddTypeConversion` or `AddNullableTypeConversion` ([`SqlTypeConverter.cs:103-127`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/SqlTypeConverter.cs#L103-L127))

4. **Update DataTypeHelpers** with factory method and classification

### Adding a New Collation

1. **Add entry** to `CollationNameToLCID.txt` embedded resource
2. Collation parsing automatically handles CI/CS/AI/AS/KS/WS flags
3. Use `Collation.GetAllCollations()` to generate variant combinations

---

## Configuration

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| throwOnTruncate | bool | false | Throw error vs. silently truncate on string overflow |
| style | int | 0 | DateTime conversion style (0-131) |

DateTime conversion styles follow SQL Server's CONVERT function specification, including Hijri calendar support (styles 130, 131).

---

## Testing

### Acceptance Criteria

- [ ] Type precedence matches SQL Server for all supported types
- [ ] Implicit conversions succeed where SQL Server allows
- [ ] Explicit conversions fail at compile time where SQL Server disallows
- [ ] Collation-aware string comparisons respect case/accent sensitivity
- [ ] Null propagation works correctly through all operators
- [ ] Date/time scale correctly rounds fractional seconds

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| Null + any value | `NULL + 5` | `NULL` |
| String to int overflow | `CAST('999999999999' AS int)` | Overflow error |
| DateTime2 scale truncation | `CAST('12:30:45.1234567' AS time(3))` | `12:30:45.123` |
| SmallDateTime rounding | `1900-01-01 12:30:29` | `1900-01-01 12:30:00` |
| SmallDateTime rounding | `1900-01-01 12:30:30` | `1900-01-01 12:31:00` |
| Collation conflict | `col1 COLLATE A = col2 COLLATE B` | Error |
| EntityReference type check | `CAST(guid AS EntityReference(account))` with contact GUID | Runtime error |

### Test Examples

```csharp
[TestMethod]
public void ImplicitConversion_IntToDecimal()
{
    // Arrange
    var intType = DataTypeHelpers.Int;
    var decimalType = DataTypeHelpers.Decimal(18, 2);

    // Act
    var canConvert = SqlTypeConverter.CanChangeTypeImplicit(intType, decimalType);

    // Assert
    Assert.IsTrue(canConvert);
}

[TestMethod]
public void CollationComparison_CaseInsensitive()
{
    // Arrange
    var collation = Collation.USEnglish; // CI_AS
    var str1 = collation.ToSqlString("Hello");
    var str2 = collation.ToSqlString("HELLO");

    // Act
    var result = str1 == str2;

    // Assert
    Assert.IsTrue(result.Value);
}
```

---

## Related Specs

- [architecture.md](./architecture.md) - Overall system architecture, Context Pattern
- [execution-plan-nodes.md](./execution-plan-nodes.md) - How nodes use type system for schema
- [expression-evaluation.md](./expression-evaluation.md) - Expression compilation using types
- [query-compilation.md](./query-compilation.md) - Type inference during compilation

---

## Roadmap

- Support for hierarchyid type if Dataverse adds support
- Additional datetime format styles
- Performance optimization for high-volume type conversions
- Extended collation support for international deployments
