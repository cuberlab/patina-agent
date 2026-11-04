# patina-agent

Patina 的 agent kit：圖庫格式、agent 整理守則、命令列工具 `patina` 的用法，以及回報 bug 的方式。CLI 從本 repo 的 [Release](https://github.com/cuberlab/patina-agent/releases/latest) 下載。

## Patina 是什麼

Patina 是一個 Mac 圖庫 app，照片、影片和字體可以放在一起整理。圖庫就是一個普通的資料夾：子資料夾是分類，檔名就是名稱，檔案一直留在原處。描述、標籤、網址和評分存在圖庫根目錄的隱藏資料夾 `.patina/`，每個檔案一份 JSON 側檔。完整格式見 [docs/vault-format.md](docs/vault-format.md)。

```
~/Patina/
├── 2025/旅行/IMG_0001.jpg
├── 字體/SomeFont.otf
└── .patina/
    ├── vault.json                            圖庫 id 與 format
    ├── meta/2025/旅行/IMG_0001.jpg.json      側檔，路徑跟著檔案
    ├── removed/                              丟進垃圾桶的檔案的側檔
    └── log/                                  寫入紀錄、錯誤、Eagle 匯入紀錄
```

## 安裝 CLI

需要 macOS 14 以上。

```sh
cd "$(mktemp -d)"
curl -fLO https://github.com/cuberlab/patina-agent/releases/latest/download/patina-cli-macos.zip
shasum -a 256 patina-cli-macos.zip
unzip patina-cli-macos.zip
mkdir -p ~/.local/bin && mv patina ~/.local/bin/
patina --version
```

- `shasum` 印出的值要和[最新 Release](https://github.com/cuberlab/patina-agent/releases/latest) 說明裡的 SHA-256 完全一樣；不一樣就停下來，不要安裝。
- `~/.local/bin` 不在 PATH 時，把 `export PATH="$HOME/.local/bin:$PATH"` 加進 `~/.zshrc`，或放到 PATH 裡的其他目錄。
- 更新：同樣步驟再做一次，蓋掉舊的 `patina`。
- Release 裡的 `patina` 是 universal（Apple 晶片和 Intel 都能跑），有 Developer ID（Cuber Lab Ltd.）簽章並經 Apple 公證。

## Agent 守則

1. **不改照片、影片、字體檔本身**：不寫 EXIF／IPTC、不重新存檔、不轉檔。備份用 MD5 驗證；側檔靠檔案大小加頭尾各 64 KB 的 SHA-256（`fingerprint`）認檔，內容一改就接不回。
2. **整理資料優先用 `patina` CLI，並帶 `--by <agent 名>`**（例如 `--by openclaw`）。CLI 會一起搬側檔、更新指紋、寫紀錄；沒帶 `--by` 時紀錄寫成 `cli`。
3. **一定要直接寫側檔時，整份原子寫入**：先在同一個資料夾寫暫存檔（檔名不要以 `.json` 結尾），再改名蓋過去。`format` 維持 `1`，`source`、`fingerprint` 照原樣保留。寫完跑 `patina check`。
4. **在 Finder 或 shell 直接搬檔、改名之後，跑 `patina reconcile`** 把側檔接回新位置。CLI 沒有改名或搬移資料夾的指令，資料夾用這個方式處理。
5. **`.patina/log/` 只讀**：不改、不刪、不重寫紀錄檔。每台電腦只寫檔名含自己主機名稱的檔，別台電腦的紀錄檔更不要動。
6. **不動 `.patina/vault.json` 與 `.patina/removed/`**：`vault.json` 的 id 是圖庫身分；`removed/` 是丟進垃圾桶的檔案的側檔，檔案放回來時靠它接回。
7. **刪檔案或資料夾請使用者在 Patina app 裡做**（會進 macOS 垃圾桶，可以復原）。CLI 沒有刪除指令；用 shell 刪掉的檔案，側檔會變成「找不到檔案的側檔」。
8. **色票、尺寸、片長這類從檔案算得出來的資料不寫進側檔**，app 自己算。
9. **`patina check` 列出衝突副本（`.sync-conflict-`）時回報使用者**，不要自己挑一份刪掉。
10. **改完跑 `patina check`**，印「沒有問題」、exit 0 才算完成。

`.` 開頭、`~$` 開頭、含 `.sync-conflict-` 的檔案和不認得的副檔名不算圖庫裡的檔案；認得的副檔名見 [docs/vault-format.md](docs/vault-format.md)。

## 指令

每個指令的用法：`patina help <指令>`。全部指令：`patina help`。

- 圖庫依序找：`--vault <路徑>` → 環境變數 `PATINA_VAULT` → 從目前目錄往上找 `.patina/vault.json`。
- `FILE`、`FOLDER` 寫圖庫相對路徑（例如 `2025/旅行/IMG_0001.jpg`），也可以寫絕對路徑或相對目前目錄的路徑。資料夾寫 `/` 或 `""` 是根目錄。
- `--vault`、`--by`、`--json` 寫在指令前後都可以；`--json` 輸出機器可讀格式。
- 結束碼：0 成功；1 執行失敗，或 `check`／`verify-eagle` 找到問題；2 參數寫錯。

```sh
export PATINA_VAULT=~/Patina
```

### 查看

```sh
patina info                                      # 檔案數（依類型）、資料夾數、有側檔數、未標籤數、標籤數
patina ls "2025/旅行"                             # 這個資料夾裡的檔案（不含子資料夾）
patina ls "2025" --recursive --kind video        # 含子資料夾，只列影片；--kind 是 image、video、font、other，可給多個
patina ls / --recursive --untagged --json        # 整個圖庫沒標籤的檔案，含整理資料
patina show "2025/旅行/IMG_0001.jpg"              # 尺寸、片長、codec、字體家族與可變軸，加上側檔內容
patina search 拱門 庭院                           # 每個詞都要出現在檔名、資料夾、描述、標籤或網址（不分大小寫）
patina search --tag 城堡 --tag 拱門 --kind image --min-rating 3   # 每個 --tag 都要有
patina search --untagged
patina tags                                      # 全部標籤與使用次數，次數多到少
```

### 標籤、描述、網址、評分

```sh
patina set "2025/旅行/IMG_0001.jpg" --add-tag 拱門 --add-tag 城堡 --by openclaw
patina set "2025/旅行/IMG_0001.jpg" --remove-tag 城堡 --by openclaw
patina set "2025/旅行/IMG_0001.jpg" --tags "拱門,街景" --by openclaw     # 整組換掉；逗號分隔，全形逗號也可以
patina set "2025/旅行/IMG_0001.jpg" --desc "從拱門內看向庭院。" --by openclaw
patina set "2025/旅行/IMG_0001.jpg" --url "https://example.com" --rating 4 --by openclaw   # 評分 0–5
patina tag-rename 街景 街道 --by openclaw         # 全圖庫標籤改名；改成既有標籤就是合併
```

- `set` 沒給的欄位不動，至少要給一個欄位。標籤自動去頭尾空白、去重複。
- `--desc ""`、`--url ""`、`--tags ""` 清空該欄。描述、標籤、網址、評分全部空了，側檔會被刪掉。
- 側檔 JSON 壞掉時 `set` 會拒絕寫入，先跑 `patina check`。
- `tag-rename` 會略過找不到檔案的側檔，先跑 `patina reconcile`。

### 改名、建資料夾、搬移

```sh
patina rename "2025/旅行/IMG_0001.jpg" "城堡拱門" --by openclaw     # 印出新路徑 2025/旅行/城堡拱門.jpg
patina mkdir "2025/城堡" --by openclaw                               # 可以一次建多層
patina mv "2025/旅行/城堡拱門.jpg" "2025/城堡" --by openclaw         # 印出新路徑 2025/城堡/城堡拱門.jpg
patina mv "2025/旅行/IMG_0002.jpg" / --by openclaw                   # 搬到根目錄
```

- `rename` 只給主檔名，副檔名自動保留；寫成 `城堡拱門.jpg` 會變成 `城堡拱門.jpg.jpg`。
- 目的地已有同名檔案時自動改成 `名字 (2).jpg`，以 CLI 印出的路徑為準。
- 名稱裡的 `/`、`:` 換成 `-`，開頭的 `.` 會被拿掉。
- `mv` 只搬檔案，不搬資料夾；目的資料夾要先存在。

### 接回側檔、檢查格式

```sh
patina reconcile --dry-run                        # 試跑：列出會接回的側檔，不改東西
patina reconcile --by openclaw                    # 在 Finder／shell 搬檔或改名之後跑
patina check                                      # 壞掉的 JSON、找不到檔案的側檔、衝突副本、format 不是 1
```

- `reconcile` 只處理「側檔找不到檔案」且「新位置還沒有側檔」的情況，用指紋比對；檔案內容改過就接不回。
- 接不回的列在「找不到檔案的側檔」，留在原地不刪。

### 從 Eagle 匯入

一般用 app 的「從 Eagle 匯入」。要用 CLI 時：

```sh
patina import-eagle "<Eagle>.library" --folder "參考" --dry-run --log ~/eagle-dry.jsonl --by openclaw   # 試跑，只算每個項目會放哪
patina import-eagle "<Eagle>.library" --folder "參考" --log ~/eagle-import.jsonl --by openclaw          # 正式搬
patina verify-eagle "<Eagle>.library" --folder "參考"                                                   # 逐一比對，有不符 exit 1
```

- 只讀 Eagle library。`--folder` 是 Eagle 資料夾名稱（完全相符，含子資料夾），可以給多個；不給就是整個 library。
- 標籤、註解（變成描述）、網址、評分寫進側檔；重跑會跳過已搬的項目。有項目出錯時 exit 1。
- Eagle 垃圾桶裡的項目不搬。名稱以 `.syncthing.` 或 `~syncthing~` 開頭的項目也不搬，這是檔案同步途中留下的暫存紀錄。
- `--log` 每個項目寫一行 JSONL，不能放在 Eagle library 裡。

## 回報 bug

到本 repo [開 issue](https://github.com/cuberlab/patina-agent/issues/new/choose)，選「Bug 回報」範本，照欄位填。

**這個 repo 是公開的，issue 誰都看得到。**

- 不要貼私人照片，也不要貼截圖裡的私人內容。
- 不要貼描述內容。
- 家目錄路徑改成 `~`（寫 `~/Patina`，不要寫出帳號名稱）。
- 檔名是私人的就改成代稱（例如 `照片A.jpg`）。

要附的資料：

| 欄位 | 怎麼拿 |
|---|---|
| Patina app 版本 | app 側欄左下角，例如 `Patina 0.2.1 (20261104083511)` |
| CLI 版本 | `patina --version` |
| macOS 版本 | `sw_vers -productVersion` |
| 錯誤紀錄 | 見下方指令；只貼和這個問題有關的幾行 |

```sh
tail -n 20 ~/Patina/.patina/log/errors-*.jsonl                       # app 遇到的錯誤，每行一筆
grep -h '"status":"error"' ~/Patina/.patina/log/eagle-import-*.jsonl  # app「從 Eagle 匯入」出錯的項目
```

沒有這個檔（zsh 印 `no matches found`）就是沒有這類紀錄。紀錄裡的 `path`、`src`、`dest` 含檔名與家目錄路徑，貼上前照上面的規則改。用 `gh issue create --repo cuberlab/patina-agent --label bug` 開 issue 時，內文照 bug 範本的欄位順序寫。
