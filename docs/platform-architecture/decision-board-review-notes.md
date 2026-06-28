# Decision Board Review Notes

此文件記錄 Codex 對 decision-board.md 的第一輪 review。這不是新決策，而是交給 Fortes / Claude 修 decision board 前的 blocker 清單。

## Blockers Before Treating Board As Settled

### 1. R2 public/private 邊界需釐清

目前 board 拍板 per-tenant bucket，但同時提到 product 圖 public、member 私密檔 private。若某 bucket 掛 public custom domain，就不能把 private object 混在同 bucket 裡只靠 prefix 擋。

建議二選一：

- 每 tenant 拆 	enant-public-assets / 	enant-private-assets。
- 或明確規定所有物件都經 Worker/API gate，不直接公開整個 bucket。

### 2. Cloudflare WAF per-tenant rate limit 需要可信 tenant identity

Cloudflare 邊界要做 per-tenant rate limit，必須知道 tenant 是誰。

需明確定義：

- subdomain tenant：可用 hostname 判斷。
- JWT claim tenant：Cloudflare WAF 未必可直接做精細 per-tenant，需 platform middleware。
- header tenant：不能信 client 直傳，只能由可信 edge/origin 注入。

### 3. Origin protection 必須補進 board

若所有服務都基於 Cloudflare，Hetzner origin 不應可被直接打。否則 WAF/rate limit 只保護正門，攻擊者可繞 Cloudflare 打 origin。

需決策：firewall allow Cloudflare IP、Caddy origin auth、Cloudflare Tunnel，或其他明確 origin protection strategy。

### 4. 金流 ledger 要明確寫 double-entry / money invariant

需補：ledger entry 採 double-entry 或至少有可驗證 balance invariant；money 禁止 float；provider transaction id unique；refund / chargeback 是 reversal event，不覆蓋原 payment。

### 5. Discord 全攤告警要補資料分類限制

學習環境中告警全攤可以接受，但不能把敏感資料攤開。

需補：不貼 raw payload、token、provider secret、email、phone、payment amount 明細；只貼 tenant、endpoint、rate、error class、log id。

## Recommendation

上述五點補完後，decision-board.md 可作為平台 v1 的 settled decision board。補完前不建 schema、不建 API、不建 migration。
