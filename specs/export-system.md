# Export System

**Status:** Implemented
**Version:** 1.0
**Last Updated:** 2026-01-26
**Code:** [MarkMpn.Sql4Cds.Export/](../MarkMpn.Sql4Cds.Export/)

---

## Overview

The Export System provides a flexible data export library for saving query results from a `DbDataReader` to files in multiple formats. It is a fork of classes from Microsoft SQLToolsService, extended with SQL4CDS-specific enhancements like hyperlink support for `SqlEntityReference` values in Excel and Markdown exports.

### Goals

- **Multi-format Export**: Support CSV, JSON, XML, Excel (XLSX), and Markdown table formats
- **Streaming Architecture**: Process large result sets without loading entire data into memory
- **Configurable Output**: Per-format options for headers, encoding, formatting, and styling
- **Type-aware Formatting**: Proper handling of SQL types, dates, decimals, and binary data

### Non-Goals

- Import/parsing of data from external files (read operations are only for internal service buffer)
- Real-time streaming to multiple consumers
- Modification of source data during export

---

## Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                        Export Request                              │
│  SaveResultsAs[Format]RequestParams                               │
└───────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│                     IFileStreamFactory                            │
│  SaveAs[Format]FileStreamFactory                                  │
└───────────────────────────────────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
        ┌─────────────────┐       ┌─────────────────┐
        │ IFileStreamWriter│       │ IFileStreamReader│
        │ (export formats) │       │ (service buffer) │
        └─────────────────┘       └─────────────────┘
                    │
        ┌───────────┼───────────┬───────────┬───────────┐
        ▼           ▼           ▼           ▼           ▼
    ┌───────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌──────────┐
    │  CSV  │  │  JSON  │  │  XML   │  │ Excel  │  │ Markdown │
    └───────┘  └────────┘  └────────┘  └────────┘  └──────────┘
```

The export system uses the **Factory Pattern** to create format-specific writers and the **Strategy Pattern** for format implementations sharing a common interface.

### Components

| Component | Responsibility |
|-----------|----------------|
| `IFileStreamFactory` | Creates readers/writers for a specific format |
| `IFileStreamWriter` | Writes formatted data rows to output files |
| `IFileStreamReader` | Reads data from service buffer (internal format) |
| `SaveAsStreamWriter` | Base class with shared writer logic |
| `DbCellValue` | Cell data with display and raw values |
| `DbColumnWrapper` | Column metadata with SQL type information |
| `ValueFormatter` | Type-aware value formatting utility |

### Dependencies

- Depends on: [type-system.md](./type-system.md) for `SqlEntityReference` handling
- Uses patterns from: [architecture.md](./architecture.md)

---

## Specification

### Core Requirements

1. All export formats must implement `IFileStreamWriter` and provide a corresponding `IFileStreamFactory`
2. Writers must support partial export via row/column range selection
3. NULL values must be represented appropriately per format
4. Writers must properly dispose resources and flush buffers

### Primary Flows

**Export to File:**

1. **Create Factory**: Instantiate format-specific factory with configuration parameters
2. **Get Writer**: Call `factory.GetWriter(filePath, columns)` to create the writer
3. **Write Rows**: For each row, call `writer.WriteRow(row, columns)`
4. **Dispose**: Writer flushes buffer and closes file on disposal

**Two-Stage Export (via Service Buffer):**

1. **Cache Results**: Write query results to service buffer using `ServiceBufferFileStreamWriter`
2. **Re-read Data**: Use `ServiceBufferFileStreamReader` to read cached rows
3. **Export**: Write to final format using format-specific writer

### Constraints

- Excel limits: 16,384 columns, 1,048,576 rows
- CSV fields must escape delimiters, newlines, and quote characters
- XML element names derived from column names (must be valid XML identifiers)
- Markdown tables require escaping pipe characters and HTML entities

### Validation Rules

| Field | Rule | Error |
|-------|------|-------|
| FilePath | Must be non-null, valid path | ArgumentNullException |
| Columns | Must be non-null list | ArgumentNullException |
| ColumnStartIndex | Must be valid index when IsSaveSelection | IndexOutOfRangeException |

---

## Core Types

### IFileStreamFactory

Factory interface for creating format-agnostic readers and writers ([`IFileStreamFactory.cs:14-22`](../MarkMpn.Sql4Cds.Export/DataStorage/IFileStreamFactory.cs#L14-L22)).

```csharp
public interface IFileStreamFactory
{
    IFileStreamReader GetReader(string fileName);
    IFileStreamWriter GetWriter(string fileName, IReadOnlyList<DbColumnWrapper> columns = null);
    void DisposeFile(string fileName);
}
```

### IFileStreamWriter

Writer interface for outputting formatted data ([`IFileStreamWriter.cs:17-23`](../MarkMpn.Sql4Cds.Export/DataStorage/IFileStreamWriter.cs#L17-L23)).

```csharp
public interface IFileStreamWriter : IDisposable
{
    int WriteRow(StorageDataReader dataReader);
    void WriteRow(IList<DbCellValue> row, IReadOnlyList<DbColumnWrapper> columns);
    void Seek(long offset);
    void FlushBuffer();
}
```

### SaveAsStreamWriter

Abstract base class providing common functionality for all SaveAs writers ([`SaveAsWriterBase.cs:19-179`](../MarkMpn.Sql4Cds.Export/DataStorage/SaveAsWriterBase.cs#L19-L179)). Handles column range selection, encoding parsing, and resource disposal.

### DbCellValue

Cell data contract with display and raw values ([`DbCellValue.cs:13-55`](../MarkMpn.Sql4Cds.Export/Contracts/DbCellValue.cs#L13-L55)).

```csharp
public class DbCellValue
{
    public string DisplayValue { get; set; }
    public bool IsNull { get; set; }
    public string InvariantCultureDisplayValue { get; set; }
    public object RawObject { get; set; }
    public long RowId { get; set; }
}
```

### DbColumnWrapper

Extended column metadata wrapper ([`DbColumnWrapper.cs:21-325`](../MarkMpn.Sql4Cds.Export/Contracts/DbColumnWrapper.cs#L21-L325)).

```csharp
public class DbColumnWrapper : DbColumn
{
    public SqlDbType SqlDbType { get; private set; }
    public bool IsBytes { get; private set; }
    public bool IsChars { get; private set; }
    public bool IsSqlVariant { get; private set; }
    public bool IsXml { get; set; }
    public bool IsUpdatable { get; }
}
```

---

## Error Handling

### Error Types

| Error | Condition | Recovery |
|-------|-----------|----------|
| `ArgumentNullException` | Null stream or columns parameter | Validate inputs before calling |
| `InvalidOperationException` | Calling deprecated WriteRow overload | Use cell-based WriteRow instead |
| `IOException` | File system write failure | Check file permissions/disk space |
| `EncoderFallbackException` | Invalid encoding name | Falls back to UTF-8 |

### Recovery Strategies

- **Encoding Failures**: `ParseEncoding` returns fallback encoding (UTF-8) when requested encoding is invalid
- **Write Failures**: Exceptions propagate to caller for handling
- **Disposal**: Double-dispose is safe (tracked via `disposed` flag)

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Empty result set | Valid file with headers only (if requested) |
| NULL cell value | Format-specific NULL representation ("NULL" for CSV, JSON null, etc.) |
| Binary data | Hexadecimal string prefixed with "0x" |
| Date before 1900-03-01 | Excel stores as string (epoch limitation) |

---

## Design Decisions

### Why Factory Pattern for Format Selection?

**Context:** Need to support multiple export formats with a unified interface while allowing format-specific configuration.

**Decision:** Each format has a dedicated factory class that creates appropriately configured readers/writers.

**Alternatives considered:**
- Single factory with format enum: Rejected - would require switch statements and limit extensibility
- Direct writer instantiation: Rejected - couples consumers to specific implementations

**Consequences:**
- Positive: Adding new formats requires only new factory/writer classes
- Positive: Format-specific options encapsulated in request params
- Negative: More classes to maintain

### Why Two-Stage Export via Service Buffer?

**Context:** Query results may be exported to multiple formats or re-exported after user selection changes.

**Decision:** Cache query results in a compact binary service buffer, then read and re-export as needed.

**Alternatives considered:**
- Re-execute query for each export: Rejected - expensive for complex queries
- Hold results in memory: Rejected - large result sets would exhaust memory

**Consequences:**
- Positive: Single query execution, multiple export formats
- Positive: Supports row/column range selection without re-query
- Negative: Temporary disk space usage for service buffer

### Why SkiaSharp for Excel Column Width Measurement?

**Context:** Excel supports auto-sizing columns, requiring accurate text width measurement.

**Decision:** Use SkiaSharp (`SKPaint.MeasureText`) for cross-platform font measurement.

**Alternatives considered:**
- GDI+ (System.Drawing): Rejected - Windows-only, not cross-platform compatible
- Fixed column widths: Rejected - poor user experience

**Consequences:**
- Positive: Works on Windows, Linux, macOS
- Positive: Accurate measurements using actual font metrics
- Negative: Additional NuGet dependency

### Why Inline Strings for Excel Instead of Shared String Table?

**Context:** ECMA-376 (XLSX format) allows two string storage methods: inline strings or shared string table.

**Decision:** Use inline strings directly in cells.

**Alternatives considered:**
- Shared string table: Rejected - adds complexity for deduplication with minimal benefit for typical exports

**Consequences:**
- Positive: Simpler implementation
- Positive: Single-pass writing without string table management
- Negative: Potentially larger file size for highly repetitive data

---

## Extension Points

### Adding a New Export Format

1. **Create Request Params**: Add `SaveResultsAs{Format}RequestParams` class extending `SaveResultsRequestParams`
2. **Implement Writer**: Create `SaveAs{Format}FileStreamWriter` extending `SaveAsStreamWriter`
3. **Implement Factory**: Create `SaveAs{Format}FileStreamFactory` implementing `IFileStreamFactory`
4. **Register Handler**: Add format-specific handler method in consuming code

**Example skeleton:**

```csharp
public class SaveAsNewFormatFileStreamWriter : SaveAsStreamWriter
{
    public SaveAsNewFormatFileStreamWriter(Stream stream,
        SaveResultsAsNewFormatRequestParams requestParams,
        IReadOnlyList<DbColumnWrapper> columns)
        : base(stream, requestParams, columns)
    {
        // Format-specific initialization
    }

    public override void WriteRow(IList<DbCellValue> row,
        IReadOnlyList<DbColumnWrapper> columns)
    {
        // Write row in new format
    }
}
```

---

## Configuration

### CSV Export Options

| Setting | Type | Required | Default | Description |
|---------|------|----------|---------|-------------|
| IncludeHeaders | bool | No | false | Include column names as first row |
| Delimiter | string | No | "," | Field separator character |
| LineSeperator | string | No | Environment.NewLine | Row terminator (CR, LF, or CRLF) |
| TextIdentifier | string | No | "\"" | Quote character for escaping |
| Encoding | string | No | "utf-8" | File encoding name or codepage |

### Excel Export Options

| Setting | Type | Required | Default | Description |
|---------|------|----------|---------|-------------|
| IncludeHeaders | bool | No | false | Include column header row |
| FreezeHeaderRow | bool | No | false | Freeze top row for scrolling |
| BoldHeaderRow | bool | No | false | Bold formatting for headers |
| AutoFilterHeaderRow | bool | No | false | Enable auto-filter dropdowns |
| AutoSizeColumns | bool | No | false | Auto-fit column widths to content |

### XML Export Options

| Setting | Type | Required | Default | Description |
|---------|------|----------|---------|-------------|
| Formatted | bool | No | false | Pretty-print with indentation |
| Encoding | string | No | "utf-8" | File encoding name or codepage |

### Markdown Export Options

| Setting | Type | Required | Default | Description |
|---------|------|----------|---------|-------------|
| IncludeHeaders | bool | No | true | Include column header row |
| LineSeparator | string | No | Environment.NewLine | Row terminator |
| Encoding | string | No | "utf-8" | File encoding name or codepage |

---

## Testing

### Acceptance Criteria

- [ ] CSV export produces valid CSV with proper escaping of delimiters and quotes
- [ ] JSON export produces valid JSON array of objects
- [ ] XML export produces well-formed XML document
- [ ] Excel export produces valid XLSX readable by Excel
- [ ] Markdown export produces valid GitHub-flavored Markdown table
- [ ] All formats handle NULL values appropriately
- [ ] Binary data exported as hexadecimal strings
- [ ] Date/time values formatted according to SQL type
- [ ] Column range selection works across all formats
- [ ] SqlEntityReference values create clickable hyperlinks in Excel/Markdown

### Edge Cases

| Scenario | Input | Expected Output |
|----------|-------|-----------------|
| CSV with embedded delimiter | "Hello, World" | "\"Hello, World\"" |
| CSV with embedded quotes | Say "Hi" | "Say \"\"Hi\"\"" |
| Markdown with pipe character | A\|B | A\\\|B |
| Excel date before epoch | 1899-01-01 | String value (not date) |
| Empty string vs NULL | "" vs null | Empty vs "NULL" |

### Test Examples

```csharp
[Fact]
public void CsvWriter_Should_EscapeDelimiters()
{
    var writer = new SaveAsCsvFileStreamWriter(stream, params, columns);
    var encoded = writer.EncodeCsvField("Hello, World");
    Assert.Equal("\"Hello, World\"", encoded);
}

[Fact]
public void ExcelWriter_Should_CreateValidXlsx()
{
    using var stream = new MemoryStream();
    using var writer = new SaveAsExcelFileStreamWriter(stream, params, columns, null);
    writer.WriteRow(row, columns);
    writer.Dispose();

    stream.Position = 0;
    using var zip = new ZipArchive(stream, ZipArchiveMode.Read);
    Assert.NotNull(zip.GetEntry("xl/worksheets/sheet1.xml"));
}
```

---

## Related Specs

- [type-system.md](./type-system.md) - SqlEntityReference type used for hyperlink generation
- [ado-net-provider.md](./ado-net-provider.md) - DbDataReader source for export data
- [language-server.md](./language-server.md) - LSP protocol for export commands

---

## Roadmap

- JSON export configuration options (TODO in codebase)
- Additional export formats (Parquet, Arrow)
- Streaming export for very large result sets
- Custom value formatters per column
