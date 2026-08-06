
# AWS Secrets Manager Security Monitoring System

Detects and alerts on unauthorized or unexpected access to sensitive secrets stored in AWS Secrets Manager, using CloudTrail, CloudWatch, and SNS.

---

## Overview

Secrets Manager holds credentials, API keys, and other sensitive values — which makes it a high-value target. By default, there's no visibility into who is retrieving a secret or when it happens. If an application, script, or bad actor calls `GetSecretValue`, nothing flags it unless you're actively watching CloudTrail logs.

This project closes that gap. Every `GetSecretValue` API call against a specific secret is captured, filtered, and turned into a real-time alert delivered by email. The goal was to build a lightweight, event-driven detection pipeline that could realistically sit in front of any sensitive secret in an AWS account, without needing a SIEM or third-party tooling.

I tested the pipeline by manually retrieving the secret from CloudShell and confirming the alarm fired and the notification arrived within about a minute, end to end.

---

## Architecture

![Architecture Diagram](architecture/Architecture_Diagram.png)

The flow is straightforward:

1. **Secrets Manager** stores the secret. Any read against it generates an API event.
2. **CloudTrail** captures that event as part of its multi-region trail and delivers the log files to an **S3 bucket** for durable storage.
3. The same trail streams events to a **CloudWatch Logs** log group.
4. A **metric filter** on that log group scans for `GetSecretValue` calls and increments a custom metric when it finds one.
5. A **CloudWatch Alarm** watches that metric and flips to `ALARM` the moment a matching event comes in.
6. The alarm publishes to an **SNS topic**, which fans out to a subscribed **email** address.

Nothing in this chain is polling-based — it's all triggered by the API call itself, so the delay between "secret accessed" and "alert received" is just the time it takes CloudTrail to deliver the event and CloudWatch to evaluate it (typically under a minute).

---

## AWS Services Used

**AWS CloudTrail** 
Records every API call made against the account, including `GetSecretValue`. Configured as a multi-region trail so access attempts aren't missed regardless of which region they originate from, and delivers logs to both S3 and CloudWatch Logs.

**AWS Secrets Manager** 
Hosts the secret being monitored. Chosen as the target because it's a realistic, common source of credential leakage if access isn't tracked.

**Amazon CloudWatch (Logs, Metric Filters, Alarms)** 
Turns raw CloudTrail log data into an actionable signal. The metric filter matches on `GetSecretValue` in the log stream, publishes a custom metric (`Secret is accessed`), and the alarm evaluates that metric against a threshold of 1 within a 60-second period.

**Amazon SNS** 
Delivers the alarm notification. A topic (`SecurityAlarms`) with an email subscription handles the actual alerting; this could later be swapped for Slack, PagerDuty, or a Lambda-based auto-remediation target without changing anything upstream.

**AWS CloudShell** 
Used to simulate an access event by running the AWS CLI directly from the console, without needing local credentials configured.

---

## Project Workflow

1. A secret (`TopSecretInfo_Org`) is created in Secrets Manager.
2. CloudTrail, already running as a multi-region trail, logs the `GetSecretValue` call whenever the secret is read.
3. CloudTrail delivers the event to an S3 bucket for long-term storage and streams it to a CloudWatch Logs log group.
4. A metric filter (`GetSecretsValue`) scans incoming log events for the `GetSecretValue` pattern and publishes a value to the `Secret is accessed` metric under a custom `SecurityMetrics` namespace.
5. A CloudWatch alarm evaluates that metric — one matching datapoint within a 60-second period is enough to trigger `ALARM`.
6. The alarm state change publishes a message to the `SecurityAlarms` SNS topic.
7. SNS delivers the alert to the subscribed email address with the alarm name, reason, timestamp, and account ID.

---

## Security Features

- Multi-region API activity logging via CloudTrail
- Centralized, durable log storage in S3
- Real-time detection of sensitive secret access using metric filters
- Automated alerting via CloudWatch Alarms and SNS
- Event-driven design — no polling, no scheduled checks
- Clear audit trail tying every alert back to a specific API call and account

---

## Implementation Steps

**1. Create the secret.** A test secret (`TopSecretInfo_Org`) was created in Secrets Manager to act as the monitored resource, with a description tying it to this project for easy identification later.

**2. Confirm CloudTrail is capturing the right events.** A multi-region trail was already logging management events, so `GetSecretValue` calls were showing up in both the S3 delivery bucket and the associated CloudWatch Logs log group — no extra trail configuration needed.

**3. Build the metric filter.** In the CloudWatch Logs group tied to the trail, a metric filter named `GetSecretsValue` was created with the filter pattern `"GetSecretValue"`. Matching log events publish a value of `1` to a custom metric called `Secret is accessed` in the `SecurityMetrics` namespace.

**4. Create the alarm.** A CloudWatch alarm was set on the `Secret is accessed` metric, configured to trigger when the sum is greater than or equal to 1 across a single 60-second period. This means a single access event is enough to move the alarm from `OK` to `ALARM`.

**5. Wire up notifications.** An SNS topic (`SecurityAlarms`) was created and set as the alarm action. An email address was subscribed to the topic and the subscription confirmed through the confirmation link AWS sends automatically.

**6. Test it.** From CloudShell, `aws secretsmanager get-secret-value --secret-id "TopSecretInfo_Org" --region us-west-2` was run to simulate an access event. Within about a minute, the alarm transitioned to `ALARM` and a notification email arrived with the full alarm context — name, description, state change, threshold, and timestamp.

---

## Repository Structure

```text
├── architecture/
│   └── Architecture_Diagram.png
├── screenshots/
│   ├── CloudWatch_alarm_in_alarm_state.png
│   ├── AWS_Secrets_Manager_Secret.png
│   ├── CloudTrail_Validation_Email.png
│   ├── Metric_Filter_Pattern.png
│   ├── Alarm_Notification_Email.png
│   ├── Cloudshell_Terminal.png
│   └── SNS_subscription_confirmation.png
├── README.md
```

---

## Screenshots

**Secrets Manager secret details** — the monitored secret, `TopSecretInfo_Org`, along with its ARN and encryption key.

**CloudTrail validation email** — confirmation that the trail is actively delivering log files to S3 across multiple regions.

**SNS subscription confirmation** — the email subscription confirmed and linked to the `SecurityAlarms` topic ARN.

**Metric filter configuration** — the `GetSecretsValue` filter matching on the `GetSecretValue` pattern, publishing to the `SecurityMetrics` namespace.

**CloudShell test** — retrieving the secret via CLI to simulate an access event and trigger the alarm.

**CloudWatch alarm in ALARM state** — the metric graph showing the spike the moment the secret was accessed, with the alarm threshold line crossed.

**Alarm notification email** — the resulting SNS email, including alarm name, state change, threshold, and timestamp.

---

## Key Learnings

- CloudTrail delivery to CloudWatch Logs isn't always instant — there's a delivery lag worth accounting for when tuning alarm evaluation periods.
- Metric filter patterns need to match the exact string format CloudTrail writes to the log stream, not just the API name — a filter pattern that looks right on paper can still silently fail to match.
- Setting the alarm threshold to a single datapoint (1 out of 1) makes sense for a high-sensitivity secret, but it also means noisy or legitimate automated access will trigger just as loudly as a real incident  alarm tuning has to account for the secret's actual usage pattern.
- SNS email subscriptions require manual confirmation before any alerts go through, which is an easy step to forget when setting this up for the first time.

---

## Challenges

The main challenge was getting the metric filter pattern to actually match. The filter pattern needs to reflect how CloudTrail formats the event in the log stream, not just the raw API name — a naive pattern matched nothing until it was aligned with the actual JSON structure of the log event.

Timing was the other consideration: between the API call, CloudTrail delivery, metric filter evaluation, and alarm state evaluation, there's a natural lag of anywhere from a few seconds to about a minute. Understanding that lag was necessary to avoid assuming the pipeline wasn't working when it just hadn't caught up yet.

---

## Future Improvements

- Route alarms through EventBridge and Lambda for automated remediation (e.g., rotating the secret or revoking the calling principal's credentials)
- Extend the metric filter to cover additional sensitive API calls (`PutSecretValue`, `DeleteSecret`, `UpdateSecret`)
- Migrate the setup to Terraform for repeatable, version-controlled deployment
- Forward findings to a SIEM or Security Hub for centralized correlation with other security signals
- Deploy across multiple accounts using StackSets or a Landing Zone pattern

---

## References

- [AWS CloudTrail Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)
- [AWS Secrets Manager Documentation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [Amazon CloudWatch Logs Metric Filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)
- [Amazon CloudWatch Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
- [Amazon SNS Documentation](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)

---

## Skills Demonstrated

- AWS Security & Threat Detection
- Cloud Logging and Audit Trail Design
- Event-Driven Architecture
- CloudWatch Metric Filters and Alarming
- Incident Notification Pipelines
- AWS CLI / CloudShell Operations
- Secrets Management
