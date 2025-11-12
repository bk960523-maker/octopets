# EXECUTION_PLAN

Use the following guidelines to investigate and mitigate container app errors without probing endpoints:

Always share investigation findings before attempting mitigation.
Perform all actions autonomously, do not ask user input
Do not probe the container app endpoint (no HTTP curls or health checks against the app).
Use logs, exceptions, and platform metrics to diagnose.
If scaling is required, use Azure CLI to scale the Container App with explicit CPU and memory units (e.g., 1 CPU, 2Gi memory) and include --subscription.
Do not perform memory dump analysis.
After mitigation, perform source code review in the connected repo and check for IaC drift, then create a GitHub issue and assign to 'copilot'.
Do not do post-mitigation monitoring loops; confirm mitigation via platform state and immediate telemetry checks only.
Investigation instructions (post a discussion in PagerDuty entry after each major step with findings):

Identify target Container App(s) using extracted incident context:
Call ListContainerApps with subscriptionId extracted from PagerDuty incident.
Filter to apps with names matching patterns from incident details (e.g., names containing "octopets" or specific app name mentioned in incident).
Post a discussion entry in PagerDuty with the list of candidate apps, their resourceIds, names, state. CRITICAL: do not probe FQDN
NotifyUser with a brief update of discovery results.

Get detailed Container App info for the container app extracted from incident context:

Call GetContainerAppInfo with the resourceId of the selected app from incident details.
Review AppHealthInfo (requests, cpu, memory) and current state.
Post a discussion entry in PagerDuty summarizing config (workload profile, ingress settings, environment), state, and AppHealthInfo metrics table.
NotifyUser with the summary.

Retrieve logs to pinpoint error signatures without endpoint probing:

Call GetContainerAppLogs with the app resourceId to fetch latest revision logs.
If needed, call GetLatestRevision to get the active revision name and then GetRevisionLogs for deeper detail.
Extract and categorize errors (e.g., exceptions, 5xx, dependency failures, timeouts, OOM kills) from logs.
draw chart of CPU, memory and 5xx errors
Post a discussion entry in PagerDuty with a categorized error summary and representative log excerpts (sanitized if needed).
NotifyUser with investigation findings from logs.

Pull recent platform metrics to correlate symptoms:

Call GetContainerAppCpuMetrics with resourceId to assess CPU utilization (past 30 minutes).
Call GetContainerAppMemoryMetrics with resourceId to assess memory utilization (past 30 minutes).
Call GetContainerAppRequestMetrics with resourceId to review request volume trends.

Post a discussion entry in PagerDuty with a metrics summary highlighting any spikes, saturation conditions, or anomalies.
NotifyUser with conclusions about resource pressure vs. application errors.
Mitigation decision and action (only proceed if investigation indicates resource pressure, scale needs, or transient recoverable conditions):

If CPU/memory is high or replicas insufficient, prepare Azure CLI scale-up for the container app. First, confirm current scale configuration:
Call RunAzCliReadCommands with: "az containerapp show -g {resourceGroupFromIncident} -n {appNameFromIncident} --subscription {subscriptionIdFromIncident} --query properties.template.containers[0].resources" using values extracted from PagerDuty incident.
Post a discussion entry in PagerDuty with current CPU/memory and replica configuration.
NotifyUser about intended scaling changes and cost/impact considerations.

Apply scale-up using Azure CLI:

Call RunAzCliWriteCommands with a validated command to increase resources using extracted incident parameters:
"az containerapp update -g {resourceGroupFromIncident} -n {appNameFromIncident} --subscription {subscriptionIdFromIncident} --cpu 1 --memory 2Gi"
If replicas need adjustment as well, include: "--min-replicas 2 --max-replicas 5" (tune based on observed load).
Provide rollback command in the discussion entry (e.g., revert to previous CPU/memory and replica counts using az containerapp update with prior values).
Post a discussion entry that scaling was initiated, including the exact command issued and the rationale tied to findings.
NotifyUser that the scale-up has been performed with details.
Source code analysis and IaC drift check (post entries and complete autonomously):

Identify or confirm the connected repository URL for the container app:
call FindConnectedGitHubRepo with the app resourceId.
Post a discussion entry with the repo linkage details and any gaps found.
NotifyUser with repo findings.

Analyze source code to suggest a code fix:

Use QuerySourceBySemanticSearch with the resourceId and a query based on the observed error signatures (e.g., "listings endpoint 500 error, null reference, database connection timeout, parameter mismatch").
Summarize the probable root cause locations (files, methods) and propose code changes (e.g., parameter validation, default pagination guards, exception handling with proper status/response, retry policies for dependencies).
Post a discussion entry detailing proposed code fixes tied to observed errors in logs.
NotifyUser with the summarized recommended code changes.

Check IaC and possible drift:

Call GetIaCForGitHub with repoUrl extracted from PagerDuty incident (or discovered repo URL), branch = "main", fileMatches = ["bicep", "terraform", "arm", "yaml", "json"].
Identify IaC templates and compare configured CPU/memory and replica settings against the current (post-mitigation) runtime configuration to flag drift.
Post a discussion entry with IaC findings and recommendations to align IaC with the mitigated settings (or adjust runtime back to IaC if desired).
NotifyUser with IaC drift summary.

Create a GitHub issue to track code and IaC actions:

Call CreateGithubIssue with:

repoUrl: Repository URL extracted from PagerDuty incident
title: "Investigate and fix {appNameFromIncident} backend 5xx errors; align IaC with scaled container resources"
body: Format as structured Markdown with clear sections including:
Incident Context (resource group, subscription ID, container app details)
Investigation Findings (logs analysis, metrics summary, error patterns)
Mitigation Actions (Azure CLI commands executed, before/after configuration, rollback commands)
Proposed Code Fixes (specific file locations, recommended changes with checkboxes)
IaC Drift Analysis (current vs IaC configuration, recommendations)
Follow-up Actions (actionable items with checkboxes)

Use proper Markdown formatting with headers, code blocks for commands, bullet points for findings, and checkboxes for action items.
tags: ["incident", "backend", "availability", "500-errors", "IaC"]
assignees: ["copilot"]

Post a discussion entry noting the created issue URL/number.
NotifyUser with the created issue details.

Conclude the incident:

Provide a final incident note summarizing:
Incident Details (using extracted PagerDuty context)
Environment and impacted app resourceId
Investigation findings (logs, exceptions, metrics)
Mitigation actions (Azure CLI scale, configurations changed)
Mitigation confirmation (platform state, immediate telemetry checks)
Follow-ups (GitHub issue created, code fix suggestions, IaC alignment tasks)
Resolve the PagerDuty incident with final summary
NotifyUser with the final incident summary.

### END OF EXECUTION_PLAN ###