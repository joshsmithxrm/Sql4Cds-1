# FetchXML Translation

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-26
**Code:** [MarkMpn.Sql4Cds.Engine/](../MarkMpn.Sql4Cds.Engine/)

---

## Overview

FetchXML Translation is the bi-directional conversion system between T-SQL and Dataverse's native FetchXML query language. This system enables SQL4CDS to translate SQL queries into efficient FetchXML for server-side execution and convert FetchXML back to SQL for debugging and interoperability.

### Goals

- **SQL to FetchXML**: Convert SELECT queries into optimized FetchXML for Dataverse execution
- **FetchXML to SQL**: Convert existing FetchXML queries back to readable T-SQL
- **Operator Fidelity**: Support 80+ FetchXML-specific operators via SQL function syntax
- **Schema Preservation**: Maintain type information and aliases across translation

### Non-Goals

- Full SQL Server syntax support (some constructs have no FetchXML equivalent)
- Direct FetchXML editing UI (handled by host applications)
- FetchXML validation against Dataverse schema (handled at execution time)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SQL → FetchXML Flow                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────┐    ┌────────────────────┐    ┌──────────────────┐   │
│  │ ExecutionPlan  │───▶│  FetchXmlScan      │───▶│  FetchXml.       │   │
│  │ Builder        │    │  Node Creation     │    │  FetchType       │   │
│  └────────────────┘    └────────────────────┘    └──────────────────┘   │
│         │                      │                          │              │
│         ▼                      ▼                          ▼              │
│  ┌────────────────┐    ┌────────────────────┐    ┌──────────────────┐   │
│  │ WHERE to       │───▶│  JOIN to           │───▶│  XmlSerializer   │   │
│  │ <filter>       │    │  <link-entity>     │    │  (Serialize)     │   │
│  └────────────────┘    └────────────────────┘    └──────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         FetchXML → SQL Flow                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────┐    ┌────────────────────┐    ┌──────────────────┐   │
│  │ FetchXml2Sql   │───▶│  Element           │───▶│ TSqlFragment     │   │
│  │ Static Class   │    │  Conversion        │    │ AST              │   │
│  └────────────────┘    └────────────────────┘    └──────────────────┘   │
│         │                      │                          │              │
│         ▼                      ▼                          ▼              │
│  ┌────────────────┐    ┌────────────────────┐    ┌──────────────────┐   │
│  │ XmlSerializer  │───▶│  Visitor Pattern   │───▶│ Sql160Script     │   │
│  │ (Deserialize)  │    │  (Simplify/Quote)  │    │ Generator        │   │
│  └────────────────┘    └────────────────────┘    └──────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Responsibility |
|-----------|----------------|
| **FetchXml2Sql** | Static converter: FetchXML string → T-SQL string |
| **FetchXmlScan** | Execution plan node: executes FetchXML against Dataverse |
| **FetchType** | Auto-generated schema classes from FetchXml.xsd |
| **FetchXmlExtensions** | Helper methods for navigating FetchXML object model |
| **FetchXmlConditionMethods** | SQL functions for FetchXML-specific operators |
| **BaseDataNode** | Translates SQL WHERE to FetchXML filter/condition |

### Dependencies

- Depends on: [type-system.md](./type-system.md) for value formatting
- Depends on: [execution-plan-nodes.md](./execution-plan-nodes.md) for FetchXmlScan node
- Uses patterns from: [architecture.md](./architecture.md)

---

## Specification

### Core Requirements

1. Parse FetchXML using .NET XmlSerializer with auto-generated schema classes
2. Convert FetchXML elements to equivalent T-SQL constructs
3. Support all 85+ FetchXML condition operators
4. Preserve entity aliases and column mappings across translation
5. Handle special join types (exists, in, matchfirstrowusingcrossapply)

### SQL to FetchXML Flow

**Entry Point:** `ExecutionPlanBuilder.ConvertSelectQuerySpec()`

1. **FROM Clause**: Create `FetchXmlScan` node with `<entity>` element
2. **JOIN Clauses**: Fold joins as `<link-entity>` elements via `FoldableJoinNode.FoldFetchXmlJoin()`
3. **WHERE Clause**: Translate predicates to `<filter>` and `<condition>` via `TranslateFetchXMLCriteria()`
4. **SELECT Clause**: Add `<attribute>` elements for projected columns
5. **ORDER BY**: Add `<order>` elements with ascending/descending flags
6. **TOP/OFFSET**: Set `top`, `count`, `page` attributes on `<fetch>` root

### FetchXML to SQL Flow

**Entry Point:** `FetchXml2Sql.Convert()`

1. **Parse**: Deserialize FetchXML string to `FetchType` object
2. **SELECT**: Build column list from `<attribute>` and `<all-attributes>` elements
3. **FROM**: Build table reference from `<entity>` name attribute
4. **JOIN**: Recursively build JOINs from `<link-entity>` elements
5. **WHERE**: Convert `<filter>` and `<condition>` to boolean expressions
6. **ORDER BY**: Process `<order>` elements
7. **Simplify**: Apply visitor patterns to clean up identifiers
8. **Generate**: Output SQL via `Sql160ScriptGenerator`

### Constraints

- Maximum 10 link-entities (15 in Dataverse version 9.2.22043+)
- Aggregate queries limited to 5,000 records before paging required
- FULL OUTER JOIN not supported in FetchXML
- Virtual/Elastic tables cannot participate in joins
- Cross-datasource joins not supported (e.g., archive + regular data)

### Element Mapping

| FetchXML Element | SQL Construct |
|------------------|---------------|
| `<fetch>` | SELECT statement root |
| `<entity name="...">` | FROM table |
| `<attribute name="...">` | SELECT column |
| `<all-attributes />` | SELECT table.* |
| `<filter type="and">` | WHERE ... AND ... |
| `<filter type="or">` | WHERE ... OR ... |
| `<condition>` | WHERE comparison |
| `<link-entity linktype="inner">` | INNER JOIN |
| `<link-entity linktype="outer">` | LEFT OUTER JOIN |
| `<link-entity linktype="exists">` | WHERE EXISTS (subquery) |
| `<link-entity linktype="in">` | WHERE column IN (subquery) |
| `<order>` | ORDER BY |
| `count` attribute | OFFSET/FETCH (paging) |
| `distinct` attribute | SELECT DISTINCT |
| `aggregate` attribute | Aggregate function |
| `groupby` attribute | GROUP BY |

---

## Core Types

### FetchType

Root element representing a complete FetchXML query. Auto-generated from `FetchXml.xsd` using xsd.exe.

```csharp
[XmlRootAttribute("fetch", Namespace = "", IsNullable = false)]
public partial class FetchType
{
    public object[] Items { get; set; }  // entity, order elements
    public string version { get; set; }
    public string count { get; set; }
    public string page { get; set; }
}
```

The implementation ([`FetchXml.cs:1441`](../MarkMpn.Sql4Cds.Engine/FetchXml.cs#L1441)) contains 30+ attributes controlling query behavior including `aggregate`, `distinct`, `top`, `nolock`, `returntotalrecordcount`, and `utcoffset`.

### FetchEntityType

Represents the primary `<entity>` element in a FetchXML query.

```csharp
public partial class FetchEntityType
{
    public object[] Items { get; set; }  // attributes, filters, link-entities, orders
    public string name { get; set; }      // Entity logical name
    public string enableprefiltering { get; set; }
}
```

The `Items` array ([`FetchXml.cs:1046-1050`](../MarkMpn.Sql4Cds.Engine/FetchXml.cs#L1046-L1050)) holds child elements of types: `allattributes`, `FetchAttributeType`, `filter`, `FetchLinkEntityType`, and `FetchOrderType`.

### FetchLinkEntityType

Represents JOIN relationships via `<link-entity>` elements.

```csharp
public partial class FetchLinkEntityType
{
    public string name { get; set; }      // Related entity logical name
    public string @from { get; set; }     // Foreign key attribute
    public string to { get; set; }        // Primary key attribute
    public string linktype { get; set; }  // inner, outer, exists, in, etc.
    public string alias { get; set; }
}
```

The `linktype` attribute ([`FetchXml.cs:493`](../MarkMpn.Sql4Cds.Engine/FetchXml.cs#L493)) controls join behavior with values: `inner`, `outer`, `exists`, `in`, `matchfirstrowusingcrossapply`, `any`, `not any`, `all`.

### condition

Represents a single filter condition within a `<filter>` element.

```csharp
public partial class condition
{
    public string attribute { get; set; }
    public @operator @operator { get; set; }
    public string value { get; set; }
    public string entityname { get; set; }
    public conditionValue[] Items { get; set; }  // For IN operators
}
```

The `@operator` enumeration ([`FetchXml.cs:1108-1424`](../MarkMpn.Sql4Cds.Engine/FetchXml.cs#L1108-L1424)) defines 85+ operators for comparisons, date ranges, user context, fiscal periods, and hierarchy queries.

### FetchXml2SqlOptions

Controls FetchXML to SQL conversion behavior.

```csharp
public class FetchXml2SqlOptions
{
    public FetchXmlOperatorConversion ConvertFetchXmlOperatorsTo { get; set; }
        = FetchXmlOperatorConversion.Functions;
    public bool UseParametersForLiterals { get; set; } = false;
    public bool ConvertDateTimeToUtc { get; set; }
}
```

The `FetchXmlOperatorConversion` enum ([`FetchXml2Sql.cs:2687`](../MarkMpn.Sql4Cds.Engine/FetchXml2Sql.cs#L2687)) offers three modes: `Functions` (SQL function syntax), `Literals` (inline values), and `SqlCalculations` (DATEADD/DATEPART expressions).

---

## Operator Support

### Comparison Operators

| FetchXML | SQL Equivalent | Notes |
|----------|----------------|-------|
| `eq` | `=` | Equality |
| `ne`, `neq` | `<>` | Not equal |
| `gt` | `>` | Greater than |
| `ge` | `>=` | Greater than or equal |
| `lt` | `<` | Less than |
| `le` | `<=` | Less than or equal |
| `null` | `IS NULL` | Null check |
| `not-null` | `IS NOT NULL` | Not null check |
| `like` | `LIKE` | Pattern matching |
| `not-like` | `NOT LIKE` | Negated pattern |
| `in` | `IN (...)` | List membership |
| `not-in` | `NOT IN (...)` | List exclusion |
| `between` | `BETWEEN ... AND ...` | Range check |
| `not-between` | `NOT BETWEEN ... AND ...` | Range exclusion |

### Date/Time Operators

SQL4CDS exposes FetchXML date operators as SQL functions via `FetchXmlConditionMethods` ([`FetchXmlConditionMethods.cs:15-226`](../MarkMpn.Sql4Cds.Engine/FetchXmlConditionMethods.cs#L15-L226)).

```sql
-- Relative date operators
WHERE createdon = yesterday()
WHERE createdon = today()
WHERE createdon = tomorrow()
WHERE createdon = lastsevendays()
WHERE createdon = thisweek()
WHERE createdon = lastmonth()
WHERE createdon = thisyear()

-- Parameterized date operators
WHERE createdon = lastxdays(7)
WHERE createdon = nextxhours(24)
WHERE createdon = olderthanxmonths(6)
```

| Operator Category | Examples |
|-------------------|----------|
| **Fixed Periods** | `yesterday`, `today`, `tomorrow`, `lastweek`, `thisweek`, `nextweek`, `lastmonth`, `thismonth`, `nextmonth`, `lastyear`, `thisyear`, `nextyear` |
| **Relative Ranges** | `lastxhours(n)`, `nextxdays(n)`, `lastxweeks(n)`, `nextxmonths(n)`, `lastxyears(n)`, `nextxyears(n)` |
| **Older Than** | `olderthanxminutes(n)`, `olderthanxhours(n)`, `olderthanxdays(n)`, `olderthanxweeks(n)`, `olderthanxmonths(n)`, `olderthanxyears(n)` |
| **Specific Dates** | `on(date)`, `onorbefore(date)`, `onorafter(date)` |

### Fiscal Period Operators

```sql
-- Current fiscal periods
WHERE closedate = thisfiscalyear()
WHERE closedate = thisfiscalperiod()

-- Relative fiscal periods
WHERE closedate = lastxfiscalyears(2)
WHERE closedate = nextxfiscalperiods(3)

-- Specific fiscal periods
WHERE closedate = infiscalyear(2024)
WHERE closedate = infiscalperiodandyear(3, 2024)
```

Fiscal period operators ([`FetchXml2Sql.cs:1636-1808`](../MarkMpn.Sql4Cds.Engine/FetchXml2Sql.cs#L1636-L1808)) query the organization's fiscal calendar settings via `GetFiscalPeriodSettings()` to calculate date boundaries.

### User Context Operators

```sql
-- User identity
WHERE ownerid = equserid()
WHERE ownerid = neuserid()

-- Team membership
WHERE ownerid = equserteams()
WHERE ownerid = equseroruserteams()

-- Hierarchy
WHERE ownerid = equseroruserhierarchy()
WHERE ownerid = equseroruserhierarchyandteams()

-- Business unit
WHERE owningbusinessunit = eqbusinessid()
WHERE languagecode = equserlanguage()
```

### Hierarchy Operators

```sql
-- Descendants in hierarchy
WHERE accountid = under(@parentid)
WHERE accountid = eqorunder(@parentid)
WHERE accountid = notunder(@parentid)

-- Ancestors in hierarchy
WHERE accountid = above(@childid)
WHERE accountid = eqorabove(@childid)
```

Hierarchy operators ([`FetchXml2Sql.cs:1810-1921`](../MarkMpn.Sql4Cds.Engine/FetchXml2Sql.cs#L1810-L1921)) generate recursive CTEs when converted to SQL to traverse parent-child relationships.

### Text Pattern Operators

```sql
-- String patterns
WHERE name = beginswith('Contoso')
WHERE name = endswith('Inc')
WHERE name = notbeginwith('Test')

-- Multi-select picklist
WHERE tags = containvalues(1, 2, 3)
WHERE tags = notcontainvalues(4, 5)
```

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| `NotSupportedQueryFragmentException` | SQL construct has no FetchXML equivalent | Rewrite query or use client-side processing |
| `QueryExecutionException` | FetchXML execution failed | Check Dataverse error, retry if transient |
| `InvalidPagingException` | Aggregate query exceeds 5,000 records | Add WHERE filters to reduce result set |
| `XmlException` | Malformed FetchXML input | Validate FetchXML structure |

### Operator Restrictions

FetchXML condition methods throw `NotImplementedException` at runtime if they cannot be folded into FetchXML ([`FetchXmlConditionMethods.cs:221-224`](../MarkMpn.Sql4Cds.Engine/FetchXmlConditionMethods.cs#L221-L224)):

```csharp
private static SqlBoolean ThrowException()
{
    throw new NotImplementedException(
        "Custom FetchXML filter conditions must only be used where they can be folded into a FetchXML Scan operator");
}
```

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Empty FetchXML | Return empty SELECT statement |
| No attributes specified | Generate `SELECT *` equivalent |
| Null condition values | Convert to `IS NULL` check |
| Invalid operator for type | Throw validation exception at compile time |
| Exceeded link-entity limit | Fall back to client-side join |

---

## Design Decisions

### Why Bi-directional Translation?

**Context:** Users need to both execute SQL against Dataverse and understand/debug FetchXML queries.

**Decision:** Implement separate SQL→FetchXML (for execution) and FetchXML→SQL (for readability) paths.

**Alternatives considered:**
- Single direction only: Rejected - Limits debugging and interoperability
- AST sharing: Rejected - FetchXML and SQL have fundamentally different semantics

**Consequences:**
- Positive: Users can write SQL and view the generated FetchXML
- Positive: Users can convert existing FetchXML to SQL for editing
- Negative: Two codepaths to maintain for similar operations

### Why Auto-generated Schema Classes?

**Context:** FetchXML schema is complex with 30+ element types and 85+ operators.

**Decision:** Use xsd.exe to generate C# classes from FetchXml.xsd, then extend with partial classes.

**Implementation:** The schema classes ([`FetchXml.cs:1-2703`](../MarkMpn.Sql4Cds.Engine/FetchXml.cs#L1-L2703)) are auto-generated with `auto-generated` comment at line 2. Extensions are added in [`FetchXml.Extensions.cs:7-88`](../MarkMpn.Sql4Cds.Engine/FetchXml.Extensions.cs#L7-L88).

**Consequences:**
- Positive: Schema stays synchronized with Dataverse FetchXML definition
- Positive: Type-safe manipulation of FetchXML structure
- Negative: Must maintain partial classes for custom properties
- Negative: Auto-generated code style doesn't match project conventions

### Why SQL Function Syntax for FetchXML Operators?

**Context:** FetchXML has operators like `lastxdays` with no SQL equivalent.

**Decision:** Expose FetchXML operators as SQL functions that are recognized and folded into FetchXML at compile time.

**Usage:**
```sql
-- This SQL:
SELECT * FROM account WHERE createdon = lastxdays(7)

-- Becomes this FetchXML:
<condition attribute="createdon" operator="last-x-days" value="7" />
```

**Alternatives considered:**
- Special syntax (e.g., `createdon @lastxdays 7`): Rejected - Non-standard SQL
- Only support during optimization: Rejected - User must understand folding rules

**Consequences:**
- Positive: Natural SQL syntax for FetchXML-specific features
- Positive: Functions are discoverable via autocomplete
- Negative: Functions throw at runtime if not folded into FetchXML

### Why Three Conversion Modes for Operators?

**Context:** Different use cases need different SQL representations of FetchXML operators.

**Decision:** `FetchXml2SqlOptions.ConvertFetchXmlOperatorsTo` offers three modes:
- `Functions`: `createdon = lastxdays(7)` - Best for SQL4CDS re-execution
- `Literals`: `createdon >= '2024-01-19' AND createdon < '2024-01-26'` - Best for debugging
- `SqlCalculations`: `createdon >= DATEADD(day, -7, GETDATE())` - Best for SQL Server compatibility

**Consequences:**
- Positive: Users choose representation matching their use case
- Negative: Three code paths for date operator conversion

---

## Extension Points

### Adding a New FetchXML Operator

1. **Add to schema** in `FetchXml.xsd`:
```xml
<xs:enumeration value="new-operator" />
```

2. **Regenerate classes** using xsd.exe:
```bash
xsd.exe FetchXml.xsd /classes /namespace:MarkMpn.Sql4Cds.Engine.FetchXml
```

3. **Add SQL function** in `FetchXmlConditionMethods.cs`:
```csharp
[Description("Description of what the operator matches")]
public static SqlBoolean NewOperator(SqlDateTime field, SqlInt32 value)
    => ThrowException();
```

4. **Add conversion logic** in `FetchXml2Sql.cs` inside `GetCondition()`:
```csharp
case @operator.newoperator:
    return new BooleanComparisonExpression
    {
        ComparisonType = BooleanComparisonType.Equals,
        FirstExpression = field,
        SecondExpression = BuildNewOperatorExpression(value)
    };
```

5. **Add folding logic** in `BaseDataNode.TranslateFetchXMLCriteria()` to recognize the function call and convert to FetchXML condition.

### Customizing FetchXML Generation

Override `FoldQuery()` in execution plan nodes to customize how SQL constructs map to FetchXML:

```csharp
class CustomNode : BaseDataNode
{
    public override IDataExecutionPlanNodeInternal FoldQuery(
        NodeCompilationContext context, IList<OptimizerHint> hints)
    {
        // Custom folding logic
        return base.FoldQuery(context, hints);
    }
}
```

---

## Testing

### Acceptance Criteria

- [ ] Convert FetchXML with all standard element types to valid SQL
- [ ] Convert SQL SELECT to FetchXML that executes successfully
- [ ] Round-trip SQL → FetchXML → SQL produces semantically equivalent queries
- [ ] All 85+ FetchXML operators recognized and converted correctly
- [ ] Date operators respect user timezone settings
- [ ] Fiscal period operators use organization fiscal calendar

### Test Examples

```csharp
[TestMethod]
public void FetchXmlToSql_SimpleSelect()
{
    var fetchXml = @"
        <fetch>
            <entity name='account'>
                <attribute name='name' />
                <filter>
                    <condition attribute='statecode' operator='eq' value='0' />
                </filter>
            </entity>
        </fetch>";

    var sql = FetchXml2Sql.Convert(org, metadata, fetchXml, new FetchXml2SqlOptions());

    Assert.IsTrue(sql.Contains("SELECT"));
    Assert.IsTrue(sql.Contains("FROM account"));
    Assert.IsTrue(sql.Contains("statecode = 0"));
}

[TestMethod]
public void FetchXmlToSql_DateOperator_LastXDays()
{
    var fetchXml = @"
        <fetch>
            <entity name='contact'>
                <attribute name='fullname' />
                <filter>
                    <condition attribute='createdon' operator='last-x-days' value='7' />
                </filter>
            </entity>
        </fetch>";

    var options = new FetchXml2SqlOptions
    {
        ConvertFetchXmlOperatorsTo = FetchXmlOperatorConversion.Functions
    };
    var sql = FetchXml2Sql.Convert(org, metadata, fetchXml, options);

    Assert.IsTrue(sql.Contains("lastxdays(7)"));
}

[TestMethod]
public void SqlToFetchXml_JoinFolding()
{
    using var con = new Sql4CdsConnection(context.GetOrganizationService());
    using var cmd = con.CreateCommand();
    cmd.CommandText = @"
        SELECT a.name, c.fullname
        FROM account a
        INNER JOIN contact c ON a.accountid = c.parentcustomerid";

    var plan = ((Sql4CdsCommand)cmd).GeneratePlan(out _);
    var scan = FindNode<FetchXmlScan>(plan);

    // Verify JOIN was folded into link-entity
    Assert.IsTrue(scan.FetchXmlString.Contains("<link-entity"));
    Assert.IsTrue(scan.FetchXmlString.Contains("linktype=\"inner\""));
}
```

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| Empty filter | `<filter />` | No WHERE clause |
| Multiple link-entities | 5 nested joins | All converted to `<link-entity>` hierarchy |
| Aggregate with groupby | `GROUP BY name` | `<attribute groupby="true">` |
| EXISTS subquery | `WHERE EXISTS (SELECT ...)` | `<link-entity linktype="exists">` |

---

## Related Specs

- [architecture.md](./architecture.md) - Overall system architecture and patterns
- [type-system.md](./type-system.md) - SQL type conversions for FetchXML values
- [execution-plan-nodes.md](./execution-plan-nodes.md) - FetchXmlScan node details
- [query-compilation.md](./query-compilation.md) - How ExecutionPlanBuilder creates FetchXML nodes
- [expression-evaluation.md](./expression-evaluation.md) - FetchXML condition function evaluation

---

## Roadmap

- Support additional FetchXML operators as Dataverse adds them
- Improve fiscal period calculation performance with caching
- Add FetchXML validation against entity metadata at compile time
- Support FetchXML `options` attribute for Dataverse query hints
