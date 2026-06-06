# Self-Hosted Blog Design

**Date:** 2026-06-06  
**Status:** Approved

## Overview

A mixed general-purpose blog (personal posts, tech writing, project showcases) built with Astro and hosted on AWS S3 + CloudFront. Start with an official Astro blog template, customize over time. EC2 kept as a fallback if server-side services are needed later.

## Goals

- Get something live quickly using familiar AWS tooling
- Write posts in Markdown, deploy with a git push
- Minimal ongoing maintenance
- Easy to customize incrementally

## Non-Goals

- Self-hosted comments or analytics (use third-party services if needed)
- Server-side rendering or dynamic backend (defer to EC2 if this ever changes)
- Multi-author support

## Stack

| Layer | Tool | Notes |
|---|---|---|
| Site framework | Astro | Blog starter template (`npm create astro@latest`) |
| Content | Markdown | Posts in `src/content/blog/*.md` with frontmatter |
| Hosting | AWS S3 | Static file storage, not public website endpoint |
| CDN + HTTPS | AWS CloudFront | Custom domain, SSL termination |
| SSL cert | AWS ACM | Free, auto-renewing, must be provisioned in `us-east-1` |
| DNS | Domain registrar or Route 53 | CNAME/ALIAS pointing to CloudFront distribution |
| CI/CD | GitHub Actions | Build + sync + cache invalidation on push to `main` |
| EC2 fallback | AWS EC2 (t3a.micro) | Available if server-side services are needed later |

## Architecture

```
Developer
   │
   │  git push main
   ▼
GitHub Actions
   │  npm run build
   │  aws s3 sync dist/ s3://bucket --delete
   │  aws cloudfront create-invalidation
   ▼
S3 Bucket (origin)
   │
   ▼
CloudFront Distribution ──── ACM SSL cert
   │
   ▼
yourdomain.com (HTTPS)
```

## Astro Project Structure

```
blog/
├── src/
│   ├── content/
│   │   └── blog/          # Markdown posts go here
│   │       └── *.md
│   ├── components/        # Reusable .astro components
│   ├── layouts/           # Page layout templates
│   └── pages/
│       ├── index.astro    # Homepage / post listing
│       └── blog/
│           └── [...slug].astro  # Individual post pages
├── public/                # Static assets (images, favicon)
├── astro.config.mjs
└── package.json
```

Starting point: `npm create astro@latest` → select "Blog" template. Customize components and layouts incrementally as needed.

## Deployment Pipeline

GitHub Actions workflow triggers on push to `main`:

1. Checkout repo
2. Install Node dependencies (`npm ci`)
3. Build site (`npm run build`) — outputs to `dist/`
4. Sync to S3 (`aws s3 sync dist/ s3://$BUCKET --delete`)
5. Invalidate CloudFront cache (`aws cloudfront create-invalidation --distribution-id $CF_ID --paths "/*"`)

AWS credentials stored as GitHub Actions secrets (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`). Use a least-privilege IAM user scoped to S3 write + CloudFront invalidation only.

## AWS Setup Steps (one-time)

1. Create S3 bucket — disable public access, do NOT enable static website hosting (CloudFront will be the origin)
2. Create CloudFront distribution — origin is the S3 bucket via OAC (Origin Access Control), not the website endpoint
3. Request ACM certificate in `us-east-1` for your domain
4. Attach ACM cert to CloudFront distribution
5. Point domain DNS to CloudFront distribution URL
6. Create IAM user with scoped policy, generate access keys for GitHub Actions

## Content Model

Each post is a Markdown file with frontmatter:

```markdown
---
title: "Post Title"
description: "Short description for SEO and post listing"
pubDate: 2026-06-06
tags: ["tech", "personal"]
draft: false
---

Post content here...
```

The Astro blog template handles rendering, post listing, and RSS feed out of the box.

## Cost Estimate

| Service | Estimated monthly cost |
|---|---|
| S3 | < $0.10 |
| CloudFront | < $1.00 (low traffic) |
| ACM | Free |
| GitHub Actions | Free (public repo or within free tier) |
| **Total** | **~$1/mo** |

## Future Considerations

- **Comments:** giscus (GitHub Discussions-backed, no server needed) is the recommended first choice
- **Analytics:** Plausible Cloud or Fathom (privacy-friendly, no server needed)
- **EC2 migration path:** If server-side services are needed, provision a `t3a.micro` in the same AWS account, run nginx, and either mirror the static site there or split traffic
