# Debugging Power BI issues with Capacity Metrics + Workspace Monitoring

Diagnosing Power BI performance and reliability problems on Fabric capacity usually takes two tools,
because each one holds a different layer of the story.

The Capacity Metrics app surfaces capacity-level signals: CU consumption, throttling, and overload,
together with the operation, item, and user behind them. It does not expose lower-level detail such
as the DAX query text that ran.

Workspace Monitoring covers that gap. Its `SemanticModelLogs` table holds the per-operation record:
DAX query text, durations, status, the user, and the report or model. There are two ways to use it,
and this repo covers both:

- **Part A then Part B:** identify an operation in Capacity Metrics, then carry its identifiers into
  a KQL lookup against `SemanticModelLogs` to read the full detail behind it.
- **Part B on its own:** query `SemanticModelLogs` directly with KQL for standalone triage, without
  going through the Capacity Metrics app.

## Prerequisites

- A Power BI Premium or a Fabric capacity.
- Capacity admin on the capacity, to install the Capacity Metrics app.
- A Power BI license (Pro, Premium Per User, or a Power BI individual trial), to use the Capacity Metrics app.
- The *Workspace admins can turn on monitoring for their workspaces* tenant setting turned on, which a Fabric administrator sets.
- [Workspace monitoring enabled ](https://learn.microsoft.com/fabric/fundamentals/enable-workspace-monitoring).
- At least the **contributor** role in the workspace, to query the read-only Eventhouse KQL database that monitoring creates.
- The [Microsoft Fabric Capacity Metrics app](https://learn.microsoft.com/fabric/enterprise/metrics-app-install), installed and pointed at the capacity you are investigating.

---

## Part A: Capacity Metrics (Compute page -> Timepoint page)

1. Open the **Microsoft Fabric Capacity Metrics** app, pick the capacity.
2. Use the **Capacity utilization and throttling** visual to find overload (CU% above the limit
   line) or applied throttling.
3. Click into a **timepoint** to open the timepoint page.
4. In the **Interactive operations** or **Background operations** table, sort by **Total CU (s)**
   descending to find the offending operation, then drill through to the **Timepoint Item Detail**
   page for that item/operation.
5. `Operation ID` is **not shown by default**. On the item-detail table visual it is documented as
   an **optional field**: add it using the **Select optional columns** dropdown menu. The same page also has an **Operation ID** slicer in its filter pane.
6. With the column added, take the **Operation ID** value: select the cell and **Copy > Copy value**, or use **... (More options) > Export data**. Also note the
   **User**, **Item name**, and the **timepoint (UTC)**.

---

## Part B: Workspace Monitoring (KQL on `SemanticModelLogs`)

Open the workspace monitoring query set in Fabric, or paste the blocks from `queries/`.

### `queries/01_find_offenders.kql`

Standalone triage with no Capacity Metrics needed. Six independent blocks (each carries its own
`Lookback`, so blocks can be run one at a time):

1. **Event census** - `summarize` count and total CPU by `OperationName` / `OperationDetailName`.
   Run this first to see which trace events actually exist in the window before filtering on a
   specific operation.
2. **Top 50 by CPU** - highest `CpuTimeMs` operations, projecting identity, item, mode, and
   `EventText` (DAX text for query events).
3. **Top 50 by duration** - highest `DurationMs` operations.
4. **Errors / warnings** - rows where `Level` is error/warning or `StatusCode != 0`.
5. **Rollup by report** - total CPU / duration and distinct users grouped by `ItemName` / `ItemId`.
6. **Rollup by user** - total CPU / duration, distinct reports, and max CPU grouped by `ExecutingUser`.

### `queries/02_drill_by_operationid.kql`

Set `TargetOperationId` to the **Operation ID** from Part A and run. Two blocks:

1. Returns every `SemanticModelLogs` row for that `OperationId` (a request emits begin/step/end rows),
   carrying the DAX text in `EventText` alongside user, item, mode, status, duration and CPU, ordered
   by `Timestamp`.
2. Reads the `CorrelationId` off that operation and returns every row sharing it across the monitoring
   tables, to see the surrounding activity for the same request.

### Key `SemanticModelLogs` columns 

| Question | Column |
| --- | --- |
| DAX text that ran | `EventText` |
| Cost (proxy) | `CpuTimeMs`, `DurationMs` |
| Who | `ExecutingUser`, `User`, `Identity`, `CallerIpAddress` |
| Which report / model | `ApplicationContext` (report id), `ItemName` / `ItemId`, `ApplicationName` |
| Model storage mode | `DatasetMode` (documented values: Import, DirectQuery, Composite) |
| Outcome | `Status`, `StatusCode`, `Level` |
| Operation type | `OperationName`, `OperationDetailName` |
| Scope | `WorkspaceId` / `WorkspaceName`, `CapacityId` |
| Cross-table correlation | `CorrelationId` |

---

## References

- Workspace monitoring overview: https://learn.microsoft.com/fabric/fundamentals/workspace-monitoring-overview
- Enable workspace monitoring: https://learn.microsoft.com/fabric/fundamentals/enable-workspace-monitoring
- Semantic model operation logs (schema): https://learn.microsoft.com/power-bi/enterprise/semantic-model-operations
- Install the Capacity Metrics app: https://learn.microsoft.com/fabric/enterprise/metrics-app-install
- Metrics app compute page: https://learn.microsoft.com/fabric/enterprise/metrics-app-compute-page
- Metrics app timepoint page: https://learn.microsoft.com/fabric/enterprise/metrics-app-timepoint-page
- Metrics app timepoint item-detail page: https://learn.microsoft.com/fabric/enterprise/metrics-app-timepoint-item-detail-page
- Copy a cell value from a Power BI table visual: https://learn.microsoft.com/power-bi/visuals/power-bi-visualization-tables#copy-single-cell
- Export data from a Power BI visual: https://learn.microsoft.com/power-bi/visuals/power-bi-visualization-export-data
- Official sample queries: https://github.com/microsoft/fabric-samples/tree/main/workspace-monitoring
