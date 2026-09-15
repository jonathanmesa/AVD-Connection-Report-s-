# Vanguard AVD Transport Investigation Guide

## Purpose

**Vanguard AVD Transport Investigation** is a transport-health and connection-reliability workbook for Azure Virtual Desktop (AVD). It helps determine whether connection behavior is associated with the client path, RDP Shortpath transport, Azure gateway region, session host, client version, logon phase, graphics delivery, or an AVD error.

Use it for a single user, a host pool, a gateway region, a client population, a time window, or an incident. It is an AVD diagnostic workbook; it does not directly measure application transaction time.

## Load and scope the workbook

1. Open the target **Log Analytics workspace** in the Azure portal.
2. Open **Workbooks** and create a workbook draft.
3. Open **Advanced Editor** and replace the draft JSON with [Vanguard AVD Transport Investigation.workbook](Vanguard%20AVD%20Transport%20Investigation.workbook).
4. Apply the JSON and select the workspace in the workbook parameter bar.
5. Set the **Time range**, then optionally select a **Host pool**, **Gateway region**, **User**, and **Transport outcome**.
6. Start with the summary panels before using the detailed tables.

The workbook does not require a fixed workspace identifier. The workspace picker controls where every query runs.

## How the data is collected

The workbook reads Azure Virtual Desktop resource logs from the selected Azure Monitor Log Analytics workspace. The normal collection path is:

1. An AVD resource such as a host pool, workspace, or application group emits a diagnostic event.
2. Azure Monitor diagnostic settings route the selected AVD log categories to the chosen Log Analytics workspace.
3. The event is stored in a strongly typed `WVD*` table, with `TimeGenerated` recording the event or measurement time in UTC.
4. Workbook queries filter that table by the selected time range and correlate related records with `CorrelationId`, user, host pool, session host, gateway, or client fields.

The workbook does not collect new telemetry and does not measure application transactions. It reads whatever diagnostic categories are already enabled and retained in the selected workspace. A missing row can mean that a category is disabled, the resource is not sending that category, the time range is outside retention, or the event did not occur.

Configure collection on the relevant Azure Virtual Desktop resource under **Diagnostic settings**. Select the required AVD log categories and send them to the Log Analytics workspace used by this workbook. The Microsoft Learn supported-logs pages identify which tables belong to host pools, workspaces, and application groups.

Official collection and schema references:

- [Azure Monitor diagnostic settings](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-settings)
- [Supported logs for AVD host pools](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-logs/microsoft-desktopvirtualization-hostpools-logs)
- [Supported logs for AVD workspaces](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-logs/microsoft-desktopvirtualization-workspaces-logs)
- [Supported logs for AVD application groups](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-logs/microsoft-desktopvirtualization-applicationgroups-logs)
- [Azure Monitor Logs table reference: WVD](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvd)
- [Azure Monitor Log Analytics query overview](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-query-overview)

### How joins work

- `WVDConnections` is the session identity anchor.
- `CorrelationId` connects a session to `WVDMultiLinkAdd`, `WVDConnectionNetworkData`, `WVDErrors`, checkpoints, and related event records.
- `UserName`, `SessionHostName`, `_ResourceId`, and `GatewayRegion` support user, host, host-pool, and gateway comparisons.
- `TimeGenerated` aligns measurements and errors to the complaint window.
- Percentiles such as P95 RTT and P10 bandwidth describe the degraded tail; they are not individual-user measurements unless the query is filtered to one user.

If a join produces no rows, check collection coverage and field availability before concluding that the session was healthy.

## Filter behavior

The Host pool, Gateway region, User, and Transport outcome filters apply to the investigation panels. The User filter is particularly useful: select one user to follow that user through transport, network, errors, connected time, and session evidence.

The Diagnostic coverage panel is intentionally workspace-wide when present. It answers whether diagnostic data is being collected, not whether one user is healthy.

## Transport meanings

| Result | Meaning | What to investigate |
| --- | --- | --- |
| Direct UDP, managed | RDP Shortpath direct over a managed/private network; documented `UdpUse=1` | Preferred path; compare RTT, bandwidth, and stability |
| Direct UDP, public/STUN | RDP Shortpath direct over the public network; documented `UdpUse=2` | Direct internet path; compare with managed UDP and TURN |
| UDP via TURN | UDP is working through a Microsoft-managed TURN relay; documented `UdpUse=4` or TURN evidence | Functional but relayed path; check RTT, bandwidth, and regional concentration |
| TCP fallback | Reverse-connect TCP/TLS 443 path; UDP Shortpath was not established or not reported | Check client support, policy, firewall, NAT, and network path |
| Unknown | Available diagnostic records do not identify the transport | Check `WVDConnections`, `WVDMultiLinkAdd`, `WVDErrors`, and collection coverage |

TCP fallback is not automatically a failure. It becomes a strong hypothesis when it is concentrated in affected users, hosts, client versions, gateway regions, or complaint windows and has worse RTT, bandwidth, or connected-time results than UDP.

## Diagnostic tables

### `WVDConnections`

**Role:** primary session identity and connection-state table.

**Important fields:**

- `CorrelationId`: joins a connection to network, transport, checkpoint, graphics, and error records.
- `UserName`: user filter and user-level correlation key.
- `SessionHostName`: host-level comparison key.
- `_ResourceId`: host-pool resource identifier.
- `GatewayRegion`: Azure gateway region used by the connection.
- `ClientOS`, `ClientType`, `ClientVersion`: client-population analysis.
- `TransportType`, `State`, `UdpUse`: connection outcome and transport evidence.
- `PredecessorConnectionId`: reconnect lineage.

**Use it for:** connected versus incomplete attempts, session identity, client mix, host-pool/gateway filtering, and joining all other connection-scoped evidence.

### `WVDMultiLinkAdd`

**Role:** transport-link evidence for RDP Shortpath and related paths.

**Important fields:**

- `CorrelationId`: joins links to `WVDConnections`.
- `ClientTransportType` and `ServerTransportType`: observed client/server transport types.
- `ClientTURNIP` and `ServerTURNIP`: evidence that a TURN relay was involved.
- `LinkId`: individual link identity.
- `UdpUse`: documented transport-use classification when populated.

**Use it for:** distinguishing direct UDP, TURN relay, TCP fallback, and unknown transport. Treat this table as link evidence and combine it with the connection record rather than interpreting one link row in isolation.

### `WVDConnectionNetworkData`

**Role:** in-session estimated network quality.

**Important fields:**

- `CorrelationId`: joins measurements to a connection.
- `EstRoundTripTimeInMs`: estimated AVD RTT.
- `EstAvailableBandwidthKBps`: estimated available bandwidth.
- `TimeGenerated`: measurement timestamp.

**Use it for:** average and percentile RTT, lower-end bandwidth, 30-minute complaint windows, protocol comparisons, host comparisons, and user-level network evidence.

Recommended interpretation:

- P95 RTT above 250 ms: high-risk tail latency.
- P95 RTT above 120 ms: warning-level latency.
- P10 bandwidth below 5 Mbps: high-risk constrained tail.
- P10 bandwidth below 10 Mbps: warning-level constrained tail.
- Low sample coverage: unknown, not healthy.

### `WVDErrors`

**Role:** AVD error activity correlated to connections and attempts.

**Important fields:**

- `CorrelationId`: ties errors to a connection.
- `CodeSymbolic` or `Code`: error identity.
- `Message`: contextual error text.
- `ActivityType`, `Operation`, and `Source`: error classification.
- `ServiceError`: separates service-originated errors from environment/configuration errors.
- `TimeGenerated`: incident timing.

**Use it for:** top errors by affected attempts, error trends, user/device groups, session error details, and deciding whether an error pattern is service-side or environment-side.

Count distinct `CorrelationId` values when measuring affected attempts. Raw error counts can overstate impact when one session retries repeatedly.

### `WVDCheckpoints`

**Role:** connection and logon phase checkpoints.

**Important fields:**

- `CorrelationId`: joins phases to a session.
- `Name`: checkpoint type, including `LogonDelay`.
- `Parameters`: dynamic phase measurements.
- `Source` and `ActivityType`: checkpoint context.
- `UserName` and `TimeGenerated`: user and timing correlation.

**Use it for:** authentication, Group Policy, profile, FSLogix, session environment, terminal services, and shell-start timing. A missing `LogonDelay` result means the telemetry is unavailable for that range.

### `WVDAgentHealthStatus`

**Role:** session-host health and active-session state.

**Important fields:**

- `EndpointState`: overall endpoint health.
- `SessionHostHealthCheckResult`: individual checks.
- `SessionHostName`: host identity.
- `AgentVersion` and `SxSStackVersion`: version comparison.
- `ActiveSessions`: host session count.
- `TimeGenerated`: latest health evidence.

**Use it for:** domain trust, FSLogix, SxS stack, URL checks, WebRTC redirector, and other health-check failures. An unhealthy host is a remediation target, but it does not by itself prove application causality.

### `WVDConnectionGraphicsDataPreview`

**Role:** optional graphics and frame-delivery quality.

**Important fields:**

- `ServerSkippedFramesPercentage`: frames skipped because the server was busy.
- `NetworkSkippedFramesPercentage`: frames skipped because of network conditions.
- `ClientSkippedFramesPercentage`: frames skipped during client decoding.
- `EncodingDelayOnServerInMs`, `DecodingTimeOnClientInMs`, and `RenderingTimeOnClientInMs`: end-to-end rendering components.
- `EndToEndDelayInMs`: highest observed frame end-to-end delay in the interval.

**Use it for:** separating network frame loss from server encoding and client decoding/rendering delay. This is preview telemetry; no rows mean unavailable evidence, not healthy graphics.

### `WVDFeeds`

**Role:** workspace feed and published-resource retrieval.

**Important fields:** `UserName`, `ClientType`, `ClientVersion`, `RDPFail`, `RDPTotal`, `IconFail`, `IconTotal`, and `TimeGenerated`.

**Use it for:** users who cannot see, refresh, or launch published desktops or RemoteApps. It diagnoses resource discovery, not in-session application speed.

### `WVDHostRegistrations`

**Role:** session-host registration activity.

**Important fields:** `SessionHostName`, `SessionHostIPAddress`, `IsSessionHostPrivateLink`, `CorrelationId`, and `TimeGenerated`.

**Use it for:** hosts added, rebuilt, re-registered, or changed near the start of an incident. Registration activity is context for host readiness, not a direct user-experience metric.

### `WVDManagement`

**Role:** AVD management operations and resource changes.

**Important fields:** `ArmObjectScope`, `Route`, `ObjectsCreated`, `ObjectsUpdated`, `ObjectsDeleted`, `ProvisioningCorrelationId`, `CorrelationId`, `UserName`, and `TimeGenerated`.

**Use it for:** finding resource changes before a regression and linking top-level provisioning operations to session-host management records.

### `WVDSessionHostManagement`

**Role:** session-host provisioning and update activity.

**Important fields:** `ProvisioningType`, `ProvisioningStatus`, `FromSessionHostConfigVer`, `ToSessionHostConfigVer`, `FromInstanceCount`, `ToInstanceCount`, `UpdateMethod`, `ScheduledDateTime`, and `CorrelationId`.

**Use it for:** image/configuration rollouts, host-count changes, failed provisioning, canary behavior, and identifying the deployment window to compare with affected users.

### `WVDAutoscaleEvaluationPooled`

**Role:** pooled-host Autoscale decisions.

**Important fields:** `ResultType`, `ConfigScheduleName`, `ConfigSchedulePhase`, `SessionCount`, `ActiveSessionHostCount`, `TotalSessionHostCount`, `SessionOccupancyPercent`, `ConfigCapacityThresholdPercent`, `UnhealthySessionHostCount`, `ScalingReasonMessage`, `ScalingPlanResourceId`, and `CorrelationId`.

**Use it for:** time-of-day complaints, capacity transitions, host start/deallocation behavior, occupancy pressure, unhealthy capacity, and Autoscale failures. For failed evaluations, join to `WVDErrors` on `CorrelationId`.

## Investigation order

1. Confirm all 12 tables in **Diagnostic coverage and data freshness**.
2. Find the complaint window in **30-minute AVD network risk at complaint time**.
3. Compare affected users with peers.
4. Determine whether risk clusters by host, gateway, client, location, or transport.
5. Inspect logon, health, graphics, and error evidence.
6. If timing aligns with an operational change, inspect feeds, registrations, management, provisioning, and Autoscale.
7. Validate the affected application's own telemetry before assigning application causality.
8. Test one remediation at a time and repeat the same measurement.

## How the connection-report queries work

The workbook queries follow a consistent pattern so the result can be traced back to individual AVD connection records.

### 1. Apply the workbook scope

Most queries begin with a time filter and the selected workbook parameters:

```kusto
WVDConnections
| where TimeGenerated {TimeRange}
| where ('All' in ({HostPoolFilter}) or HostPool in ({HostPoolFilter}))
| where ('All' in ({GatewayFilter}) or Gateway in ({GatewayFilter}))
| where ('All' in ({UserFilter}) or User in ({UserFilter}))
```

`{TimeRange}`, `{HostPoolFilter}`, `{GatewayFilter}`, `{UserFilter}`, and `{TransportFilter}` are replaced by Azure Workbooks with the current parameter values. Do not paste the braces into Log Analytics as standalone KQL; they are workbook parameters.

### 2. Build one connection row

Connection-based panels use this pattern:

```kusto
WVDConnections
| where TimeGenerated {TimeRange}
| where State == "Connected"
| summarize arg_max(TimeGenerated, *) by CorrelationId
```

`arg_max` keeps the latest record for each connection so state and client fields are not duplicated. `CorrelationId` is the connection-level key used to join network, transport, and error data.

### 3. Classify transport

The transport logic combines `WVDConnections` with `WVDMultiLinkAdd`. The workbook classifies a connection in this order:

1. TURN evidence or `UdpUse=4` becomes **UDP via TURN**.
2. Shortpath, Multipath, direct-link, or `UdpUse=1/2` evidence becomes **UDP direct**.
3. WebSocket or TCP transport without UDP evidence becomes **TCP fallback**.
4. Anything without decisive evidence becomes **Unknown**.

This ordering prevents a connection with both historical and current link records from being incorrectly labelled as direct UDP when TURN evidence is present. Unknown is an evidence state, not a healthy result.

### 4. Join network measurements

Network panels join `WVDConnectionNetworkData` to the filtered connection set on `CorrelationId`:

```kusto
WVDConnectionNetworkData
| where TimeGenerated {TimeRange}
| join kind=inner Connections on CorrelationId
| summarize AvgRTTms=avg(EstRoundTripTimeInMs),
    P95RTTms=percentile(EstRoundTripTimeInMs,95),
    P10BandwidthMbps=percentile(EstAvailableBandwidthKBps,10) * 8.0 / 1000
    by CorrelationId
```

The bandwidth conversion is `KBps * 8 / 1000 = Mbps`. P95 RTT exposes the slow tail; P10 bandwidth exposes the constrained tail. Network coverage is calculated separately as the share of sessions with at least one network sample.

### 5. Count errors by impact

Error panels join `WVDErrors` to the connection set on `CorrelationId`. They report both:

- **Errors:** raw error records, useful for understanding message volume.
- **AffectedAttempts:** distinct connection IDs, useful for measuring how many attempts were affected.

Use affected attempts to rank impact. A single retrying session can create many raw error records without representing many affected users.

### 6. Explain each workbook query family

| Query family | Tables and method | What the result answers |
| --- | --- | --- |
| **Transport health summary** | `WVDConnections` + `WVDMultiLinkAdd`; counts classified connected sessions | What percentage used direct UDP, TURN, TCP fallback, or unknown transport? |
| **Transport trend** | The same transport classification grouped into time buckets | Did the transport mix change during the incident? |
| **Session-host transport and bandwidth** | Connections joined to `WVDConnectionNetworkData`, grouped by host | Is one session host carrying worse transport, RTT, bandwidth, or coverage? |
| **Host-pool connected time** | Connection start/disconnect states grouped by host pool and protocol | Are sessions on one protocol or pool shorter or less stable? |
| **Gateway transport mix** | Connections grouped by `GatewayRegion`, with network aggregates | Does one Azure gateway region have a different transport or network pattern? |
| **RTT and bandwidth trends** | Network samples joined to classified connections and binned over time | Is the path degrading at particular times rather than only in the overall average? |
| **Protocol comparison** | Connected duration plus network percentiles grouped by transport | Which observed path is faster and more stable in this scope? |
| **Client OS and version mix** | Connections grouped by `ClientOS`, `ClientType`, and `ClientVersion` | Is TCP fallback or poor network quality concentrated in one client population? |
| **Location analysis** | Network samples joined to approximate client-IP geolocation | Does a regional client pattern exist? Locations are approximate and may be unavailable. |
| **Logon phases** | `WVDConnections` joined to `WVDCheckpoints` where `Name == "LogonDelay"` | Which authentication, GPO, profile, FSLogix, or shell phase is slow? |
| **Graphics performance** | `WVDConnectionGraphicsDataPreview` summarized by host pool | Are skipped frames or encode/decode/render delays server, network, or client-side? |
| **Errors** | `WVDErrors` joined to connections and grouped by code, origin, user, device, or time | Which errors affect the most attempts, and do they align with network conditions? |
| **Users without UDP** | User-level transport classification with an error join | Which users never establish an observed UDP path, and are errors also present? |
| **Session detail** | One connection row joined to links, network samples, and errors | What exactly happened for a selected user's individual sessions? |

### 7. Read aggregate results carefully

- `count()` counts rows or events; `dcount(CorrelationId)` counts distinct attempts.
- `avg()` is useful for overall direction but can hide a degraded tail.
- `percentile(...,95)` and `percentile(...,10)` expose tail behavior; use them with sample counts and coverage.
- `make_set()` provides examples, not a complete inventory.
- `arg_max()` chooses the latest record in a group; it does not reconstruct every state transition.
- `join kind=inner` keeps only records with a matching connection; `leftouter` preserves the base connection even when enrichment is missing.
- Empty or missing enrichment is an evidence gap. It should not be converted to healthy or zero without an explicit reason.

For official KQL syntax, see [Kusto Query Language overview](https://learn.microsoft.com/en-us/kusto/query/), [join operator](https://learn.microsoft.com/en-us/kusto/query/join-operator), [summarize operator](https://learn.microsoft.com/en-us/kusto/query/summarize-operator), [arg_max aggregation](https://learn.microsoft.com/en-us/kusto/query/arg-max-aggregation-function), and [percentile aggregation](https://learn.microsoft.com/en-us/kusto/query/percentiles-aggregation-function).

## Deployment

```powershell
.\Deploy-Workbook.ps1 `
    -ResourceGroupName '<resource-group>' `
    -WorkbookFileName 'AVD Connection Report - Vanguard' `
    -DisplayName 'Vanguard AVD Transport Investigation'
```

## Sources

- [Microsoft Learn: Azure Monitor WVD tables](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvd)
- [Microsoft Learn: WVDConnections](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdconnections)
- [Microsoft Learn: WVDMultiLinkAdd](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdmultilinkadd)
- [Microsoft Learn: WVDConnectionNetworkData](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdconnectionnetworkdata)
- [Microsoft Learn: WVDErrors](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvderrors)
- [Microsoft Learn: WVDCheckpoints](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdcheckpoints)
- [Microsoft Learn: WVDAgentHealthStatus](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdagenthealthstatus)
- [Microsoft Learn: WVDConnectionGraphicsDataPreview](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdconnectiongraphicsdatapreview)
- [Microsoft Learn: WVDFeeds](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdfeeds)
- [Microsoft Learn: WVDHostRegistrations](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdhostregistrations)
- [Microsoft Learn: WVDManagement](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdmanagement)
- [Microsoft Learn: WVDSessionHostManagement](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdsessionhostmanagement)
- [Microsoft Learn: WVDAutoscaleEvaluationPooled](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdautoscaleevaluationpooled)
- [Microsoft Learn: Configure RDP Shortpath](https://learn.microsoft.com/en-us/azure/virtual-desktop/configure-rdp-shortpath)
