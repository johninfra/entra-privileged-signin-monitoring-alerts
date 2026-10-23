# Architecture and Design Decisions

## Flow

1. Microsoft Entra records interactive user sign-ins.
2. An Entra **tenant-level diagnostic setting** routes `SignInLogs` to the Azure Log Analytics workspace.
3. `SigninLogs` becomes queryable using KQL after ingestion.
4. An Azure Monitor scheduled query rule detects matching events.
5. The rule triggers an action group configured for email delivery.

**Important:** Diagnostic settings on the Log Analytics workspace itself describe telemetry about the workspace. They are not required to route Entra identity logs into the workspace.

## Boundary of the validated implementation

- **Verified:** Workspace deployment completed; Entra diagnostic setting selected `AuditLogs` and `SignInLogs`; `SigninLogs` query returned successful Azure Portal sign-in records.
- **Not yet verified:** Alert rule creation, firing, end-to-end email delivery, both emergency accounts covered, and dynamic Global Administrator coverage.
- **Earlier issue:** A write attempt was denied for `microsoft.insights/scheduledqueryrules/write`. Before retrying, confirm active Azure RBAC at the correct scope and refresh the sign-in session.

## Identity and permissions

The identity viewing logs may use a read-only workspace role such as **Log Analytics Reader**. Creating alert rules requires permissions to write scheduled query rules in the selected resource group, as well as required read access to the workspace and action group. A Microsoft Entra **Global Administrator** role is not equivalent to Azure subscription RBAC.

The starter alert can use the normal workspace access path. Managed identity is an option when a query needs explicitly controlled access to additional resources; grant required read permissions and validate execution.

## Detection design

Emergency account detection uses Entra user Object IDs in a private configuration, not account display names or UPNs. Account renames do not change Object IDs.

For **all Global Administrators**, implement an automated, timestamp-aware role-inventory pipeline (e.g. Microsoft Graph with PIM state and assignment history) and join that inventory to sign-in events. Current role membership alone cannot prove historical role membership. Do not label a static Object ID list as comprehensive Global Administrator coverage.

## Alert behavior

A log search alert may evaluate over a rolling window. Choose an alert mode and query window appropriate to ingestion latency, and test for duplicate notifications. A **Single event** rule targets matching log events but must still be validated for real-world delivery.

## Publishing safety

Screenshots and documentation should contain no live principal names, tenant domains, personal IP addresses, user Object IDs, subscription IDs, tokens, or recovery details. Use opaque, irreversible black-box redactions rather than blur for public evidence.
