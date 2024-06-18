---
title: "Streaming GitHub audit log to S3 with OpenID Connect - Troubleshooting"
date: 2024-06-02
description: "Solving problems with setting up Streaming GitHub audit log to S3 through OpenID Connect."
tags: ["github", "s3", "audit log", "streaming"]
---

Streaming GitHub audit log to one centralized place is quite often used to maintain compliance or improve security within the organization, or both.

If you are reading this post, you are having trouble setting up this stream to your AWS S3. I will show you two common mistakes that I experienced and solved.


## First steps
GitHub has good documentation on how to configure communication between GitHub and AWS.

You can find it in [Streaming the audit log for your enterprise](https://docs.github.com/en/enterprise-cloud@latest/admin/monitoring-activity-in-your-enterprise/reviewing-audit-logs-for-your-enterprise/streaming-the-audit-log-for-your-enterprise#setting-up-streaming-to-amazon-s3).
The first step I would recommend is to go through the steps again and check if you did everything listed there. If you did and still have a problem, carry on reading.

## What does the successful setup look like?
Before you will be able to save streaming, you need to go through the "Check endpoint" process. Successful communication will be shown as:

![Streaming GitHub audit log to S3 with OpenID Connect - Successful communication](/img/streaming-github-audit-log-to-s3/successful-communication.png)

Also, in your bucket, you should have a _check object:

![Streaming GitHub audit log to S3 with OpenID Connect - _check object](/img/streaming-github-audit-log-to-s3/_check_object.png)


## Errors
So as we know what a successful setup looks like, let us check what potential issues we can face.

### There was an error trying to connect to XYZXYZXYZ

![Streaming GitHub audit log to S3 with OpenID Connect - Connection Issues](/img/streaming-github-audit-log-to-s3/connection-issues.png)

This error can indicate that there is something wrong with your OpenID configuration or IAM role configuration. 
To confirm that, just go to the IAM role that you've used and check its "Last activity." If its value is set to `-`, it means it was never assumed by a third party (in this case GitHub).

So double-check that:
- trusted entities are set correctly
- your thumbprints in the Identity provider are correct. How to get one you can read in [Obtain the thumbprint for an OpenID Connect identity provider](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc_verify-thumbprint.html)

### Authorization error accessing "XYZXYZXYZ" bucket.

![Streaming GitHub audit log to S3 with OpenID Connect - Connection Issues](/img/streaming-github-audit-log-to-s3/authorization-error.png)

So you definitely confirmed that your OpenID connection is working correctly? Make sure that the role you want to use has "Last activity" set to some real date.

If you did that, and you see this "Authorization error accessing "XYZXYZXYZ" bucket." it might indicate at least these two things:

#### Your role does not have access to S3 bucket
Make sure that your role has the permission set:

```json
{
    "Effect": "Allow",
    "Action": [
        "s3:PutObject"
    ],
    "Resource": "arn:aws:s3:::{your-bucket-name}/*"
}
```

#### You use custom KMS as default encryption
If you use "Server-side encryption with AWS Key Management Service keys (SSE-KMS)" for your S3 bucket, the role that will be assumed by GitHub needs to have access to it as well.

Make sure that your role has:
```json
{
    "Effect": "Allow",
    "Action": [
        "kms:Encrypt",
        "kms:ReEncrypt",
        "kms:GenerateDataKey"
    ],
    "Resource": "arn:aws:kms:{aws-region}:{aws-account-id}:key/{kms-key-id}"
}
```

## The End
That's it! I hope you found something useful in this post! If your company needs some help with AWS, [get in touch]({{< ref "/contact" >}} "Get in touch").
