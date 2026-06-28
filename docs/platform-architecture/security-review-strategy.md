# Security Review Strategy

Akasa Platform 需要兩層 security review。

## 1. Public Git Boundary Review

每次 commit / push / public docs 前必跑。檢查 env files、token、key material、正式 hostname、IP、deployment path、internal runbook、commit message。

## 2. Application / Cloudflare Security Audit

Milestone / release 前執行。檢查 auth/session/cookie/CSRF、tenant isolation、R2 bucket/object key/upload policy、webhook signature/idempotency、rate limit、origin protection、Cloudflare WAF/Access/DNS/cache/R2 settings。

Cloudflare security-audit-skill 可作為正式漏洞審計流程參考，但不取代 Public Git boundary review。
