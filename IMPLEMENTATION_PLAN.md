# SQL4CDS Specification Generation Plan

**Repository:** SQL4CDS
**Created:** 2026-01-25
**Status:** In Progress

---

## Exploration Summary

- **Projects explored:** 13 C# projects + 1 TypeScript extension
- **Total C# files:** ~460
- **Interfaces identified:** 20 interface files
- **Subdirectories explored:** 40+
- **Systems requiring specs:** 11 (+ 1 optional)
- **Systems excluded:** 5 (with justification)

---

## Phase 1.2: Project Inventory Table

| Project | Subdirectory | Files | Key Interfaces | Notes |
|---------|--------------|-------|----------------|-------|
| MarkMpn.Sql4Cds.Engine | / | 39 | IAttributeMetadataCache, IQueryExecutionOptions | Core engine root |
| MarkMpn.Sql4Cds.Engine | Ado/ | 15 | None | ADO.NET implementation |
| MarkMpn.Sql4Cds.Engine | ExecutionPlan/ | 100 | 13 interfaces (IExecutionPlanNode hierarchy) | Core execution nodes |
| MarkMpn.Sql4Cds.Engine | Visitors/ | 28 | None | Visitor pattern classes |
| MarkMpn.Sql4Cds.Export | / | 2 | None | Export root |
| MarkMpn.Sql4Cds.Export | Contracts/ | 3 | None | Export contracts |
| MarkMpn.Sql4Cds.Export | DataStorage/ | 20 | IFileStreamFactory, IFileStreamReader, IFileStreamWriter | File storage |
| MarkMpn.Sql4Cds.Export | Utility/ | 3 | None | Utilities |
| MarkMpn.Sql4Cds.LanguageServer | / | 3 | IJsonRpcMethodHandler | LSP root |
| MarkMpn.Sql4Cds.LanguageServer | Admin/Contracts/ | 12 | None | Admin contracts |
| MarkMpn.Sql4Cds.LanguageServer | Autocomplete/ | 3 | None | Autocomplete |
| MarkMpn.Sql4Cds.LanguageServer | Connection/ | 6 | None | Connection handling |
| MarkMpn.Sql4Cds.LanguageServer | Connection/Contracts/ | 11 | None | Connection contracts |
| MarkMpn.Sql4Cds.LanguageServer | ObjectExplorer/Contracts/ | 16 | None | OE contracts |
| MarkMpn.Sql4Cds.LanguageServer | QueryExecution/Contracts/ | 40 | None | QE contracts |
| MarkMpn.Sql4Cds.XTB | / | 38 | IDocumentWindow | XTB plugin root |
| MarkMpn.Sql4Cds.XTB | Autocomplete/ | 12 | IAutocompleteListView, ITextBoxWrapper | Autocomplete UI |
| MarkMpn.Sql4Cds.XTB | CDSLookupDialog/ | 13 | None | Lookup dialog |
| MarkMpn.Sql4Cds.SSMS | / | 11 | None | SSMS shared code |
| MarkMpn.Sql4Cds.SSMS | Reflection/ | 6 | None | Reflection helpers |
| MarkMpn.Sql4Cds | / | 3 | None | XrmToolBox host |
| MarkMpn.Sql4Cds.Controls | / | 2 | None | UI controls |
| MarkMpn.Sql4Cds.DebugVisualizer.* | / | 7 | None | Debug visualizers |
| MarkMpn.Sql4Cds.Engine.Tests | / | 39 | None | Engine tests |
| AzureDataStudioExtension | src/ | ~5 | None | TypeScript extension |

---

## Phase 2: Significance Matrix

| Subdirectory | Files | Interface? | Verdict | Proof |
|--------------|-------|------------|---------|-------|
| Engine/ (root) | 39 | Yes (2) | SPEC NEEDED | New spec: architecture.md, type-system.md |
| Engine/ExecutionPlan/ | 100 | Yes (13) | SPEC NEEDED | New spec: execution-plan-nodes.md |
| Engine/Visitors/ | 28 | No | SPEC NEEDED | New spec: query-compilation.md |
| Engine/Ado/ | 15 | No | SPEC NEEDED | New spec: ado-net-provider.md |
| Export/DataStorage/ | 20 | Yes (3) | SPEC NEEDED | New spec: export-system.md |
| LanguageServer/ (all) | 118 | Yes (1) | SPEC NEEDED | New spec: language-server.md |
| XTB/ | 65 | Yes (3) | SPEC NEEDED | New spec: host-integrations.md |
| SSMS/ | 17 | No | COVERED | host-integrations.md will cover |
| MarkMpn.Sql4Cds/ | 3 | No | SKIP | <5 files, thin host wrapper |
| Controls/ | 2 | No | SKIP | <5 files, minimal UI |
| DebugVisualizer.*/ | 7 | No | SKIP | Dev tooling only |
| Engine.Tests/ | 39 | No | SKIP | Test code, not spec target |
| AzureDataStudioExtension/ | ~5 | No | COVERED | host-integrations.md will cover |

---

## Tasks

### Spec Generation (Priority Order)

- [x] **Generate spec: architecture.md** <!-- id: spec-arch -->
    - Source: Cross-cutting (all projects)
    - Priority: P0 (Blocking - all specs reference this)
    - Key files: MarkMpn.Sql4Cds.Engine.csproj, SessionContext.cs, NodeContext.cs

- [x] **Generate spec: type-system.md** <!-- id: spec-types -->
    - Source: MarkMpn.Sql4Cds.Engine/ (SqlTypeConverter, DataTypeHelpers, SqlVariant, SqlDateTypes)
    - Priority: P1
    - Key files: ExecutionPlan/SqlTypeConverter.cs, DataTypeHelpers.cs, SqlVariant.cs, SqlDateTypes.cs

- [x] **Generate spec: execution-plan-nodes.md** <!-- id: spec-nodes -->
    - Source: MarkMpn.Sql4Cds.Engine/ExecutionPlan/
    - Priority: P1
    - Key files: IExecutionPlanNode.cs, IDataExecutionPlanNode.cs, BaseNode.cs, NodeSchema.cs

- [x] **Generate spec: query-compilation.md** <!-- id: spec-compile -->
    - Source: MarkMpn.Sql4Cds.Engine/ (ExecutionPlanBuilder, Visitors/)
    - Priority: P2
    - Key files: ExecutionPlanBuilder.cs, Visitors/RewriteVisitor.cs

- [x] **Generate spec: expression-evaluation.md** <!-- id: spec-expr -->
    - Source: MarkMpn.Sql4Cds.Engine/ (ExpressionExtensions, ExpressionFunctions)
    - Priority: P2
    - Key files: ExecutionPlan/ExpressionExtensions.cs, ExpressionFunctions.cs

- [x] **Generate spec: fetchxml-translation.md** <!-- id: spec-fetchxml -->
    - Source: MarkMpn.Sql4Cds.Engine/ (FetchXml*, FetchXmlScan)
    - Priority: P2
    - Key files: FetchXml2Sql.cs, ExecutionPlan/FetchXmlScan.cs, FetchXml.cs

- [x] **Generate spec: query-optimization.md** <!-- id: spec-optim -->
    - Source: MarkMpn.Sql4Cds.Engine/ (ExecutionPlanOptimizer)
    - Priority: P3
    - Key files: ExecutionPlanOptimizer.cs, FetchXmlElementComparer.cs

- [x] **Generate spec: ado-net-provider.md** <!-- id: spec-ado -->
    - Source: MarkMpn.Sql4Cds.Engine/Ado/
    - Priority: P3
    - Key files: Sql4CdsConnection.cs, Sql4CdsCommand.cs, Sql4CdsDataReader.cs

- [ ] **Generate spec: metadata-caching.md** <!-- id: spec-meta -->
    - Source: MarkMpn.Sql4Cds.Engine/ (AttributeMetadataCache, MessageCache, TableSizeCache)
    - Priority: P3
    - Key files: AttributeMetadataCache.cs, MessageCache.cs, TableSizeCache.cs

- [ ] **Generate spec: language-server.md** <!-- id: spec-lsp -->
    - Source: MarkMpn.Sql4Cds.LanguageServer/
    - Priority: P4
    - Key files: Program.cs, IJsonRpcMethodHandler.cs, Connection/ConnectionManager.cs

- [ ] **Generate spec: export-system.md** <!-- id: spec-export -->
    - Source: MarkMpn.Sql4Cds.Export/
    - Priority: P4
    - Key files: DataStorage/IFileStreamFactory.cs, various writers

- [ ] **Generate spec: host-integrations.md** (Optional) <!-- id: spec-hosts -->
    - Source: MarkMpn.Sql4Cds.XTB/, MarkMpn.Sql4Cds.SSMS/, AzureDataStudioExtension/
    - Priority: P5
    - Key files: XTB/*.cs, SSMS/*.cs

---

## Spec Dependency Graph

```
                          architecture.md
                                |
                  +-------------+-------------+
                  |                           |
           type-system.md              metadata-caching.md
                  |
       execution-plan-nodes.md
                  |
       +---------+---------+
       |                   |
query-compilation.md  expression-evaluation.md
       |                   |
       +--------+----------+
                |
    fetchxml-translation.md
                |
    query-optimization.md
                |
     ado-net-provider.md
                |
       +-------+-------+
       |               |
language-server.md  export-system.md
       |               |
       +-------+-------+
               |
     host-integrations.md
```

---

## Justified Exclusions

| Component | Files | Reason for Exclusion |
|-----------|-------|---------------------|
| MarkMpn.Sql4Cds (root) | 3 | Thin XrmToolBox host wrapper, <5 files, no interfaces |
| MarkMpn.Sql4Cds.Controls | 2 | Minimal UI controls, <5 files, no complexity |
| DebugVisualizer.* | 7 total | Developer tooling only, not core functionality |
| Test projects | 50+ | Test code documents behavior but doesn't need own spec |
| Individual 90+ nodes | n/a | Covered by execution-plan-nodes.md as extension points |

---

## Verification

### How to Test Each Spec

1. **architecture.md**: Verify project structure matches spec, test build across target frameworks
2. **type-system.md**: Write type conversion tests using documented precedence rules
3. **execution-plan-nodes.md**: Verify interface hierarchy matches, test node lifecycle (Init/GetNext/Close)
4. **query-compilation.md**: Parse sample SQL and verify execution plan structure
5. **expression-evaluation.md**: Test expression evaluation with documented functions
6. **fetchxml-translation.md**: Round-trip SQL->FetchXML->SQL and verify equivalence
7. **query-optimization.md**: Enable/disable optimizer and compare query plans
8. **ado-net-provider.md**: Execute queries via Sql4CdsConnection and validate behavior
9. **metadata-caching.md**: Test cache behavior with mock metadata
10. **language-server.md**: Connect via LSP client and verify protocol compliance
11. **export-system.md**: Export to each format and verify output structure
12. **host-integrations.md**: Load plugins in each host application

---

## Key Statistics

| Metric | Count |
|--------|-------|
| Phase 1.2 rows with Files > 5 | 15 |
| Phase 1.2 rows with interfaces | 7 |
| Total qualifying directories | 17 (deduplicated) |
| Phase 2 matrix rows | 13 |
| New specs needed | 11 (+ 1 optional) |
| Existing specs needing expansion | 0 |
