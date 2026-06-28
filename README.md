# Akasa Platform Backend

Akasa Platform Backend 是 Rust 寫的萬用前後台 / commerce capability platform backend。

此 repo 是 public-safe 作品集與平台骨架，不是 production ops repo。正式部署所需的 secret、provider credential、Cloudflare token、server IP、runbook 與 tenant data 不會進入本 repo。

## Scope

目標平台提供 commerce capability primitives：

- product
- cart
- order
- payment（後階段）
- notification
- member
- storage（Cloudflare R2，後續）
- access policy

## Current Non-goals

- 不在第一階段導入 Kafka / gRPC / microservices。
- 不在第一階段實作金流。
- 不在第一階段實作 R2 upload。
- 不把 meetup demo 的 kline / Binance / trading-pairs domain 搬進來。

## Architecture Entry

請先讀：

- docs/platform-architecture/decision-board.md
- docs/platform-architecture/decision-board-review-notes.md
- docs/platform-architecture/reusable-tech-inventory.md
- .claude/docs/SESSION_HANDOFF_NEXT.md
