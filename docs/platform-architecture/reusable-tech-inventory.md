# Reusable Tech Inventory

此文件盤點從 socket-meetup-backend / socket-meetup-frontend 可沿用到 Akasa Platform 的技術資產。此文件只描述可沿用方向，不代表已實作。

## Backend Candidates

- ErrorBody + AppError：固定錯誤碼、JSON error、原始錯誤只進 log。
- strict extractor pattern：StrictJson / StrictQuery / StrictPath。
- transaction + audit + commit-after-emit 的流程觀念。
- committed_broadcast_failed 語意：資料已成功，通知失敗。
- SQLx offline / migration / CI gates。

## Frontend Candidates

- axios transport + error normalization。
- Zod runtime validation + ts-rest type-level contract。
- TanStack Query key factory，不把 token 放 query key。
- mutation retry:false 與錯誤分流。
- BroadcastChannel control / jitter dedupe 作為 optional realtime kit。
- feature-based view structure。

## Not Directly Reusable

- Binance / kline / trading-pairs demo domain。
- demo-only static admin token。
- klinecharts adapter。
