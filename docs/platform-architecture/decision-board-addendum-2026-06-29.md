# Decision Board Addendum — 2026-06-29

此文件補足 decision-board.md 第一輪 review 擋下的五個缺口。性質與 decision board 相同：這是架構拍板層，不是 schema / endpoint / code spec。

本 addendum 優先於 decision-board-review-notes.md。Claude 後續要把這份內容整合回 decision-board.md，但整合前不得寫 Rust code、migration、schema、API handler、payment code、R2 implementation、Kafka 或 gRPC。

## A1. R2 Public / Private Bucket Boundary

✅ **每個 tenant 至少拆 public / private 兩個 R2 bucket。**

- public bucket：商品圖、公開展示資源，可掛 custom domain / CDN。
- private bucket：會員私密檔、訂單附件、非公開資源，不掛 public custom domain，只能經 API / Worker / signed URL 授權讀取。

禁止把 public 與 private object 混在同一個公開 bucket 裡靠 prefix 紀律區分。bucket 是 R2 的權限邊界；public/private 是安全邊界，不是命名規則。

### 翻盤 Trigger

此決議不因 bucket 數量或成本重開。只有 Cloudflare R2 未來提供更強的 bucket-level policy / object-level auth primitive，且通過 security review 證明不退化，才可重開。

## A2. Cloudflare WAF Tenant Identity Boundary

✅ **Cloudflare 層的 tenant identity 只能來自可信邊界。**

- subdomain tenant：Cloudflare 可用 hostname / route 做粗粒度 per-tenant rate limit。
- JWT claim tenant：以 platform middleware 為準；Cloudflare WAF 不假裝能完整理解應用層 tenant。
- header tenant：不得信任 client 直傳。若使用 header，只能由可信 edge / origin gateway 注入，且 origin 必須拒絕外部 client 直接偽造。

結論：Cloudflare WAF 負責邊界粗擋；platform middleware 負責權威 tenant resolution + 精細限流。兩層互補，不能把應用層 tenant policy 全推給 WAF。

## A3. Origin Protection

✅ **Hetzner origin 不可被 Internet 直接打。Cloudflare 是唯一 public ingress。**

所有 public HTTP(S) 流量必須先經 Cloudflare。否則 WAF / rate limit 只保護正門，攻擊者仍可繞過 Cloudflare 直打 origin。

初期裁決：

1. 首選 Cloudflare Tunnel。origin 不開 public inbound HTTP(S)，由 cloudflared 主動連出。
2. 若 Tunnel 在維運上不可接受，才改用 firewall allow Cloudflare IP ranges + origin auth header / mTLS 類方案。
3. 不接受 origin 直接裸露 80/443，只靠 Caddy 或 app 自己擋。

### 翻盤 Trigger

Cloudflare Tunnel 實測造成不可接受的穩定性或維運問題，才重開選 firewall allowlist / mTLS。即使翻盤，origin 不可被直接打這個原則不重開。

## A4. Ledger / Money Invariant

✅ **Ledger 必須具備 double-entry 或等價 balance invariant。**

內部幣、退款、chargeback、人工調帳都不能只改 balance 欄位；必須產生可追溯 ledger entry。若 v1 未採完整 double-entry，也必須有可驗證的不變式與 reconciliation 查核。

硬規則：

- money 禁止 float。採 minor unit integer 或 decimal type，並明確記錄 currency。
- 跨幣別不可直接加總。
- provider transaction id 必須唯一且可查。同 provider 的交易識別不可重複套用到不同 payment。
- refund / chargeback 是 reversal event，不覆蓋原 payment。
- 原始 payment、callback、reversal 都要保留。

## A5. Discord Alert Data Classification

✅ **告警全攤不等於資料全攤。**

Discord 公開頻道只允許低敏摘要：

- tenant
- endpoint
- rate / count
- error class
- time window
- log id

禁止貼：

- raw payload
- request body / response body
- token
- provider secret
- email / phone 等 PII
- payment amount 明細，除非另有明確決策
- provider transaction raw data

若需要查細節，admin 透過 log id 到受控機器查 raw log。這條是資料隔離與教學透明能共存的前提。

## A6. CORS / Tenant Origin Note

per-tenant CORS origin 必須由 tenant 註冊資料或後端管理資料決定，不可信 client header 宣告。CORS 是瀏覽器保護，不是後端授權；真正授權仍靠 access policy + auth。

這一點後續應升級成正式章節，而不是只留在備註。

## Claude Next Action

Claude 只做文件整理：

1. 讀 decision-board.md。
2. 讀本 addendum。
3. 讀 decision-board-review-notes.md。
4. 把本 addendum 的決策整合回 decision-board.md，或保留 addendum 但更新 review notes 標示 blockers resolved。
5. 更新 handoff。

禁止寫 Rust code / migration / schema / API handler。
