# Drift Detection

> Requirements for detecting and handling configuration drift.

---

## Purpose

Detect when deployed infrastructure differs from CloudFormation templates due to:

- Manual console changes
- Out-of-band API calls
- Failed/partial deployments
- External automation

---

## Detection Schedule

| Environment | Schedule       | Notification              |
| ----------- | -------------- | ------------------------- |
| dev         | Daily 6 AM UTC | Slack #dev-alerts         |
| stage       | Daily 6 AM UTC | Slack #stage-alerts       |
| prod        | Daily 6 AM UTC | Slack #prod-alerts + page |

---

## Detection Process

### Trigger Drift Detection

```bash
DETECTION_ID=$(aws cloudformation detect-stack-drift \
  --stack-name "$STACK_NAME" \
  --query 'StackDriftDetectionId' \
  --output text)
```

### Wait for Completion

```bash
while true; do
  STATUS=$(aws cloudformation describe-stack-drift-detection-status \
    --stack-drift-detection-id "$DETECTION_ID" \
    --query 'DetectionStatus' \
    --output text)

  if [[ "$STATUS" == "DETECTION_COMPLETE" ]]; then
    break
  elif [[ "$STATUS" == "DETECTION_FAILED" ]]; then
    exit 1
  fi

  sleep 10
done
```

### Check Results

```bash
DRIFT_STATUS=$(aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id "$DETECTION_ID" \
  --query 'StackDriftStatus' \
  --output text)

if [[ "$DRIFT_STATUS" != "IN_SYNC" ]]; then
  # Get detailed drift info
  aws cloudformation describe-stack-resource-drifts \
    --stack-name "$STACK_NAME" \
    --stack-resource-drift-status-filters MODIFIED DELETED \
    > drift-details.json
fi
```

---

## Drift Status Values

| Status        | Meaning                                  | Action       |
| ------------- | ---------------------------------------- | ------------ |
| `IN_SYNC`     | All resources match template             | None         |
| `DRIFTED`     | Some resources modified                  | Investigate  |
| `NOT_CHECKED` | Detection incomplete                     | Retry        |
| `UNKNOWN`     | Resource doesn't support drift detection | Manual check |

---

## Response Procedures

### DRIFTED Status

1. Review drift-details.json for affected resources.
2. Determine if drift is:
   - **Intentional**: Document exception and update template
   - **Accidental**: Plan remediation
   - **Critical**: Immediate remediation required

3. For critical resources (buckets, tables), escalate immediately.

### Remediation Options

| Option          | When to Use                             |
| --------------- | --------------------------------------- |
| Re-deploy stack | Minor drift, no data impact             |
| Import resource | Resource created outside CloudFormation |
| Update template | Drift is intentional/desired            |
| Manual fix      | Specific property change needed         |

---

## Excluded Resources

Some resources don't support drift detection:

- Lambda function code (only configuration)
- Lambda layers
- Some IAM policy details

For these, use additional validation:

```bash
# Verify Lambda code hash
aws lambda get-function --function-name "$FUNCTION_NAME" \
  --query 'Configuration.CodeSha256'
```

---

## Alerting

### Drift Detected Alert

```json
{
  "type": "drift_detected",
  "environment": "prod",
  "stack": "processing-app-prod",
  "status": "DRIFTED",
  "resources": [
    {
      "logicalId": "IntakeQueue",
      "physicalId": "arn:aws:sqs:...",
      "driftStatus": "MODIFIED"
    }
  ],
  "detectedAt": "2026-05-13T06:00:00Z"
}
```

### Alert Routing

| Environment | Drift Severity | Routing             |
| ----------- | -------------- | ------------------- |
| dev         | Any            | Slack #dev-alerts   |
| stage       | Any            | Slack #stage-alerts |
| prod        | Minor          | Slack #prod-alerts  |
| prod        | Critical       | PagerDuty + Slack   |

### Critical Resources

Drift on these resources triggers immediate escalation:

- ProcessingJobsTable
- ProcessingTasksTable
- ProcessingBucket
- WikisBucket
- AppEnvSecret
- ProcessingStateMachine

---

## Changelog

| Date       | Change                         |
| ---------- | ------------------------------ |
| 2026-05-13 | Initial requirements extracted |
