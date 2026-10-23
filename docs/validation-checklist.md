# Deployment and Validation Checklist

Use this checklist to record actual outcomes. **Do not mark an item as completed until validated.**

## Data pipeline
- [x] Log Analytics workspace deployed successfully.
- [x] Entra diagnostic setting enabled `SignInLogs` and `AuditLogs` export to the workspace.
- [x] Interactive sign-in rows appeared in `SigninLogs`.
- [ ] Verify `AuditLogs` ingestion, if needed for role-change monitoring.

## Alert
- [ ] Verify active Azure RBAC permits `Microsoft.Insights/scheduledQueryRules/write` at the rule resource-group scope.
- [ ] Create the scheduled query alert successfully.
- [ ] Confirm the rule is enabled and healthy.
- [ ] Include both emergency account Object IDs in the **private** rule configuration.
- [ ] Attach and verify the intended email action group and recipient.
- [ ] Trigger one authorized interactive sign-in for a monitored account.
- [ ] Verify a corresponding `SigninLogs` record.
- [ ] Verify a fired alert in Azure Monitor alert history.
- [ ] Verify an email notification arrives.
- [ ] Evaluate duplicate notifications and latency.
- [ ] Decide whether failed sign-ins need a separate rule.
- [ ] Implement and validate dynamic Global Administrator membership coverage, if required.

## Operations
- [ ] Remove unnecessary temporary subscription or root-scope elevated access.
- [ ] Configure spending alerts and review ingestion/alert costs.
- [ ] Document an owner, validation date, and test cadence for each alert.
- [ ] Redact screenshots before upload to the public repository.
