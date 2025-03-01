+++
author = "Daniel Ancuta"
title = "Secrets Management in AWS: Using Secrets Manager"
date = "2025-03-01"
description = "A guide to effective secrets management in AWS using Secrets Manager, with best practices for microservices."
tags = ["aws", "secrets manager", "devops", "microservices", "security", "aws lambda", "kms", "ci/cd"]
+++

Sharing secrets across microservices creates security risks. Not only when one of the microservices is compromised, as then during rotation of single secret all of your microservices will be affected. Leading to potential downtimes.

But also if we talk about database access, API tokens, different microservices require different set of permissions.

Using a single secret grants excessive privileges, increasing security risks and violating the principle of least privilege.

That's why it's important to keep them separated between microservices.

But if you use AWS, don't you worry, you have two tools that will help you manage secrets across microservices properly!

## AWS Secrets Manager & AWS KMS: What They Are
- **[AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)** is a service that helps you store, manage, and retrieve secrets securely. It provides **encryption at rest** using AWS KMS, **automatic rotation** of secrets, and **fine-grained access control** via IAM policies.
- **[AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html)** is a managed encryption service that allows you to create and control cryptographic keys used for securing secrets and other sensitive data.

This duet helps you to handle sensitive data in a secure manner.

## Single Secret per Microservice Encrypted with a Dedicated KMS Key
One of the core principles of microservices is their encapsulation - each service should be independent and self-contained. The same applies to secrets.

Each microservice should have a **dedicated secret**, encrypted and decrypted by a **separate KMS key**. This ensures that only the microservice (Lambda, ECS, Fargate, etc.) has access to its own secrets.

### Why does this matter?

* If a service is compromised, only its secrets need to be rotated.

* There’s no risk of one service accidentally accessing another’s credentials.

* This follows the principle of the least privilege, reducing the attack surface.

## Naming Convention for Secrets and KMS Keys
Having a consistent naming convention helps teams quickly locate and manage secrets without confusion.

With the teams I’ve worked with, we always aimed for simplicity and common sense when naming secrets and KMS keys.

### Glossary
- `{service}`: Name of the microservice, typically the repository name (if using a multi-repo approach) or the directory name (if using a monorepo approach).
- `{stage}`: Environment name, e.g., `dev`, `staging`, `prod`, etc.

### Secret Naming Convention
```plaintext
/services/{service-name}/{stage}

# Example
/services/user-service/dev
/services/payment-service/prod
```

### KMS Alias Naming Convention
```plaintext
{service-name}-{stage}

# Example
user-service-dev
payment-service-prod
```

By looking at repository name, you can quickly find its related KMS and Secret in AWS Console quickly.

## Creating Secrets and KMS Keys is a Microservice Responsibility
Managing secrets should not be a manual process. The creation of secrets and KMS keys should be automated within the CI/CD pipeline of the microservice itself.

I’ve seen companies where secrets were managed in a separate repository or team—leading to approval bottlenecks and deployment delays. This gatekeeping approach contradicts the principles of independent and self-sufficient microservices.

Automating this process within your microservice’s CI/CD pipeline makes it smoother and safer.

## Creating and Updating Secrets with SOPS
[SOPS](https://github.com/getsops/sops) is an editor for encrypted files that supports YAML, JSON, ENV, INI, and BINARY formats and encrypts with AWS KMS, GCP KMS, Azure Key Vault, Age, and PGP.

With SOPS, secrets are **stored in version control**, encrypted using the **dedicated KMS key** of each microservice. The CI/CD pipeline for the microservice reads these encrypted secrets and creates a record in AWS Secrets Manager.

If you're not rotating secrets frequently, you can ensure that updates to secrets happen within the CI/CD pipeline whenever secret files are modified.

### Encrypting a Secrets File with SOPS
```sh
sops --encrypt --kms arn:aws:kms:region:account-id:key-id secrets.json > secrets.enc.json
```

## Secret Retrieval in Python with AWS Lambda Powertools
If you are using AWS Lambda and Python, check out [powertools-lambda-python](https://docs.powertools.aws.dev/lambda/python/latest/utilities/parameters/).

AWS Lambda Powertools for Python provides an easy SDK to access secrets, including built-in caching to avoid excessive API calls to Secrets Manager.

### Example: Caching a Secret for 5 Minutes in Python
```python
from aws_lambda_powertools.utilities.parameters import get_secret

# Retrieve secret and cache for 300 seconds (5 minutes)
db_credentials = get_secret("/services/user-service/prod", max_age=300)
```

## Final Thoughts
Microservices should be independent and secure, and that includes how they handle secrets. By isolating secrets per service, enforcing structured naming, and automating secret management in CI/CD, we enhance security without adding complexity.

Hope you enjoyed reading this article!
