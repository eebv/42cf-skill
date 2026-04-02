---
name: 42cf
description: Use when deploying, retrying, or auditing Cloudflare Email Routing plus Worker mailbox setups from account credentials, including batched subdomain creation, one-time confirmation, strict gate checks, and end-to-end receive verification.
---

# 42CF

## Overview

Alias skill for cfmail-deploy-pipeline. Use the same deployment workflow and gates.
Goal: ask once, execute end-to-end, verify all gates, and leave reusable artifacts.

## Defaults (unless user overrides)

- Default subdomain count: `42`
- Default naming: `3 random letters + mail` (example: `abcmail.example.com`)
- Keep DNS safety margin (normally target `<=198` records)
- Ask only once at the beginning, then run full flow without extra confirmations

## One-time confirmation contract

Do one compact confirmation block before execution:

1. Select target zone (if account has multiple zones)
2. If zone already has DNS records, ask delete or keep
3. Compute max creatable subdomains, then confirm final count (default 42)
4. Confirm naming rule (default 3letters+mail)

After this block, do not ask repeated process questions.

## Execution workflow

### 0) Preflight

- Check account: `/user`
- Check zone: `/zones/{zone_id}`
- Pull DNS list: `/zones/{zone_id}/dns_records?per_page=5000`
- Read account workers subdomain: `/accounts/{account_id}/workers/subdomain`
- Estimate max creatable subdomains with `4 records per subdomain (TXT + 3 MX)`

### 1) Generate subdomains

- Generate by default rule `3letters+mail`
- Skip collisions with existing records
- Save created list files

### 2) Create DNS

For each subdomain create:

- 1 TXT record
- 3 MX records:
  - `route1.mx.cloudflare.net`
  - `route2.mx.cloudflare.net`
  - `route3.mx.cloudflare.net`

Use retries and allow idempotent resume on API jitter.

### 3) Worker base

- If account workers subdomain is missing, initialize it first
- Build final worker URL only from API readback:
  - `https://{script}.{workersSubdomain}.workers.dev`
- Never guess this URL manually

### 4) Upload worker and settings

- Reuse validated email-handler source
- Bind:
  - `ADMIN_AUTH`
  - `ALLOWED_DOMAINS`
  - `EMAIL_STORE`
  - `ENABLE_LATEST_DEBUG=false`
  - `ENABLE_LEGACY_PREFIX=false`

### 5) Script subdomain and routing

- Enable script subdomain: try POST first, fallback to PUT if needed
- Enable Email Routing
- Set catch-all to worker `email-handler`
- `actions.value` must be array: `["email-handler"]`

### 6) Mandatory gates

All must pass before completion:

- account workers subdomain exists
- script subdomain is enabled
- ALLOWED_DOMAINS count equals final subdomain count
- each subdomain has TXT>=1 and MX>=3
- Email Routing is enabled and ready
- catch-all points to `worker -> email-handler`
- DNS stays inside budget (or no 81045 and creation still succeeds)

### 7) Artifacts

Write deployment outputs into both folders:

- `<MAIL_ROOT>\\worker\\<account_prefix>`
- `<MAIL_ROOT>\\worker\\<account_email>`

Include:

- deploy info
- workers subdomain / script subdomain snapshots
- routing / catch-all / settings snapshots
- final subdomain list and removed list (if trimmed)
- operator guide with query URLs and admin auth

## Receive verification flow

When user asks for random receive test:

1. Randomly pick N mailboxes (usually 3 or 5)
2. Send test mails through available sender API
3. Query worker `/get`
4. If proxy TLS jitter causes false negative, verify via KV directly
5. Report `send_ok / recv_ok / code_match` and save json/csv report

Known batch verifier:

- `D:\codex\...\serv00-chatqiqi666-cfmail\tools\cf_42_batch_send_and_verify.ps1`

## Failure playbook

- `10007` on workers subdomain: initialize account workers subdomain, then resume
- `method_not_allowed` on script subdomain: switch POST/PUT method
- `Bad JSON input` on catch-all: use array value for actions
- TLS/proxy query failures: mark as transport issue, then do KV fallback check
- If interrupted: resume from current stage, do not restart full deployment

## Scope discipline

- Touch only target CF account/zone and mail-system workspace files
- Do not modify unrelated projects
- Put temp files under project temp folders; clean up after task

