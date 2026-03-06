# Alarm Configuration Validation

## Purpose
This steering file guides validation of alarm definitions in infrastructure-as-code (CDK, CloudFormation, Terraform) against CloudWatch alarm recommendations, identifying gaps and misconfigurations.

## When to Apply
Apply this guidance when the user:
- Asks to review or validate alarm definitions in IaC code
- Is writing or modifying CloudWatch alarm resources in CDK, CloudFormation, or Terraform
- Asks if their alarms match best practices or recommendations
- Wants to find missing alarms or coverage gaps

## Validation Workflow

### Step 1: Fetch Alarm Recommendations

Call `get_alarm_recommendations_for_account` to retrieve the baseline recommendations for the account.

If `generation_status` is `IN_PROGRESS`, inform the user and suggest retrying later.

### Step 2: Parse Alarm Definitions from Code

Identify alarm resources in the user's IaC code. Look for:

**CDK (TypeScript/Python):**
- `new cloudwatch.Alarm(...)` or `cloudwatch.Alarm(...)`
- `new cloudwatch.CompositeAlarm(...)` or `cloudwatch.CompositeAlarm(...)`
- `.addAlarm(...)` method calls
- Metric references: `.metric(...)`, `new cloudwatch.Metric(...)`

**CloudFormation (YAML/JSON):**
- `AWS::CloudWatch::Alarm` resources
- `AWS::CloudWatch::CompositeAlarm` resources

**Terraform (HCL):**
- `aws_cloudwatch_metric_alarm` resources
- `aws_cloudwatch_composite_alarm` resources

For each alarm found, extract:
- Namespace
- Metric name
- Dimensions
- Statistic
- Threshold
- Comparison operator
- Period
- Evaluation periods

### Step 3: Compare Against Recommendations

For each recommendation from the API, check if a matching alarm exists in the code:

**Match criteria** (in order of specificity):
1. Same namespace AND metric name AND dimensions — exact match
2. Same namespace AND metric name — partial match (dimensions differ)
3. Same namespace only — namespace covered but specific metric missing

### Step 4: Report Results

Present a gap analysis with three sections:

**1. Covered — Alarms that match recommendations:**
List each IaC alarm that maps to a recommendation. Flag configuration differences:
- Statistic mismatch (e.g., code uses `Average` but recommendation says `Sum`)
- Threshold significantly different (>50% deviation from recommendation)
- Period or evaluation periods differ from recommendation

**2. Missing — Recommendations with no matching alarm in code:**
List each recommendation that has no corresponding alarm in the IaC code, grouped by namespace. Include the recommended configuration so the user can add it.

**3. Extra — Alarms in code with no matching recommendation:**
List alarms defined in code that don't correspond to any recommendation. These aren't necessarily wrong — they may be custom alarms for business logic. Note them as informational.

## Output Format

```
## Alarm Validation Report

### Summary
- Recommendations: {total} across {namespace_count} namespaces
- Alarms in code: {count}
- Covered: {covered_count}
- Missing: {missing_count}
- Configuration mismatches: {mismatch_count}

### Coverage by Namespace
| Namespace | Recommended | In Code | Missing | Mismatched |
|-----------|-------------|---------|---------|------------|
| AWS/Lambda | 4 | 3 | 1 | 1 |
| AWS/EC2 | 3 | 0 | 3 | 0 |
| ... | ... | ... | ... | ... |

### Missing Alarms
#### AWS/EC2
- **CPUUtilization** — Average > 80%, period 300s, eval 3
- **StatusCheckFailed** — Maximum > 0, period 60s, eval 2

### Configuration Mismatches
#### AWS/Lambda — Errors
- **Statistic**: Code uses `Average`, recommendation is `Sum`
- **Threshold**: Code uses `10`, recommendation is `5`

### Extra Alarms (no matching recommendation)
- custom-business-metric-alarm (informational, no action needed)
```

## Statistic Validation Rules

When comparing statistics, flag these common mistakes:
- Count metrics (Errors, Faults, Throttles, Invocations) should use `Sum`, not `Average`
- Utilization metrics (CPUUtilization, MemoryUtilization) should use `Average`, not `Maximum`
- Latency metrics (Duration, Latency) should use `Average` or a percentile (p95, p99), not `Sum`

Refer to the statistic guidance in `alerting-setup.md` for the full mapping.

## Edge Cases

- **No recommendations available**: Inform the user that the account may not be onboarded to alarm recommendations. Suggest using `alerting-setup.md` patterns as a fallback.
- **No IaC alarm files found**: Ask the user to point to the relevant files or directories.
- **Mixed IaC formats**: Handle each format independently and merge results.
- **Modules/constructs that wrap alarms**: Look inside imported constructs or modules if the source is available locally.
