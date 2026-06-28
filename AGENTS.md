# AGENTS.md

# Akasa Platform Backend AI Guardrails

這份規則是 always-on hard boundary。即使沒有載入任何 skill，也必須遵守。

## 進入專案後先讀

1. AGENTS.md
2. CLAUDE.md
3. .claude/rules/
4. .claude/docs/*HANDOFF*.md
5. docs/platform-architecture/decision-board.md
6. 與任務相關的 architecture、contract、code 與目前 Git diff

文件互相衝突時，停止並請 Fortes 拍板。

## 硬性 Guardrails

- 未經 Fortes 明確授權，不得 commit、push、merge、deploy 或 release。
- Contract、schema、DTO、event payload、permission、domain rule 不明時，停止並列 decision list。
- 不得靜默擴張 scope 或修改 bounded context 以外的檔案。
- 不得把 domain rule、permission rule 或 server state 複製成第二份來源。
- 沒有 delivery evidence，不得宣稱 done。
- Failed、skipped、not-run checks 必須揭露。
- 未解 risk、contract drift、未驗證範圍必須保留在 handoff。
- AI 產出必須有人類 owner。AI 不能自行放行。

## Public Git 安全邊界

- tracked file、commit message 與 Git history 一律視為外部可讀。
- Secret、主機 IP、正式部署細節、部署目錄、內部 runbook、公司專案路徑與私有 handoff 不得進 public Git。
- .env.example 只能放 placeholder；.env 真值必須 ignore。
- Secret 曾進 history 時立即 rotate；rewrite history 不能取代 rotation。

## 建議使用 Skills

- 實作前固定 contract：contract-review
- Review AI-assisted diff：pr-review
- 交付前收集證據：delivery-verify
- 跨 session 交接：handoff-maintainer
- Commit / push / 公開文件前：security-boundary-review
