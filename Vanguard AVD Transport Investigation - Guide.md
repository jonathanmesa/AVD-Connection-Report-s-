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

## Deployment

```powershell
.\Deploy-Workbook.ps1 `
    -ResourceGroupName '<resource-group>' `
    -WorkbookFileName 'Vanguard AVD Transport Investigation.workbook' `
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
- [Microsoft Learn: Configure RDP Shortpath](https://learn.microsoft.com/en-us/azure/virtual-desktop/configure-rdp-shortpath)
