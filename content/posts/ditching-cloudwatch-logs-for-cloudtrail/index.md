+++
author = "Daniel Ancuta"
title = "Ditching CloudWatch Logs for CloudTrail: How I Cut CloudWatch Costs by Over Half on a Recent Project"
date = "2026-08-22"
description = "How replacing CloudWatch log groups with S3 lifecycle policies and EventBridge+Lambda alerts cut CloudWatch costs by over half on a recent project."
tags = ["aws", "cloudtrail", "cloudwatch", "cost-optimization", "eventbridge", "lambda", "s3"]
+++

If you've ever set up CloudTrail and just... left the default CloudWatch Logs integration
running, you're probably paying more than you need to. I know I was.

The setup is almost too easy - CloudTrail ships events to a CloudWatch log group, you get
a unified place to search logs, maybe you wire up a few metric filters for alerts. Done.
Except the bill quietly grows every month and you don't really notice until you actually
look at it.

## The Challenge

CloudWatch charges you for log ingestion **and** storage. CloudTrail generates a lot of
events - API calls, console logins, resource changes - and all of that flows into your log
group and sits there. We were pushing roughly **6.6 TB of CloudTrail logs per month** into
CloudWatch. At that volume, you're talking **hundreds of dollars a month** just on log
ingestion and storage.

The other thing that bothered me: CloudWatch metric filters for alerting are kind of
clunky. You write a filter pattern, create an alarm, wire it to SNS... it works, but
changing what you alert on means touching metric filters and alarms, and the feedback loop
is slow.

So I started asking: do I actually need CloudWatch here?

Turns out, not much.

## The Replacement Stack

Three moving parts replaced the whole CloudWatch setup:

**S3 for storage** - CloudTrail already supports delivering logs directly to S3. Same
audit trail, but now you control retention with lifecycle policies instead of paying
CloudWatch's storage rates.

**EventBridge for real-time events** - CloudTrail integrates natively with EventBridge.
Any API call, any console action - match on it with an EventBridge rule and route it
wherever you want.

**Lambda for alerting** - Instead of metric filters and alarms, a Lambda function handles
the alert logic and creates a ticket automatically. Way more flexible.

## Setting It Up

### S3: Lifecycle Policies and Versioning

First, point your CloudTrail trail directly at S3 and remove the CloudWatch Logs
configuration. Then enable two things on the bucket: **versioning** (your safety net
against accidental deletion) and **SSE-KMS encryption** with a customer-managed key
(gives you key rotation control and an audit trail of who decrypted what).

Then add a lifecycle policy. The key here is tiering - you want fast access for recent
logs during incident investigations, then progressively cheaper storage as logs age:

```json
{
  "Rules": [
    {
      "ID": "cloudtrail-lifecycle",
      "Status": "Enabled",
      "Filter": { "Prefix": "AWSLogs/" },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 365
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30
      }
    }
  ]
}
```

Adjust retention to match your compliance requirements - HIPAA needs 6 years minimum, SOC 2 typically 1 year.

One more thing worth enabling on the trail itself - log file integrity validation:

```
aws cloudtrail update-trail \
  --name your-trail-name \
  --enable-log-file-validation
```

CloudTrail will generate hourly digest files with SHA-256 hashes of every log delivered. Free to enable, and it comes up in every SOC 2 and HIPAA audit. Just turn it on.

### EventBridge: Matching the Events You Care About

Instead of CloudWatch metric filter patterns, you write EventBridge rules in JSON that match against the actual CloudTrail event structure. Much more readable.

One thing to keep in mind: don't mix different event sources in a single rule. A rule for root console logins and a rule for IAM policy changes should be two separate rules - mixing `aws.signin` and `aws.iam` sources in one rule is unreliable because the event structures differ.

Root account logins:

```json
{
  "source": ["aws.signin"],
  "detail-type": ["AWS Console Sign In via CloudTrail"],
  "detail": {
    "userIdentity": {
      "type": ["Root"]
    }
  }
}
```

IAM policy changes:

```json
{
  "source": ["aws.iam"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventName": [
      "AttachRolePolicy",
      "DetachRolePolicy",
      "AttachUserPolicy",
      "DetachUserPolicy",
      "PutRolePolicy",
      "DeleteRolePolicy"
    ]
  }
}
```

Two events worth calling out specifically: `DeleteTrail` and `StopLogging`. Someone disabling CloudTrail is one of the first things an attacker does after gaining access. Add those to your rules and make sure they're high priority.

AWS management events on EventBridge are free - CloudTrail events fall into this category, so the rules above cost nothing to run. Compare that to CloudWatch alarms at $0.10 per alarm per month, and the alerting side of this migration pays for itself too.

### Lambda: Turning Events into Tickets

The Lambda function receives the full CloudTrail event payload from EventBridge. Here's the handler skeleton - wire `create_ticket()` to whatever escalation tool your team uses: Squadcast, Linear, Jira, or anything else with an API:

```python
import json

def handler(event, context):
    detail = event.get("detail", {})
    event_name = detail.get("eventName", "Unknown")
    principal = detail.get("userIdentity", {}).get("arn", "Unknown")
    region = detail.get("awsRegion", "Unknown")

    summary = f"[{event_name}] triggered by {principal} in {region}"

    create_ticket(
        summary=summary,
        description=json.dumps(detail, indent=2),
        priority=get_priority(event_name)
    )
```

If you're storing API credentials for your ticketing system, use Secrets Manager - don't put them in environment variables.

### The Numbers

Before the migration, CloudWatch was costing nearly 3x what S3 cost every month.

After the migration, CloudWatch and S3 costs are now roughly equal. The CloudWatch bill dropped by about 58% compared to the monthly average from the first half of the year. At 6.6 TB of monthly logs, that's hundreds of dollars back every month.

S3 did spike for one day when the lifecycle policy first kicked in and started moving objects between storage classes - that's expected and normal. After that it settled flat.

The EventBridge rules are free - AWS management events don't cost anything on EventBridge, so the alerting side of the migration costs essentially nothing to run.

Annualized, it adds up fast. Pull your CloudWatch log group sizes with the AWS CLI and see what you're actually storing before writing this off as not worth the effort.

## Final Thoughts

If you're running CloudTrail with CloudWatch Logs and haven't looked at the bill lately, go look. S3 + EventBridge gets you the same coverage for a lot less, and the alerting setup is more flexible than metric filters anyway.

A few hours of work, meaningful monthly savings, done.

That's it! If you're staring at your own CloudWatch bill and want a hand figuring out where the savings are, feel free to [get in touch](https://whisller.dev/about/).

---
> **Note:** This article was written with assistance from Claude (Anthropic). The experiences, code, and opinions are my own, but AI helped structure and articulate them.
