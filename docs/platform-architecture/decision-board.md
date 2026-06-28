# 萬用前後台 API — 平台架構 Decision Board

> 性質:**架構拍板層**,非實作規格。每條決議 = 拍板內容 + 理由 + 翻盤 trigger。
> schema / endpoint / code 細節一律不在此文件,留待實作期。
> 狀態:四輪收斂後落地(2026-06-28)。✅ = 已拍板、不再回頭重開;✏️ = 實作期再定。

---

## 此文件怎麼讀(給審閱者)

這份是 **decision board**,不是 spec。它回答的是「為什麼這樣選、什麼情況下會翻盤」,不是「怎麼實作」。每條決議都附**翻盤 trigger**——只有 trigger 成立,該決議才重開;trigger 沒成立就照著走,不要因為「業界都用 X」而動搖,那正是本文件要防的 over-engineering。

驅動所有決議的現實前提(讀任何一條前先吃進去):

- **使用者**:約 15 人的 side project + 教室學生。多數非後端工程師,會用 AI vibe coding 拼前台。
- **機器**:單台 Hetzner,**不再加機器**(預算限制)。
- **維運**:單人(架構者本人)。
- **性質**:pre-production 共用開發平台。**正式商用 = 各自租機器 migrate 出去**,不是把這台當 production 單點。
- **這台機器跑真錢金流的風險**:side project 階段金流多為測試 / 內部幣;真錢上線是 migrate 後的事。

這四點是後面每條「排除 Kafka / gRPC」「db-per-tenant」「夠用就好」的根。脫離這四點,很多決議會看起來保守——那是刻意的。

---

## §0 產品定義(北極星,凌駕所有技術決議)

✅ **本平台 = 給非後端使用者的 commerce capability primitives。**

提供六個「有意見的」商務原語,每個留 structured 擴充點:

| Primitive | 核心職責 | 擴充點 |
|-----------|---------|--------|
| product | 商品展示、TAG 關聯(列表 / 推薦同源)、rich-text 介紹 | 平台核心欄位(id/price/currency/status)+ tenant 自定 structured attributes |
| cart | 購物車流程 | line item 形狀允許 tenant 擴充 |
| order | 訂單,走平台統一 state machine | line item 擴充;state machine 不可改 |
| payment | 金流,provider adapter 抽象 | per-provider 實作(TapPay / NewebPay / 內部幣帳本) |
| notification | 站內即時(SSE)+ email(樣板可編輯) | channel 可擴充 |
| member | 會員、收藏 collection、auth | — |

**設計目標**:讓非後端使用者用 AI vibe 出「commerce 形狀內」的任何前台——商城、預約、訂閱、作品集詢價、純後台訂單系統、遊戲道具交易(遊戲幣扭蛋換要素)。

**「萬用」的精確定義**:不是什麼都能做,是 **commerce 這個形狀內什麼都能做**。完全非 commerce 的需求(純作品集無交易)= 只拼部分積木(只用 product 展示 + notification,不碰 order/payment),這是允許的用法,不是缺陷。

**遊戲場景驗證**:遊戲幣扭蛋 = currency 是遊戲幣、扭蛋是 order(花幣換隨機 item)、累積金幣是 ledger、道具是 product。走 commerce 同一套 primitive,只是 payment provider 是「內部金幣帳本」而非外部金流。**六個 primitive 已覆蓋,不為遊戲開新核心。**

**理由**:UIUX 要的是「打一個 endpoint 就拿到購物車」,不在乎背後是什麼。有意見的原語(幫他想好 cart / order 概念)正是他們不想自己想的部分;但留擴充點,否則「課程預約」「訂閱制」會被寫死的 schema 卡死。

**此定義反向確認**:microservices / gRPC 對 UIUX 使用者是**負價值**(多一層他要理解、會壞、要 debug 的東西)。modular monolith 不只省成本,是本產品定義下的正解。

> ✏️ structured attributes 的具體 schema(JSON column? EAV? typed extension table?)實作期定。

---

## §1 架構起點

✅ **modular monolith + designed seams + outbox in-process relay。**

- 單一部署單元(Rust binary),內部用 module 切分。
- module 邊界用 trait / interface 切乾淨(designed seams),讓未來「可以」變成獨立 service,但現在不付網路代價。
- outbox 指向一個**可替換的 transport interface**,初期用 in-process 輪詢 relay。

✅ **Kafka / gRPC / microservices — 排除(非延後)。**

| 技術 | 裁決 | 理由 | 翻盤 trigger |
|------|------|------|-------------|
| Kafka | 排除 | 單機已跑 N 個 tenant DB + API,再塞 JVM event broker = RAM 自殺。此規模 outbox in-process relay 即足夠 | 真實多-consumer 高吞吐事件流 + 跨 process consumer 出現 |
| gRPC | 排除 | 一台不加機器 =「獨立部署 / 獨立 scale」前提不存在。同 binary 內走 gRPC = 純 overhead(serialization + network hop + 多一種失敗模式) | 真的出現需要獨立部署的 service |
| microservices | 排除 | 同上。單人維運 + 單機,拆分只增負擔 | 上述任一 trigger 成立 + 團隊擴編 |

**outbox 的價值不綁 Kafka**:outbox pattern(DB transaction 內寫 outbox_events,commit 後 relay 推送)本身是 transport-agnostic。relay 可以推去 in-process handler、NATS、Redis Streams、未來的 Kafka——語意都是 at-least-once + idempotent consumer。所以現在用最輕的 in-process,interface 留著,升級不用改 outbox 邏輯。

**沿用 meetup repo 的觀念**:「commit 後才 emit」「committed_broadcast_failed(資料已成功、通知失敗)語意」「audit / event table」「strict error contract」可直接沿用。但金流場景要從「即時 emit」升級成「outbox 可補償」(見 §3)。

> 為什麼這是 board 最該守住的一條:整個 brainstorm 最大的 over-engineering 風險就是把 enterprise 事件骨架無條件塞進單人單機專案。判斷標準永遠是「它有沒有讓使用者更容易 vibe 出東西 / 讓維運更輕」,不是「業界都用」。

---

## §2 多租戶模型

✅ **database-per-tenant(literal 分開的 Postgres database)。**

**理由(分階段演進,最終定錨在 migration)**:

1. blast radius 隔離:15 個獨立開發者,無法 review 每人每條 SQL。db-per-tenant 下一人手殘 migration / 爛 query 最多炸自己那庫,不跨讀別人資料。schema-per-tenant 靠 search_path 紀律,一個 bug 就跨 tenant。
2. **最終定錨 = clean migration seam**:本平台是 pre-production,tenant 正式商用要租機器搬走。db-per-tenant 的搬遷 = 「dump 一個 database 搬過去」;schema-per-tenant = 「從共用庫把一個 tenant 摳出來」,後者是惡夢。**db-per-tenant 現在的首要正當理由就是它讓 migration 乾淨。**

✅ **per-tenant provider secrets(加密落庫、per-request 載入)。**

每個 tenant 有自己的金流商店帳號(TapPay merchant / NewebPay 商店代號 / 遊戲幣帳本)。provider secret 是 per-tenant 的,不是一份平台 secret 打天下。理由雙重:(1) 測試時不會互相打到對方金流帳號;(2) tenant 搬走時 secret 一起帶走、不殘留本平台。

✅ **支援件「夠用就好」(因 pre-production 定位降級):**

| 件 | 階段裁決 | 翻盤 trigger |
|----|---------|-------------|
| PgBouncer | 15 tenant 規模連不爆,正規該裝但非 survival 等級。可之後嫌連線多再加 | tenant 數量級成長 / 連線吃緊 |
| backup | 整機 backup 即可,不為「單 tenant restore 不波及其他」做設計 | tenant 上真錢 production |
| observability | 夠 grep、夠 debug 即可,非 production-grade per-tenant 監控 | 同上 |

> 註:per-tenant rate limit **不在此降級表內**——它從原本的「夠用就好」升級為 v1 必做,見 §8。升級理由不是 production 韌性,是防 vibe 失誤(request loop)炸垮共用機器。

**第一原則明文**:本平台是 pre-production 共用開發平台,**不是 15 個產品的 production 單點**。設計第一原則 = **per-tenant clean migration**(任何 tenant 能以「搬一個 database + 自己的 provider secrets + 跟最新 API」獨立搬出)。production-grade 的 backup 隔離 / API 灰度 / 法遵隔離全部**不在此階段做**,留給 migrate 後各自的機器。(例外:per-tenant rate limit 因防 vibe 失誤之需,提前到 v1,見 §8。)

✅ **Accepted risk(明文揭露,工程消不掉)**:單機單人維運多個 tenant,機器掛 = 全部 tenant 停。side project 階段接受此風險。此為 board 明寫的已知代價,非可用架構消除的問題。

> ✏️ tenant resolution seam(身分從 subdomain / JWT claim / header 哪來 → 解析到哪個 DB)實作期定;migration fan-out runner(schema change 跑遍所有 tenant DB、記錄各庫版本、失敗可續跑)實作期定。

---

## §3 金流(後階段,平台地基做完才進場)

> 排序:平台先做完,金流後上。金流真正商業價值在 AI 應用,但 AI 也排在平台之後。

### §3.1 Correctness 層 — ✅ 全做滿,無討價空間

這層跟 Kafka / gRPC 無關,是金流真正的地基:

- ✅ **PostgreSQL ACID = payment source of truth**。order / payment / ledger / provider callback / refund 皆以 ACID table 為唯一真相。
- ✅ **Transactional outbox**。禁止「update payment 後直接 publish event」(中間任一步失敗 = DB 與下游不一致)。正解:DB transaction 內 update state + insert event + insert outbox,commit 後 relay 推送、標記 published。
- ✅ **idempotency key**。— 必須定義 scope(per-provider + per-order)、TTL、並發同 key 的鎖。**(GPT 原稿只說「要有」,未定義這三項;補完才算數)**
- ✅ **provider callback raw payload 落庫 + signature verification**。webhook 一律 HTTP endpoint(TapPay / NewebPay 不可能是 gRPC),先驗簽 → 存 raw → idempotency check → tx 更新 → 寫 outbox。
- ✅ **payment state machine**:`created → pending → authorized → paid → failed → expired → refunded / partially_refunded / disputed(chargeback)`。**(GPT 原稿只給 created→pending→paid→failed→refunded,漏 authorized / partially_refunded / expired / chargeback。金流 state machine 漏 chargeback 是大洞,補完才拍板)**
- ✅ **reconcile worker**:不能只信 webhook。— 必須定對帳來源(主動打 provider query API)、頻率、對不上時的人工介入路徑。**(GPT 原稿只說「不能只信 webhook」,未給這三項)**
- ✅ **server order 為準**:amount / currency / items 一律以 server 端 order 為準,不信前端。
- ✅ **provider secret 絕不進 frontend / public Git / client bundle**。

### §3.2 PaymentProvider adapter

✅ **provider 抽象成 interface,TapPay / NewebPay / 內部幣各自實作。**

實作期須吸收 provider 差異(細節對最新官方文件 verify):TapPay(frontend prime → backend pay-by-prime)、NewebPay(AES 加密 trade info + SHA256 hash、notify URL 重送行為)、內部幣(無外部 secret、結算走內部 ledger)。三者都是同一個 `PaymentProvider` 的 implementation。

### §3.3 TimescaleDB — ✏️ 延後

- 不做 payment source of truth(報表查詢不可污染金流交易模型)。
- 報表先走 Postgres materialized view。是 extension,晚採用不痛。
- **翻盤 trigger**:report volume 真的上來、materialized view 撐不住。

> ✏️ ledger 設計、reconcile 頻率、state machine 各 transition 的觸發條件,全部實作期定。

---

## §4 Access Policy(訪客存取控制)

✅ **Default deny:所有 API 預設需登入。**

anonymous 是 policy 表裡逐條 **explicit allow** 出來的例外,不是反過來逐條 deny。沒被 policy 明確開放的 resource,訪客打 = 401。

**理由**:漏寫一條的後果倒向安全——「訪客被擋」(安全)而非「訪客看到不該看的」(漏洞)。與 §6 的 sanitize、strict validation 同精神:漏寫倒向安全那邊。

✅ **集中式 policy 表,per-tenant 可設定,per-resource 細分。**

- 一張 per-tenant policy 表定義:anonymous 能存取哪些 resource、各 resource 的訪客上限(如商品最多展示幾個)。
- 一處定義,六個 primitive 查同一張表套用,**不散在各 endpoint 寫 `if 訪客 then`**(那正是會長成技術債的形狀)。
- 落地策略:先做**骨架 + 最常用兩三條**(product read、cart write),policy 表結構留著,其他 resource 之後往表裡加。不一開始把每個 resource 每個動作都列滿(那是 over-build)。

**翻盤 trigger**:若實證上多數 tenant 只需「整站 public / 需登入」全域開關,可簡化;但「逛得到、買要登入」是商城最常見行為,預期集中式 policy 會被用到,故先做骨架不做全域開關。

---

## §5 Auth 契約(框架無關的統一契約)

✅ **選項 X:後端持有 httpOnly cookie 契約,Nuxt / 純 Vue 一律遵守同一份。**

> **refresh token 永遠在 httpOnly + Secure + SameSite cookie,JS 永遠讀不到。**
> **access token 短命(~15min),在記憶體(Pinia / 變數),不落地。**
> refresh 流程 = 瀏覽器自動帶 cookie 打 refresh endpoint → 後端驗 cookie → 發新 access token。前端 JS 全程碰不到 refresh token。

**為什麼框架無關 + 為什麼選 X**:平台給非後端使用者 vibe,auth 這種「錯了就帳號被接管」的東西**必須讓他們不可能寫錯,而非教他們寫對**。X 之下前端拿不到 refresh token,vibe 的人就算想犯錯也沒有材料。選項 Y(Nuxt 走 Nitro BFF、純 Vue 走後端 cookie 兩套並存)= 兩份 auth 契約、兩種 refresh 流程、兩套要防的雷,把安全賭在 vibe 者的 auth 知識上,與「default deny、漏寫倒向安全」矛盾。故排除 Y。

**要防的 vibe 反面教材(明文寫進 board 警告)**:把 refresh token 放 header / localStorage / sessionStorage / 任何 JS 讀得到處 = 把長期憑證暴露在 XSS 面前。配合 §6 的 rich-text 注入面,這會構成一條完整的帳號接管鏈。**此寫法在本平台屬 critical 錯誤,非風格偏好。**

✅ **X 的必然伴生決議(X 成立即強制成立):**

1. **Rust API 是唯一 cookie 設定者**。Caddy / Cloudflare 不碰 auth cookie(單一責任源)。
2. **access token 只在記憶體**,不落 localStorage / sessionStorage(落地 = 又一個 XSS 可達面)。
3. **refresh endpoint 是唯一靠 cookie 認證的 endpoint**,其餘 API 一律 Bearer access token。
4. **CSRF 防護必補**:用 cookie 就有 CSRF 面。SameSite=Strict/Lax 擋多數,跨站場景加 double-submit 或 origin check。與 default-deny 同級,不可漏。
5. **logout = 後端撤銷 + 清 cookie**,非前端刪 token 了事(前端刪不掉 httpOnly cookie,本就只能後端清)。

> ✏️ access token TTL 精確值、refresh token 輪替策略(rotation / reuse detection)、session 撤銷的儲存方式,實作期定。

---

## §6 跨章 Constraints

✅ **rich-text server-side sanitize(mandatory)。**

TinyMCE 6(商品介紹)和 Mail 樣板都讓 user 產出 HTML,存 DB 再渲染給訪客。允許 rich HTML = 允許存任意 HTML;不 server-side sanitize(allowlist tag/attr)= 一個能打 admin 的人就能種 stored XSS,訪客中標。對非後端使用者特別危險(他們不會想到「編輯器打的字會變攻擊面」)。與 §5 auth 鏈直接相關。**非 optional。**

✏️ **email deliverability(待辦,ops 非 v1 code)。**

Mail 樣板可編輯 = 第二個 rich-text 注入面(同上 sanitize),外加變數插值(`{{order.total}}`)要防注入、寄信要 SPF/DKIM 否則進垃圾桶。記一筆待辦,免得上線才發現信全進垃圾桶。

---

## §7 物件儲存(Cloudflare R2)

> 現況:R2 尚未開通。Cloudflare 帳號、網址、CDN、subdomain 已就緒。趁未開通先把架構釘進 board,避免實作期 vibe 出「client 直傳、key 隨便給」的洞。
> 適用範圍:跨 §0 多個 primitive——product(商品圖)、member(頭像)、notification(mail 附圖)。commerce 平台必踩。

### §7.1 隔離模型 — ✅ per-tenant bucket(隔離,非混存)

✅ **每個 tenant 一個獨立 R2 bucket。禁止共用 bucket + key prefix 混存。**

R2 實際限制查證(2026-06,Cloudflare 官方 limits 頁):

| 約束 | 數值 | 對本決策的意義 |
|------|------|---------------|
| bucket 數 / account | 1,000,000 | 15 tenant 連零頭都不到,數量完全不是問題 |
| 免費額度(10GB 儲存 + Class A/B ops) | **account 層級 pool,非 per-bucket** | per-tenant bucket **零額外成本**,15 個 bucket 與 1 個計費相同 |
| custom domain / bucket | 100 | per-tenant bucket 各掛自己的 domain 無壓力(15 << 100);**反例**:全塞一個 bucket 靠 prefix 才會撞限制 |

**理由(三條,與 §2 db-per-tenant 同構)**:

1. **隔離靠邊界,非靠紀律**:bucket 是 R2 的權限邊界,一個 API token scope 到自己 bucket = 物理上不可能跨租戶讀。共用 bucket + prefix = 又回到「靠紀律」,等同 schema 混存,已否決。
2. **migration 對稱**:tenant 搬走 = 整個 bucket 用 Super Slurper / rclone 搬走,對稱於 §2「dump 一個 database」。共用 bucket 要 list + filter + copy 一個 prefix 的所有 object,是惡夢。
3. **零成本**:額度 account pool,分開不多花錢,所以隔離沒有成本藉口。

**翻盤 trigger**:無實際翻盤條件(數量、計費、隔離三面都支持 per-tenant bucket)。唯一例外是 tenant 數成長到需考慮 bucket 管理自動化,屆時是「加 provisioning 自動化」,不是「改回混存」。

### §7.2 上傳安全契約 — ✅(與 §5 auth 選 X 同精神)

✅ **上傳憑證 server 簽發,object key 不由 client 決定,public/private 分離。**

- **server-issued upload**:用 R2 presigned URL 或 Worker 簽發的 scoped 上傳憑證。client 不持有長期 R2 token。
- **object key 由 server 決定**:client 不能任意指定完整路徑(否則可覆寫他人物件 / 路徑穿越)。server 依 tenant + resource type 生成 key。
- **public / private 區隔**:商品圖(public,掛 custom domain 經 CDN)與會員私密檔(private,需授權才取)分開處理,不可全 public。
- **content-type / size / extension allowlist**:上傳前 server 驗證,拒絕非預期類型。
- **r2.dev 公開 URL 僅開發用**(官方明示有 rate limit、非 production),production 一律掛 custom domain。本平台已有 domain + CDN,順走。

**理由**:與 §5 同——讓 vibe 的人不可能寫錯。client 直傳 + client 決定 key = 把儲存安全賭在前端,與「不信前端、default deny」矛盾。

> ✏️ presigned URL vs Worker 簽發憑證的選擇、key 命名規則(`{tenant}/{resource}/{uuid}`)、lifecycle policy(過期清理)、private 檔的授權機制(Cloudflare Access vs WAF token vs 自家 API gate),實作期定。

### §7.3 圖片處理 — ✅ 前端壓縮(optimization)/ server max size(security)兩層分離

✅ **核心原則:壓縮是 optimization(前端)、max size 是 security(server)。max size 不可依賴前端壓縮執行。**

前端壓縮對「乖乖用網頁的人」有效;但 §7.2 已定上傳憑證 server 簽發,會繞過前端的人可直接拿 presigned URL 傳原圖。故 max size 的硬限制必須在 server / presigned URL policy,前端壓縮只減少正常使用者打到上限的機會。與 §5 auth 選 X、§4 default deny 同原則:安全邊界不放前端。

**前端壓縮(optimization 層)** — ✅ 工程正解,無選擇空間:

- 壓縮 library **用到才 `import()` 動態載入**,不進 main bundle(庫常數百 KB,塞首屏拖慢)。
- 輸出 **WebP**,但保留 source 可讀性判斷(HEIC / 透明 PNG 的轉換邊界要處理;iPhone HEIC 部分瀏覽器 decode 不了,壓縮前先確認 source 可讀)。
- 壓縮**在 web worker 跑,不在 main thread**(canvas / WASM 壓大圖會凍 UI 數百 ms,使用者以為當機)。
- 前端 size 檢查 = UX 層(及早提示「太大」),**非安全層**。

**max size(security 層)** — ✅ per-tenant 可設定:

- **執行點在 server / presigned URL policy**:簽發 presigned URL / Worker 憑證時帶 `Content-Length` 上限,超過直接拒,不讓落地。
- **per-tenant 可設定**(與 §4 / §7.1 per-tenant 哲學一致):平台給預設值,tenant 可覆寫。某 4K 商品攝影站與頭像站的上限不同。
- 預設值(佔位,實作期 / 產品判斷再覆寫):✏️ 商品圖 `__ MB`、會員頭像 `__ MB`。

**理由**:前端壓完的結果大小不可預測(取決於圖片內容),故不能當 max size 執行者——這是「壓縮 ≠ 限制」的根本。

> ✏️ 壓縮 library 選型(browser-image-compression 或自寫 canvas/WASM)、壓縮品質參數、預設 max size 數字,實作期定。

---

## §8 監控與告警

> 動機:這是學習環境,但很接近 production。使用者用 AI vibe coding,程式碼品質失誤(寫壞的 `useEffect` 無限觸發、retry 無上限、polling 沒清)會在後端表現為 request loop。共用單機下,一個 tenant 的失控會威脅全機。監控/防護設在**後端入口**——不管前端在 Vercel 還是 admin 自己的機器,失控 client 都從外面打這台後端,後端是唯一收得住所有來源的點。

### §8.1 止血層 — ✅ per-tenant rate limit(v1 必做,兩層)

✅ **per-tenant rate limit,Cloudflare WAF + platform 入口兩層。從 §2「夠用就好」升級為 v1 必做。**

- **Cloudflare 邊界(WAF rate limiting rule)**:打到機器前就擋,最省機器資源。所有流量本來就過 Cloudflare,此層近乎免費。擋大流量洪水。
- **platform 入口(middleware)**:Cloudflare 擋不掉的、或需 per-tenant 精細邏輯的,在此收。擋精細的 per-tenant 異常。

**為什麼是 per-tenant 而非全域**:request loop 不是 noisy neighbor(某人流量大),是**單一 tenant 異常暴衝**。粗的全域保護會讓暴衝 tenant 吃光全域額度、波及其他人。per-tenant 限制 = 擋住那一個、不擋別人。

**升級理由(與 production 韌性無關)**:防 vibe 失誤炸垮共用機器。這與「不讓 vibe 的人有犯錯材料」一貫——只是這次防的不是安全,是程式碼品質失誤,rate limit 是讓他們的 bug 炸不到別人、也炸不垮機器的牆。**止血靠自動擋,不靠人盯著看。**

### §8.2 觀測層 — ✅ 結構化 log

✅ **platform 寫結構化 log,per-tenant + per-endpoint 的 request count / rate。**

request loop 一發生,某 tenant×endpoint 的數字爆表,一眼可辨。v1 階段「擋得住 + 查得到」即足夠,不上 Prometheus + Grafana 全套(對單人單機是負擔)。

### §8.3 告警出口 — ✅ Discord webhook,單一公開頻道、全攤

✅ **Discord webhook,Akasa Lab 共用頻道。所有使用者都在此頻道,告警全攤、不分級、含 tenant 識別 + endpoint。**

技術形狀:Discord channel webhook。platform 在事件發生時 HTTP POST 一則 JSON 到 webhook URL。不養 bot、不 OAuth、不常駐連線。比 TG bot 輕。(不採 LINE:LINE Notify 已於 2025 停止服務。)

**刻意的設計選擇(非疏漏,寫明以免被誤判)**:告警**不做 tenant 隔離**,某 tenant request loop 的細節(誰、哪個 endpoint)全頻道可見。這是按學習環境定位的決定——犯錯被看到是學習過程的一部分,不替使用者遮。

**與 §2 資料隔離的關係(關鍵區分)**:**資料隔離是安全,告警透明是教學文化,兩者不衝突。** db-per-tenant 的資料牆照樣硬隔離(誰都讀不到誰的庫);只有「告警訊息」這一層共用。不可因告警全攤而誤推「租戶不隔離」。

**告警 vs 止血的分工**:止血(§8.1 rate limit)是自動擋、不靠人;告警(§8.3)是讓 admin 事後知道是誰 loop 了。因為 loop 已被 rate limit 擋住、機器不會垮,admin 晚一點看到也不釀災——故先擋、再通知,不靠告警把人半夜叫起來救火。

### §8.4 告警格式紀律 — ✅ 摘要 + dedup + 分級(防 Elastic 反模式)

✅ **核心原則:Discord 頻道的訊息密度要低到「每一則都值得看」。** 一旦有人因為太吵開始忽略它,它就死了——而它死的那天正好是真出大事、訊號淹沒在噪音裡的那天。與 meetup「不是每個問題都要用最炫的工具解」同源:不是每個事件都值得發一則告警。

要主動避開兩個常見反模式(Elastic 團隊頻犯):

**反模式一:raw dump。** 把整包 log object / stack trace / request context 原封不動丟進去,一則訊息洗版整個畫面,人眼無法 parse。
→ ✅ **告警是 log 的「結論」,不是 log 本身。** raw log 留機器上可查,丟進 Discord 的只能是「一眼讀懂要不要管」的摘要。一則告警 = 固定欄位結論:

- 嚴重度(INFO / WARNING / CRITICAL)
- tenant
- endpoint
- 一句話講發生什麼(「request rate 超閾值」「5xx 連續發生」)
- 關鍵數字(rate / count / 時間窗)
- 查 raw log 的指引(log id / 時間戳,讓 admin 去機器撈,**不把 raw 貼進來**)

**反模式二:no dedup。** 同一錯誤每次發生都發一則。在 request loop 場景致命——一個 loop 每秒 800 次,若每次都發,頻道三秒內被同一錯誤洗 2400 則,且 Discord webhook 自身會被 rate limit 掉。**做 rate limit 是為擋 loop,告警卻被 loop 觸發成另一場洪水,荒謬但常見。**
→ ✅ **dedup + 節流**:同一 `tenant × endpoint × 錯誤類型`,一個時間窗內聚合成一則,訊息寫「過去 N 秒發生 X 次」,窗內不重發。此條把「告警自己變洪水」掐死。

✅ **分級過濾在發送前做,不是全發到頻道再讓人忽略**(「全發了再忽略」正是 Elastic 頻道最後沒人看的死法):

- INFO(如部署完成):可發可不發
- WARNING(rate 接近閾值):發
- CRITICAL(rate limit 已熔斷某 tenant):必發

> ✏️ 原則寫死、數字留實作期:**必須摘要不 raw、必須 dedup、必須分級**為 board 釘死的架構原則;dedup 時間窗(如 60 秒)、各級 rate 閾值為參數,實作期 / 用量上來調。
> ✏️ 告警閾值、是否要 per-tenant thread(目前單一頻道)、是否補 admin-only 私密細節頻道,用量上來再定。
> ✏️ rate limit 的 per-tenant 計數狀態存哪:單機 modular monolith 下傾向 in-memory(夠用、重啟丟失可接受),是否落 Redis 待實證。

---

## §9 跨章交互與已知張力

> 本章不是新決策,是**主動揭露決策之間的接縫問題**。前八章每章單獨成立,但放在一起會產生跨章張力。一份完整方案應自列缺口,不留洞讓 reviewer 踩(與全文一貫的 disclosure 原則一致)。
> 標記:**[有答案待回填]** = 答案已清楚、僅未寫成實作 spec;**[開放]** = 真未決,需 review / 實作期定。

### §9.1 tenant 完整搬遷的協調 — [開放]

§2(db 隔離)、§7(bucket 隔離)、§5(provider secrets)各自宣稱「能獨立搬」,但一個 tenant 真要 migrate 出去 = **database + R2 bucket + provider secrets 三者要一致地搬**。三章都假設自己那塊獨立搬,**無一章講三者如何作為整體搬遷、搬到一半失敗如何回滾**。

未解問題:搬遷順序(先 db 還是先 bucket?)、中途失敗的補償(secrets 搬了 db 沒搬完怎麼辦)、搬遷期間 tenant 是否凍結寫入(避免搬到一半又有新資料進來)。

這是 board 目前最大的真空。**migration 友善是 §2 第一原則,但「友善」目前只證明到單一資源層,未證明到三資源協調層。**

### §9.2 outbox 在 migration 時的狀態 — [開放]

§3 outbox 會有未處理事件(in-flight)。§2 tenant 搬走時,那些未 relay 的 outbox event 怎麼處理?三種可能:跟著搬、丟棄、搬前先 drain 到空。

傾向:**搬前先 drain**(讓 outbox relay 跑到該 tenant 無 pending event,再凍結搬遷),但這跟 §9.1 的「搬遷期間凍結寫入」是同一個凍結窗口,需一起設計。未寫死,列開放。

### §9.3 idempotency key 鎖範圍 × db-per-tenant — [有答案待回填]

§3 要求「並發同 key 的鎖」,§2 是每 tenant 獨立 database。

答案清楚:**鎖在 tenant 自己的 database 內**。理由:金流本來就 per-tenant(每個 tenant 自己的 order / payment),同一 idempotency key 的並發只可能發生在同一 tenant 內,無跨庫並發場景。故 unique constraint + 鎖落在 tenant db 即足夠,不需要跨庫協調鎖。

board 原本未明寫,此處補上立場,實作期回填為 spec。

### §9.4 rate limit 計數狀態 × modular monolith — [有答案待回填]

§8.1 的 per-tenant 計數存哪。

答案傾向:**in-memory**。理由:modular monolith 是單一部署單元(§1),無多 instance 共享計數的問題;重啟丟失計數可接受(重啟後重新累積,request loop 會再次觸發、再次被擋,無永久漏洞)。

例外觸發:若未來 platform 跑多 worker process(非多機,而是單機多 worker),則 in-memory 計數會分裂,屆時需共享存儲(Redis)。**trigger = 多 worker 部署**。此前 in-memory 足夠。

### §9.5 章際依賴小結(給 reviewer 的地圖)

| 交互 | 涉及章 | 狀態 |
|------|--------|------|
| tenant 完整搬遷協調 | §2 + §5 + §7 | [開放] — 最大真空 |
| outbox migration 狀態 | §2 + §3 | [開放] |
| idempotency 鎖範圍 | §2 + §3 | [有答案待回填] |
| rate limit 計數狀態 | §1 + §8 | [有答案待回填] |
| 告警透明 vs 資料隔離 | §2 + §8 | 已於 §8.3 釐清(不衝突) |
| CORS origin × Vercel 前端 | §5 + 外部架構 | ✏️ 未進 board(見下方備註) |

**備註(CORS / Vercel 前端前提)**:使用者前端多部署於 Vercel(admin 自己的前端可進機器 Docker),platform 機器上僅後端 + tenant databases。此架構前提下,**per-tenant CORS / allowed origin 管理**是必要的存取邊界(每 tenant 註冊允許的 Vercel origin,platform 按 tenant 驗,非全域開放)。此條尚未寫成正式章節,列此提醒 reviewer:**判斷隔離設計時須知前端不在機器上,隔離牆在 db-per-tenant + per-tenant API 憑證 + per-tenant CORS origin 三處,皆為邏輯隔離而非檔案系統隔離。**

---

## 落地後的下一步

1. 此 board 給審閱者(Codex)做獨立 review。
2. review 過的決議轉成實作期的 spec / schema(各 primitive 一份)。
3. 本 board 的 ✏️ 項在實作期逐一收斂,收斂後回填。
4. 已 ✅ 項為 settled decision,**不再回頭重開**;要動須有對應 trigger 成立。
5. §9 的 **[開放]** 項(tenant 完整搬遷協調、outbox migration 狀態)是 review 的優先標的——它們是跨章真空,需在實作 migration 機制前定案;**[有答案待回填]** 項照所列立場轉 spec 即可。

---

## 附:時程定錨

目標:**9 月以前完成平台 v1**。v1 範圍 = product+tag / cart / member / collection / notification(email + SSE)/ access policy / 物件儲存(§7 per-tenant bucket 基礎上傳契約)。**不含金流**(§3 後階段)、**不含 AI 應用**(平台後)。

從 2026-06 底算約 10 週。瓶頸不在工程量(對單一架構者 + reusable kit,此範圍寬裕),在「定義定不定得清」——而定義正是本 board 在做的事。
