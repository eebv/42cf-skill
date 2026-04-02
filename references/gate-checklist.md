# CFMail Gate Checklist

## Post-deploy checks

1) Account workers subdomain
- GET `/accounts/{account_id}/workers/subdomain`
- Expect a valid `result.subdomain`

2) Script subdomain
- GET `/accounts/{account_id}/workers/scripts/{script}/subdomain`
- Expect `enabled=true`

3) Worker settings
- GET `/accounts/{account_id}/workers/scripts/{script}/settings`
- Expect ALLOWED_DOMAINS count = final domain count

4) Email Routing
- GET `/zones/{zone_id}/email/routing`
- Expect `enabled=true` and `status=ready`

5) Catch-all
- GET `/zones/{zone_id}/email/routing/rules/catch_all`
- Expect worker target `email-handler`

6) Domain completeness
- Every subdomain has TXT>=1 and MX>=3

## Query URL templates

- Domains: `{worker_url}/domains`
- Get mail by email: `{worker_url}/get?email=prefix@sub.domain`
- Get mail by prefix+domain: `{worker_url}/get?prefix=prefix&domain=sub.domain`
- Clear mail: `DELETE {worker_url}/clear?email=prefix@sub.domain` with `X-Admin-Auth`

## Proxy and batch verification

- Proxy: `http://127.0.0.1:42006`
- Batch script:
  - `D:\codex\...\serv00-chatqiqi666-cfmail\tools\cf_42_batch_send_and_verify.ps1`
