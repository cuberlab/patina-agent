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

## 給 agent

- 改名、搬檔時檔案和側檔一起搬；寫描述、標籤的做法見 [README](../README.md)。
- 在 Finder 搬檔案也可以，Patina 開著時（或下次打開圖庫時）會自己接回側檔。
- 不要改照片或影片檔本身的內容（不寫 EXIF／IPTC），備份用 MD5 驗證。
