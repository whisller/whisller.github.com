+++
author = "Daniel Ancuta"
title = "GitHub Actions: Security Hardening With OpenID Connect"
date = "2025-03-06"
description = "Enhancing CI/CD security in GitHub Actions using OpenID Connect for AWS deployments."
tags = ["github actions", "openid connect", "aws", "ci/cd", "security", "iam", "devops", "enterprise security", "security hardening"]
+++

The easiest way to configure your CI/CD pipeline with AWS is to just create long-lived credentials, access and secret key, and store them in GitHub Actions.

While this method works, it has **several security risks**:

1. **Exposure Risk** – If credentials are leaked or compromised, an attacker can use them indefinitely.
2. **Most likely no existing key rotation** – Let's face it, in the majority of cases when access and secret key are created and stored in GitHub they will never be rotated.
3. **Overprivileged Access** – Static credentials in CI/CD pipelines often have full access to everything, violating the **principle of least privilege**.
4. **Storage in CI/CD System** – Secrets stored within GitHub Actions' secrets or environment variables can be accidentally logged.

To mitigate these risks, GitHub Actions provides a better alternative: OpenID Connect (OIDC).

## OpenID Connect: A Secure Alternative

[GitHub Actions supports OpenID Connect (OIDC)](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect), which allows GitHub to establish a trust-based authentication mechanism with cloud providers **without long-lived secrets**.

### Why OIDC is Better Than Long-Lived Secrets

1. **No Static Credentials** – AWS credentials are **never stored** in GitHub Actions. No need to rotate or store access keys. Authentication is handled dynamically.
2. **Automatic Credential Expiry** – AWS issues temporary credentials per job execution.
3. **Fine-Grained Access Control** – You can define IAM roles **with specific permissions per repository, branch, or workflow**.
4. **Stronger Security Model** – Uses **federated authentication**, no static credentials.

## Configuring GitHub Actions with OIDC for AWS

GitHub Actions has detailed documentation on how to set up such a connection - [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services). Rather than repeating the steps, I recommend referring to that guide.

Additionally, I previously wrote about [Streaming GitHub audit log to S3 with OpenID Connect - Troubleshooting]({{< ref "/posts/streaming-github-audit-log-to-s3-with-openid-connect-troubleshooting" >}}), which might help with some common OIDC setup issues.

## Fine-Grained Access Control with Subject Claim
One of the biggest advantages of using OpenID Connect in GitHub Actions is the ability to assign **fine-grained permissions** through defining **multiple IAM roles**. This approach ensures that workflows operate under the **principle of least privilege**.

You can build your permissions based on claims from JWT, like Audience (`aud`), Issuer (`iss`) or Subject (`sub`) and so on. Have a look at whole list in [Understanding the OIDC token](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token) documentation.

Let us take Subject as an example, where we can filter by environment, specified workflow, branch etc.

### Example use case
Let's consider scenario where we would want to have three separate roles, per environment we deploy our code to.

Typical hierarchy of such environments are like:
1. `dev`, most permissive and liberal
2. `staging`, trying to be as close to `production` as possible
3. `prod`, we want to limit potential blast radius of damage done through CI/CD

### **Using Different IAM Roles in GitHub Actions**
To use appropriate IAM role in our workflow we will use **`github.ref` and `github.environment`**

#### **GitHub Actions Workflow Example**
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Required for OIDC authentication
      contents: read
    steps:
      - name: Assume AWS Role for dev Deployment
        if: github.ref == 'refs/heads/develop'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/DevDeploymentRole
          aws-region: us-east-1

      - name: Assume AWS Role for staging Deployment
        if: github.ref == 'refs/heads/staging'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/StagingDeploymentRole
          aws-region: us-east-1

      - name: Assume AWS Role for prod Deployment
        if: github.ref == 'refs/heads/main'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/ProdDeploymentRole
          aws-region: us-east-1
```

### **IAM Role Configurations for Different Environments**
Each IAM role has **different permissions** and is associated with **specific conditions** in its trust policy.

Below you can find few "assumeRole" policies, that you can attach to your corresponding roles.

#### 1. "dev" Deployment Role
- Used to deploy changes to the `dev` environment.
- Only accessible from workflows on the `develop` branch.

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:ref:refs/heads/develop"
    }
  }
}
```

#### 2. "staging" Deployment Role
- Used for deployments in `staging` environment.
- Only accessible from workflows on the `staging` branch.
- On top of IAM protection you can also introduce GitHub Rulesets, which I mentioned in [GitHub Rulesets: Your Safeguard for Your Repositories]({{< ref "/posts/github-rulesets-your-safeguard-for-your-repositories" >}})

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:ref:refs/heads/staging"
    }
  }
}
```

#### 3. "prod" Deployment Role
- Used for deployments in `prod` environment.
- Accessible only from workflows triggered by the `main` branch.
- In addition to IAM protection you can also require [Manual Approval](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment#required-reviewers) before your workflow on `main` is executed

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:ref:refs/heads/main"
    }
  }
}
```

## Final Thoughts
Moving from long-lived AWS credentials to **OIDC authentication** in GitHub Actions significantly improves security, reduces credential management overhead, and allows for **fine-grained access control**.

Hope you have enjoyed reading it!
