# Azure Virtual Desktop Investigation Report

## Purpose and decision statement

This report investigates reported application slowdowns after a move from legacy VDI to Azure Virtual Desktop (AVD). It defines how to determine whether the issue is caused by the AVD delivery path, a migrated-session configuration, the endpoint, the application, or its service and content path.

No causal conclusion should be made from the current complaint statement alone. The deliverable from this investigation is a ranked evidence record for each complaint cohort, a remediation target, and a measured before/after outcome.

## Executive approach

1. Segment affected employees by application, location, endpoint, AVD host pool, session host, AVD client version, and complaint time.
2. Gather the affected application's operation time, failures, service health, and client diagnostic data. Do not use user perception alone.
3. Correlate the affected users and exact event times with AVD connection, network, transport, logon, error, health, and graphics events.
4. Compare each affected cohort to peer users using the same time window and, where possible, the same application operation outside AVD.
5. Change one suspected condition at a time for a pilot cohort; measure the same operation and the relevant application and AVD metrics.

## Time-correlation limitation

Use the exact complaint timestamp to correlate with AVD events, normally in 30-minute buckets for triage. An AVD event that occurs near an application complaint is a hypothesis. It becomes actionable when the condition repeats across affected users or disappears after a controlled change.

## Evidence model

| Evidence pattern | Interpretation | Priority action |
| --- | --- | --- |
| Poor application telemetry and poor AVD RTT, bandwidth, TCP fallback, graphics, errors, or logon events at the same time | AVD delivery, client, host, or profile hypothesis is strengthened | Segment by host, client version, gateway, location, and transport; pilot the targeted fix |
| Poor application telemetry, but healthy and well-covered AVD diagnostics | AVD is not exonerated absolutely, but the strongest next hypotheses are application service, proxy, DNS, TLS inspection, application configuration, or endpoint | Inspect the affected application's client and service telemetry |
| Poor AVD telemetry without a matching application symptom | An AVD condition exists but is not yet established as the reported cause | Monitor and remediate according to AVD operational priority; do not attribute the application complaint to it |
| Affected cohort is worse than peer cohort on the same host pool, client, and time period | Stronger, scoped remediation target | Change only the distinguishing condition and rerun the comparison |
| All cohorts show the same decline after a release or migration change | Broad change hypothesis | Correlate with image, AVD agent/stack, policy, proxy, security, and autoscale changes |
| Low diagnostic coverage | Unknown, not healthy | Enable or repair the missing AVD diagnostic category before relying on the dashboard |

## Complete AVD diagnostic catalog

The investigation uses the full Azure Monitor Azure Virtual Desktop catalog below. Tables marked **primary** belong in the complaint-time correlation path. Tables marked **context** explain a migration, provisioning, or resource-publication change that may precede the symptoms.

| Table | Role | Investigation use |
| --- | --- | --- |
| `WVDConnections` | Primary | Join key source: user, correlation ID, host pool, session host, client OS/type/version, gateway region, transport, reconnect lineage, and connection state |
| `WVDMultiLinkAdd` | Primary | Connection transport-link details; use with connections to expose UDP, TURN, or TCP fallback behavior |
| `WVDConnectionNetworkData` | Primary | Estimated round-trip time and available bandwidth by correlation ID and time |
| `WVDConnectionGraphicsDataPreview` | Primary, optional | End-to-end render delay, encode/decode/render time, bandwidth, and client/network/server skipped-frame signals |
| `WVDErrors` | Primary | Failure volume and error codes/messages for the same user, correlation ID, host, and period |
| `WVDCheckpoints` | Primary | Connection and logon checkpoints for authentication, profile, shell, and published-resource timing analysis |
| `WVDAgentHealthStatus` | Primary | Host health checks, active sessions, FSLogix, domain, URL, WebRTC redirector, stack, and related health state |
| `WVDFeeds` | Context | Workspace feed, RDP-file, and icon retrieval failures by user/client; investigate slow or incomplete resource discovery |
| `WVDHostRegistrations` | Context | Session-host registration events, host identity, and Private Link state; useful for new-host or registration-pattern changes |
| `WVDManagement` | Context | Management operations, route, object changes, and provisioning correlation IDs that precede a regression |
| `WVDSessionHostManagement` | Context | Provisioning and update status, session-host configuration version, instance-count changes, canary settings, and update method |
| `WVDAutoscaleEvaluationPooled` | Context | Autoscale evaluation, active-session count, occupancy, scaling decision, and reason; use when dissatisfaction clusters after capacity behavior changes |

`WVDConnectionGraphicsDataPreview` is preview telemetry. Treat its absence as unavailable evidence, not a healthy graphics result. `WVDAgentHealthStatus` is categorized as Virtual Machines in Azure Monitor but is essential AVD session-host telemetry.

## Correlation contract

Use these fields without attempting to join every record only on display name:

| Dataset | Required fields |
| --- | --- |
| External application telemetry | User identity, device identity, application name, operation, timestamp, latency or failure result, and session identity where available |
| AVD | `UserName`, `CorrelationId`, `TimeGenerated`, `SessionHostName`, host-pool resource ID, client version/type/OS, `GatewayRegion`, and transport type |

Normalize UPN casing and time zones. Start with `UserName` plus a bounded time window, then enrich with `CorrelationId` and `SessionHostName`. Never treat an IP address or a display name alone as a durable identity join.

## Required application analysis

For every affected cohort, collect:

- Exact operation timing, failure details, and client diagnostic events.
- Application service health and applicable tenant or regional incident records.
- Browser network traces, proxy, DNS, identity, and TLS-inspection evidence for web workloads.
- Endpoint and virtual-session context needed to reproduce the issue.

## AVD query sequence

Use the included [AVD Microsoft 365 Experience.workbook](AVD%20Microsoft%20365%20Experience.workbook) for all 12 WVD sources and its 21 investigation panels. The [workbook guide](AVD%20Microsoft%20365%20Experience%20-%20Guide.md) maps every panel to its purpose and investigation order. Run these query examples only when standalone analysis is required.

```kusto
let StartTime = ago(7d);
let EndTime = now();
WVDFeeds
| where TimeGenerated between (StartTime .. EndTime)
| extend FailureRate = 100.0 * todouble(RDPFail + IconFail) / todouble(max_of(RDPTotal + IconTotal, 1))
| summarize Requests = count(), FailedRdp = sum(RDPFail), FailedIcons = sum(IconFail), P95FailureRate = percentile(FailureRate, 95)
    by UserName, ClientType, ClientVersion, ClientOS
| order by P95FailureRate desc, Requests desc
```

```kusto
let StartTime = ago(7d);
let EndTime = now();
WVDHostRegistrations
| where TimeGenerated between (StartTime .. EndTime)
| summarize Registrations = count(), FirstSeen = min(TimeGenerated), LastSeen = max(TimeGenerated)
    by SessionHostName, IsSessionHostPrivateLink
| order by LastSeen desc
```

```kusto
let StartTime = ago(7d);
let EndTime = now();
WVDManagement
| where TimeGenerated between (StartTime .. EndTime)
| project TimeGenerated, UserName, Route, ArmObjectScope, ObjectsCreated, ObjectsUpdated, ObjectsDeleted, ProvisioningCorrelationId, CorrelationId
| order by TimeGenerated desc
```

```kusto
let StartTime = ago(7d);
let EndTime = now();
WVDSessionHostManagement
| where TimeGenerated between (StartTime .. EndTime)
| project TimeGenerated, CorrelationId, ProvisioningType, ProvisioningStatus, FromInstanceCount, ToInstanceCount,
    FromSessionHostConfigVer, ToSessionHostConfigVer, UpdateMethod, ScheduledDateTime
| order by TimeGenerated desc
```

```kusto
let StartTime = ago(7d);
let EndTime = now();
WVDAutoscaleEvaluationPooled
| where TimeGenerated between (StartTime .. EndTime)
| project TimeGenerated, SessionCount, SessionOccupancyPercent, MaxSessionLimitPerSessionHost,
    ResultType, ScalingReasonMessage, ScalingPlanResourceId, Properties
| order by TimeGenerated desc
```

Before publishing any supplemental query, run `getschema` or inspect a small sample in the target workspace to confirm field availability; Azure table schemas can evolve.

## Remediation hypotheses and tests

| Hypothesis | Evidence to require | Controlled remediation test | Success measure |
| --- | --- | --- | --- |
| UDP Shortpath is unavailable or TCP fallback is common | High TCP share or repeated degraded transport for affected users; poor RTT/bandwidth relative to peers | Validate network/firewall routing and UDP path for a pilot location | Higher direct-UDP share, lower AVD RTT, and no application-operation regression |
| AVD client release/regression | Poor cohort clusters on one `ClientVersion` or client type | Pilot a supported current client version while retaining a comparison group | Better connection/graphics metrics and improved application operation time |
| Host/agent/stack/image change | Symptoms start after registration, management, session-host management, or agent-version change and concentrate on new hosts | Roll back or exclude a small set of affected hosts; validate image and agent/stack consistency | Lower affected-user rate and restoration of the application operation time |
| Profile or session initialization issue | Logon checkpoints or agent health expose FSLogix, domain, URL, or stack failures | Correct only the identified profile, name-resolution, or health issue for a cohort | Faster visible-desktop/logon phase and improved application operation time |
| Render/graphics issue | High end-to-end delay or skipped frames; client/server/network pattern differentiates affected cohort | Test client, display, codec, policy, or supported graphics configuration in a pilot | Reduced frame delay/skips and improved input/responsiveness feedback |
| Autoscale or capacity behavior change | Dissatisfaction aligns with autoscale decision/occupancy and specific sessions/hosts | Adjust only the relevant autoscale schedule, buffer, or host allocation for a pilot period | Fewer complaints and a stable affected-cohort score trend |
| Application, proxy, DNS, security, or content-path cause | AVD data is healthy or does not covary; application nodes identify page load, transaction, or web reliability impact | Compare direct/bypass-approved path and identical operation outside AVD; inspect the applicable service health | Improved application raw timings and application-score impact without relying on an AVD change |

## 30-day execution plan

| Period | Work | Exit criterion |
| --- | --- | --- |
| Days 1-3 | Verify diagnostic settings for all 12 tables; define affected and peer cohorts; capture five to ten reproducible complaints | Every selected source has a coverage result and every complaint has user, timestamp, application operation, location, and session context |
| Days 4-7 | Establish pre/post migration baseline; run workbook and supplemental queries; identify application and AVD evidence patterns | Ranked hypotheses with evidence coverage and a named owner per test |
| Days 8-14 | Execute one or two low-risk pilot changes; preserve a comparable control cohort | Exact operation measured before and after; no broad, simultaneous change obscures the result |
| Days 15-21 | Expand a successful pilot or reject the hypothesis; investigate next ranked path | Improvement replicated across more than one time period or cohort |
| Days 22-30 | Stabilize configuration and document the final target state | Trend shows sustained improvement in application timing, AVD evidence, and complaint volume |

## Customer-ready success criteria

Declare improvement only when all are true for the pilot cohort:

- The named application operation improves against the pre-change baseline.
- AVD evidence shows the expected associated improvement, or explicitly demonstrates that the solved issue was outside AVD.
- The result is repeatable for comparable users, locations, and operating periods.
- No compensating regression appears in connection failures, logon experience, collaboration quality, or user-reported satisfaction.

## Sources

- [Microsoft Learn: Azure Monitor WVD tables](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvd)
- [Microsoft Learn: WVDConnections](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdconnections)
- [Microsoft Learn: WVDConnectionNetworkData](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdconnectionnetworkdata)
- [Microsoft Learn: WVDConnectionGraphicsDataPreview](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdconnectiongraphicsdatapreview)
- [Microsoft Learn: WVDCheckpoints](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/wvdcheckpoints)
