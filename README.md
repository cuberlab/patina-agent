# patina-agent

Patina 的 agent kit：圖庫格式、agent 直接改圖庫資料夾的做法與守則，以及回報 bug 的方式。不用安裝任何程式，範例是只用標準函式庫的 Python 3。

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

## Agent 守則

1. **不改照片、影片、字體檔本身**：不寫 EXIF／IPTC、不重新存檔、不轉檔。備份用 MD5 驗證；Patina 靠檔案內容認檔，內容一改，搬走後側檔就接不回。
2. **只改 `.patina/meta/` 裡的側檔，整份原子寫入**：先在同一個資料夾寫 `.tmp` 暫存檔，再改名蓋過去。做法見「側檔工具」。
3. **不寫 `fingerprint`**：新建側檔不寫，改既有側檔時原樣保留；寫錯、缺少或檔案換過，Patina 0.2.2 起都會自己重算蓋掉。`format`、`source` 也原樣保留。
4. **改名、搬檔時，檔案和側檔一起搬**；資料夾改名或搬移時，`.patina/meta/` 底下對應的資料夾一起搬。
5. **`.patina/log/` 只讀**：不改、不刪、不新增紀錄檔。
6. **不動 `.patina/vault.json` 與 `.patina/removed/`**：`vault.json` 的 id 是圖庫身分；`removed/` 是丟進垃圾桶的檔案的側檔，檔案放回來時靠它接回。
7. **刪檔案或資料夾請使用者在 Patina app 裡做**：會進 macOS 垃圾桶，可以復原，側檔移到 `.patina/removed/`。
8. **色票、尺寸、片長這類從檔案算得出來的資料不寫進側檔**，app 自己算。
9. **看到衝突副本（名稱含 `.sync-conflict-`）就告訴使用者**，不要自己挑一份刪掉。
10. **Syncthing 不能忽略 `.patina/`**：設定或建議忽略規則時照「Syncthing 設定」那段，不要加 `.*`、`.patina` 這類規則。
11. **改完跑 `check()`**，印「沒有問題」才算完成。

## 側檔在哪

一個檔案最多一份側檔，路徑是 `<圖庫>/.patina/meta/<檔案的相對路徑>.json`：檔名含原本的副檔名，後面再加 `.json`。

| 檔案 | 側檔 |
|---|---|
| `~/Patina/2025/旅行/IMG_0001.jpg` | `~/Patina/.patina/meta/2025/旅行/IMG_0001.jpg.json` |
| `~/Patina/字體/SomeFont.otf` | `~/Patina/.patina/meta/字體/SomeFont.otf.json` |

沒有任何整理資料的檔案沒有側檔。側檔是 UTF-8 的 JSON：

```json
{
  "description": "從拱門內看向庭院。",
  "fingerprint": {"sha256_edges": "7496dc3a…5f34f80", "size": 1016085},
  "format": 1,
  "rating": 4,
  "source": {"eagle": "MPPS9SYI7KGB6"},
  "tags": ["拱門", "城堡", "庭院"],
  "updated_at": "2026-11-04T08:00:00Z",
  "updated_by": "app",
  "url": "https://example.com"
}
```

| 欄位 | 內容 | agent 怎麼處理 |
|---|---|---|
| `format` | 整數 `1` | 新建時寫 `1`，既有的不動 |
| `description` | 字串 | 可以改 |
| `tags` | 字串陣列 | 可以改 |
| `url` | 字串 | 可以改 |
| `rating` | 0 到 5 的整數，0 是沒評分 | 可以改 |
| `source` | 從哪裡匯入，例如 `{"eagle": "<Eagle 項目 id>"}` | 原樣保留 |
| `fingerprint` | Patina 用來認檔的指紋 | 新建不寫，既有的原樣保留 |
| `updated_at` | 最後修改時間 | 改的時候寫 UTC 的 `YYYY-MM-DDTHH:MM:SSZ`，例如 `2026-11-04T08:00:00Z` |
| `updated_by` | 誰改的 | 改的時候寫 `"<agent 名>"` |

- 沒寫的欄位當成空的。`format` 一定要寫。
- 不要在側檔放自己的欄位：Patina 改寫側檔時，上表以外的欄位會被丟掉。
- 型別不對（例如 `rating` 寫成 `"4"`、`tags` 寫成字串），整份側檔 Patina 都讀不了，畫面上會像沒有整理資料，而且不會報錯。

### 哪些檔案算圖庫裡的檔案

只有 Patina 會列出的檔案能有側檔。

- 認得的副檔名（不分大小寫）：
  - 照片：jpg jpeg png heic heif webp gif tif tiff bmp avif，RAW（cr3 cr2 arw nef dng raf orf rw2）
  - 影片：mov mp4 m4v avi mkv webm mts m2ts 3gp
  - 字體：ttf otf ttc otc woff woff2
  - 其他：pdf svg psd ai eps
- 略過：名稱以 `.` 或 `~$` 開頭、名稱含 `.sync-conflict-` 的檔案和資料夾（資料夾裡的東西一起略過）、捷徑（symlink）、Finder 的自訂圖示檔 `Icon\r`、其他副檔名。

## 側檔工具

把下面整段存成圖庫外面的 `patina_meta.py`，改最上面的 `VAULT`（圖庫根目錄）和 `AGENT`（自己的名字，會寫進 `updated_by`）。需要 Python 3.9 以上，只用標準函式庫。

```python
"""Patina 側檔工具：加標籤、寫描述、網址、評分，改名、搬檔、資料夾改名，檢查格式。"""
import json
import os
import re
import time
import unicodedata
from pathlib import Path

VAULT = Path("~/Patina").expanduser()  # 圖庫根目錄
AGENT = "agent"  # 自己的名字，寫進 updated_by

META = VAULT / ".patina" / "meta"
EXTS = {
    "jpg", "jpeg", "png", "heic", "heif", "webp", "gif", "tif", "tiff", "bmp", "avif",
    "cr3", "cr2", "arw", "nef", "dng", "raf", "orf", "rw2",
    "mov", "mp4", "m4v", "avi", "mkv", "webm", "mts", "m2ts", "3gp",
    "ttf", "otf", "ttc", "otc", "woff", "woff2",
    "pdf", "svg", "psd", "ai", "eps",
}
if not (VAULT / ".patina" / "vault.json").is_file():
    raise SystemExit(f"{VAULT} 不是 Patina 圖庫（找不到 .patina/vault.json）")


def nfc(s):
    return unicodedata.normalize("NFC", s)


def sidecar(rel):
    """檔案相對路徑 → 側檔：2025/旅行/A.jpg → .patina/meta/2025/旅行/A.jpg.json"""
    return META / (rel + ".json")


def ignored(name):
    return name.startswith((".", "~$")) or ".sync-conflict-" in name or name == "Icon\r"


def is_media(rel):
    return Path(rel).suffix[1:].lower() in EXTS


def check_name(rel):
    """相對路徑每一段都要是 Patina 會列出的名稱。"""
    parts = Path(rel).parts
    if not parts or Path(rel).is_absolute():
        raise ValueError(f"要寫圖庫相對路徑：{rel!r}")
    for part in parts:
        if ignored(part) or ":" in part:
            raise ValueError(f"Patina 不會列出這個名稱：{rel}")


def exact(rel):
    """路徑大小寫要和磁碟上一樣：磁碟不分大小寫，Patina 分。"""
    d = VAULT
    for part in Path(rel).parts:
        if not d.is_dir() or nfc(part) not in {nfc(n) for n in os.listdir(d)}:
            return False
        d = d / part
    return True


def normalize_tags(tags):
    """去頭尾空白、去掉空的、去重複（保留第一次出現的順序，分大小寫）。"""
    out, seen = [], set()
    for t in tags:
        t = t.strip()
        if t and nfc(t) not in seen:
            seen.add(nfc(t))
            out.append(t)
    return out


def problems(meta):
    """側檔格式對不對：回傳問題清單，空的就是沒問題。"""
    if not isinstance(meta, dict):
        return ["不是 JSON 物件"]
    out = []
    if type(meta.get("format")) is not int or meta["format"] != 1:
        out.append("format 要是 1")
    for k in ("description", "url", "updated_by"):
        if meta.get(k) is not None and not isinstance(meta[k], str):
            out.append(f"{k} 要是字串")
    tags = meta.get("tags")
    if tags is not None and not (isinstance(tags, list) and all(isinstance(t, str) for t in tags)):
        out.append("tags 要是字串陣列")
    r = meta.get("rating")
    if r is not None and not (type(r) is int and 0 <= r <= 5):
        out.append("rating 要是 0 到 5 的整數")
    src = meta.get("source")
    if src is not None and not (isinstance(src, dict) and all(isinstance(v, str) for v in src.values())):
        out.append("source 要是字串對字串的物件")
    fp = meta.get("fingerprint")
    if fp is not None and not (isinstance(fp, dict) and type(fp.get("size")) is int
                               and isinstance(fp.get("sha256_edges"), str)):
        out.append("fingerprint 格式不對")
    ts = meta.get("updated_at")
    if ts is not None and not (isinstance(ts, str) and re.fullmatch(
            r"\d{4}-\d\d-\d\dT\d\d:\d\d:\d\d(\.\d+)?(Z|[+-]\d\d:\d\d)", ts)):
        out.append("updated_at 要像 2026-11-04T08:00:00Z")
    return out


def write_json(path, data):
    """原子寫入：同一個資料夾先寫 .tmp 暫存檔（Patina 只讀 .json），再改名蓋過去。"""
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_name(f"{path.name}.{os.getpid()}.tmp")
    try:
        with open(tmp, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2, sort_keys=True)
            f.write("\n")
        os.replace(tmp, path)
    finally:
        if tmp.exists():
            tmp.unlink()


def edit(rel, add_tags=(), remove_tags=(), tags=None, description=None, url=None, rating=None):
    """改一個檔案的整理資料。沒給的欄位不動；tags=[...] 整組換掉；description="" 清空。"""
    if any(isinstance(x, str) for x in (add_tags, remove_tags, tags)):
        raise TypeError('標籤要用 list，例如 ["拱門"]')
    check_name(rel)
    if not (is_media(rel) and exact(rel) and (VAULT / rel).is_file()):
        raise FileNotFoundError(f"圖庫裡沒有這個檔案（大小寫也要一樣）：{rel}")
    path = sidecar(rel)
    if path.exists():
        with open(path, encoding="utf-8") as f:
            meta = json.load(f)  # JSON 壞掉會丟錯：停下來告訴使用者，不要蓋掉
        bad = problems(meta)
        if bad:
            raise ValueError(f"側檔格式不對，先告訴使用者：{path}：{'、'.join(bad)}")
    else:
        meta = {"format": 1}  # 新側檔；fingerprint 由 Patina app 補上
    current = list(meta.get("tags") or []) if tags is None else list(tags)
    drop = {nfc(t.strip()) for t in remove_tags}
    meta["tags"] = normalize_tags([t for t in current if nfc(t.strip()) not in drop] + list(add_tags))
    if description is not None:
        meta["description"] = description.strip()
    if url is not None:
        meta["url"] = url.strip()
    if rating is not None:
        if type(rating) is not int or not 0 <= rating <= 5:
            raise ValueError("rating 要是 0 到 5 的整數")
        meta["rating"] = rating
    if not any(meta.get(k) for k in ("description", "tags", "url", "rating")) and meta.get("source") is None:
        if path.exists():
            path.unlink()  # 沒有整理資料就不留側檔，和 Patina app 一樣
            prune(path.parent)
        return None
    meta["updated_at"] = time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
    meta["updated_by"] = AGENT
    write_json(path, meta)
    return meta


def prune(d):
    """刪掉 .patina/meta 底下搬空的資料夾（只刪空的）。"""
    while d != META and d.is_dir() and not any(d.iterdir()):
        d.rmdir()
        d = d.parent


def move(rel, new_rel):
    """改名或搬檔，側檔一起搬。new_rel 是新的相對路徑，副檔名不變；目的資料夾不存在就建。"""
    check_name(rel)
    check_name(new_rel)
    src, dst = VAULT / rel, VAULT / new_rel
    if not (exact(rel) and src.is_file()):
        raise FileNotFoundError(f"圖庫裡沒有這個檔案（大小寫也要一樣）：{rel}")
    if Path(rel).suffix.lower() != Path(new_rel).suffix.lower():
        raise ValueError("副檔名要和原本一樣")
    if dst.exists() and not os.path.samefile(src, dst):
        raise FileExistsError(f"已經有 {new_rel}，換個名字")
    dst.parent.mkdir(parents=True, exist_ok=True)
    os.rename(src, dst)
    old, new = sidecar(rel), sidecar(new_rel)
    if old.exists():
        new.parent.mkdir(parents=True, exist_ok=True)
        os.replace(old, new)
        prune(old.parent)
    return new_rel


def move_folder(rel, new_rel):
    """資料夾改名或搬移，.patina/meta 底下對應的資料夾一起搬。"""
    check_name(rel)
    check_name(new_rel)
    src, dst = VAULT / rel, VAULT / new_rel
    if not (exact(rel) and src.is_dir()):
        raise FileNotFoundError(f"圖庫裡沒有這個資料夾（大小寫也要一樣）：{rel}")
    if dst.exists() and not os.path.samefile(src, dst):
        raise FileExistsError(f"已經有 {new_rel}，換個名字")
    dst.parent.mkdir(parents=True, exist_ok=True)
    os.rename(src, dst)
    msrc, mdst = META / rel, META / new_rel
    if msrc.is_dir():
        if mdst.exists() and not os.path.samefile(msrc, mdst):
            # 新位置留有舊的側檔資料夾：逐份搬過去，同名的蓋掉，和 Patina app 一樣
            for f in sorted(msrc.rglob("*.json")):
                t = mdst / f.relative_to(msrc)
                t.parent.mkdir(parents=True, exist_ok=True)
                os.replace(f, t)
            for d in sorted((d for d in msrc.rglob("*") if d.is_dir()), reverse=True):
                prune(d)
            prune(msrc)
        else:
            mdst.parent.mkdir(parents=True, exist_ok=True)
            os.rename(msrc, mdst)
            prune(msrc.parent)
    return new_rel


def check():
    """每份側檔讀一次並檢查格式，列出找不到檔案的側檔與衝突副本。印「沒有問題」才算完成。"""
    files, conflicts, bad, orphans = set(), [], [], []
    for d, dirs, names in os.walk(VAULT):
        here = Path(d).relative_to(VAULT)
        conflicts += [(here / n).as_posix() for n in dirs + names if ".sync-conflict-" in n]
        dirs[:] = [n for n in dirs if not ignored(n)]
        files.update(nfc((here / n).as_posix()) for n in names
                     if not ignored(n) and is_media(n) and not os.path.islink(os.path.join(d, n)))
    conflicts += [p.relative_to(VAULT).as_posix() for p in (VAULT / ".patina").rglob("*.sync-conflict-*")]
    for f in sorted(META.rglob("*.json")):
        if ".sync-conflict-" in f.name:
            continue
        shown = f.relative_to(VAULT).as_posix()
        try:
            with open(f, encoding="utf-8") as fh:
                found = problems(json.load(fh))
        except (OSError, ValueError) as e:
            found = [f"讀不了（{e}）"]
        if found:
            bad.append(f"{shown}：{'、'.join(found)}")
        elif nfc(f.relative_to(META).as_posix()[:-5]) not in files:
            orphans.append(shown)
    for title, rows in (("格式不對的側檔", bad), ("找不到檔案的側檔", orphans), ("衝突副本", conflicts)):
        if rows:
            print(f"{title}：{len(rows)}")
            for row in sorted(rows):
                print(f"  {row}")
    ok = not (bad or orphans or conflicts)
    if ok:
        print("沒有問題")
    return ok
```

在放 `patina_meta.py` 的資料夾執行，路徑都寫圖庫相對路徑：

```sh
python3 - <<'EOF'
from patina_meta import edit, move, move_folder, check
edit("2025/旅行/IMG_0001.jpg", add_tags=["拱門", "城堡"])
edit("2025/旅行/IMG_0001.jpg", remove_tags=["城堡"])
edit("2025/旅行/IMG_0001.jpg", description="從拱門內看向庭院。", url="https://example.com", rating=4)
edit("2025/旅行/IMG_0002.jpg", tags=["街景", "夜景"])        # 整組換掉
edit("2025/旅行/IMG_0002.jpg", description="")               # 清空描述
move("2025/旅行/IMG_0001.jpg", "2025/城堡/城堡拱門.jpg")      # 改名＋搬檔，側檔跟著走
move_folder("2025/旅行", "2025/歐洲旅行")                     # 資料夾改名，側檔資料夾跟著走
check()
EOF
```

## 加標籤、寫描述、網址、評分

`edit()` 照下面的規則做，和 Patina app 存檔時一樣：

- **已有側檔**：讀出 JSON，只改要改的欄位，整份寫回；其他欄位原樣保留。側檔讀不了（JSON 壞掉、型別或範圍不對）就停下來告訴使用者，不要蓋掉。
- **沒有側檔**：新建 `{"format": 1, ...}`，只寫要改的欄位，不寫 `fingerprint`。
- **標籤**：去頭尾空白、去掉空的、去重複，保留第一次出現的順序；新加的放在最後。分大小寫：`Cat` 和 `cat` 是兩個標籤。
- **描述、網址**：去頭尾空白。
- **評分**：0 到 5 的整數，0 是沒評分。
- **全部空了**：描述、標籤、網址、評分都是空的，而且沒有 `source`，就刪掉側檔（app 也這樣做）；有 `source` 就留著，寫回空的欄位。
- **原子寫入**：暫存檔放在側檔同一個資料夾，名稱以 `.tmp` 結尾（Patina 只讀 `.json` 結尾的側檔），寫完用 `os.replace` 改名蓋過去，等於 `mv`。
- **`updated_by`、`updated_at`**：`updated_by` 寫自己的名字；`updated_at` 寫 UTC 的 `YYYY-MM-DDTHH:MM:SSZ`，例如 `2026-11-04T08:00:00Z`。

## 改名、搬檔、資料夾改名

- **檔案和側檔一起搬**：`2025/旅行/A.jpg` 搬成 `2025/城堡/B.jpg`，側檔從 `.patina/meta/2025/旅行/A.jpg.json` 搬到 `.patina/meta/2025/城堡/B.jpg.json`。目的資料夾不存在就建。`move()` 一起做。
- **資料夾改名或搬移**：整個資料夾和 `.patina/meta/` 底下對應的資料夾一起搬。`2025/旅行` 改成 `2025/歐洲旅行` 時，`.patina/meta/2025/旅行` 也改成 `.patina/meta/2025/歐洲旅行`。`move_folder()` 一起做。
- **名稱**：副檔名不改；不要以 `.` 或 `~$` 開頭（Patina 不會列出），不要含 `.sync-conflict-`（會被當成衝突副本），不要含 `:`（Finder 會顯示成 `/`）。
- **撞名**：目的地已經有同名檔案或資料夾就換個名字，不要蓋掉。
- **大小寫**：路徑大小寫要和磁碟上一樣。磁碟不分大小寫，Patina 分；檔案是 `IMG_0001.jpg`，側檔寫成 `IMG_0001.JPG.json`，Patina 就對不上。
- **建空資料夾**：`mkdir -p` 就好。
- **漏搬側檔**：Patina 開著時（或下次打開圖庫時）會用指紋把側檔接回新位置。還沒被 app 補上指紋的側檔、內容改過的檔案都接不回，所以一定要一起搬。

## 改完確認

1. 跑 `check()`，印「沒有問題」才算完成。它會：
   - 每份側檔用 `json.load` 讀一次，檢查欄位型別與範圍；
   - 列出找不到檔案的側檔：通常是搬檔時漏搬側檔，把側檔搬到檔案的新位置；不知道是哪個檔案就告訴使用者，不要刪；
   - 列出衝突副本。
2. Patina 開著的話，畫面會自己更新：app 會監看整個圖庫資料夾，有變動就重讀。
3. Patina 讀不了的側檔不會跳錯誤，也不會寫進 `.patina/log/`，只會當成沒有整理資料，所以一定要跑 `check()`。

## Syncthing 設定

圖庫用 Syncthing 同步時，整個資料夾都要同步，包括 `.patina/`：標籤、描述、評分、資料夾整理都在裡面，擋掉的話每台電腦會各改各的。Syncthing 自己的 `.stfolder`、`.stversions` 和下載中的 `.syncthing.*.tmp` 它本來就不會同步，不用寫。

Syncthing 這個資料夾的「忽略模式」（資料夾根目錄的 `.stignore`）只放系統產生的檔案：

```
// Patina 圖庫：.patina 不能忽略
(?d).DS_Store
(?d)._*
(?d).Spotlight-V100
(?d).Trashes
(?d).fseventsd
(?d).TemporaryItems
(?d)Thumbs.db
(?d)desktop.ini
```

- `(?d)` 讓 Syncthing 刪資料夾時可以順便刪掉這些檔，不會卡住。
- 不要寫 `.*`、`*`、`.patina`、`/.patina` 這類會擋到 `.patina/` 的規則。圖庫放在更大的 Syncthing 資料夾裡也一樣（例如同步整個 `~/Sync`，圖庫是裡面的 `Patina`）。
- `.stignore` 不會在裝置之間同步，每台都要設。
- Patina 0.2.4 起打開圖庫時會檢查；擋到 `.patina/` 會跳錯誤提示，並寫進 `.patina/log/errors-<主機>.jsonl`。

## 同步衝突

圖庫用 Syncthing 在多台電腦同步時，兩邊同時改同一份檔案，會多一份名稱含 `.sync-conflict-` 的副本（例如 `IMG_0001.jpg.sync-conflict-20261104-083000-ABCDEFG.json`）。Patina 和 `check()` 都不讀它，`check()` 會列出來。看到就告訴使用者是哪幾份，不要自己挑一份刪掉或合併。

## 從 Eagle 匯入

請使用者在 Patina app 裡用「從 Eagle 匯入」，agent 不自己從 Eagle library 搬檔案。匯入的檔案側檔有 `source`（Eagle 項目 id），改這些側檔時原樣保留。

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
| macOS 版本 | `sw_vers -productVersion` |
| 錯誤紀錄 | 見下方指令；只貼和這個問題有關的幾行 |

```sh
tail -n 20 ~/Patina/.patina/log/errors-*.jsonl                       # app 遇到的錯誤，每行一筆
grep -h '"status":"error"' ~/Patina/.patina/log/eagle-import-*.jsonl  # app「從 Eagle 匯入」出錯的項目
```

沒有這個檔（zsh 印 `no matches found`）就是沒有這類紀錄。紀錄裡的 `path`、`src`、`dest` 含檔名與家目錄路徑，貼上前照上面的規則改。

### 很慢、卡住

另外附：

- **圖庫規模**：檔案數、影片數、總大小。
- **當時在做什麼**：Syncthing 正在同步嗎、加了多少檔案、卡了多久、哪個畫面（開影片、標籤頁、切資料夾…）。
- **卡住當下的 sample**：Patina 開著、卡住時跑，10 秒。

```sh
cd ~/Patina   # 圖庫根目錄
find . -type f -not -path './.patina/*' | wc -l                                          # 檔案數
find . -type f -not -path './.patina/*' \( -iname '*.mov' -o -iname '*.mp4' -o -iname '*.m4v' \) | wc -l   # 影片數
du -sh .                                                                                   # 總大小

sample Patina 10 -file /tmp/patina-sample.txt > /dev/null
sed -n '/Sort by top of stack/,/Binary Images/p' /tmp/patina-sample.txt | head -25
grep -E '\(in Patina' /tmp/patina-sample.txt | sed -E 's/^[ +!:|]*//; s/  \(in .*//' \
  | awk '{n=$1; $1=""; s[$0]+=n} END {for (k in s) print s[k] k}' | sort -nr | head -25
```

sample 的兩段輸出只有函式名稱和次數，沒有檔名，整段照原文貼進「卡住時的 sample」。`/tmp/patina-sample.txt` 本身含帳號名稱和路徑，不要整份貼上。

用 `gh` 開 issue 時，先查有沒有一樣的：`gh issue list --repo cuberlab/patina-agent --search "<關鍵字>"`；沒有再用 `gh issue create --repo cuberlab/patina-agent --label bug` 開，內文照 bug 範本的欄位順序寫。
