# Metadata Caching

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-26
**Code:** [MarkMpn.Sql4Cds.Engine/](../MarkMpn.Sql4Cds.Engine/)

---

## Overview

The metadata caching system provides efficient access to Dataverse entity metadata, table size estimates, and SDK message definitions. It reduces API calls by caching entity schema, attribute definitions, relationship metadata, and record counts used throughout query compilation and optimization.

### Goals

- **Performance**: Minimize Dataverse API calls by caching metadata on first access
- **Non-Blocking Access**: Support background loading to prevent UI freezes during autocomplete
- **Query Optimization**: Provide accurate row count estimates for query plan cost analysis

### Non-Goals

- Real-time metadata synchronization (caches are per-session)
- Cross-session cache sharing (each connection maintains its own caches)
- Persistent caching (all caches are in-memory only)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DataSource                                │
│  ┌───────────────┬───────────────┬───────────────┬───────────┐ │
│  │ IOrganization │  IAttribute   │ ITableSize    │ IMessage  │ │
│  │   Service     │ MetadataCache │   Cache       │   Cache   │ │
│  └───────┬───────┴───────┬───────┴───────┬───────┴─────┬─────┘ │
└──────────┼───────────────┼───────────────┼─────────────┼───────┘
           │               │               │             │
           ▼               ▼               ▼             ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Dataverse API Calls                           │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────────────┐│
│  │RetrieveEntity  │ │RetrieveTotal   │ │RetrieveMultiple        ││
│  │Request         │ │RecordCount     │ │(sdkmessagerequest)     ││
│  └────────────────┘ └────────────────┘ └────────────────────────┘│
└──────────────────────────────────────────────────────────────────┘
```

The metadata caches flow through the system via `DataSource` → `SessionContext` → `NodeCompilationContext`, making metadata available during both query compilation and execution.

### Components

| Component | Responsibility |
|-----------|----------------|
| **IAttributeMetadataCache** | Entity schema, attributes, and relationships |
| **ITableSizeCache** | Record count estimates for query optimization |
| **IMessageCache** | SDK message definitions for stored procedure support |
| **DataSource** | Container holding all caches for a Dataverse instance |

### Dependencies

- Depends on: [architecture.md](./architecture.md) - Overall system layering
- Used by: [query-optimization.md](./query-optimization.md) - Row count estimates
- Used by: [query-compilation.md](./query-compilation.md) - Schema validation

---

## Specification

### Core Requirements

1. Cache entity metadata (attributes, relationships) on first access
2. Support non-blocking TryGetValue for autocomplete scenarios
3. Provide minimal metadata variant for faster initial loads
4. Cache table row counts for query cost estimation
5. Cache SDK message definitions for EXEC statement support

### Primary Flows

**Metadata Retrieval (Blocking):**

1. **Check Cache**: Look up entity in `_metadata` dictionary
2. **Cache Hit**: Return cached `EntityMetadata` immediately
3. **Cache Miss**: Execute `RetrieveEntityRequest` against Dataverse
4. **Store & Return**: Cache result and return to caller

**Metadata Retrieval (Non-Blocking):**

1. **Check Full Cache**: If available, return immediately
2. **Check Minimal Cache**: If available, return minimal data
3. **Start Background Load**: Queue `Task.Run()` to load metadata
4. **Return False**: Caller can retry later when data is ready

**Table Size Estimation:**

1. **Check Cache**: Look up in `_tableSize` dictionary
2. **Virtual Entity Check**: If `DataProviderId != null`, return 1000
3. **API Method (v9+)**: Try `RetrieveTotalRecordCountRequest`
4. **FetchXML Fallback**: Use aggregate query `COUNT(primaryid)`

### Constraints

- All caches use case-insensitive string comparison for entity names
- Failed metadata lookups cache the exception to avoid repeated failures
- Background loading uses `HashSet.Add()` atomicity to prevent duplicate loads

### Validation Rules

| Cache | Input | Validation | Error |
|-------|-------|------------|-------|
| AttributeMetadataCache | Entity name | Must be valid Dataverse entity | FaultException with cached exception |
| TableSizeCache | Entity name | Requires metadata lookup first | Propagates metadata errors |
| MessageCache | Message name | Case-insensitive lookup | Returns false from TryGetValue |

---

## Core Types

### IAttributeMetadataCache

The primary interface for entity metadata access ([`IAttributeMetadataCache.cs:13-111`](../MarkMpn.Sql4Cds.Engine/IAttributeMetadataCache.cs#L13-L111)).

```csharp
public interface IAttributeMetadataCache
{
    EntityMetadata this[string name] { get; }
    EntityMetadata this[int otc] { get; }
    bool TryGetValue(string logicalName, out EntityMetadata metadata);
    bool TryGetMinimalData(string logicalName, out EntityMetadata metadata);
    string[] RecycleBinEntities { get; }
    IEnumerable<EntityMetadata> GetAllEntities();
}
```

The indexers provide blocking retrieval that ensures metadata is available, while `TryGetValue` and `TryGetMinimalData` support non-blocking patterns for UI responsiveness.

### AttributeMetadataCache

Default implementation with background loading support ([`AttributeMetadataCache.cs:17-303`](../MarkMpn.Sql4Cds.Engine/AttributeMetadataCache.cs#L17-L303)).

```csharp
public class AttributeMetadataCache : IAttributeMetadataCache
{
    private readonly IDictionary<string, EntityMetadata> _metadata;
    private readonly IDictionary<string, EntityMetadata> _minimalMetadata;
    private readonly ISet<string> _loading;
    private readonly IDictionary<string, Exception> _invalidEntities;
}
```

### ITableSizeCache

Simple interface for row count lookup ([`ExecutionPlan/ITableSizeCache.cs:12-20`](../MarkMpn.Sql4Cds.Engine/ExecutionPlan/ITableSizeCache.cs#L12-L20)).

```csharp
public interface ITableSizeCache
{
    int this[string logicalName] { get; }
}
```

### TableSizeCache

Implementation with multi-tier count retrieval ([`TableSizeCache.cs:19-121`](../MarkMpn.Sql4Cds.Engine/TableSizeCache.cs#L19-L121)).

```csharp
public class TableSizeCache : ITableSizeCache
{
    private readonly IDictionary<string, int> _tableSize;
    private static readonly string[] _unreliableRetrieveTotalRecordCountEntities;
}
```

### IMessageCache

Interface for SDK message access ([`MessageCache.cs:17-41`](../MarkMpn.Sql4Cds.Engine/MessageCache.cs#L17-L41)).

```csharp
public interface IMessageCache
{
    bool TryGetValue(string name, out Message message);
    IEnumerable<Message> GetAllMessages(bool lazy);
    bool IsMessageAvailable(string entityLogicalName, string messageName);
}
```

### Usage Pattern

```csharp
// DataSource creates and wires caches together
var dataSource = new DataSource(orgService);
// Caches are auto-initialized: Metadata, TableSizeCache, MessageCache

// Blocking access during compilation
var accountMetadata = dataSource.Metadata["account"];

// Non-blocking for autocomplete
if (dataSource.Metadata.TryGetValue("contact", out var meta))
{
    // Use metadata immediately
}
else
{
    // Show loading indicator, retry later
}

// Row count for optimization
var rowEstimate = dataSource.TableSizeCache["account"];
```

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| FaultException | Invalid entity name | Cached in `_invalidEntities`, thrown on retry |
| RetrieveMetadataException | Network/permission error | Not cached, retry may succeed |
| ErrorCode -2147164125 | Table size aggregation exceeds batch limit | Returns conservative estimate of 50,000 |

### Recovery Strategies

- **Invalid entity**: Exception is cached to prevent repeated failed API calls
- **Network errors**: Not cached, subsequent requests may succeed
- **Unreliable count entities**: Uses FetchXML fallback instead of RetrieveTotalRecordCountRequest

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Entity doesn't exist | FaultException cached, rethrown on subsequent access |
| Virtual entity table size | Returns 1000 (avoids timeout on external data sources) |
| Recycle bin not enabled | `RecycleBinEntities` returns null (lazy-loaded) |
| OTC lookup not cached | Uses `RetrieveMetadataChangesRequest` to find entity |

---

## Design Decisions

### Why Lazy Background Loading?

**Context:** Autocomplete needs metadata for many entities but blocking the UI is unacceptable.

**Decision:** `TryGetValue` returns false immediately and starts a background `Task.Run()` to load metadata.

**Implementation ([`AttributeMetadataCache.cs:149-161`](../MarkMpn.Sql4Cds.Engine/AttributeMetadataCache.cs#L149-L161)):**
```csharp
if (_loading.Add(logicalName))
{
    var task = Task.Run(() => this[logicalName]);
    OnMetadataLoading(new MetadataLoadingEventArgs(logicalName, task));
}
return false;
```

**Alternatives considered:**
- Pre-load all metadata: Rejected - Too slow for large orgs (1000+ entities)
- Synchronous load: Rejected - Freezes UI during autocomplete
- External cache file: Rejected - Complexity, staleness issues

**Consequences:**
- Positive: Responsive UI during initial metadata loading
- Positive: MetadataLoading event allows hosts to show loading indicators
- Negative: First autocomplete may show incomplete results

### Why Minimal Metadata Variant?

**Context:** Full metadata includes all attributes, relationships, and option sets—often more than needed for autocomplete.

**Decision:** `TryGetMinimalData` retrieves only essential properties via `RetrieveMetadataChangesRequest`.

**Minimal properties ([`AttributeMetadataCache.cs:187-234`](../MarkMpn.Sql4Cds.Engine/AttributeMetadataCache.cs#L187-L234)):**
- Entity: LogicalName, DisplayName, Description, IsIntersect
- Attributes: LogicalName, AttributeOf, AttributeType, DisplayName, IsValidForCreate/Update/Read
- Relationships: ReferencedEntity/Attribute, ReferencingEntity/Attribute

**Consequences:**
- Positive: Faster initial loads for autocomplete
- Positive: Reduced API payload size
- Negative: Two cache tiers to manage (`_metadata` vs `_minimalMetadata`)

### Why Exception Caching?

**Context:** Invalid entity names would cause repeated API calls if not cached.

**Decision:** Store exceptions in `_invalidEntities` dictionary and rethrow on subsequent access ([`AttributeMetadataCache.cs:89-90`](../MarkMpn.Sql4Cds.Engine/AttributeMetadataCache.cs#L89-L90)).

**Consequences:**
- Positive: Failed lookups don't repeatedly hit the API
- Negative: Renamed entities require new connection to pick up changes

### Why Multi-Tier Table Size Retrieval?

**Context:** Different Dataverse versions and entity types require different approaches.

**Decision:** Three-tier strategy in `TableSizeCache` indexer ([`TableSizeCache.cs:65-119`](../MarkMpn.Sql4Cds.Engine/TableSizeCache.cs#L65-L119)):

| Tier | Condition | Method |
|------|-----------|--------|
| 1 | Virtual entity | Return 1000 (estimate) |
| 2 | CRM v9+ (non-unreliable) | `RetrieveTotalRecordCountRequest` |
| 3 | Fallback | FetchXML aggregate COUNT |

**Unreliable entities ([`TableSizeCache.cs:29-42`](../MarkMpn.Sql4Cds.Engine/TableSizeCache.cs#L29-L42)):**
```csharp
private static readonly string[] _unreliableRetrieveTotalRecordCountEntities = new[]
{
    "attribute", "attributeimageconfig", "commitment", "connector",
    "entity", "entityimageconfig", "entitykey", "entityrelationship",
    "environmentvariabledefinition", "environmentvariablevalue", "relationship"
};
```

**Consequences:**
- Positive: Most efficient method used for each scenario
- Positive: Handles edge cases (virtual entities, unreliable counts)
- Negative: Complexity of multi-tier fallback logic

### Why Thread-Safe Loading State?

**Context:** Multiple threads may request the same entity metadata simultaneously.

**Decision:** Use `HashSet<string>.Add()` atomicity to prevent duplicate background loads.

**Implementation ([`AttributeMetadataCache.cs:154`](../MarkMpn.Sql4Cds.Engine/AttributeMetadataCache.cs#L154)):**
```csharp
if (_loading.Add(logicalName))  // Atomic - returns false if already present
{
    var task = Task.Run(() => this[logicalName]);
}
```

**Consequences:**
- Positive: Lock-free thread safety for common case
- Positive: No duplicate API calls for same entity
- Negative: Potential race between full and minimal loads (benign)

---

## Extension Points

### Implementing Custom Metadata Cache

For scenarios requiring pre-loaded metadata or alternative sources:

1. **Implement IAttributeMetadataCache**:
```csharp
public class PreloadedMetadataCache : IAttributeMetadataCache
{
    private readonly Dictionary<string, EntityMetadata> _metadata;

    public EntityMetadata this[string name] => _metadata[name];
    public bool TryGetValue(string name, out EntityMetadata meta)
        => _metadata.TryGetValue(name, out meta);
}
```

2. **Pass to DataSource constructor**:
```csharp
var dataSource = new DataSource(
    org,
    customMetadataCache,
    new TableSizeCache(org, customMetadataCache),
    new MessageCache(org, customMetadataCache));
```

### XrmToolBox Integration

`SharedMetadataCache` ([`MarkMpn.Sql4Cds.XTB/SharedMetadataCache.cs`](../MarkMpn.Sql4Cds.XTB/SharedMetadataCache.cs)) wraps `AttributeMetadataCache` and falls back to XrmToolBox's global metadata cache when available.

### Language Server Integration

`CachedMetadata` ([`MarkMpn.Sql4Cds.LanguageServer/Connection/CachedMetadata.cs`](../MarkMpn.Sql4Cds.LanguageServer/Connection/CachedMetadata.cs)) wraps `AttributeMetadataCache` with persistent metadata from background tasks.

---

## Configuration

The metadata caching system has no explicit configuration—behavior is determined by Dataverse version detection and entity characteristics.

| Behavior | Condition | Automatic Detection |
|----------|-----------|---------------------|
| Use RetrieveTotalRecordCountRequest | CRM version ≥ 9.0 | Version queried on DataSource init |
| Virtual entity handling | `DataProviderId != null` | Checked in metadata |
| Recycle bin support | Entity exists in recyclebinconfig | Lazy-loaded on first access |

---

## Testing

### Acceptance Criteria

- [ ] Metadata retrieved on first access and cached for subsequent requests
- [ ] TryGetValue returns false and starts background load for uncached entities
- [ ] Invalid entity names throw and cache the exception
- [ ] Table size returns accurate counts for standard entities
- [ ] Virtual entities return 1000 estimate without timeout
- [ ] Messages loaded from sdkmessagerequest/sdkmessageresponse

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| Valid entity first access | "account" | EntityMetadata loaded and cached |
| Invalid entity | "notanentity" | FaultException thrown and cached |
| OTC lookup | 1 (Account OTC) | EntityMetadata via RetrieveMetadataChanges |
| Virtual entity size | External table | 1000 (estimate) |
| Unreliable count entity | "attribute" | FetchXML aggregate used instead |

### Test Examples

```csharp
[TestMethod]
public void MetadataCache_CachesOnFirstAccess()
{
    var context = new XrmFakedContext();
    context.InitializeMetadata(Assembly.GetExecutingAssembly());

    var cache = new AttributeMetadataCache(context.GetOrganizationService());

    // First access loads from service
    var meta1 = cache["account"];
    Assert.IsNotNull(meta1);

    // Second access returns cached value (no API call)
    var meta2 = cache["account"];
    Assert.AreSame(meta1, meta2);
}

[TestMethod]
public void TryGetValue_ReturnsFalseAndStartsBackgroundLoad()
{
    var context = new XrmFakedContext();
    var cache = new AttributeMetadataCache(context.GetOrganizationService());

    Task backgroundTask = null;
    cache.MetadataLoading += (s, e) => backgroundTask = e.Task;

    // First call returns false, starts background load
    var result = cache.TryGetValue("account", out var meta);
    Assert.IsFalse(result);
    Assert.IsNull(meta);
    Assert.IsNotNull(backgroundTask);

    // Wait for background load
    backgroundTask.Wait();

    // Now metadata is available
    result = cache.TryGetValue("account", out meta);
    Assert.IsTrue(result);
    Assert.IsNotNull(meta);
}
```

---

## Related Specs

- [architecture.md](./architecture.md) - System layering and DataSource role
- [query-optimization.md](./query-optimization.md) - Uses TableSizeCache for cost estimation
- [query-compilation.md](./query-compilation.md) - Uses metadata for schema validation
- [execution-plan-nodes.md](./execution-plan-nodes.md) - Nodes access metadata via NodeCompilationContext
- [ado-net-provider.md](./ado-net-provider.md) - Sql4CdsConnection initializes DataSource with caches

---

## Roadmap

- Persistent metadata cache for faster connection startup
- Cross-session cache sharing for multi-connection scenarios
- Incremental metadata refresh for long-running connections
- Extended minimal metadata for improved autocomplete coverage
