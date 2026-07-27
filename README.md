# GitHub OIDC with AWS — Complete Setup Guide

## What is GitHub OIDC?

GitHub OIDC (OpenID Connect) lets your GitHub Actions pipeline authenticate with AWS **without storing any AWS credentials** (no access keys, no secrets) in your repository.

Instead, GitHub issues a short-lived JWT token to the pipeline, AWS verifies it against a registered Identity Provider, and if the conditions match, AWS hands back temporary credentials.

**Flow:**

GitHub Actions → generates JWT token → sends to AWS STS
AWS STS → verifies token against OIDC provider → checks trust policy conditions
Trust policy matches → returns temporary credentials → pipeline uses them


## Prerequisites

- AWS account with IAM permissions to create roles and OIDC providers
- GitHub repository you want to authenticate from
- AWS CloudShell or AWS CLI configured locally


## Part 1 — Register GitHub as an OIDC Identity Provider in AWS

This is a one-time setup per AWS account. You only need one OIDC provider for GitHub, regardless of how many repos or roles you create.

### Option A — AWS Console

1. Go to **IAM → Identity providers → Add provider**
2. Select **OpenID Connect**
3. Provider URL: `https://token.actions.githubusercontent.com`
4. Click **Get thumbprint**
5. Audience: `sts.amazonaws.com`
6. Click **Add provider**

### Option B — AWS CLI

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list ffffffffffffffffffffffffffffffffffffffff
```

### Verify it exists

```bash
aws iam list-open-id-connect-providers
```

Expected output:
```json
{
    "OpenIDConnectProviderList": [
        {
            "Arn": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
        }
    ]
}
```

---

## Part 2 — Create the IAM Role

### Step 1 — Start with a broad trust policy (Phase 1)

When setting up for the first time, use `repo:*` as a temporary placeholder.
This lets the pipeline reach STS so you can capture the real `sub` claim from CloudTrail.

```bash
aws iam create-role \
  --role-name github-oidc-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringEquals": {
            "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
          },
          "StringLike": {
            "token.actions.githubusercontent.com:sub": "repo:*"
          }
        }
      }
    ]
  }'
```

Replace `<ACCOUNT_ID>` with your 12-digit AWS account number.

### Step 2 — Attach permissions to the role

Attach whatever permissions the pipeline needs. Example for S3 read access:

```bash
aws iam attach-role-policy \
  --role-name github-oidc-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

For Terraform, you will likely need broader permissions depending on what resources you manage.

---

## Part 3 — GitHub Actions Pipeline Setup

Create `.github/workflows/oidc.yml` in your repository:

```yaml
name: OIDC AWS Auth

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  id-token: write    # Required — allows the pipeline to request an OIDC token
  contents: read     # Required — allows checkout

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source
        uses: actions/checkout@v4

      - name: Configure AWS Credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ap-south-1
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/github-oidc-role

      - name: Verify AWS Identity
        run: aws sts get-caller-identity
```

> **Important:** `permissions: id-token: write` is mandatory. Without it, GitHub will not issue an OIDC token and the pipeline will fail immediately.

---

## Part 4 — Get the Real `sub` Claim from CloudTrail

### Why this step exists

GitHub's OIDC `sub` claim includes numeric IDs that you cannot know in advance:

```
repo:rxvivekpatel@307057061/terraform-oidc@1313452850:ref:refs/heads/main
```

- `307057061` = your GitHub user/org numeric ID (permanent, never changes)
- `1313452850` = your repository numeric ID (permanent, never changes)

AWS console wizard and most documentation examples use the old format without these IDs (`repo:owner/repo:*`). That format no longer works. You must get the real value from CloudTrail.

### Run the pipeline once, then check CloudTrail

Trigger the pipeline (push a commit or use workflow_dispatch). It will fail with `Not authorized to perform sts:AssumeRoleWithWebIdentity` — that is expected at this stage.

Then run:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
  --max-results 1 \
  --query 'Events[0].Username' \
  --output text
```

You will get back the exact `sub` value:

```
repo:yourorg@123456789/yourrepo@987654321:ref:refs/heads/main
```

Take everything up to and including the repo ID, then add `:*`:

```
repo:yourorg@123456789/yourrepo@987654321:*
```

---

## Part 5 — Lock Down the Trust Policy (Phase 2)

Now update the role with the real `sub` value:

```bash
aws iam update-assume-role-policy \
  --role-name github-oidc-role \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringLike": {
            "token.actions.githubusercontent.com:sub": "repo:<OWNER>@<USER_ID>/<REPO>@<REPO_ID>:*"
          },
          "StringEquals": {
            "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
          }
        }
      }
    ]
  }'
```

Trigger the pipeline again — it should succeed now.

---

## Trust Policy Field Reference

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:<OWNER>@<ID>/<REPO>@<ID>:*"
                },
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

| Field | Description | Changes? |
|---|---|---|
| `Version` | IAM policy language version | Never change |
| `Principal.Federated` | ARN of the OIDC provider in your account | Account ID only |
| `Action` | The STS API call for OIDC auth | Never change |
| `aud` condition | Who the token was issued for | Never change |
| `sub` condition | Which repo and context sent the token | Changes per repo |

---

## Condition Operator Rules

| Situation | Operator to use |
|---|---|
| Value contains `*` wildcard | `StringLike` |
| Value is a fixed exact string | `StringEquals` |

Never use `StringEquals` with a `*` wildcard — the asterisk is treated as a literal character, not a wildcard, and the condition will never match.

---

## Why AWS Console Role Wizard Generates a Broken Policy

The AWS console wizard generates the old `sub` format:

```json
"token.actions.githubusercontent.com:sub": "repo:owner/repo:*"
```

This is outdated. GitHub now includes numeric user and repo IDs in the `sub` claim. The console has not been updated to account for this.

**Rule:** Use the wizard only to create the role structure and attach permissions. Always overwrite the trust policy manually using the Phase 1 → Phase 2 approach described above.

---

## Troubleshooting

### Error: `Not authorized to perform sts:AssumeRoleWithWebIdentity`

The trust policy conditions are not matching the token. Run CloudTrail to see the exact `sub` being sent:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
  --max-results 1 \
  --query 'Events[0].Username' \
  --output text
```

Compare the output to the `sub` value in your trust policy. They must match.

### Error: `Could not assume role with OIDC` with repeated retries

The action retries up to 12 times before giving up. This is normal retry behavior on auth failures — it is not a network issue. The root cause is always the trust policy.

### OIDC provider already exists

If you get a conflict error when creating the provider, it already exists. Verify with:

```bash
aws iam list-open-id-connect-providers
```

One provider per AWS account is enough for all GitHub repos.

### Pipeline permission error on token request

Ensure your workflow has this at the top level (not inside the job):

```yaml
permissions:
  id-token: write
  contents: read
```

---

## Checklist for Every New Repo

```
[ ] OIDC provider exists in the AWS account (one-time, reuse for all repos)
[ ] IAM role created with Phase 1 broad trust policy (repo:*)
[ ] Pipeline has permissions: id-token: write
[ ] Pipeline triggered once to generate a CloudTrail entry
[ ] Real sub claim retrieved from CloudTrail
[ ] Trust policy updated with exact sub including numeric IDs
[ ] Pipeline triggered again — confirms success
```
```
