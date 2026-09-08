# AVD Connection Report — Query Reference & Customer Walkthrough

This document explains, panel by panel, how the **AVD Connection Report** workbook classifies transport, what each query measures, and how to interpret the results. Use it as a talking-points guide when walking a customer through the dashboard.

## Table of contents

- [1. Data sources](#1-data-sources)
- [2. Global filters (top of the workbook)](#2-global-filters-top-of-the-workbook)
- [3. The transport classification formula (used by every panel)](#3-the-transport-classification-formula-used-by-every-panel)
- [4. Panel-by-panel reference (summary)](#4-panel-by-panel-reference-summary)
- [5. Query-by-query technical breakdown](#5-query-by-query-technical-breakdown)
- [6. Suggested customer talking points](#6-suggested-customer-talking-points)

---

## 1. Data sources

The workbook queries four Azure Monitor / Log Analytics tables, all populated by Azure Virtual Desktop diagnostics. All four share `CorrelationId` (a per-session-attempt GUID), which is the join key used to stitch them together everywhere in the workbook.

| Table | What it provides | Reference |
|---|---|---|
| `WVDConnections` | One row per connection state change (Started/Connected/Completed). Contains `TransportType`, `UdpUse`, host pool/gateway/session host/user info. | [Docs](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdconnections) |
| `WVDMultiLinkAdd` | Link-level detail for RDP Shortpath negotiation (client/server transport type, TURN IPs). Used as corroborating evidence alongside `WVDConnections`. | [Docs](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdmultilinkadd) |
| `WVDConnectionNetworkData` | Periodic network samples per session: `EstRoundTripTimeInMs`, `EstAvailableBandwidthKBps`. | [Docs](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdconnectionnetworkdata) |
| `WVDErrors` | Error/diagnostic events: `Code`/`CodeSymbolic`, `ServiceError`, `Source`, `ActivityType`, `Operation`, `Message`. | [Docs](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvderrors) |

### 1.1 `WVDConnections` — the backbone table

Every panel starts here. One row is written **per state change** of a connection attempt (e.g. `Started`, `Connected`, `Completed`, `ClientDisconnected`), so a single session normally produces multiple rows sharing the same `CorrelationId`. This is why almost every query first does `where State=="Connected"` and then `summarize arg_max(TimeGenerated,*) by CorrelationId` — collapsing each session down to its single most-recent "Connected" snapshot before doing anything else.

Columns the workbook actually reads:
- **`CorrelationId`** — unique ID for one connection attempt; the join key to all other tables.
- **`TimeGenerated`** — event timestamp; used for the time-range filter and time-chart binning.
- **`State`** — lifecycle stage of the row (`Started`, `Connected`, `ClientDisconnected`, `Completed`, etc.). Most panels only care about `Connected`; §5.8's error-correlation panel intentionally does **not** filter on this, so failed/never-connected attempts are included too.
- **`TransportType`** — the primary signal for classification (`Shortpath`, `Multipath`, `TURN`, `Websocket`/`TCP Websocket`, or blank).
- **`UdpUse`** — documented numeric code (`1`/`2`/`4`) refining *which kind* of UDP path was used; see §3 for the full mapping.
- **`UserName`** — the signed-in user (UPN); normalized to `"Unknown"` when blank.
- **`GatewayRegion`** — which AVD gateway region handled the connection; normalized to `"Unknown"` when blank.
- **`ClientOS` / `ClientType` / `ClientVersion`** — client device details, surfaced in the per-session detail table (§5.9) for forensics.
- **`_ResourceId`** — the Azure resource ID of the host pool; `HostPool` is derived from it via `extract("/hostpools/([^/]+)",1,_ResourceId)` since there's no plain `HostPool` column.
- **`SessionHostName`** (aliased `SessionHost`) — which VM served the session; used for the per-host breakdown in §5.4.

> 📌 **Accuracy note:** cross-checked against the [official `WVDConnections` schema reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdconnections) — as of this writing, `UdpUse` is **not** listed in that published column reference, even though it is a real field emitted by the service and used throughout this workbook (and documented separately in the [RDP Shortpath configuration guide](https://learn.microsoft.com/azure/virtual-desktop/configure-rdp-shortpath)). This is a known gap/lag in the generic table-schema docs, not an error in the workbook — `UdpUse` reliably appears in live data. Worth mentioning to a customer who goes looking for it in the schema reference and can't find it.

### 1.2 `WVDMultiLinkAdd` — corroborating link-negotiation evidence

Written when RDP Shortpath negotiates one or more "links" (network paths) for a session — a session can have zero, one, or several rows here depending on how many paths were attempted (e.g. a direct path attempt plus a TURN fallback attempt). The workbook never treats this table as authoritative on its own; it's always used as **secondary evidence** alongside `WVDConnections.TransportType`/`UdpUse` (see the `HasTurn`/`HasShort`/`HasWeb` counters described throughout §5).

Columns the workbook reads:
- **`CorrelationId`** — join key back to `WVDConnections`.
- **`ClientTransportType` / `ServerTransportType`** — the transport negotiated on each side of the link (values like `DIRECT`, `TURN`, `WEBSOCKET`). Both are checked because client and server can occasionally disagree/report differently.
- **`ClientTURNIP` / `ServerTURNIP`** — populated only when a TURN relay was actually used; a non-blank value is itself treated as TURN evidence even if the transport-type string doesn't say "TURN".
- **`LinkId`** — distinct network path identifier; `dcount(LinkId)` in §5.9 shows how many paths were attempted for a session (multiple links can indicate network instability or a client that's retrying different NICs/adapters).
- **`GatewayRegion`** — present here too, though the workbook sources gateway region from `WVDConnections` instead.
- **`ClientNatIP` / `ServerNatIP`** and **`ClientTransportIP` / `ServerTransportIP`** — the client's/session host's public NAT address and the actual transport-connection IP for the link. Not currently read by any workbook query, but they're the underlying evidence Microsoft's own diagnostics use to determine STUN-vs-TURN outcomes (public/STUN direct UDP relies on successful NAT traversal between these IPs; a TURN relay is negotiated when it fails) — useful if you ever need to explain to a customer *why* a session fell back to TURN instead of public/STUN direct.

### 1.3 `WVDConnectionNetworkData` — periodic in-session network samples

Written repeatedly *during* an active session (not just once), giving a time series of network quality per `CorrelationId`. Because there are multiple samples per session, every query aggregates with `avg()`/`percentile()` rather than reading a single row.

Columns the workbook reads:
- **`CorrelationId`** — join key back to `WVDConnections`.
- **`TimeGenerated`** — sample timestamp (not separately surfaced, but implicitly bounded by the workbook's time-range filter).
- **`EstRoundTripTimeInMs`** — estimated round-trip latency at time of sample; aggregated into `AvgRTTms` and `P95RTTms` (95th percentile — a worst-case-realistic view, not skewed by one outlier).
- **`EstAvailableBandwidthKBps`** — estimated available bandwidth in **kilobytes/sec**; the workbook converts this to **megabits/sec** everywhere it's displayed (`KBps * 8.0 / 1000`) since Mbps is the unit most customers expect. Aggregated into `AvgBandwidthMbps` and `P10BandwidthMbps` (10th percentile — a conservative floor).

Because this table is joined with `kind=inner` in most panels, only sessions that produced **at least one** sample contribute to RTT/bandwidth numbers — this is exactly what the "Network Telemetry Coverage %" column in §5.4 is warning about: a session with no rows here contributes nothing to the averages, so low coverage means those averages are based on a smaller, possibly unrepresentative slice of sessions.

### 1.4 `WVDErrors` — diagnostic/error events

Written whenever AVD's client or service stack logs a warning/error during connection setup or an active session. Unlike the network table, a session can have **zero** error rows (the common case) or many.

Columns the workbook reads:
- **`CorrelationId`** — join key back to `WVDConnections` (used to scope errors to in-filter sessions).
- **`TimeGenerated`** — event timestamp; bounded by the workbook's time-range filter.
- **`Code` / `CodeSymbolic`** — numeric and (when available) human-readable error identifiers; the workbook falls back to `"Error <Code>"` when `CodeSymbolic` is blank.
- **`ServiceError`** — a boolean documented by Microsoft to indicate whether the error originated in the Microsoft-managed service (`true`) versus the customer's environment/client/network (`false`); this directly drives the red/orange color-coding in §5.7 and the `Origin` label used elsewhere.
- **`Source`** — which component logged the error (e.g. client, agent, gateway, broker).
- **`ActivityType` / `Operation`** — what the client/service was doing when the error occurred; used to derive the friendlier `Category`/`Issue` buckets in §5.8 (e.g. "Host or agent", "Connection setup").
- **`Message`** — free-text diagnostic message; the same `Code` can carry different `Message` text depending on context, which is why §4.8/§5.8 collect up to 5 distinct sample messages per group instead of assuming one message per code.

Source for the RDP Shortpath transport model: [Configure RDP Shortpath for Azure Virtual Desktop](https://learn.microsoft.com/azure/virtual-desktop/configure-rdp-shortpath).

---

## 2. Global filters (top of the workbook)

| Filter | Type | Behavior |
|---|---|---|
| **Log Analytics workspace** | Resource picker, multi-select, required | Not fixed to one workspace — the customer picks one or more Log Analytics workspaces at run time. Bound via `crossComponentResources`, so every panel automatically fans out and merges results across all selected workspaces. |
| **Time range** | Time picker | Standard Workbooks time range control; every query filters on `TimeGenerated {TimeRange}`. |
| **Host pool** | Dropdown, multi-select | Populated dynamically from `WVDConnections` host pool names (parsed from `_ResourceId`). |
| **Gateway region** | Dropdown, multi-select | Populated dynamically from `WVDConnections.GatewayRegion`. |
| **Search or select user** | Dropdown, multi-select | Populated dynamically from `WVDConnections.UserName`. |
| **Transport outcome** | Dropdown, multi-select | Static list: All / UDP direct / UDP via TURN / TCP fallback / Unknown. |

**How multi-select filtering works in the KQL:** every filter includes an `"All"` sentinel value. Each query's `where` clause uses the pattern:
```kql
where ('All' in ({FilterName}) or FieldName in ({FilterName}))
```
If the customer selects "All" (alone or together with other values), the filter is bypassed and every row passes. Otherwise, only rows whose field value is in the selected set pass. This is the standard Azure Workbooks pattern for multi-select dropdown parameters (values arrive pre-quoted/comma-delimited, so `in (...)` is used instead of `==`).

---

## 3. The transport classification formula (used by every panel)

Every panel derives a `Mode`/`Transport` field using the same priority-ordered logic, combining `WVDConnections.TransportType` / `UdpUse` with link-level evidence from `WVDMultiLinkAdd`:

1. **UDP via TURN (relayed)** — if `TransportType == "TURN"`, OR any `WVDMultiLinkAdd` row for the session shows a TURN transport type or TURN IP, OR `UdpUse == "4"`.
2. **UDP direct** — else if `TransportType` is `Shortpath`/`Multipath`, OR `UdpUse` is `"1"` (managed/private) or `"2"` (public/STUN), OR `WVDMultiLinkAdd` shows a Shortpath/DIRECT link.
3. **TCP fallback** — else if `TransportType` is `Websocket`/`TCP Websocket`, OR `WVDMultiLinkAdd` shows a WEBSOCKET link.
4. **Unknown** — else (no decisive evidence in either table).

**Why TURN is checked first:** a session can show early Shortpath/DIRECT evidence and still fail over to TURN partway through, so any TURN signal — including the documented `UdpUse=4` code — overrides a lower-priority direct-path match.

**Documented `UdpUse` codes** (per Microsoft docs — any other value means "not using UDP, connected via TCP"):
- `1` = RDP Shortpath for managed networks (private ExpressRoute/VPN)
- `2` = RDP Shortpath for public networks via STUN (direct)
- `4` = RDP Shortpath for public networks via TURN (relayed)

Several panels further split "UDP direct" into **managed** (`UdpUse=1`) vs **public/STUN** (`UdpUse=2`) sub-buckets for clarity.

> ⚠️ **Known limitation:** the "Session host transport and bandwidth" table currently has a labeled column **"Direct UDP - Unspecified %"** intended to capture Direct-UDP sessions where `UdpUse` is neither `1` nor `2` (e.g., classified as direct purely from `TransportType`/link evidence). That column's underlying query calculation has not yet been wired up, so it will render blank. The **managed % + public/STUN %** in that table may not always sum exactly to the total **Direct UDP %** for this reason. This is flagged here so it can be fixed in an upcoming update — it does not affect the TURN/TCP/Unknown numbers.

---

## 4. Panel-by-panel reference (summary)

### 4.1 UDP health summary (tiles)
Top-line KPI tiles for the current filter selection: **Sessions**, **Direct UDP %**, **Direct UDP – managed %**, **Direct UDP – public/STUN %**, **Relayed UDP %**, **TCP fallback %**, **Unknown %**. All percentages are of total *connected* sessions matching the filters.

### 4.2 Transport outcome (pie chart)
Same classification, grouped into a friendlier `Detail` label (e.g., "UDP direct (managed/private)") and shown as a pie so the customer can see the overall transport mix at a glance.

### 4.3 Session transport trend (time chart)
Same `Detail` breakdown, binned over time (`bin(TimeGenerated, {TimeRange:grain})`) so the customer can see whether TCP fallback/Unknown spikes correlate with specific time windows (e.g., a deployment or network change).

### 4.4 Transport and bandwidth by session host (table)
One row per session host. Columns: Sessions, Users, Direct UDP % (+ managed/public/STUN sub-splits), Relayed via TURN %, TCP Fallback %, Unknown Transport %, **Network Telemetry Coverage %** (share of sessions with at least one RTT/bandwidth sample — low coverage means the bandwidth/RTT numbers are based on a small, possibly unrepresentative subset), Average Bandwidth (Mbps), P10 Bandwidth (Mbps). Sorted with the worst TCP Fallback %/Unknown % hosts first — these are the ones most worth investigating.

### 4.5 P95 RTT by transport (bar chart)
One bar per transport sub-type. Height = P95 (95th percentile) round-trip time in ms from `WVDConnectionNetworkData`. P95 reflects realistic worst-case latency rather than a single spike or a hidden-by-average number. Lower is better.

### 4.6 P10 bandwidth by transport (bar chart)
One bar per transport sub-type. Height = P10 (10th percentile) estimated available bandwidth, converted from KBps to **Mbps** (`KBps * 8.0 / 1000`). P10 reflects a conservative floor — 90% of samples were at or above this value. Higher is better; a much lower P10 on one transport type suggests a subset of sessions with poor connectivity.

### 4.7 Top errors by affected connection attempts (bar chart)
One bar per `WVDErrors` error code (`CodeSymbolic`, or `Error <Code>` if no symbolic name exists). Height = **distinct connection attempts** affected (not raw event count, so one noisy retrying session can't skew the ranking). Color-coded by the documented `ServiceError` field: **red = Microsoft service** (not the customer's fault), **orange = Environment/configuration** (client, network, firewall, DNS, or host pool policy). Top 10 only.

### 4.8 Which users/devices are hitting errors, and their network conditions (table)
Groups `WVDErrors` by User + Client IP + Outcome + Transport to surface repeat offenders. Key columns:
- **Top errors** — up to 5 most frequent error codes for that row with counts, most-frequent-first.
- **Sample messages** — up to 5 distinct raw `WVDErrors.Message` strings (the same code can carry different message text depending on context).
- **Issues / Categories / Sources / Session hosts / Gateways** — deduplicated sets (`make_set`, capped at 5–10) for quick scanning.
- Correlated network stats (Network sessions/samples, Avg/P95 RTT, Avg/P10 bandwidth) so you can tell whether errors coincide with poor connectivity.

### 4.9 Session transport, RTT, and bandwidth details (table)
One row **per session** (not aggregated), sorted by user then most-recent-first, so all of one user's sessions appear together. Includes a plain-English `TransportDetail` and `Interpretation` column, `ObservedEvidence` (what data justified the classification), network stats, error count, and an **Error details** column listing up to 10 raw WVDErrors entries (`code | activity | operation | source | message`) for that session — so the customer doesn't need to cross-reference `WVDErrors` separately.

### 4.10 Users who have never had a UDP connection (table)
Lists every user whose **every session** in the selected range was TCP fallback or Unknown — i.e., zero UDP sessions (direct or relayed). Deliberately **ignores the Transport outcome filter**, since it needs to see all of a user's sessions to determine whether UDP was ever used. Shows **Top errors** (same top-5-with-counts pattern) so you can see whether their lack of UDP correlates with other problems. A user appearing here across **multiple** session hosts and gateway regions suggests a persistent client-side/network issue (firewall, NAT type, Windows/RDP UDP settings) rather than a one-off blip.

---

## 5. Query-by-query technical breakdown

This section walks through the actual KQL logic behind each panel, `let`-statement by `let`-statement, so you have the full technical picture — not just the summary.

### 5.1 UDP health summary (tiles)
1. **`let L`** — queries `WVDMultiLinkAdd`, groups by `CorrelationId`, and produces three counters per session: `HasTurn` (rows where client/server transport type is TURN, or a TURN IP is present), `HasShort` (rows showing Shortpath/DIRECT), `HasWeb` (rows showing WEBSOCKET). This is per-session corroborating evidence from the link-negotiation table.
2. **`let B`** (`materialize`d, since it's reused by 6 downstream `toscalar` calls) — starts from `WVDConnections`, keeps only `State == "Connected"` rows, collapses duplicate rows per `CorrelationId` with `arg_max(TimeGenerated,*)` (keeps the latest snapshot of each session), left-joins the `L` evidence, coalesces nulls to `0` (sessions with no MultiLinkAdd rows at all), applies the `Mode` classification formula (see §3), derives `HostPool` from `_ResourceId` via regex `extract("/hostpools/([^/]+)",1,_ResourceId)`, normalizes blank `GatewayRegion`/`UserName`/`HostPool` to `"Unknown"`, then applies all four filter-pill `where` clauses.
3. Six `let ... = toscalar(B | summarize countif(...))` statements compute `Total`, `Direct`, `ManagedDirect` (`Mode=="UDP direct" and UdpUse=="1"`), `PublicDirect` (`UdpUse=="2"`), `Turn`, `TCP`, `Unknown` as single scalar numbers.
4. A final `union` of seven `print` statements builds the tile rows: each has `Metric` (title), `Value` (percentage, `round(100.0*count/iff(Total==0,1,Total),1)` — the `iff` guards against divide-by-zero when there are no sessions), `Description` (subtitle text), and `Sort` (1–7) — then `order by Sort asc` fixes the tile order regardless of query engine ordering.

### 5.2 Transport outcome (pie chart)
Same `L`/`Mode` logic as §5.1 but **not** materialized (single-pass query, no reuse needed). Adds an `extend Detail=case(...)` that turns `Mode` into a more descriptive label (`"UDP direct (managed/private)"`, `"UDP direct (public/STUN)"`, `"UDP direct (evidence-based)"` for direct sessions with no matching `UdpUse` code, `"UDP via TURN (relay)"`, `"TCP fallback"`, `"Unknown"`). Ends with `summarize Sessions=count() by Detail | order by Sessions desc` to feed the pie chart's slices.

### 5.3 Session transport trend (time chart)
Identical query to §5.2, except the final `summarize` groups `by bin(TimeGenerated,{TimeRange:grain}), Detail` instead of just `by Detail`. `{TimeRange:grain}` is a Workbooks built-in that auto-picks a sensible bucket size (minutes/hours/days) based on the selected time range, so the same query adapts whether you're looking at 1 hour or 30 days.

### 5.4 Transport and bandwidth by session host (table)
Four building blocks:
1. **`let L`** — same three evidence counters as before, but aliased `T`/`S`/`W` (shorter names since this query is denser).
2. **`let C`** (`materialize`d) — `WVDConnections` filtered to `State=="Connected"`, deduplicated per session, joined with `L`, classified into `Mode` (`"TURN"`/`"Direct"`/`"TCP"`/`"Unknown"` — short internal labels), with `HostPool`/`Gateway`/`User`/`SessionHost` derived, then re-mapped to a friendlier `Transport` field (used only for the Transport-outcome filter pill), filtered by all four pills, and projected down to just the 6 columns needed downstream (`CorrelationId,HostPool,SessionHost,UserName,Mode,UdpUse`).
3. **`let Mix`** — groups `C` `by HostPool,SessionHost` and counts `Sessions`, `Users` (`dcount(UserName)`), and per-mode counts (`Direct`, `ManagedDirect`, `PublicDirect`, `Relayed`, `TCP`, `Unknown`), then `extend`s six `_Pct` columns as `round(100.0*count/iff(Sessions==0,1,Sessions),1)`.
4. **`let Bandwidth`** — `WVDConnectionNetworkData` **inner**-joined to `C` (so only sessions with at least one network sample count), grouped `by HostPool,SessionHost`, computing `SampledSessions` (`dcount(CorrelationId)`), `AvgBandwidthMbps` and `P10BandwidthMbps` (both converted from the raw `EstAvailableBandwidthKBps` KBps value via `*8.0/1000` — multiply by 8 to convert bytes→bits, divide by 1000 to convert kilo→mega).
5. **Final stitch** — `Mix | join kind=leftouter Bandwidth on HostPool,SessionHost` (left join keeps hosts with zero network samples, leaving `AvgBandwidthMbps`/`P10BandwidthMbps` blank for them) `| extend SampledSessions=coalesce(SampledSessions,0) | extend NetworkCoverage_Pct=round(100.0*SampledSessions/iff(Sessions==0,1,Sessions),1) | project [13 columns] | order by TCP_Pct desc, Unknown_Pct desc, Sessions desc` — sorts the worst-behaving hosts to the top of the grid.

> Note: as flagged in §3, this panel's `labelSettings` currently defines a `"Direct UDP - Unspecified %"` column (`OtherDirect_Pct`) that the query above does **not** yet compute or `project` — that column will render blank until the query is updated to add an `OtherDirect=countif(Mode=="Direct" and tostring(UdpUse) !in ("1","2"))` counter and corresponding `OtherDirect_Pct` field.

### 5.5 P95 RTT by transport (bar chart)
1. Same `T`/`S`/`W` evidence counters from `WVDMultiLinkAdd`.
2. `WVDConnections` classified into `Mode` and the friendlier `Detail` label, filtered by all four pills, then `project CorrelationId,Detail` — trimmed down to just the join key and the grouping label, since nothing else is needed.
3. `WVDConnectionNetworkData | join kind=inner C on CorrelationId | summarize SampledSessions=dcount(CorrelationId), Samples=count(), AvgRTTms=round(avg(EstRoundTripTimeInMs),1), P95RTTms=round(percentile(EstRoundTripTimeInMs,95),1), AvgBandwidthMbps=..., P10BandwidthMbps=... by Detail | order by SampledSessions desc` — the **inner** join means only sessions with at least one network sample contribute to any bar; each `Detail` value becomes one bar in the chart, bound to `P95RTTms` on the Y axis.

### 5.6 P10 bandwidth by transport (bar chart)
This panel's KQL is **byte-for-byte identical** to §5.5 — it computes the same five aggregate columns (`SampledSessions`, `Samples`, `AvgRTTms`, `P95RTTms`, `AvgBandwidthMbps`, `P10BandwidthMbps`) by `Detail`. Only the chart's `yAxis` binding differs (`P10BandwidthMbps` instead of `P95RTTms`), so this is effectively "the same dataset, a different column charted" — each Workbooks panel runs its own independent query, so the computation is duplicated rather than shared.

### 5.7 Top errors by affected connection attempts (bar chart)
1. Evidence counters + `Mode`/`Transport` classification as before, filtered by all four pills, then `project CorrelationId` **only** — this panel doesn't care about a session's transport for display, just needs the ID to scope which errors count (the Transport filter pill already narrowed `C` to the right sessions before this projection).
2. `WVDErrors | where TimeGenerated {TimeRange} | extend Error=iff(isempty(CodeSymbolic),strcat("Error ",tostring(Code)),CodeSymbolic), Origin=iff(ServiceError==true,"Microsoft service","Environment/configuration") | join kind=inner C on CorrelationId | summarize AffectedAttempts=dcount(CorrelationId) by Error,Origin | top 10 by AffectedAttempts desc` — builds a human-readable `Error` label (falls back to the numeric code when there's no symbolic name), buckets the documented `ServiceError` boolean into two plain-English categories, inner-joins to keep only errors on in-scope sessions, counts **distinct** `CorrelationId`s (not raw error-event rows, so a retry storm on one session doesn't inflate the ranking), and keeps only the top 10 error+origin combinations.

### 5.8 Which users/devices are hitting errors, and their network conditions (table)
The densest query in the workbook — four independent aggregations stitched together:
1. **`let C`** (`materialize`d) — unlike most other panels, this one does **not** restrict to `State=="Connected"`. It keeps every attempt and derives `Outcome=iff(isnotempty(ConnectedAt),"Connected","Not connected")` from `minif(TimeGenerated,State=="Connected")`, so failed connection attempts are included too (their errors matter just as much). Projects `CorrelationId,Outcome,Transport,User,ClientIP,HostPool,Gateway,SessionHost`.
2. **`let ErrorGroups`** — `WVDErrors` with an added `Category=case(...)` bucketing `ActivityType`/`Operation` into `"Host or agent"`, `"Connection setup"`, `"Resource management"`, `"Workspace or assignment"`, or `"Other AVD activity"`, plus the same `Issue` label as §5.7. Inner-joins `C`, then `summarize`s **by `User,ClientIP,Outcome,Transport`**: `Errors=count()` (raw event count), `AffectedAttempts=dcount(CorrelationId)`, `ServiceErrors=countif(ServiceError==true)`, and six `make_set(...)` columns (`Issues`, `Messages`, `Categories`, `Sources`, `HostPools`, `SessionHosts`, `Gateways`, capped at 5–10 each) plus `LastSeen=max(TimeGenerated)` and `SampleCorrelationIds`.
3. **`let TopErrorsPerGroup`** — a separate pass over `WVDErrors` that first computes `IssueCount=count() by User,ClientIP,Outcome,Transport,Issue` (per-issue-code counts within each group), then `order by ...,IssueCount desc` followed by `summarize TopErrorsList=make_list(strcat(Issue," (",IssueCount,")"),5) by User,ClientIP,Outcome,Transport`. This is the standard Kusto "top-N-per-group" idiom: `make_list` preserves the row order it receives, so sorting descending by count *before* the `summarize` gives you the top 5 issues per group in the resulting array, which is then flattened to a string with `strcat_array(...,", ")`.
4. **`let NetworkGroups`** — `WVDConnectionNetworkData` inner-joined to `C`, grouped by the same `User,ClientIP,Outcome,Transport` key, computing `NetworkSessions`, `NetworkSamples`, `AvgRTTms`, `P95RTTms`, `AvgBandwidthMbps`, `P10BandwidthMbps`.
5. **Final stitch** — `ErrorGroups | join kind=leftouter NetworkGroups on (key) | join kind=leftouter TopErrorsPerGroup on (key) | extend NetworkSessions=coalesce(...,0), NetworkSamples=coalesce(...,0), TopErrors=iff(isempty(TopErrors),"None",TopErrors) | top 50 by Errors desc | project [22 columns]`. Both joins are **left** joins so a row is never dropped just because it has no network samples or no computed top-error string — it gets `0`/`"None"` defaults instead. Capped at the top 50 rows (by raw error count) to keep the grid readable.

### 5.9 Session transport, RTT, and bandwidth details (table)
Row-per-session (no cross-session aggregation), enriched with three left-joins:
1. **`L`** — the usual `T`/`S`/`W` evidence counters, **plus** `LinkCount=dcount(LinkId)`, `ClientTransports`/`ServerTransports` (`make_set` of the raw transport-type values seen), and `ClientTURNIPs`/`ServerTURNIPs` (`make_set_if(...,isnotempty(...))`, capturing only non-blank TURN IPs). This panel surfaces more raw `WVDMultiLinkAdd` detail than any other, since it's meant for deep per-session forensics.
2. **`N`** — per-`CorrelationId` network stats (`NetworkSamples`, `AvgRTTms`, `P95RTTms`, `AvgBandwidthMbps`, `P10BandwidthMbps`) — no grouping by transport, just raw per-session numbers.
3. **`E`** — per-`CorrelationId` error rollup: `ErrorCount=count()` and `ErrorDetails=make_set(strcat(Code-or-CodeSymbolic," | ",ActivityType," | ",Operation," | ",Source," | ",Message),10)` — up to 10 fully-formatted raw error strings per session.
4. **Base query** — `WVDConnections` filtered to `State=="Connected"`, deduplicated per session, left-joined to `L`, `N`, and `E` (coalescing all the count-type fields to `0` when a session has no matching rows), classified into `Mode`, then three additional descriptive fields: `TransportDetail` (friendlier label), `ObservedEvidence` (explains *why* it was classified that way — e.g. `"TURN transport, TURN IP, or documented UdpUse=4 present"`), and `Interpretation` (a plain-English sentence for the customer, e.g. `"Reverse-connect TCP/TLS 443 used; UDP Shortpath not established or not reported"`). Filtered by all four pills, projected to 19 columns including `ClientOS`/`ClientType`/`ClientVersion` and the raw `ErrorDetails`/`CorrelationId`, then `order by User asc, TimeGenerated desc` — so every session belonging to one user is grouped together, most-recent first.

### 5.10 Users who have never had a UDP connection (table)
1. **`L`** — the usual three evidence counters.
2. **`C`** (`materialize`d) — `WVDConnections` filtered to `State=="Connected"`, deduplicated, joined to `L`, classified into `Mode`, `HostPool`/`Gateway`/`User` derived, then filtered by **only the Host pool, Gateway, and User pills — deliberately not the Transport outcome pill** (explained in the panel description: the whole point is to see a user's full session history regardless of the current Transport filter selection). Projected to `CorrelationId,User,HostPool,Gateway,Mode,TimeGenerated`.
3. **`UserSummary`** — groups `C` by `User`: `Sessions=count()`, `UdpSessions=countif(Mode in ("UDP direct","UDP via TURN"))`, `TcpSessions=countif(Mode=="TCP fallback")`, `UnknownSessions=countif(Mode=="Unknown")`, `HostPools`/`Gateways` (`make_set`, capped at 5), `LastSeen=max(TimeGenerated)`, then **`where UdpSessions==0`** — the core filter: only users with *zero* UDP sessions (direct or TURN) across their entire history in the selected range survive.
4. **`UserErrorCounts`/`UserErrors`/`TopErrorsPerUser`** — the same "top-N-per-group" idiom as §5.8, but grouped only **by `User`** (not by ClientIP/Outcome/Transport, since this panel is purely user-centric): per-issue counts, summed into a total `ErrorCount`, and a formatted `TopErrors` string of the top 5 issue codes with counts.
5. **Final stitch** — `UserSummary | join kind=leftouter UserErrors | join kind=leftouter TopErrorsPerUser | extend ErrorCount=coalesce(...,0), TopErrors=iff(isempty(...),"None",...) | project [9 columns] | order by ErrorCount desc, Sessions desc` — the most error-prone, most-active no-UDP users surface at the top of the list.

---

## 6. Suggested customer talking points

1. Start with the **UDP health summary** tiles for the headline number: "X% of sessions used UDP directly, Y% via TURN relay, Z% fell back to TCP."
2. Use **Session transport trend** to check whether TCP fallback/Unknown correlates with a specific date (deployment, network change, ISP issue).
3. Drill into **Transport and bandwidth by session host** to find specific hosts with high TCP Fallback %/Unknown % — these often point to per-host firewall or NIC configuration issues.
4. Compare **P95 RTT** and **P10 bandwidth** across transport types to quantify whether TURN/TCP sessions are meaningfully worse than direct UDP for this customer's network.
5. Use **Top errors by affected attempts** and the **Which users/devices are hitting errors** table to correlate error codes with poor network conditions — red tiles (Microsoft service) vs orange (customer-side) tells you who owns the next troubleshooting step.
6. Finish with **Users who have never had a UDP connection** — this is usually the most actionable list, since persistent no-UDP users across multiple hosts/gateways point to a fixable client-side or firewall/NAT configuration problem rather than random network noise.

---

*This document reflects the workbook's query logic as of the current version of `AVD Connection Report.workbook`. If the queries are modified, update this document accordingly.*
