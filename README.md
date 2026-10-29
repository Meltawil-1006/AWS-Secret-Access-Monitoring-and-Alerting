# AWS Secret Access Monitoring and Alerting

Get an email alert the moment someone reads a secret in AWS Secrets Manager.

This project builds a detection pipeline using **AWS Secrets Manager, CloudTrail, CloudWatch Logs, a CloudWatch metric filter, a CloudWatch alarm, and SNS**.

## Architecture

```
Secret read (GetSecretValue)
        |
        v
   CloudTrail  --->  CloudWatch Logs (log group)
                            |
                            v
                  Metric filter ("GetSecretValue")
                            |
                            v
                  Custom metric (value = 1)
                            |
                            v
                  CloudWatch alarm (Sum >= 1)
                            |
                            v
                  SNS topic  --->  Email
```

## Services used

| Service | Role in the project |
|---|---|
| Secrets Manager | Stores the secret being monitored |
| CloudTrail | Records every API call, including `GetSecretValue` |
| S3 | Stores CloudTrail log files |
| CloudWatch Logs | Receives CloudTrail events as a log group |
| CloudWatch metric filter | Turns matching log events into a metric |
| CloudWatch alarm | Fires when the metric reaches the threshold |
| SNS | Sends the email notification |

## Setup

### Stage 1: Secret and logging

1. **Create a secret** in Secrets Manager.
2. **Create a CloudTrail trail** that logs management events (choose **Read** or **Read + Write**, because `GetSecretValue` is a read event) and stores logs in an S3 bucket.
3. **Test the trail** by retrieving the secret value, then confirm the `GetSecretValue` event appears in CloudTrail event history.

### Stage 2: Monitoring and alerts

4. **Send CloudTrail to CloudWatch Logs.** Edit the trail and enable CloudWatch Logs with a new log group.
5. **Create a metric filter** on that log group:
   - Filter pattern: `"GetSecretValue"`
   - Metric namespace: `SecurityMetrics`
   - Metric name: `Secret is accessed`
   - Metric value: `1`
   - Default value: `0`
6. **Create a CloudWatch alarm** on the metric:
   - Statistic: `Sum`
   - Period: `1 minute`
   - Threshold: `>= 1`
   - Missing data: treat as not breaching
7. **Create an SNS topic**, subscribe your email, and **confirm the subscription** from the email link.
8. Attach the SNS topic to the alarm's **In alarm** action.

### Test

Retrieve the secret value again. Within a few minutes the alarm moves to **In alarm** and an email arrives.

## Troubleshooting notes

Problems I hit and how I fixed them:

- **No events in the log group.** The trail was not sending to CloudWatch Logs, or it was set to log write events only. Read events must be enabled.
- **JSON filter pattern matched nothing.** I was testing it against the wrong sample (an S3 `GetBucketAcl` event). Testing against a real `GetSecretValue` event fixed the confusion. The simple text pattern `"GetSecretValue"` also works and is easier.
- **Metric stayed empty.** A metric filter only counts events that arrive **after** it is created, so earlier events are never counted. Retrieve the secret again after creating the filter.
- **Alarm stayed in OK.** The alarm must use the exact same namespace and metric name as the filter. Allow about 5 minutes for CloudTrail delivery.
- **No email.** The SNS subscription must show **Confirmed** (not "Pending confirmation"). Also check spam and make sure the alarm's action points to the right topic.
- **Region mismatch.** The trail, log group, alarm, and secret must be in the same region (here `us-west-1`), or the trail must be multi-region.

## What I learned

- How CloudTrail, CloudWatch, and SNS fit together into one detection pipeline.
- Metric filters only see new log events, so test after creating them.
- Debugging a pipeline is easiest one link at a time, starting from the source.
- The pattern can be tightened for fewer false alarms:
  ```
  { ($.eventSource = "secretsmanager.amazonaws.com") && ($.eventName = "GetSecretValue") }
  ```

## Secret mission: CloudTrail direct SNS notifications

Sending CloudTrail notifications straight to SNS (without CloudWatch) produces an email for every log file delivery, which is noisy. CloudWatch filters and alarms let you alert only on the specific event you care about.

## Possible improvements

- Use the JSON filter pattern to reduce false positives.
- Alert only on access from unexpected principals or IP addresses.
- Add EventBridge as an alternative to the metric filter and alarm.
- Define the whole stack with Terraform.
- Add a KMS key policy for the SNS topic if encryption is enabled.

## Cleanup

Delete these to avoid charges:

1. CloudTrail trail
2. S3 bucket for CloudTrail logs
3. CloudWatch alarm and log group
4. Secrets Manager secret
5. SNS topic and subscription

## Author

Mostafa Tarek, IT Specialist, Riyadh, Saudi Arabia
