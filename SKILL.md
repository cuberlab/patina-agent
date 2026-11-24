---
name: patina
description: >
  整理 Patina 圖庫（Mac app，一個資料夾就是一個圖庫）：直接改圖庫裡的 JSON 側檔來加標籤、寫描述、網址、評分，
  改名、搬檔、資料夾改名時連側檔一起搬，檢查側檔格式；讀 board 上使用者標的標籤並回覆，在 board 放圖、加文字、標籤、畫線；
  產生 QR 讓 iPhone 版配對，回報 Patina 的 bug。
  觸發詞：Patina、圖庫、參考圖、加標籤、寫描述、整理照片、board、標註、回覆標籤、Eagle 匯入、iPhone 配對、手機掃 QR、Patina bug。
---

# Patina 圖庫整理

照片、影片、字體是圖庫資料夾裡的一般檔案；描述、標籤、網址、評分放在 `<圖庫>/.patina/meta/<相對路徑>.json` 側檔（例如 `2025/旅行/IMG_0001.jpg` → `.patina/meta/2025/旅行/IMG_0001.jpg.json`）。不用安裝任何程式。格式見 [docs/vault-format.md](docs/vault-format.md)，完整規則與 Python 工具見 [README.md](README.md)。

## 開始前

1. 確認圖庫根目錄有 `.patina/vault.json`。
2. 把 README「側檔工具」那段 Python 存成圖庫外面的 `patina_meta.py`，改 `VAULT`、`AGENT`。
3. 跑 `check()`：列出格式不對的側檔就停下來告訴使用者。列出衝突副本（`.sync-conflict-`）時：側檔的請使用者打開 Patina 0.2.4 以上讓它自動合併，照片、影片的告訴使用者。

## 做事

- 標籤、描述、網址、評分：`edit()`。已有側檔就讀出、只改要改的欄位、整份寫回；沒有就新建 `{"format": 1, ...}`。
- 新建側檔不寫 `fingerprint`，改既有側檔時原樣保留（Patina 0.2.2 起會自己重算）；`format`、`source` 也原樣保留。不要在側檔放自己的欄位，Patina 改寫時會丟掉。
- 寫側檔要原子寫入：同資料夾先寫 `.tmp` 暫存檔，再改名蓋過去。
- 標籤去頭尾空白、去重複，分大小寫；`rating` 是 0 到 5 的整數；`updated_by` 寫自己的名字，`updated_at` 寫 UTC 的 `YYYY-MM-DDTHH:MM:SSZ`。
- 描述、標籤、網址、評分全空而且沒有 `source`：刪掉側檔。
- 改名、搬檔：`move()`，檔案和側檔一起搬，副檔名不改，撞名就換名字。資料夾改名或搬移：`move_folder()`，`.patina/meta/` 底下對應的資料夾一起搬。路徑大小寫要和磁碟上一樣。
- 刪檔案或資料夾、從 Eagle 匯入：請使用者在 Patina app 裡做。

## Boards

board 是 `.patina/boards/<id>.json`，一個 board 一個檔，放圖、文字、線、標籤。把 README「Board 工具」那段存成 `patina_board.py`，改 `VAULT`、`AGENT`。

1. 使用者在 board 上標了東西：`notes(<board>)` 列沒回覆的標籤（編號、文字、誰、哪張圖、`px` 原圖像素）。
2. 直接讀圖庫裡的那張圖看標籤指的地方（`px` 是原圖像素，不是 board 上的大小），做事。
3. `reply(<board>, n, "done"|"skip"|"note", "…")` 回覆每個處理過的標籤。
4. 要指給使用者看：`add_line()`、`add_label()`（座標是原圖像素）；也可以 `add_image()`、`add_text()`。
5. 守則：只經過 `update()` 寫（拿 `.patina/` 的鎖、寫前比對、被改過就重做、整份原子寫入）；不認得的欄位和元素原樣留著；改了元素就更新它和 board 的 `updated_at`／`updated_by`，回覆不動標籤的 `updated_at`；標籤編號不重用；拿掉元素用 `remove()` 留墓碑；`format` 不是 1 不寫；刪 board 請使用者在 app 裡做。

## 不能碰

- 照片、影片、字體檔本身：不寫 EXIF／IPTC、不重新存檔、不轉檔。
- `.patina/vault.json`、`.patina/removed/`（含刪掉的 board）、`.patina/log/`（只讀）。
- 衝突副本：不要自己刪或合併（側檔、board 的 Patina 會合併，照片、影片的讓使用者決定）。
- Syncthing 忽略規則：不能擋到 `.patina/`（不要加 `.*`、`.patina`）；要建議規則時照 README「Syncthing 設定」。

## 收尾

- `check()` 印「沒有問題」才算完成；改了 board 也跑 `check_boards()`。Patina 讀不了的側檔不會報錯，只會當成沒有整理資料。
- 告訴使用者改了哪些檔案、加了哪些標籤、回覆了 board 上哪幾個標籤。

## iPhone 配對

使用者要讓 iPhone 的 Patina 同步 Mac 的圖庫時，照 README「iPhone 配對」：

1. 產生 Mac 的 Syncthing 裝置 ID 的 QR 給使用者看（QR 內容就是裝置 ID）。
2. 手機掃完，`syncthing cli show pending devices` 確認是使用者的手機，再加裝置、把圖庫資料夾分享給它。
3. 告訴使用者在手機上勾要放的圖庫。

## 回報 bug

Patina app 出錯，或照 README 改圖庫時遇到問題：

1. 先找有沒有一樣的 issue：`gh issue list --repo cuberlab/patina-agent --search "<關鍵字>"`。
2. 沒有就開：`gh issue create --repo cuberlab/patina-agent --label bug`，內文照 bug 範本的欄位（app 版本、macOS 版本、做了什麼、預期、實際、錯誤原文、相關紀錄、重現步驟）。
3. 很慢、卡住：另外附圖庫規模、當時在做什麼，和卡住當下的 sample 摘要（指令見 README「很慢、卡住」）。
4. repo 是公開的：不貼私人照片與描述內容，家目錄路徑寫 `~`，私人檔名改成代稱。
