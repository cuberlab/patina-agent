> Patina 圖庫格式。Patina 改格式時這份一起更新。

# 圖庫格式（format 1）

Patina 把任何資料夾當成圖庫打開。照片、影片、字體就是資料夾裡的一般檔案；Patina 只在根目錄加一個隱藏資料夾 `.patina/`。Agent 可以直接讀寫，做法見 [README](../README.md)。

## 檔案

| 路徑 | 內容 |
|---|---|
| `<圖庫>/**` | 照片、影片、字體本身。資料夾就是分類。名稱就是檔名，改名就是改檔案。 |
| `.patina/vault.json` | `{"format": 1, "id": "<uuid>", "created_at": "..."}` |
| `.patina/meta/<相對路徑>.json` | 一個檔案一份側檔，路徑跟著檔案。例如 `2025/Kruje/A.jpg` → `.patina/meta/2025/Kruje/A.jpg.json`。沒有任何整理資料的檔案沒有側檔。 |
| `.patina/removed/<相對路徑>.json` | 從 Patina 丟進垃圾桶的檔案（含刪除資料夾時裡面的所有檔案）的側檔。檔案從垃圾桶放回後會自動接回。 |
| `.patina/boards/<id>.json` | 一個 board 一個檔（見「Boards」）。 |
| `.patina/removed/boards/<id>.json` | 刪掉的 board，可以在 app 裡還原。 |
| `.patina/log/<年-月>-<主機>.jsonl` | 每次寫入的紀錄：時間、誰、做了什麼、改前改後。每台電腦寫自己的檔，Syncthing 不會衝突。 |
| `.patina/log/errors-<主機>.jsonl` | app 遇到的錯誤。 |
| `.patina/log/eagle-import-<UTC 時間>.jsonl` | app「從 Eagle 匯入」每次一個檔（時間如 `20261104T013642Z`）。每個項目一行：`eagle_id`、`src`（Eagle 裡的實體檔）、`dest`（圖庫相對路徑）、`status`（`copied`／`exists`／`error`）、`warnings`、`error`。 |

略過的檔案：`.` 開頭、`~$` 開頭、Syncthing 的 `.sync-conflict-` 副本、不認得的副檔名。

認得的副檔名：

- 照片：jpg jpeg png heic heif webp gif tif tiff bmp avif，RAW（cr3 cr2 arw nef dng raf orf rw2）
- 影片：mov mp4 m4v avi mkv webm mts m2ts 3gp（播放靠 AVFoundation，mkv／webm 只列出不播）
- 字體：ttf otf ttc otc woff woff2
- 其他：pdf svg psd ai eps

## 側檔

```json
{
  "format": 1,
  "description": "從陰暗拱門內看向城堡庭院。",
  "tags": ["阿爾巴尼亞", "拱門"],
  "url": "",
  "rating": 0,
  "source": {"eagle": "MPPS9SYI7KGB6"},
  "fingerprint": {"size": 1016085, "sha256_edges": "…"},
  "updated_at": "2026-11-03T15:00:00Z",
  "updated_by": "claude"
}
```

- `description`、`tags`、`url`、`rating`（0–5）是使用者資料；全部空的時候側檔會被刪掉。
- `fingerprint` 是檔案大小加上頭尾各 64 KB 的 SHA-256，Patina 自己算：app 打開圖庫和偵測到變動時會補上缺的、校正對不上的（只改這個欄位）。外部直接寫側檔時不用寫這個欄位，寫了也會被校正。檔案在 Finder 被改名或搬走時，Patina 用它把側檔接回新位置。
- 上面以外的欄位，Patina 改寫側檔時不保留。
- 寫側檔要整份原子寫入（先寫暫存檔再改名）。
- 色票、尺寸、片長這類從檔案算得出來的資料不存側檔，app 自己放快取。

## Boards

board 是一張無限大的畫布：放圖庫裡的圖、文字、線、標籤。一個 board 一個檔 `.patina/boards/<id>.json`，id 是 `b` 加 12 個小寫十六進位字元，檔名就是 id。

```json
{
  "format": 1,
  "id": "b3f9c2a7d1e4f",
  "name": "Kruje 石造建築",
  "created_at": "2026-11-24T06:52:00Z",
  "updated_at": "2026-11-24T06:52:00Z",
  "updated_by": "user",
  "elements": [
    {"id":"e1a2b3c4d5e6f","type":"image","path":"2025/Kruje/A.jpg","fingerprint":{"size":1016085,"sha256_edges":"…"},"x":504,"y":320,"w":288,"h":192,"z":3,"created_at":"…","updated_at":"…","updated_by":"user"},
    {"id":"e…","type":"text","text":"石牆的做法","x":836,"y":548,"size":15,"color":"gray","z":9,"created_at":"…","updated_at":"…","updated_by":"user"},
    {"id":"e…","type":"stroke","points":[[0.71,0.58],[0.69,0.42]],"color":"orange","arrow":true,"on":"e1a2b3c4d5e6f","z":10,"created_at":"…","updated_at":"…","updated_by":"user"},
    {"id":"e…","type":"label","n":3,"text":"拱的比例","x":0.49,"y":0.27,"on":"e1a2b3c4d5e6f","color":"orange","by":"user","replies":[{"id":"r…","by":"claude","at":"…","status":"done","text":"改成 2:3 了"}],"z":11,"created_at":"…","updated_at":"…","updated_by":"user"}
  ],
  "removed": [{"id":"e…","n":2,"at":"…"}]
}
```

| 欄位 | 說明 |
|---|---|
| `type` | `image` 圖、`text` 文字塊、`stroke` 線、`label` 標籤。每個元素都有 `id`（`e` 加 12 個十六進位）、`z`、`created_at`、`updated_at`、`updated_by` |
| `image` | `path` 是圖庫相對路徑、`fingerprint` 同側檔的指紋；`x`、`y` 是左上角，`w`、`h` 是大小。只記參照，檔案不見時元素留著（畫成灰框加檔名） |
| `text` | `text`、`x`、`y`（左上角）、`size`（字級，預設 15）、`color`（預設 `gray`） |
| `stroke` | `points` 是 `[[x, y], …]`（一點是點、兩點是直線、多點是折線）、`arrow`（最後一點畫箭頭）、`color`（預設 `orange`）、`on`（綁的圖，可省略） |
| `label` | `n` 編號、`text`、`x`、`y`（泡泡左上角）、`on`（綁的圖或線）、`color`（預設 `orange`）、`by`（誰加的）、`replies` |
| `replies` | `id`（`r` 開頭）、`by`、`at`、`status`（`done` 已改、`skip` 沒改、`note` 只回話）、`text` |
| `removed` | 拿掉的元素的墓碑：`id`、`at`，標籤多 `n`。給合併用 |

- 座標：沒綁東西的元素是畫布的 world 單位（x 向右、y 向下）。`on` 是圖時，`x`、`y`、`points` 是那張圖的比例（x 除以圖寬、y 除以圖高，0–1），圖搬移、縮放時跟著走。`on` 是線時，跟那條線的點同一個座標系（線綁圖就是那張圖的比例）；搬那條線時標籤一起搬。
- `z` 大的在上面；移到最前是目前最大值加 1（整數）。綁在圖上的線和標籤畫在那張圖之後，圖移到最前時標註還在圖上面。
- 顏色只有 `blue`（`#0072B2`）、`orange`（`#E07B00`）、`gray`（`#2F3540`）、`white`（`#FFFFFF`）。
- 標籤編號 `n` 每個 board 從 1 起，是目前標籤和墓碑裡最大的加 1，拿掉的不重用。
- 拿掉圖時綁在上面的線和標籤一起拿掉，拿掉線時綁在上面的標籤一起拿掉，每個都留墓碑。超過 90 天的墓碑在 Patina 下次寫入時清掉，但編號最大的標籤墓碑留著（編號才不會重用）。
- 時間是 UTC、到秒；讀進來也只看到秒。
- 不認得的 `type`、欄位原樣保留，寫回時不丟；`format` 不是 1 的 board 不寫入、回報錯誤。
- 檔案的寫法：最外層一個欄位一行、`elements` 和 `removed` 一個項目一行、鍵照字母排；同樣的內容寫出來的位元組一樣。整份原子寫入（先寫暫存檔再改名）。
- 刪 board：檔案搬到 `.patina/removed/boards/<id>.json`，圖不動。
- 在 Patina 裡改名、搬移檔案或資料夾時，所有 board 的 `path` 跟著改；在 Finder 搬的，app 打開圖庫和偵測到變動時用指紋接回（沒有指紋的圖順便補上）。檔案丟垃圾桶時元素留著。
- 紀錄（`.patina/log/`，`path` 是 `.patina/boards/<id>.json`）：`board-new`、`board-rename`、`board-delete`、`board-restore`、`board-add`（放圖）、`board-remove`（拿掉，`elements` 是拿掉的元素）、`board-reply`、`board-relink`（路徑跟著改）、`merge-conflict`，也可能看到 `board-text`、`board-line`、`board-label`、`board-move`、`board-edit`。app 的手繪線、移動、改字不逐筆記。
- 貼上的圖存成圖庫根目錄的一般檔案，檔名 `貼上 2026-11-24 14.31.08.png`（當地時間，撞名加編號），原始資料不重新編碼。
- 兩台裝置都改了同一個 board 留下的 `.sync-conflict-` 副本，app 打開圖庫和偵測到變動時逐元素自動合併：同 id 的元素取 `updated_at` 比較新的，回覆照 `id` 取聯集，標籤編號撞號時比較晚建立的改成下一個空號。`.patina/removed/boards/` 的副本不動。

直接寫 board 檔時：

- 讀、改、寫要整份原子寫入（寫到同資料夾的暫存檔再改名），不要只改一部分。寫回前再讀一次，和讀進來的不一樣就重讀重做。Patina 寫 board 時對 `.patina/` 資料夾拿 `flock` 排他鎖，外部程式也拿這把，就不會和 app 同時寫。`format` 不是 1 的不要寫。
- 不認得的欄位和元素原樣留著。
- 新元素的 `id` 用 `e` 加 12 個隨機的小寫十六進位字元，回覆用 `r` 開頭；`created_at`、`updated_at` 寫現在（UTC），`updated_by` 寫自己；改了元素就更新它的 `updated_at`、`updated_by`，也更新 board 的。
- 綁在圖上的座標是比例（像素除以圖的寬高；圖的寬高照 EXIF 方向轉正後），不是像素。
- 新標籤的 `n` 是目前標籤和 `removed` 裡最大的 `n` 加 1；`z` 是目前最大的加 1。
- 拿掉元素時從 `elements` 刪掉，`removed` 加 `{"id", "at"}`（標籤加 `n`）；綁在上面的線和標籤一起拿掉。
- 回覆只加在標籤的 `replies` 後面，不要改標籤本身的 `updated_at`。
- 圖的 `fingerprint` 可以不寫，app 會補上。

## Syncthing 設定

圖庫整個資料夾都要同步，包括 `.patina/`。忽略規則（`.stignore`）只放系統產生的檔案，建議的內容與注意事項見 [README「Syncthing 設定」](../README.md#syncthing-設定)。擋到 `.patina/` 時 Patina 0.2.4 起會跳錯誤提示。

## 給 agent

- 改名、搬檔時檔案和側檔一起搬；寫描述、標籤的做法見 [README](../README.md)。
- 在 Finder 搬檔案也可以，Patina 開著時（或下次打開圖庫時）會自己接回側檔。
- 不要改照片或影片檔本身的內容（不寫 EXIF／IPTC），備份用 MD5 驗證。
- 讀標籤、回覆、在 board 上放圖、加文字、標籤、畫線的做法見 [README「Boards」](../README.md#boards)。
