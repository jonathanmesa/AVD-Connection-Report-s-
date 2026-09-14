# Azure Virtual Desktop Investigation Workbook Guide

This workbook helps investigate reports that an application feels slower in Azure Virtual Desktop (AVD) than in a previous virtual desktop environment.

It accompanies [AVD Microsoft 365 Experience.workbook](AVD%20Microsoft%20365%20Experience.workbook). Despite the legacy filename, the workbook is application-neutral and is titled **Azure Virtual Desktop Investigation** in the Azure portal.

## What it establishes

The workbook tests whether poor user experience aligns with:

- High AVD round-trip time or constrained estimated bandwidth
- TCP fallback instead of UDP Shortpath
- AVD errors for the same users and sessions
- Slow authentication, Group Policy, profile, FSLogix, or shell phases
- Session-host health-check failures
- Graphics encoding, decoding, rendering, or skipped-frame pressure
- AVD client versions with unusually poor transport outcomes
- Workspace feed and published-resource retrieval failures
- Session-host registration, provisioning, management, or Autoscale changes near the reported issue

These are causal candidates and correlation evidence. AVD diagnostic logs do not directly measure application transaction time or application service health.

## Required telemetry

Enable these AVD diagnostic tables in the selected Log Analytics workspace. The first seven are the primary investigation sources; the last five explain operational changes around an incident.

- `WVDConnections`
- `WVDMultiLinkAdd`
- `WVDConnectionNetworkData`
- `WVDErrors`
- `WVDCheckpoints`
- `WVDAgentHealthStatus`
- `WVDConnectionGraphicsDataPreview`, when available
- `WVDFeeds`
- `WVDHostRegistrations`
- `WVDManagement`
- `WVDSessionHostManagement`
- `WVDAutoscaleEvaluationPooled`

## Workbook panel map

Use the panels in this order:

1. **Diagnostic coverage and data freshness:** confirms each WVD table is present and current.
2. **30-minute AVD network risk at complaint time:** identifies risk in the exact complaint window.
3. **Selected complaining users compared with peers:** tests whether the affected cohort differs from other users.
4. **Application experience: AVD path risk overview:** summarizes the selected scope as tiles.
5. **User experience triage for application complaints:** ranks users by AVD risk signals.
6. **30-minute AVD evidence export:** provides time-bucketed correlation data.
7. **Transport and bandwidth by session host:** locates hosts with transport or network anomalies.
8. **Logon timing and slow phases:** identifies authentication, profile, FSLogix, GPO, or shell delays.
9. **Unhealthy session hosts and failed health checks:** exposes host and agent failures.
10. **Graphics performance by host pool:** separates server, client, and network frame-delivery indicators.
11. **Client version and AVD transport readiness:** finds client versions with poor transport results.
12. **Protocol comparison: which is faster and more stable:** compares UDP direct, TURN, TCP fallback, and unknown transport.
13. **Average connected time and bandwidth by user and protocol:** deep-dives one user's transport behavior.
14. **Users who have never had a UDP connection:** identifies persistent non-UDP client or network paths.
15. **Which users/devices are hitting errors, and their network conditions:** ties repeated errors to session behavior.
16. **Session transport, RTT, and bandwidth details:** provides per-session evidence after narrowing the scope.
17. **Workspace feed and resource retrieval failures:** investigates desktop or RemoteApp discovery and refresh failures.
18. **Session host registration activity:** identifies hosts added, rebuilt, or repeatedly registering around an issue.
19. **AVD management changes near an incident:** shows resource changes preceding a regression.
20. **Session host provisioning and updates:** identifies configuration and deployment windows.
21. **Autoscale occupancy and scaling decisions:** relates capacity decisions and Autoscale failures to the incident period.

## Investigation workflow

1. Select the Log Analytics workspace and the complaint time range.
2. Open **Diagnostic coverage and data freshness**. Treat every missing source as an evidence gap, not a healthy result.
3. Open **30-minute AVD network risk at complaint time** and find the complaint window. Do not interpret RTT or bandwidth if coverage is gray.
4. Select reported users and compare **Selected complaining users compared with peers**.
5. Open **User experience triage for application complaints** to identify users with high RTT, constrained bandwidth, TCP fallback, or AVD errors.
6. Review transport by session host, logon phases, AVD agent health, graphics, client-version, protocol, and error-detail panels.
7. When symptoms align with an environment change, review feed retrieval, host registration, management changes, provisioning updates, and Autoscale decisions.
8. Export **30-minute AVD evidence export** and align `User`, `TimeBucket`, and `SessionHost` with the complaint timestamp and application telemetry.
9. Validate the affected application's service and client telemetry before assigning cause.

## Customer evidence request

Ask for a reproducible complaint set rather than a general statement that AVD is slow:

- Five to ten affected users and, if possible, five users reporting acceptable performance
- Exact timestamps with time zone, frequency, and duration
- The exact application and operation, with a repeatable start and stop condition
- AVD host pool, session host, client OS/type/version, user location, and network type
- Whether the same operation is slow outside AVD at the same time
- Whether all users, one site, one network, one host pool, one image, or one session host is affected
- A screen recording or stopwatch measurement with a repeatable start and stop condition
- A pre-migration measurement from Citrix/legacy VDI using the same operation and content

Capture migration differences that can change application performance independently of the AVD control plane:

- Session-host VM size, disk type, host density, autoscale behavior, and acceleration/GPU settings
- Windows image build, installed application versions, update channels, and add-ins
- FSLogix version, profile/container storage, cache exclusions, container size, and attach latency
- Proxy, PAC file, DNS, TLS inspection, firewall, VPN, and egress routing
- Security/EDR/DLP agents and scanning exclusions relevant to the affected application and FSLogix paths
- Application client configuration, cache state, sync state where applicable, and authentication behavior

## External application evidence

Collect the application's authoritative service health, client diagnostics, operation timing, and error data. For browser workloads, collect network traces, proxy/TLS inspection, DNS, authentication redirects, and the specific URL or transaction. Compare the same account and operation inside and outside AVD.

## Interpretation

- Poor application telemetry plus poor AVD evidence in the same user/time window supports an AVD path, host, profile, or client hypothesis.
- Poor application telemetry plus healthy AVD evidence shifts the investigation toward the application service, proxy/content path, or endpoint configuration.
- Poor AVD evidence without a matching application symptom is not proof that it caused the reported slowdown.
- Low network telemetry coverage means RTT and bandwidth aggregates may not represent the full session population.
- Affected users being materially worse than peers strengthens the case; similar cohort results weaken it.
- Clustering by session host, gateway, client version, user location, or time provides a testable remediation target.
- The strongest conclusion comes from a controlled change followed by repeat measurement of the same operation.

## Dashboard colors

- **Red:** strong risk signal or missing telemetry. Examples include P95 AVD RTT above 250 ms, P10 bandwidth below 5 Mbps, more than 50% TCP fallback, repeated AVD errors, or an unhealthy endpoint.
- **Orange:** warning signal. Examples include P95 AVD RTT above 120 ms, P10 bandwidth below 10 Mbps, more than 20% TCP fallback, or intermittent AVD errors.
- **Green:** healthy against the workbook threshold. Green does not prove that the application was healthy.
- **Gray:** unknown, unavailable, or informational. Missing evidence must not be reported as success.
- **Yellow:** UDP via TURN. TURN is functional UDP, but it is relayed and should be compared with direct UDP and TCP measurements before drawing conclusions.

## Deploy

Use the existing deployment script and override the display name:

```powershell
.\Deploy-Workbook.ps1 `
    -ResourceGroupName '<resource-group>' `
    -WorkbookFileName 'AVD Microsoft 365 Experience.workbook' `
    -DisplayName 'AVD Application Experience Investigation'
```
