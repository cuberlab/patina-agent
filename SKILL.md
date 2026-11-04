---
name: patina
description: >
  整理 Patina 圖庫（Mac app，一個資料夾就是一個圖庫）：直接改圖庫裡的 JSON 側檔來加標籤、寫描述、網址、評分，
  改名、搬檔、資料夾改名時連側檔一起搬，檢查側檔格式，回報 Patina 的 bug。
  觸發詞：Patina、圖庫、參考圖、加標籤、寫描述、整理照片、Eagle 匯入、Patina bug。
---

# Patina 圖庫整理

照片、影片、字體是圖庫資料夾裡的一般檔案；描述、標籤、網址、評分放在 `<圖庫>/.patina/meta/<相對路徑>.json` 側檔（例如 `2025/旅行/IMG_0001.jpg` → `.patina/meta/2025/旅行/IMG_0001.jpg.json`）。不用安裝任何程式。格式見 [docs/vault-format.md](docs/vault-format.md)，完整規則與 Python 工具見 [README.md](README.md)。

## 開始前

1. 確認圖庫根目錄有 `.patina/vault.json`。
2. 把 README「側檔工具」那段 Python 存成圖庫外面的 `patina_meta.py`，改 `VAULT`、`AGENT`。
3. 跑 `check()`：列出衝突副本（`.sync-conflict-`）或格式不對的側檔，就停下來告訴使用者。

## 做事

- 標籤、描述、網址、評分：`edit()`。已有側檔就讀出、只改要改的欄位、整份寫回；沒有就新建 `{"format": 1, ...}`。
- 新建側檔不寫 `fingerprint`，改既有側檔時原樣保留（Patina 0.2.2 起會自己重算）；`format`、`source` 也原樣保留。不要在側檔放自己的欄位，Patina 改寫時會丟掉。
- 寫側檔要原子寫入：同資料夾先寫 `.tmp` 暫存檔，再改名蓋過去。
- 標籤去頭尾空白、去重複，分大小寫；`rating` 是 0 到 5 的整數；`updated_by` 寫自己的名字，`updated_at` 寫 UTC 的 `YYYY-MM-DDTHH:MM:SSZ`。
- 描述、標籤、網址、評分全空而且沒有 `source`：刪掉側檔。
- 改名、搬檔：`move()`，檔案和側檔一起搬，副檔名不改，撞名就換名字。資料夾改名或搬移：`move_folder()`，`.patina/meta/` 底下對應的資料夾一起搬。路徑大小寫要和磁碟上一樣。
- 刪檔案或資料夾、從 Eagle 匯入：請使用者在 Patina app 裡做。

## 不能碰

- 照片、影片、字體檔本身：不寫 EXIF／IPTC、不重新存檔、不轉檔。
- `.patina/vault.json`、`.patina/removed/`、`.patina/log/`（只讀）。
- 衝突副本：告訴使用者，不要自己刪或合併。

## 收尾

- `check()` 印「沒有問題」才算完成。Patina 讀不了的側檔不會報錯，只會當成沒有整理資料。
- 告訴使用者改了哪些檔案、加了哪些標籤。

## 回報 bug

Patina app 出錯，或照 README 改圖庫時遇到問題：

1. 先找有沒有一樣的 issue：`gh issue list --repo cuberlab/patina-agent --search "<關鍵字>"`。
2. 沒有就開：`gh issue create --repo cuberlab/patina-agent --label bug`，內文照 bug 範本的欄位（app 版本、macOS 版本、做了什麼、預期、實際、錯誤原文、相關紀錄、重現步驟）。
3. repo 是公開的：不貼私人照片與描述內容，家目錄路徑寫 `~`，私人檔名改成代稱。
