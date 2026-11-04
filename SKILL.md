---
name: patina
description: >
  整理 Patina 圖庫（Mac app，一個資料夾就是一個圖庫）：用 patina CLI 找檔案、加標籤、寫描述、
  評分、改名、建資料夾、搬檔，在 Finder 或 shell 搬檔後接回側檔，檢查格式，從 Eagle 匯入，回報 Patina 的 bug。
  觸發詞：Patina、圖庫、參考圖、加標籤、寫描述、整理照片、Eagle 匯入、Patina bug。
---

# Patina 圖庫整理

照片、影片、字體是圖庫資料夾裡的一般檔案；描述、標籤、網址、評分放在圖庫根目錄 `.patina/` 的側檔。格式見 [docs/vault-format.md](docs/vault-format.md)，每個指令的寫法見 [README.md](README.md)。

## 開始前

1. `patina --version` 確認裝了 CLI；沒有就照 README「安裝 CLI」。
2. 指定圖庫：`export PATINA_VAULT=<圖庫路徑>`（或每個指令加 `--vault`），`patina info` 要印得出檔案數。
3. `patina check`：有問題先處理；列出衝突副本（`.sync-conflict-`）就停下來告訴使用者。

## 做事

- 找：`patina search`、`patina ls`、`patina show`、`patina tags`。
- 整理資料：`patina set <檔案> ... --by <agent 名>`；全圖庫標籤改名或合併用 `patina tag-rename`。
- 改名、搬檔：`patina rename`、`patina mkdir`、`patina mv`。資料夾改名或搬移在 shell 做，做完跑 `patina reconcile`。
- 刪檔案或資料夾：請使用者在 Patina app 裡做。
- 不改照片、影片、字體檔本身。

## 收尾

- `patina check` 印「沒有問題」、exit 0 才算完成。
- 告訴使用者改了哪些檔案、加了哪些標籤。

## 回報 bug

Patina app 或 CLI 出錯時：

1. 先找有沒有一樣的 issue：`gh issue list --repo cuberlab/patina-agent --search "<關鍵字>"`。
2. 沒有就開：`gh issue create --repo cuberlab/patina-agent --label bug`，內文照 bug 範本的欄位（app 版本、`patina --version`、macOS 版本、做了什麼、預期、實際、錯誤原文、相關紀錄、重現步驟）。
3. repo 是公開的：不貼私人照片與描述內容，家目錄路徑寫 `~`，私人檔名改成代稱。
