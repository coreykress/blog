# Blog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and deploy a personal Astro blog to AWS S3 + CloudFront with automated deploys via GitHub Actions.

**Architecture:** Astro generates static HTML from Markdown posts on every push to `main`. GitHub Actions builds the site and syncs the output to an S3 bucket. CloudFront serves the files globally over HTTPS using a free ACM certificate.

**Tech Stack:** Astro (blog template), AWS S3, AWS CloudFront, AWS ACM, AWS IAM, GitHub Actions, Node.js 20+

---

## Prerequisites

Before starting, confirm you have:
- Node.js 20+ (`node --version`)
- AWS CLI installed and configured (`aws sts get-caller-identity` should return your account)
- A registered domain name with access to its DNS records
- A GitHub account

---

## Phase 1: Local Astro Setup

### Task 1: Scaffold Astro project

**Files:**
- Create: `~/Development/blog/` (Astro project root — its own git repo)

- [ ] **Step 1: Scaffold the project**

Run from `~/Development/`:
```bash
npm create astro@latest blog -- --template blog --install --git --typescript strict --no-add-ons
```

When prompted:
- Project location: `blog` (already set by arg)
- Template: blog (already set)
- Install dependencies: yes
- Init git repo: yes (creates a new repo inside `blog/`)

- [ ] **Step 2: Verify the project structure**

```bash
ls ~/Development/blog/src/content/blog/
```

Expected: two or three sample `.md` files from the template.

- [ ] **Step 3: Start dev server and confirm it runs**

```bash
cd ~/Development/blog && npm run dev
```

Open `http://localhost:4321` in a browser. Expected: blog homepage with sample posts listed.

Stop the dev server (`Ctrl+C`).

- [ ] **Step 4: Commit**

```bash
cd ~/Development/blog
git add -A
git commit -m "feat: scaffold Astro blog from blog template"
```

---

### Task 2: Configure Astro for production

**Files:**
- Modify: `~/Development/blog/astro.config.mjs`

- [ ] **Step 1: Update astro.config.mjs**

Replace the contents of `astro.config.mjs` with:

```js
import { defineConfig } from 'astro/config';

export default defineConfig({
  site: 'https://yourdomain.com',
  output: 'static',
});
```

Replace `yourdomain.com` with your actual domain.

- [ ] **Step 2: Verify build succeeds**

```bash
cd ~/Development/blog && npm run build
```

Expected output ends with something like:
```
✓ Completed in X.XXs
```

And `dist/` exists:
```bash
ls ~/Development/blog/dist/
```

Expected: `index.html`, `404.html`, `blog/`, and static asset folders.

- [ ] **Step 3: Commit**

```bash
cd ~/Development/blog
git add astro.config.mjs
git commit -m "feat: configure Astro site URL and static output"
```

---

### Task 3: Write your first real post

**Files:**
- Create: `~/Development/blog/src/content/blog/hello-world.md`

- [ ] **Step 1: Create the post file**

Create `~/Development/blog/src/content/blog/hello-world.md`:

```markdown
---
title: "Hello World"
description: "The first post on my new blog."
pubDate: 2026-06-06
tags: ["personal"]
draft: false
---

This blog is live. More soon.
```

- [ ] **Step 2: Verify it appears in dev**

```bash
cd ~/Development/blog && npm run dev
```

Open `http://localhost:4321`. Expected: "Hello World" appears in the post list.

Stop the dev server (`Ctrl+C`).

- [ ] **Step 3: Verify it appears in build output**

```bash
cd ~/Development/blog && npm run build && ls dist/blog/
```

Expected: a `hello-world/` directory in `dist/blog/`.

- [ ] **Step 4: Commit**

```bash
cd ~/Development/blog
git add src/content/blog/hello-world.md
git commit -m "content: add hello world post"
```

---

## Phase 2: AWS Infrastructure

> All AWS CLI commands below assume your default region is `us-east-1` unless specified. Run `aws configure` and set region to `us-east-1` if needed, or append `--region us-east-1` to each command.
>
> Pick a bucket name now and use it consistently. S3 bucket names are globally unique. A good pattern: `blog-yourdomain-com` (e.g. `blog-coreykress-com`).

---

### Task 4: Create S3 bucket

**Files:** None (AWS resource)

- [ ] **Step 1: Create the bucket**

```bash
aws s3api create-bucket \
  --bucket YOUR-BUCKET-NAME \
  --region us-east-1
```

Expected: `{ "Location": "/YOUR-BUCKET-NAME" }`

- [ ] **Step 2: Block all public access**

```bash
aws s3api put-public-access-block \
  --bucket YOUR-BUCKET-NAME \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

Expected: no output (success).

- [ ] **Step 3: Confirm bucket exists and is private**

```bash
aws s3api get-public-access-block --bucket YOUR-BUCKET-NAME
```

Expected: all four values are `true`.

---

### Task 5: Request ACM SSL certificate

> Do this early — DNS validation takes a few minutes. You'll come back to confirm it's issued in Task 7.

**Files:** None (AWS resource)

- [ ] **Step 1: Request the certificate**

```bash
aws acm request-certificate \
  --domain-name yourdomain.com \
  --subject-alternative-names "www.yourdomain.com" \
  --validation-method DNS \
  --region us-east-1
```

Expected:
```json
{
  "CertificateArn": "arn:aws:acm:us-east-1:ACCOUNT:certificate/CERT-ID"
}
```

Save this ARN — you'll need it in Task 7.

- [ ] **Step 2: Get the DNS validation records**

```bash
aws acm describe-certificate \
  --certificate-arn YOUR-CERT-ARN \
  --region us-east-1 \
  --query "Certificate.DomainValidationOptions[*].ResourceRecord"
```

Expected: one or two CNAME records like:
```json
[
  {
    "Name": "_abc123.yourdomain.com.",
    "Type": "CNAME",
    "Value": "_xyz456.acm-validations.aws."
  }
]
```

- [ ] **Step 3: Add the validation CNAME to your domain's DNS**

Log into your domain registrar and add the CNAME record(s) from Step 2. The record type is CNAME, the name is the `Name` value, the value is the `Value` value.

Leave this running in the background — validation typically takes 2–5 minutes but can take up to 30 minutes.

---

### Task 6: Create CloudFront Origin Access Control

**Files:**
- Create: `~/Development/blog/infra/cloudfront-oac.json`

- [ ] **Step 1: Create the OAC config file**

Create `~/Development/blog/infra/cloudfront-oac.json`:

```json
{
  "Name": "blog-s3-oac",
  "Description": "OAC for blog S3 bucket",
  "SigningProtocol": "sigv4",
  "SigningBehavior": "always",
  "OriginAccessControlOriginType": "s3"
}
```

- [ ] **Step 2: Create the OAC**

```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config file://~/Development/blog/infra/cloudfront-oac.json
```

Expected: JSON response containing:
```json
{
  "OriginAccessControl": {
    "Id": "OAC-ID-HERE",
    ...
  }
}
```

Save the `Id` value — you'll need it in Task 7.

- [ ] **Step 3: Commit the infra config**

```bash
cd ~/Development/blog
git add infra/cloudfront-oac.json
git commit -m "infra: add CloudFront OAC config"
```

---

### Task 7: Create CloudFront distribution

> Before this step, confirm your ACM cert is issued:
> ```bash
> aws acm describe-certificate --certificate-arn YOUR-CERT-ARN --region us-east-1 --query "Certificate.Status"
> ```
> Expected: `"ISSUED"`. If still `"PENDING_VALIDATION"`, wait and retry.

**Files:**
- Create: `~/Development/blog/infra/cloudfront-distribution.json`

- [ ] **Step 1: Create the distribution config file**

Create `~/Development/blog/infra/cloudfront-distribution.json`, replacing all placeholder values:

```json
{
  "CallerReference": "blog-dist-1",
  "Comment": "Blog CloudFront distribution",
  "DefaultRootObject": "index.html",
  "HttpVersion": "http2and3",
  "Enabled": true,
  "Aliases": {
    "Quantity": 1,
    "Items": ["yourdomain.com"]
  },
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3Origin",
        "DomainName": "YOUR-BUCKET-NAME.s3.us-east-1.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        },
        "OriginAccessControlId": "YOUR-OAC-ID"
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3Origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "Compress": true,
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"]
    }
  },
  "CustomErrorResponses": {
    "Quantity": 1,
    "Items": [
      {
        "ErrorCode": 403,
        "ResponseCode": "404",
        "ResponsePagePath": "/404.html",
        "ErrorCachingMinTTL": 10
      }
    ]
  },
  "ViewerCertificate": {
    "ACMCertificateArn": "YOUR-CERT-ARN",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  }
}
```

> Note: `CachePolicyId` `658327ea-...` is AWS's managed "CachingOptimized" policy — do not change it.

- [ ] **Step 2: Create the distribution**

```bash
aws cloudfront create-distribution \
  --distribution-config file://~/Development/blog/infra/cloudfront-distribution.json \
  --query "[Distribution.Id, Distribution.DomainName]" \
  --output text
```

Expected: two values on one line:
```
E1ABCDEF12345    d1234abcd.cloudfront.net
```

Save both:
- `Distribution.Id` (e.g. `E1ABCDEF12345`) — needed for S3 bucket policy and GitHub secrets
- `Distribution.DomainName` (e.g. `d1234abcd.cloudfront.net`) — needed for DNS

> The distribution status will be `InProgress` for ~10 minutes while it deploys globally. You can continue with other tasks.

- [ ] **Step 3: Commit the infra config**

```bash
cd ~/Development/blog
git add infra/cloudfront-distribution.json
git commit -m "infra: add CloudFront distribution config"
```

---

### Task 8: Apply S3 bucket policy for CloudFront

**Files:**
- Create: `~/Development/blog/infra/s3-bucket-policy.json`

- [ ] **Step 1: Get your AWS account ID**

```bash
aws sts get-caller-identity --query Account --output text
```

Save the 12-digit account ID.

- [ ] **Step 2: Create the bucket policy file**

Create `~/Development/blog/infra/s3-bucket-policy.json`, replacing all placeholder values:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAC",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR-ACCOUNT-ID:distribution/YOUR-DISTRIBUTION-ID"
        }
      }
    }
  ]
}
```

- [ ] **Step 3: Apply the policy**

```bash
aws s3api put-bucket-policy \
  --bucket YOUR-BUCKET-NAME \
  --policy file://~/Development/blog/infra/s3-bucket-policy.json
```

Expected: no output (success).

- [ ] **Step 4: Commit the infra config**

```bash
cd ~/Development/blog
git add infra/s3-bucket-policy.json
git commit -m "infra: add S3 bucket policy for CloudFront OAC"
```

---

### Task 9: Configure DNS

- [ ] **Step 1: Add domain DNS record**

In your domain registrar's DNS panel, add a record pointing your domain to the CloudFront distribution:

| Type | Name | Value |
|------|------|-------|
| CNAME | `yourdomain.com` (or `@`) | `d1234abcd.cloudfront.net` |

> If your registrar supports ALIAS records at the root (like Cloudflare does with CNAME flattening), use CNAME. If not, check if they offer ALIAS/ANAME records for the root domain. Subdomains (e.g. `blog.yourdomain.com`) can always use a plain CNAME.

- [ ] **Step 2: Verify DNS propagation**

```bash
dig yourdomain.com CNAME +short
```

Expected: the CloudFront domain (e.g. `d1234abcd.cloudfront.net`).

DNS propagation can take minutes to hours. Continue to the next phase while it propagates.

---

## Phase 3: IAM and CI/CD

### Task 10: Create IAM deployment user

**Files:**
- Create: `~/Development/blog/infra/iam-policy.json`

- [ ] **Step 1: Create the IAM policy file**

Create `~/Development/blog/infra/iam-policy.json`, replacing placeholders:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::YOUR-ACCOUNT-ID:distribution/YOUR-DISTRIBUTION-ID"
    }
  ]
}
```

- [ ] **Step 2: Create the IAM user and policy**

```bash
aws iam create-user --user-name blog-deploy

aws iam create-policy \
  --policy-name blog-deploy-policy \
  --policy-document file://~/Development/blog/infra/iam-policy.json \
  --query "Policy.Arn" \
  --output text
```

Save the policy ARN returned.

- [ ] **Step 3: Attach the policy to the user**

```bash
aws iam attach-user-policy \
  --user-name blog-deploy \
  --policy-arn YOUR-POLICY-ARN
```

- [ ] **Step 4: Generate access keys**

```bash
aws iam create-access-key --user-name blog-deploy
```

Expected:
```json
{
  "AccessKey": {
    "AccessKeyId": "AKIA...",
    "SecretAccessKey": "..."
  }
}
```

**Save both values immediately** — the secret is only shown once.

- [ ] **Step 5: Commit the IAM config**

```bash
cd ~/Development/blog
git add infra/iam-policy.json
git commit -m "infra: add IAM deployment policy"
```

> Do NOT commit the access key values. They go into GitHub secrets only.

---

### Task 11: Create GitHub repo and push

- [ ] **Step 1: Create the GitHub repo**

```bash
gh repo create blog --public --description "Personal blog built with Astro"
```

If `gh` is not installed: go to github.com/new and create the repo manually, then copy the remote URL.

- [ ] **Step 2: Add remote and push**

```bash
cd ~/Development/blog
git remote add origin https://github.com/coreykress/blog.git
git push -u origin main
```

Expected: repo appears at `github.com/coreykress/blog`.

---

### Task 12: Create GitHub Actions deployment workflow

**Files:**
- Create: `~/Development/blog/.github/workflows/deploy.yml`

- [ ] **Step 1: Create the workflow file**

```bash
mkdir -p ~/Development/blog/.github/workflows
```

Create `~/Development/blog/.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - run: npm run build

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - run: aws s3 sync dist/ s3://${{ secrets.S3_BUCKET }} --delete

      - run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_DISTRIBUTION_ID }} \
            --paths "/*"
```

- [ ] **Step 2: Add GitHub Actions secrets**

```bash
gh secret set AWS_ACCESS_KEY_ID --body "YOUR-ACCESS-KEY-ID"
gh secret set AWS_SECRET_ACCESS_KEY --body "YOUR-SECRET-ACCESS-KEY"
gh secret set S3_BUCKET --body "YOUR-BUCKET-NAME"
gh secret set CF_DISTRIBUTION_ID --body "YOUR-DISTRIBUTION-ID"
```

Run each command separately, one at a time.

- [ ] **Step 3: Commit and push the workflow**

```bash
cd ~/Development/blog
git add .github/workflows/deploy.yml
git commit -m "ci: add GitHub Actions deploy workflow"
git push
```

- [ ] **Step 4: Watch the Actions run**

```bash
gh run watch
```

Expected: all steps pass (green). This takes about 1–2 minutes.

If it fails, check the logs:
```bash
gh run view --log-failed
```

---

## Phase 4: Verify End-to-End

### Task 13: Verify the live site

- [ ] **Step 1: Confirm CloudFront distribution is deployed**

```bash
aws cloudfront get-distribution \
  --id YOUR-DISTRIBUTION-ID \
  --query "Distribution.Status" \
  --output text
```

Expected: `Deployed` (not `InProgress`).

- [ ] **Step 2: Test via CloudFront domain first**

```bash
curl -I https://YOUR-DISTRIBUTION-DOMAIN.cloudfront.net
```

Expected: `HTTP/2 200`

- [ ] **Step 3: Test via custom domain**

```bash
curl -I https://yourdomain.com
```

Expected: `HTTP/2 200`

If you get a certificate error, DNS hasn't fully propagated yet. Wait a few minutes and retry.

- [ ] **Step 4: Verify the post is live**

Open `https://yourdomain.com` in a browser. Expected: blog homepage with "Hello World" post listed. Click through to confirm the post page loads.

- [ ] **Step 5: Verify 404 handling**

```bash
curl -I https://yourdomain.com/this-page-does-not-exist
```

Expected: `HTTP/2 404` (CloudFront maps S3's 403 to 404 via the custom error response set in Task 7).

---

## Writing New Posts

Once deployed, the workflow for every new post is:

```bash
# 1. Create the post file
touch ~/Development/blog/src/content/blog/my-new-post.md

# 2. Add frontmatter and write content
# (edit the file)

# 3. Preview locally
cd ~/Development/blog && npm run dev

# 4. Deploy
git add src/content/blog/my-new-post.md
git commit -m "content: add my new post"
git push
# GitHub Actions runs automatically, site updates in ~1-2 minutes
```
