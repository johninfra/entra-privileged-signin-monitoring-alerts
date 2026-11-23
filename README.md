# Microsoft Entra Privileged Sign-In Monitoring & Alerts

Hands-on Azure identity-security lab demonstrating how to export Microsoft Entra sign-in events to Azure Monitor Logs, investigate them with Kusto Query Language (KQL), and document privileged account monitoring. The lab also validates native Microsoft Entra PIM activation emails and investigates PIM-managed directory-role changes in `AuditLogs`. Neither constitutes proof of Azure Monitor alert delivery.

## Objectives

- Centralize Microsoft Entra interactive sign-in telemetry in a Log Analytics workspace.
- Identify successful and failed sign-ins for monitored emergency access accounts.
- Build a log search alert with an Azure Monitor email action group.
- Validate native PIM role-activation email notifications with a controlled test activation.
- Examine privileged role membership and eligibility changes using `AuditLogs` and KQL.
- Verify the Azure Monitor detection and email-delivery path independently without exposing live tenant identifiers.
- Understand the limits of fixed account lists versus dynamic Global Administrator membership.

## Architecture

```mermaid
flowchart LR
    A[Microsoft Entra ID] --> B[Diagnostic setting: SignInLogs]
    B --> C[Azure Log Analytics workspace]
    C --> D[KQL detection]
    D --> E[Azure Monitor scheduled query alert]
    E --> F[Action group]
    F --> G[Email notification]
```

The diagram is the **Azure Monitor sign-in detection design**. Entra-to-Log-Analytics ingestion has been validated, while end-to-end scheduled-query alert firing and action-group email delivery remain unverified. **Native Entra PIM role-activation email notifications are a separate, validated notification path**, not evidence that the Azure Monitor design has fired.

## Technologies

Microsoft Entra ID · Azure Monitor · Log Analytics · KQL · Azure RBAC · Azure Monitor Action Groups · Privileged Identity Management (PIM)

## Implementation

1. **Provision a Log Analytics workspace** within an Azure subscription and resource group.
2. In **Entra admin center → Entra ID → Monitoring & health → Diagnostic settings**, select `SignInLogs` and `AuditLogs`, sending them to the workspace.
3. Confirm that new records arrive in the `SigninLogs` table using `queries/01-ingestion-health.kql`.
4. Use `queries/02-privileged-signins.kql` as a **template**, replacing placeholders with actual Entra user **Object IDs** in your private environment.
5. Create an Azure Monitor **Custom log search** alert for matching sign-ins.
6. Attach an email action group, configure severity and alert behavior, and test the notification.
7. Review rule health, permissions, coverage, cost, and the results of an authorized test sign-in.

## KQL query library

Beyond the three alert templates, the `queries/` folder holds the saved investigation queries used in the lab workspace. They are **investigation and hunting queries run manually in Log Analytics**, not deployed alert rules. They were run against a personal lab tenant with lab-scale volume, so thresholds are starting points rather than tuned production values. Live identifiers have been replaced with placeholders.

| # | Query | Question it answers | Source table(s) | Window |
|---|---|---|---|---|
| 01 | `01-ingestion-health.kql` | Are sign-in events reaching the workspace? | SigninLogs | 24h |
| 02 | `02-privileged-signins.kql` | Did a monitored privileged account sign in successfully? (alert template) | SigninLogs | 15m |
| 03 | `03-failed-privileged-signins.kql` | Did sign-ins to a monitored privileged account fail? (alert template) | SigninLogs | 15m |
| 04 | `04-brute-force-per-account.kql` | Is one account receiving 5+ failed password or lockout events from one IP in 15 minutes? | SigninLogs | 24h |
| 05 | `05-password-spray-per-ip.kql` | Is one IP failing against 3+ accounts in 15 minutes? | SigninLogs | 24h |
| 06 | `06-emergency-account-signins.kql` | Has any break-glass account signed in, successfully or not? | SigninLogs | 30d |
| 07 | `07-privileged-role-assignment-changes.kql` | Which role assignments and PIM activations changed, who initiated them, and for which principal? | AuditLogs | 30d |
| 08 | `08-identity-security-config-changes.kql` | Who changed Conditional Access, authentication methods, apps, service principals, or groups? | AuditLogs | 24h |
| 09 | `09-conditional-access-failures.kql` | Which sign-ins did Conditional Access block, and under which policies? | SigninLogs | 7d |
| 10 | `10-all-failed-signins.kql` | What failed sign-ins occurred, and why? | SigninLogs | 24h |
| 11 | `11-successful-signins-outside-us.kql` | Did anyone sign in successfully from outside the US? | SigninLogs, AADNonInteractiveUserSignInLogs | 30d |
| 12 | `12-all-successful-signins.kql` | Who signed in successfully, from where, and under which authentication requirement? | SigninLogs | 24h |
| 13 | `13-latest-signin-per-user.kql` | What is each user's latest sign-in, including device context? | SigninLogs | 24h |
| 14 | `14-recently-active-sessions.kql` | Who is active right now (last hour), from which apps and IPs? | SigninLogs, AADNonInteractiveUserSignInLogs | 1h |
| 15 | `15-inactive-users.kql` | Which users have not signed in successfully for 7+ days? | SigninLogs, AADNonInteractiveUserSignInLogs | 90d |

**Query-specific limits**

- **11, 14, 15** need the `NonInteractiveUserSignInLogs` diagnostic category in addition to `SignInLogs`.
- **15** returns only users with at least one successful sign-in inside the lookback window. Accounts that never signed in are not returned, so use Microsoft Graph `signInActivity` or access reviews for a complete stale-account inventory.
- **06** matches break-glass accounts by UPN prefix for investigation convenience. For alerting, use the Object ID templates in 02 and 03.
- **04 and 05** aggregate per IP. Distributed attacks spread across many IPs need different aggregation, such as per account over a longer window.

## Validated PIM activation email notification

A controlled Microsoft Entra Privileged Identity Management (PIM) activation generated an automatic Microsoft Security notification email. The message identifies the activating account, the **Privileged Role Administrator** directory role, the directory, and the **October 20, 2026, 19:22–20:22 UTC** activation window (one hour). This validates receipt of a native PIM role-activation email notification in the lab.

This event is a **directory-role activation**, not an interactive sign-in alert. It does **not** establish that Azure Monitor's KQL scheduled-query alert or its action group sent an email. These two monitoring paths must be tested and reported independently.

## Detection scope and limitations

**Emergency account monitoring:** Explicit Object IDs provide stable detection even if a monitored account's UPN changes. The repository intentionally contains placeholders, not live Object IDs.

**All Global Administrators:** `SigninLogs` alone does **not** reliably establish who held the Global Administrator role at the moment of authentication. To cover every current and future Global Administrator, maintain a role-assignment inventory, account for PIM activation/deactivation and historical changes, and correlate it with sign-in events. A fixed list should never be presented as dynamic role detection.

**Sign-in types:** `SigninLogs` addresses interactive sign-ins. Non-interactive account activity requires appropriate diagnostic categories and separate detection logic.

**PIM audit activity:** `AuditLogs` records role-management operations, distinct from authentication events; audit records alone do not validate Azure Monitor alert delivery.

**Alert delivery:** Log ingestion, evaluation intervals, stateful/stateless behavior, and notification processing introduce latency. A matching sign-in is not guaranteed to yield exactly one email.

## Repository structure

```text
.
├── README.md
├── docs/
│   ├── architecture.md
│   └── validation-checklist.md
├── queries/
│   ├── 01-ingestion-health.kql
│   ├── 02-privileged-signins.kql
│   ├── 03-failed-privileged-signins.kql
│   └── 04–15 investigation queries (see KQL query library)
└── screenshots/
    ├── 01-alert-rule-review-redacted.png
    ├── 02-signin-log-query-results-redacted.png
    ├── 03-entra-diagnostic-settings-redacted.png
    ├── 04-pim-role-activation-email-alert-redacted.png
    └── 05-pim-role-assignment-auditlogs-redacted.png  (upload pending)
```

## Evidence

These redacted screenshots document the configuration and observed results. Screenshot 05 will render after its image is uploaded to `screenshots/`. Identifiers are obscured for public documentation.

### 1. Microsoft Entra diagnostic settings

The Entra tenant's diagnostic setting routes `SignInLogs` and `AuditLogs` to Log Analytics.

![Redacted Microsoft Entra diagnostic setting exporting sign-in and audit logs](screenshots/03-entra-diagnostic-settings-redacted.png)

### 2. KQL sign-in log verification

The `SigninLogs` query returned successful interactive sign-in records, verifying that data reached the workspace.

![Redacted Log Analytics query and successful sign-in results](screenshots/02-signin-log-query-results-redacted.png)

### 3. Azure Monitor alert rule configuration

The alert wizard review shows the configured single-event log search, selected email action group, and severity. **This screenshot documents configuration only, not successful deployment or email delivery.** The initial creation attempt encountered an RBAC write authorization error, and successful alert firing remains unverified.

![Redacted Azure Monitor alert rule Review and create screen](screenshots/01-alert-rule-review-redacted.png)

### 4. Native PIM role-activation email — validated

On October 20, 2026, Microsoft Security sent an automatic notification when the lab administrator activated the **Privileged Role Administrator** directory role. The email shows the activating identity, role, directory, and one-hour activation window. This provides evidence of successful **native PIM notification delivery**.

![Redacted Microsoft Entra PIM Privileged Role Administrator activation email](screenshots/04-pim-role-activation-email-alert-redacted.png)

**Validation boundary:** This confirms a native Microsoft Entra PIM activation email, **not** successful Azure Monitor scheduled-query alert delivery. The Azure Monitor alert configuration above remains unverified end-to-end.

### 5. Privileged role assignment changes — PIM AuditLogs

The saved [`07-privileged-role-assignment-changes.kql`](queries/07-privileged-role-assignment-changes.kql) investigation query examines `AuditLogs` over a 30-day window. The redacted Log Analytics results show successful role-management operations attributed to **MS-PIM**, including **Add member to role**, **Remove member from role**, and **Remove eligible member from role**. Displayed target roles include **Global Administrator**, **Billing Administrator**, and **Privileged Role Administrator**.

![Redacted PIM directory role assignment and eligibility changes in Azure Log Analytics AuditLogs](screenshots/05-pim-role-assignment-auditlogs-redacted.png)

**Validation boundary:** These results demonstrate that PIM-related role changes were captured and queried in centralized `AuditLogs`. They do **not** demonstrate an interactive privileged sign-in, a complete role activation workflow, or Azure Monitor alert notification delivery.

See [validation checklist](docs/validation-checklist.md) for outstanding end-to-end tests.

## Security considerations

- Never commit credentials, tokens, unredacted UPNs, IP addresses, Object IDs, tenant IDs, or subscription IDs.
- Keep emergency access accounts excluded from policies that could prevent emergency recovery; protect them with phishing-resistant credentials and monitor their use.
- Use least-privilege Azure RBAC for provisioning and monitoring. Remove temporary elevated access after use.
- Store notifications in an inbox independent of the monitored tenant, where appropriate.
- Validate action group delivery and scheduled query permissions before calling the alert operational.

## Costs

Azure Monitor log-search alert rules and Log Analytics ingestion may be billable. The Azure portal showed **$1.50** as the estimated price for the configured alert at the time of review; this is not a guarantee of total Azure spend. Confirm pricing in the subscription's billing region and monitor Cost Management.

## Related work

This repository documents **three complementary evidence paths**: Entra **sign-in activity** analyzed in Log Analytics, **PIM role-management changes** captured in `AuditLogs`, and **directory-role activation** notifications sent by native PIM. The PIM email path is validated; the Azure Monitor log-search alert delivery path is not yet verified end-to-end.

## References

- [Microsoft Entra diagnostic settings](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-configure-diagnostic-settings)
- [Azure Monitor log search alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-create-log-alert-rule)
- [Manage emergency access accounts in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)
- [Kusto Query Language](https://learn.microsoft.com/en-us/kusto/query/)
