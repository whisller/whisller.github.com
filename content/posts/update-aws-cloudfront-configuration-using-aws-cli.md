---
title: "Update AWS CloudFront configuration using AWS CLI"
date: 2024-06-18
description: "Update AWS CloudFront configuration using AWS CLI."
tags: ["aws", "cloudfront", "aws cli"]
---

Regardless of whether you just need to make a quick manual change or it's part of your CI/CD process, AWS CLI can be very useful for both scenarios.

Let's dig into updating AWS CloudFront with AWS CLI!

## Prerequisite
1. You have [AWS CLI](https://aws.amazon.com/cli/) installed
2. You have [jq](https://jqlang.github.io/jq/) installed (not required but useful)

## Update AWS CloudFront configuration using AWS CLI

### Getting ETag
A Distribution ETag is required to update its configuration

```Bash
aws cloudfront get-distribution-config --id {ID-OF-DISTRIBUTION} | jq '. | .ETag'
```

Take note of the ETag value; you will need it later.

### Getting CloudFront configuration

```Bash
aws cloudfront get-distribution-config --id {ID-OF-DISTRIBUTION} | jq '. | .DistributionConfig' > /tmp/cloudfront-{ID-OF-DISTRIBUTION}
```

Now you can modify configuration file.

### Updating CloudFront configuration
Now we will send the modified configuration file to the AWS CloudFront distribution.

```Bash
aws cloudfront update-distribution --id {ID-OF-DISTRIBUTION} --if-match {ETag-value} --distribution-config file:///tmp/cloudfront-{ID-OF-DISTRIBUTION}
```

That's it! Your changes got deployed and applied.

## Troubleshooting 

```Bash
The If-Match version is missing or not valid for the resource.
```

If you see this error, you have used the wrong ETag value for --if-match. After you run update-distribution, your CloudFront's ETag will change, and you will need to obtain it again.

## The End
That's it! I hope you found something useful in this post! If your company needs some help with AWS, [get in touch]({{< ref "/contact" >}} "Get in touch").
