# Proposed API contract

This is a design contract, not a running API. Authenticate all developer endpoints
and enforce deployment-, service-, and field-level authorization. Deployment IDs
must be unique across audience, environment, cloud, and tenant. `tenant-b` is the
short identifier used by the POC examples.

## 1. Read service configuration

```http
GET /api/deployments/tenant-b/services/service-b/configuration
```

```json
{
  "service": "service-b",
  "settings": [
    {"path": "/runtime/workers", "effectiveValue": 8, "source": "tenant-override"},
    {"path": "/retry/maxAttempts", "effectiveValue": 3, "source": "subchart-default"}
  ]
}
```

Resolve effective values using the exact chart/default/tenant layers and recorded
revisions. Provide setting types, descriptions, and allowed values to drive the
editor. Label Git desired configuration separately from deployed/live status.

## 2. Prepare a change

```http
POST /api/configuration-changes
Idempotency-Key: request-123
Content-Type: application/json
```

```json
{
  "deploymentId": "tenant-b",
  "service": "service-b",
  "changes": [
    {"operation": "set", "path": "/runtime/workers", "value": 12},
    {"operation": "set", "path": "/retry/maxAttempts", "value": 5}
  ],
  "reason": "Increase worker capacity and retry attempts"
}
```

Paths are JSON-Pointer-style service-relative setting paths, not file paths.
`set` adds or replaces a supported value. `remove` takes a path without a value
and removes an override. This is an application-specific operation format, not
RFC 6902 JSON Patch.

The API authorizes the request, deduplicates the idempotency key in the caller's
scope, and starts `ConfigurationChangeWorkflow`. Reusing the same key with a
different body is a conflict, not a second operation.

```http
202 Accepted
```

```json
{"changeId": "change-123", "status": "PLANNING"}
```

The workflow reads the pinned snapshot, edits a temporary copy, validates it,
and generates a preview. It has not created a commit or PR at this point.

## 3. Retrieve progress and preview

```http
GET /api/configuration-changes/change-123
```

```json
{
  "changeId": "change-123",
  "status": "READY_FOR_REVIEW",
  "planVersion": 1,
  "summary": [
    {"path": "service-b.runtime.workers", "action": "replace-override", "before": 8, "after": 12},
    {"path": "service-b.retry.maxAttempts", "action": "add-override", "inheritedValue": 3, "after": 5}
  ],
  "validation": {"schema": "passed", "helmRender": "passed", "policy": "passed"}
}
```

Return authorized artifact links for the values and rendered resource diffs.
The server retains the base revisions, candidate digest, and plan version.
The inherited retry value is not an existing line in the tenant YAML: this
operation adds a tenant override rather than replacing a stored value of `3`.

## 4. Confirm the preview and create a PR

```http
POST /api/configuration-changes/change-123/submit
Content-Type: application/json
```

```json
{"planVersion": 1}
```

```http
202 Accepted
```

```json
{"changeId": "change-123", "status": "CREATING_PR"}
```

The API reauthorizes and confirms the expected plan version with the workflow.
The workflow rechecks source freshness before publishing. If it detects a changed
configuration, it produces a new plan for review instead of publishing the old one.
Repeated submission must reuse the same publication attempt and PR.

The status endpoint later returns:

```json
{
  "changeId": "change-123",
  "status": "WAITING_FOR_MERGE",
  "pullRequest": {"number": 128}
}
```

## 5. Receive Git events

```http
POST /api/webhooks/github
```

This is an integration endpoint, not a developer action. Verify signatures,
deduplicate delivery IDs, and correlate repository/PR identity to a known request.
Deliver the event to the workflow, which verifies current PR state with the provider.
Periodic provider checks recover from missed events.

After merge, the workflow observes Argo CD. The status endpoint progresses through
`RECONCILING` to `SUCCEEDED`, or reports a deployment failure/deadline with links
and actionable context. A merged PR alone must not be reported as deployed.

## Errors

| Condition | Behavior |
|---|---|
| Invalid request shape or unsupported operation | Reject with validation errors before starting work |
| Caller lacks access | Reject; do not expose another tenant's configuration |
| Invalid chart value | Expose `VALIDATION_FAILED` with setting paths and messages |
| Wrong submitted plan version | Return conflict and current plan version |
| Relevant source changes after preview | Refresh the plan and require confirmation |
| Transient provider failure | Temporal retries with bounded policies and idempotent activities |
| PR closed without merge | Expose `PR_CLOSED`; nothing is deployed |
| Deployment failure or deadline | Expose failure/timeout; retain the PR and commit links |

The API should expose stable error codes and messages suitable for inline form
errors. Larger diffs must redact sensitive content and use access-controlled storage.
