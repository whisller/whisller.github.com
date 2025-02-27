+++
author = "Daniel Ancuta"
title = "GitHub Rulesets: Your Safeguard for Your Repositories"
date = "2025-02-27"
description = "GitHub Rulesets: Your Safeguard for Your Repositories"
tags = ["github", "pull requests", "code quality", "devops", "version control", "best practices", "code review", "secure coding", "compliance", "healthcare tech", "enterprise security"]
+++

Regardless of whether you're working on a product in a regulated industry like healthcare or finance, or building products that do not require such strict rules, **maintaining control over your repositories is essential**.

Ensuring high-quality pull requests (PRs), and protecting your crucial branches (especially `main` and `develop`) from accidental or intentional overwrites, is critical to the success of your company.

There's no better way to ensure this than by **automating checks and policies** that keep your repository safe, reducing the number of things you need to worry about.

If you are a GitHub user, I have good news for you!

## GitHub Rulesets

GitHub offers **rulesets**, a powerful way to enforce policies and workflows in your repository.

For **GitHub Enterprise** users, rulesets can be enforced **across the entire organization**, ensuring that all repositories follow the same security and quality standards. This is particularly useful for large teams or companies that need consistency across multiple projects.

Additionally, **GitHub allows exporting and importing rulesets**, which is essential in **multi-organization environments** or when you want to **version-control your policies**. This makes it easier to maintain a standardized approach across different teams while ensuring rulesets are easily shareable and auditable.

![GitHub Rulesets target](/img/github-rulesets/github-ruleset-1.png)

## Key GitHub Rulesets to Improve PR Quality

Below is a list of **"bare minimum" settings** I strongly recommend enabling.

1. **Restrict Deletions** – Prevents accidental or intentional deletion of the branch to maintain repository integrity.
2. **Require Signed Commits** – Ensures that all commits are verified and linked to an authorized contributor, adding a layer of security.
3. **Require a Pull Request Before Merging** – Forces changes to go through a pull request (PR) instead of being pushed directly, ensuring proper review.
4. **Required Approvals** – Mandates that a PR gets a minimum number of approvals before merging, increasing code quality.
5. **Dismiss Stale Pull Request Approvals When New Commits Are Pushed** – Revokes approvals when new commits are added to a PR, making sure all changes are reviewed again.
6. **Require Conversation Resolution Before Merging** – Ensures that all comments and discussions in a PR are resolved before merging, preventing overlooked feedback.
7. **Block Force Pushes** – Stops force-pushing on critical branches, preventing accidental overwrites and maintaining commit history.
8. **Require Code Scanning Results** – Runs automated security checks on the code before merging, reducing vulnerabilities.

## Final Thoughts
GitHub Rulesets are a simple yet powerful way to enforce best practices, improve code quality, and protect your repositories. These rules help create a structured and secure workflow.  
Don't wait—set them up on your `main` and `develop` branches now to save yourself from major headaches in the future.  
