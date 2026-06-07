---
name: record
description: 把當前 Claude 對話整理成一份任務記錄，用 canonical frontmatter push 進你的 private session-records inbox repo（由環境變數 $RECORD_INBOX_REPO 指定），供本地 reconcile 回流到 personal_os / session-board。雲端 session 結束想留痕、用戶說 /record、記錄這次對話、存成任務時使用。
---

# /personal-os:record（雲端版 · 寫入 private inbox）

把當前對話整理成結構化任務記錄，push 進你的 **private inbox repo**（環境變數 `$RECORD_INBOX_REPO`，例 `you/session-records`）的 `records/{slug}.md`。

> **為什麼寫 inbox，不寫當前 repo / 不寫 personal_os**：雲端碰不到本機 personal_os（local-only 唯一副本）；而把記錄寫進「當前工作 repo」會洩漏 public repo、污染 git 史、撞並行寫入。改 push 一個專用 private inbox 最安全。本地 `record_reconcile.py` 會把 inbox 併進 `personal_os/tasks/`，`/session-board` 才看得到。本檔**不含任何本機路徑與個資**，可安全放 public marketplace。

## inbox 目標（預設已內建，通常你什麼都不用設）
- inbox repo = `$RECORD_INBOX_REPO`（若雲端環境有設）否則用**內建預設 `atomchung/session-records`**。
- **絕不寫進當前工作 repo**——只寫 inbox（避免 public 洩漏 / git 史污染 / 並行 race）。
- **`GH_TOKEN`（選填）**：僅當雲端內建 git 認證無法 push 到 inbox 時才需要——用一把**只授權該 inbox repo、`contents:write`** 的 fine-grained PAT，設在 cloud environment settings。

## ⚠️ 兩個不可妥協的價值核心
1. frontmatter 必須是 **canonical YAML**（下方 schema）。session-board parser 只認 frontmatter，舊式 `> Status:` 引言區塊會被**靜默漏掉**。
2. status 語意精確：`paused` 只給「等外部、自己推不動」；卡點欄位**只能叫 `blocked_by`**（parser 只認這名字）。

## 工作流

### 1. 決定 slug
kebab-case，一行看得出是什麼任務。

### 2. 掃描對話整理
做了什麼 / 關鍵發現（2–5）/ 產出檔案 / 待辦 / 下一步。去重、去 noise。

### 3. 組 record（canonical frontmatter）
```yaml
---
slug: <slug>
status: in_progress        # open / in_progress / paused / done / archived
created: <YYYY-MM-DD>
last_session: <YYYY-MM-DD>
next_action: <一句話、可執行>
related_files: []
tags: []
# blocked_by: <paused 時必填>
origin: cloud
source_repo: <當前工作 repo 名>   # 你這次 session 在哪個 repo（給 reconcile 溯源）
synced: false
---
```
**status 決策樹（命中即止）**：① 用戶說做完→`done` ② 等外部事件（回信/審核/財報/他人交付）→`paused`＋`blocked_by` 必填 ③ 其他→`in_progress`（瓶頸在自己＝in_progress，不要判 paused）。
**next_action**：單一動作（動詞＋對象/判斷條件），多步只寫第一步。
內容結構：`# Title` / `## Context` / `## Sessions` →`### {date} Session`（關鍵發現 / 檔案清單 / 待辦 / 後續行動）。

### 4. 立即 push 到 inbox（**不要等 session 結束**——雲端 crash 會丟失）
先解析 inbox 與 slug；優先用 git，失敗才走 API。

```bash
INBOX="${RECORD_INBOX_REPO:-atomchung/session-records}"   # 預設已內建，通常不用設
# A. git（最佳，無需額外 token）
cd /tmp && rm -rf inbox && git clone --depth 1 "https://github.com/$INBOX" inbox && cd inbox
mkdir -p records
# 寫 records/<slug>.md；若已存在 → 在其 ## Sessions 底下「追加」新 session 區塊，不覆蓋
git add records/<slug>.md && git commit -m "record: <slug>"
git pull --rebase && git push        # 多 session 並行安全
```
```bash
# B. fallback：Contents API（需 $GH_TOKEN）。先 GET 舊 sha（檔已存在才有），再 PUT base64 內容
curl -s -H "Authorization: token $GH_TOKEN" \
  "https://api.github.com/repos/$INBOX/contents/records/<slug>.md"   # 取 .sha（若有）
curl -s -X PUT -H "Authorization: token $GH_TOKEN" \
  "https://api.github.com/repos/$INBOX/contents/records/<slug>.md" \
  -d "{\"message\":\"record: <slug>\",\"content\":\"$(base64 -w0 <slug>.md)\",\"sha\":\"<舊sha或省略>\"}"
```

### 5. 回報
inbox 路徑（`$RECORD_INBOX_REPO/records/<slug>.md`）/ status / next_action，並提醒：「本地 `record_reconcile.py`（launchd）會把它併進 personal_os，`/session-board` 就看得到。」

## 排除清單（不是 session task，不套）
`week-fit-*` / `week-review-*` / `recap-*` —— 週報走對應 skill。

## Frontmatter Schema（與 session-board 共用，務必逐字一致）
| 欄位 | 必填 | 值 |
|---|---|---|
| `slug` | ✅ | 同檔名 |
| `status` | ✅ | open / in_progress / paused / done / archived |
| `created` / `last_session` | ✅ | YYYY-MM-DD |
| `next_action` | 推薦 | ≤ 30 字 |
| `blocked_by` | paused 必填 | 卡住原因 |
| `origin` | ✅ | `cloud` |
| `source_repo` | ✅ | 你這次 session 在的 repo 名 |
| `synced` | ✅ | `false` → 本地 reconcile 後改 `true` |
| `related_files` / `tags` | 選填 | |
