# patina-agent

Patina 的 agent kit：圖庫格式、agent 直接改圖庫資料夾的做法與守則，以及回報 bug 的方式。不用安裝任何程式，範例是只用標準函式庫的 Python 3。

## Patina 是什麼

Patina 是一個 Mac 圖庫 app，照片、影片和字體可以放在一起整理。圖庫就是一個普通的資料夾：子資料夾是分類，檔名就是名稱，檔案一直留在原處。描述、標籤、網址和評分存在圖庫根目錄的隱藏資料夾 `.patina/`，每個檔案一份 JSON 側檔。完整格式見 [docs/vault-format.md](docs/vault-format.md)。0.3.0 起有 Boards：一張無限大的畫布，放圖庫裡的圖、文字、線和標籤，一個 board 一份 JSON，存在 `.patina/boards/`；使用者在圖上標的標籤，agent 可以讀、回覆，見「Boards」。iPhone 版用內建的 Syncthing 從 Mac 收圖庫，配對方式見「iPhone 配對」。

```
~/Patina/
├── 2025/旅行/IMG_0001.jpg
├── 字體/SomeFont.otf
└── .patina/
    ├── vault.json                            圖庫 id 與 format
    ├── meta/2025/旅行/IMG_0001.jpg.json      側檔，路徑跟著檔案
    ├── boards/b3f9c2a7d1e4f.json             board，一個 board 一個檔
    ├── removed/                              丟進垃圾桶的檔案的側檔、刪掉的 board
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
9. **不要自己處理衝突副本（名稱含 `.sync-conflict-`）**：側檔的副本 Patina 0.2.4 起會自動合併；照片、影片本身的副本告訴使用者，不要自己挑一份刪掉。
10. **Syncthing 不能忽略 `.patina/`**：設定或建議忽略規則時照「Syncthing 設定」那段，不要加 `.*`、`.patina` 這類規則。
11. **改完跑 `check()`**，印「沒有問題」才算完成。改了 board 也跑 `check_boards()`。

### Board 守則

改 `.patina/boards/` 的 board 時另外照這幾條，「Board 工具」都做好了：

1. **整份原子寫入**：讀出整份 JSON，改完整份寫到同一個資料夾的 `.tmp` 暫存檔，再改名蓋過去，不要只改一部分。
2. **寫回前確認檔案沒被別人改過**：讀的時候記下檔案內容，寫之前再讀一次比對；不一樣（Patina、Syncthing 或別的 agent 剛寫過）就重讀、重做這次修改。`update()` 還會拿 Patina 寫 board 用的同一把鎖（對 `.patina/` 資料夾 `flock`）。
3. **不認得的欄位和元素原樣保留**，寫回時不丟。
4. **只改自己要改的元素**：新增或改了一個元素，就更新那個元素的 `updated_at`／`updated_by`，也更新 board 的 `updated_at`／`updated_by`。回覆例外：只加在標籤的 `replies` 後面，標籤本身的 `updated_at` 不動，board 的照樣更新。
5. **標籤編號不重用**：新標籤的 `n` 是目前的標籤和 `removed` 墓碑裡最大的 `n` 加 1。
6. **拿掉元素要留墓碑**：從 `elements` 刪掉，`removed` 加 `{"id", "at"}`（標籤多 `n`）；綁在上面的線和標籤一起拿掉，各留一個墓碑。
7. **`format` 不是 1 的 board 不要寫**，告訴使用者。
8. **不動 `.patina/removed/boards/`**（刪掉的 board，可以在 app 裡還原）。刪 board 請使用者在 Patina app 裡做。

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
- **board 上的圖**：`move()`、`move_folder()` 不改 board 裡的 `path`，Patina 開著時（或下次打開圖庫時）用指紋接回，不用自己改。

## 改完確認

1. 跑 `check()`，印「沒有問題」才算完成。它會：
   - 每份側檔用 `json.load` 讀一次，檢查欄位型別與範圍；
   - 列出找不到檔案的側檔：通常是搬檔時漏搬側檔，把側檔搬到檔案的新位置；不知道是哪個檔案就告訴使用者，不要刪；
   - 列出衝突副本。
2. Patina 開著的話，畫面會自己更新：app 會監看整個圖庫資料夾，有變動就重讀。
3. Patina 讀不了的側檔不會跳錯誤，也不會寫進 `.patina/log/`，只會當成沒有整理資料，所以一定要跑 `check()`。

## Boards

Patina 0.3.0 起，每個圖庫可以有好幾個 board：一張無限大的畫布，放圖庫裡的圖、文字、線、標籤。使用者在圖上加標籤（例如在拱門上標「拱的比例」），agent 讀標籤、做事、回覆「已改」「沒改」或「只回話」；Patina 開著時，回覆幾秒內出現在 board 右欄的標註清單。完整格式見 [docs/vault-format.md「Boards」](docs/vault-format.md#boards)。

### 格式重點

一個 board 一個檔 `.patina/boards/<id>.json`，id 是 `b` 加 12 個小寫十六進位字元，檔名就是 id。最外層一個欄位一行，`elements`、`removed` 一個項目一行，鍵照字母排：

```json
{
  "created_at": "2026-11-24T06:52:00Z",
  "elements": [
    {"created_at":"2026-11-24T06:52:00Z","fingerprint":{"sha256_edges":"c6f4b8ab…6424d9cde","size":934125},"h":320,"id":"e4aabaa9f3087","path":"2025/旅行/IMG_0001.jpg","type":"image","updated_at":"2026-11-24T06:52:00Z","updated_by":"user","w":479.77,"x":0,"y":0,"z":1},
    {"by":"user","color":"orange","created_at":"2026-11-24T06:53:00Z","id":"e107b97b9d358","n":1,"on":"e4aabaa9f3087","replies":[{"at":"2026-11-24T07:10:00Z","by":"agent","id":"r5c1e0a9b7d24","status":"done","text":"改成 2:3 了"}],"text":"拱的比例","type":"label","updated_at":"2026-11-24T06:53:00Z","updated_by":"user","x":0.488281,"y":0.270864,"z":2}
  ],
  "format": 1,
  "id": "b3f9c2a7d1e4f",
  "name": "旅行參考",
  "removed": [{"at":"2026-11-24T07:00:00Z","id":"e6d0d02d5569f","n":2}],
  "updated_at": "2026-11-24T07:10:00Z",
  "updated_by": "agent"
}
```

| `type` | 是什麼 | 欄位 |
|---|---|---|
| `image` | 圖庫裡的圖，只記參照 | `path`（圖庫相對路徑）、`fingerprint`（同側檔的指紋）、`x`、`y`（左上角）、`w`、`h` |
| `text` | 文字塊 | `text`、`x`、`y`（左上角）、`size`（字級，預設 15）、`color`（預設 `gray`） |
| `stroke` | 線 | `points`（`[[x, y], …]`，兩點是直線、多點是折線）、`arrow`（最後一點畫箭頭）、`color`（預設 `orange`）、`on`（綁的圖，可省略） |
| `label` | 標籤 | `n`（編號）、`text`、`x`、`y`（泡泡左上角）、`on`（綁的圖或線）、`color`（預設 `orange`）、`by`（誰加的）、`replies` |

- 每個元素都有 `id`（`e` 加 12 個小寫十六進位字元）、`z`（大的在上面）、`created_at`、`updated_at`、`updated_by`。
- `replies` 一則一個物件：`id`（`r` 開頭）、`by`、`at`、`status`（`done` 已改、`skip` 沒改、`note` 只回話）、`text`。沒有回覆的標籤就是還沒處理的。
- `removed` 是拿掉的元素的墓碑：`id`、`at`，標籤多 `n`。Patina 合併同步衝突時靠它，標籤編號也靠它不重用。
- 圖檔不見時元素留著（畫成灰框加檔名）。在 Patina 裡改名、搬檔時 board 的 `path` 跟著改；用 `move()`、`move_folder()` 或在 Finder 搬的，Patina 開著時（或下次打開圖庫時）用 `fingerprint` 接回。
- 時間是 UTC、到秒，寫法同側檔。

### 座標

- **world**：沒綁東西的元素用畫布的 world 單位，x 向右、y 向下。圖的 `x`、`y`、`w`、`h` 都是 world。
- **綁在圖上**（`on` 是圖的 id）：`x`、`y`、`points` 是那張圖的比例，0–1（x 除以圖寬、y 除以圖高），圖搬移、縮放時跟著走。綁在線上的標籤跟那條線同一個座標系（線綁在圖上，就是那張圖的比例）。
- **比例和原圖像素互換**：像素 = 比例 × 原圖的寬（高），比例 = 像素 ÷ 原圖的寬（高）。原圖寬高是檔案本身的像素，照 EXIF 方向轉正後（直拍的照片寬高對調），不是 board 上的 `w`、`h`；`image_size()` 會算。例如原圖 2048 × 1366，標籤 `x` 0.488281、`y` 0.270864，在原圖像素 (1000, 370)。
- **比例換成 world**：圖的 `x` + 比例 × 圖的 `w`，`y` 同理。
- 寫的時候 world 到小數 2 位、比例到 6 位，整數寫成整數。

### 顏色

只有這四個，寫顏色名稱：

| 值 | 顏色 | 預設用在 |
|---|---|---|
| `blue` | 藍 `#0072B2` | |
| `orange` | 橘 `#E07B00` | 線、標籤 |
| `gray` | 深灰 `#2F3540` | 文字 |
| `white` | 白 `#FFFFFF` | |

### Board 工具

把下面整段存成圖庫外面的 `patina_board.py`，和側檔工具一樣改最上面的 `VAULT`、`AGENT`。需要 Python 3.9 以上（macOS 或 Linux），只用標準函式庫；讀原圖尺寸時 JPEG、PNG、GIF 自己讀，HEIC、TIFF、RAW 這類用 macOS 內建的 `osascript` 叫 ImageIO（Patina 也用它），Linux 上這些格式讀不到尺寸，不能放圖、加標籤、畫線。

| 函式 | 做什麼 |
|---|---|
| `boards()` | 列出 boards：id、名稱、圖數、標籤數、沒回覆的標籤數（`open`）、修改時間 |
| `board(q)` | 讀一個 board，回傳整份 JSON。`q` 是 id 或名稱（名稱完全一樣的優先，不然要剛好對到一個） |
| `notes(q)` | 沒回覆的標籤：編號、文字、誰、在哪張圖、原圖像素位置 `px`；`all=True` 全部 |
| `reply(q, n, status, text)` | 回覆標籤 `#n`，`status` 是 `done`、`skip`、`note` |
| `add_image(q, rel)` | 放圖庫裡的圖；沒給位置就接在最後一張圖右邊 |
| `add_text(q, text)` | 加文字；沒給位置就放在圖和文字下面 |
| `add_label(q, image_id, text, at=(x, y))` | 在圖上加標籤，`at` 是原圖像素，沒給是正中間 |
| `add_line(q, image_id, [(x, y), …])` | 在圖上畫線，點是原圖像素；`arrow=True` 加箭頭 |
| `remove(q, element_id, …)` | 從 board 拿掉元素，留墓碑；檔案留在圖庫 |
| `check_boards()` | 每個 board 讀一次並檢查格式，印「沒有問題」才算完成 |

改 board 的函式都經過 `update()`：拿鎖、讀、改、寫回前比對、整份原子寫入，照「Board 守則」。

```python
"""Patina board 工具：列出 boards、讀標籤、回覆，放圖、加文字、標籤、畫線，拿掉元素，檢查格式。"""
import copy
import fcntl
import hashlib
import json
import os
import re
import secrets
import struct
import subprocess
import time
import unicodedata
from pathlib import Path

VAULT = Path("~/Patina").expanduser()  # 圖庫根目錄
AGENT = "agent"  # 自己的名字，寫進 updated_by、標籤的 by、回覆的 by

PATINA = VAULT / ".patina"
BOARDS = PATINA / "boards"
COLORS = ("blue", "orange", "gray", "white")
if not (PATINA / "vault.json").is_file():
    raise SystemExit(f"{VAULT} 不是 Patina 圖庫（找不到 .patina/vault.json）")


def now():
    return time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())


def new_id(prefix):
    """元素 e、回覆 r，後面接 12 個小寫十六進位字元。"""
    return prefix + secrets.token_hex(6)


def num(v, ratio=False):
    """world 寫到小數 2 位、比例寫到 6 位；整數寫成整數，和 Patina 一樣。"""
    v = round(v, 6 if ratio else 2)
    return int(v) if v == int(v) else v


def dumps(v):
    return json.dumps(v, ensure_ascii=False, sort_keys=True, separators=(",", ":"))


def encode(b):
    """Patina 的寫法：最外層一個欄位一行，elements 和 removed 一個項目一行，鍵照字母排。"""
    rows = []
    for k in sorted(b):
        v = b[k]
        if k in ("elements", "removed") and isinstance(v, list) and v:
            text = "[\n" + ",\n".join("    " + dumps(x) for x in v) + "\n  ]"
        else:
            text = dumps(v)
        rows.append(f"  {dumps(k)}: {text}")
    return ("{\n" + ",\n".join(rows) + "\n}\n").encode()


def exact(rel):
    """路徑大小寫要和磁碟上一樣：磁碟不分大小寫，Patina 分。"""
    d = VAULT
    for part in Path(rel).parts:
        nfc = unicodedata.normalize("NFC", part)
        if not d.is_dir() or nfc not in {unicodedata.normalize("NFC", n) for n in os.listdir(d)}:
            return False
        d = d / part
    return True


IMAGEIO_JS = """ObjC.import("ImageIO")
function run(argv) {
  const src = $.CGImageSourceCreateWithURL($.NSURL.fileURLWithPath(argv[0]), $())
  const p = ObjC.castRefToObject($.CGImageSourceCopyPropertiesAtIndex(src, 0, $())).js
  return [p.PixelWidth.js, p.PixelHeight.js, p.Orientation ? p.Orientation.js : 1].join(" ")
}"""


def image_size(rel):
    """原圖的像素寬高（照 EXIF 方向轉正），和 Patina 一樣。讀不到回 None。
    JPEG、PNG、GIF 自己讀；其他格式（HEIC、TIFF、RAW…）用 macOS 內建的 osascript 叫 ImageIO，Patina 也用它。"""
    path = VAULT / rel
    try:
        with open(path, "rb") as f:
            head = f.read(26)
            if head[:8] == b"\x89PNG\r\n\x1a\n":
                return struct.unpack(">II", head[16:24])
            if head[:6] in (b"GIF87a", b"GIF89a"):
                return struct.unpack("<HH", head[6:10])
            if head[:2] == b"\xff\xd8":
                f.seek(2)
                size = jpeg_size(f)
                if size:
                    return size
        out = subprocess.run(["osascript", "-l", "JavaScript", "-e", IMAGEIO_JS, str(path)],
                             capture_output=True, text=True).stdout.split()
    except (OSError, struct.error):
        return None
    if len(out) != 3 or not all(x.isdigit() for x in out):
        return None
    w, h, orientation = map(int, out)
    return (h, w) if orientation >= 5 else (w, h)


def jpeg_size(f):
    orientation, size = 1, None
    while size is None:
        if f.read(1) != b"\xff":
            return None
        marker = f.read(1)
        while marker == b"\xff":
            marker = f.read(1)
        if not marker or marker[0] in (0xD9, 0xDA):
            return None
        if marker[0] == 0x01 or 0xD0 <= marker[0] <= 0xD8:
            continue
        data = f.read(struct.unpack(">H", f.read(2))[0] - 2)
        if marker[0] == 0xE1 and data[:6] == b"Exif\0\0":
            orientation = exif_orientation(data[6:]) or orientation
        elif 0xC0 <= marker[0] <= 0xCF and marker[0] not in (0xC4, 0xC8, 0xCC):
            h, w = struct.unpack(">HH", data[1:5])
            size = (w, h)
    return (size[1], size[0]) if orientation >= 5 else size


def exif_orientation(tiff):
    e = "<" if tiff[:2] == b"II" else ">"
    ifd = struct.unpack(e + "I", tiff[4:8])[0]
    for i in range(struct.unpack(e + "H", tiff[ifd:ifd + 2])[0]):
        entry = tiff[ifd + 2 + 12 * i: ifd + 14 + 12 * i]
        if struct.unpack(e + "H", entry[:2])[0] == 0x0112:
            return struct.unpack(e + "H", entry[8:10])[0]
    return None


def fingerprint(rel):
    """Patina 認檔用的指紋：檔案大小，加上頭尾各 64 KB 的 SHA-256。檔案在 Finder 搬走時 Patina 靠它接回。"""
    path, chunk = VAULT / rel, 64 * 1024
    size = path.stat().st_size
    h = hashlib.sha256()
    with open(path, "rb") as f:
        h.update(f.read(chunk))
        if size > chunk:
            f.seek(max(chunk, size - chunk))
            h.update(f.read(chunk))
    return {"sha256_edges": h.hexdigest(), "size": size}


def boards():
    """列出 boards：id、名稱、幾張圖、標籤數、沒回覆的標籤數、修改時間，新的在前。讀不了的 name 是「（讀不了）」。"""
    out = []
    for p in sorted(BOARDS.glob("*.json")) if BOARDS.is_dir() else []:
        if not re.fullmatch(r"b[0-9a-f]{12}\.json", p.name):
            continue  # 衝突副本這類不是 board 的檔
        try:
            b = json.loads(p.read_bytes())
            els = [e for e in b.get("elements") or [] if isinstance(e, dict)]
        except (OSError, ValueError, AttributeError):
            out.append({"id": p.stem, "name": "（讀不了）", "images": 0, "labels": 0, "open": 0, "updated_at": ""})
            continue
        labels = [e for e in els if e.get("type") == "label"]
        out.append({"id": p.stem, "name": b.get("name", ""), "images": sum(e.get("type") == "image" for e in els),
                    "labels": len(labels), "open": sum(not e.get("replies") for e in labels),
                    "updated_at": b.get("updated_at", "")})
    return sorted(out, key=lambda b: b["updated_at"], reverse=True)


def find(q):
    """board 的 id 或名稱 → id。名稱完全一樣的優先，不然要剛好對到一個（不分大小寫）。"""
    bs = boards()
    for match in (lambda b: b["id"] == q, lambda b: b["name"].casefold() == q.casefold(),
                  lambda b: q.casefold() in b["name"].casefold() or b["id"].startswith(q)):
        hit = [b for b in bs if match(b)]
        if len(hit) == 1:
            return hit[0]["id"]
        if hit:
            raise ValueError(f"「{q}」對到 {len(hit)} 個 board，用 id：" + "、".join(f"{b['id']} {b['name']}" for b in hit))
    raise ValueError(f"找不到 board：{q}")


def board(q):
    """讀一個 board，回傳整份 JSON（dict）。"""
    return json.loads((BOARDS / f"{find(q)}.json").read_bytes())


def write_bytes(path, data):
    """原子寫入：同一個資料夾先寫 .tmp 暫存檔（Patina 只讀 <id>.json），再改名蓋過去。"""
    tmp = path.with_name(f".{path.name}.{os.getpid()}.tmp")
    try:
        tmp.write_bytes(data)
        os.replace(tmp, path)
    finally:
        if tmp.exists():
            tmp.unlink()


def update(q, change):
    """讀、改、寫一個 board。change(b) 直接改 dict，它的回傳值照樣傳回；可能跑不只一次。
    整段拿著 .patina/ 資料夾的鎖（Patina app 和 CLI 寫 board 也拿這把）；寫回前再讀一次，
    和讀進來的不一樣（Syncthing 或別的程式剛寫過）就重讀重做，最多 3 次。內容沒變就不寫。"""
    path = BOARDS / f"{find(q)}.json"
    fd = os.open(PATINA, os.O_RDONLY)
    try:
        fcntl.flock(fd, fcntl.LOCK_EX)
        for _ in range(3):
            original = path.read_bytes()
            b = json.loads(original)  # JSON 壞掉會丟錯：停下來告訴使用者，不要蓋掉
            fmt = b.get("format") if isinstance(b, dict) else None
            if type(fmt) not in (int, float) or fmt != 1:
                raise ValueError(f"board 的 format 是 {fmt!r}，只寫 format 1：{path.name}")
            b.setdefault("elements", [])
            b.setdefault("removed", [])
            if not (isinstance(b["elements"], list) and all(isinstance(e, dict) for e in b["elements"])
                    and isinstance(b["removed"], list)):
                raise ValueError(f"board 的 elements 或 removed 格式不對，先告訴使用者：{path.name}")
            before = copy.deepcopy(b)
            result = change(b)
            if b == before:
                return result
            b["updated_at"], b["updated_by"] = now(), AGENT
            if path.read_bytes() == original:
                write_bytes(path, encode(b))
                return result
        raise RuntimeError(f"board 一直在被別人改，沒寫進去，等一下再試：{path.name}")
    finally:
        os.close(fd)  # 關掉就放開鎖


def element(b, eid):
    for e in b["elements"]:
        if e.get("id") == eid:
            return e
    raise ValueError(f"board 上沒有這個元素：{eid}")


def labels(b):
    return sorted((e for e in b["elements"] if e.get("type") == "label"), key=lambda e: e.get("n") or 0)


def anchor(b, e):
    """元素的座標系：綁在圖上（或綁在綁圖的線上）回傳那張圖，沒綁回 None。綁的東西不在了丟錯。"""
    ids = {x.get("id"): x for x in b["elements"]}
    seen = {e.get("id")}
    while e.get("on"):
        e = ids.get(e["on"])
        if e is None or e.get("id") in seen:
            raise ValueError("綁的元素不在了")
        seen.add(e.get("id"))
        if e.get("type") == "image":
            return e
    return None


def new_element(b, kind, **fields):
    """新元素：id、z（目前最大的加 1）、建立和修改時間。"""
    t = now()
    z = max((e["z"] for e in b["elements"] if type(e.get("z")) in (int, float)), default=0)
    return {"id": new_id("e"), "type": kind, **fields, "z": int(z // 1) + 1,
            "created_at": t, "updated_at": t, "updated_by": AGENT}


def notes(q, all=False):
    """沒回覆的標籤（all=True 全部），編號小的在前。綁在圖上的位置換成那張原圖的像素 px=(x, y)，沒綁圖的是 world。"""
    b = board(q)
    b.setdefault("elements", [])
    out = []
    for l in labels(b):
        if l.get("replies") and not all:
            continue
        note = {k: l.get(k) for k in ("n", "id", "text", "by", "created_at")}
        note.update(replies=l.get("replies") or [], image=None, path=None, px=None, world=None)
        x, y = l.get("x", 0), l.get("y", 0)
        try:
            img = anchor(b, l)
        except ValueError:
            img = "broken"  # 綁的元素不在了：位置不明
        if isinstance(img, dict):
            note["image"], note["path"] = img.get("id"), img.get("path")
            size = image_size(img.get("path") or "")
            if size:
                note["px"] = (num(x * size[0]), num(y * size[1]))
        elif img is None:
            note["world"] = (x, y)
        out.append(note)
    return out


def reply(q, n, status, text):
    """回覆標籤 #n：status 是 done（已改）、skip（沒改）、note（只回話）。標籤本身的 updated_at 不動。"""
    if status not in ("done", "skip", "note"):
        raise ValueError("status 只能是 done、skip、note")
    text = text.strip()
    if not text:
        raise ValueError("回覆不能是空的")

    def change(b):
        for l in labels(b):
            if l.get("n") == n:
                r = {"id": new_id("r"), "by": AGENT, "at": now(), "status": status, "text": text}
                l["replies"] = (l.get("replies") if isinstance(l.get("replies"), list) else []) + [r]
                return r
        raise ValueError(f"board 上沒有標籤 #{n}")
    return update(q, change)


def add_image(q, rel, at=None, width=None):
    """放圖庫裡的圖，只記參照。at=(x, y) 是 world 的左上角，沒給就接在最後一張圖右邊（空 40）；沒給 width 時高 320。"""
    if not (exact(rel) and (VAULT / rel).is_file()):
        raise FileNotFoundError(f"圖庫裡沒有這個檔案（大小寫也要一樣）：{rel}")
    size = image_size(rel)
    if not size:
        raise ValueError(f"讀不到像素尺寸，只能放圖片（影片、字體不能放）：{rel}")
    aspect, fp = size[0] / size[1], fingerprint(rel)

    def change(b):
        if at is None:
            imgs = [e for e in b["elements"] if e.get("type") == "image" and e.get("w") and e.get("h")]
            x, y = (imgs[-1].get("x", 0) + imgs[-1]["w"] + 40, imgs[-1].get("y", 0)) if imgs else (0, 0)
        else:
            x, y = at
        w = width or 320 * aspect
        e = new_element(b, "image", path=rel, fingerprint=fp, x=num(x), y=num(y), w=num(w), h=num(w / aspect))
        b["elements"].append(e)
        return e
    return update(q, change)


def add_text(q, text, at=None, size=15, color="gray"):
    """文字塊。at=(x, y) 是 world 的左上角，沒給就放在圖和文字下面（空 40）。"""
    text = text.strip()
    if not text or color not in COLORS:
        raise ValueError(f"文字不能是空的，顏色只能是 {'、'.join(COLORS)}")

    def change(b):
        if at is None:
            boxes = [(e.get("x", 0), e.get("y", 0) + e["h"]) for e in b["elements"]
                     if e.get("type") == "image" and e.get("h")]
            boxes += [(e.get("x", 0), e.get("y", 0) + e.get("size", 15) * 1.3 * (1 + (e.get("text") or "").count("\n")))
                      for e in b["elements"] if e.get("type") == "text"]
            x, y = (min(p[0] for p in boxes), max(p[1] for p in boxes) + 40) if boxes else (0, 0)
        else:
            x, y = at
        e = new_element(b, "text", text=text, x=num(x), y=num(y), size=size, color=color)
        b["elements"].append(e)
        return e
    return update(q, change)


def on_image(q, image_id):
    """綁在圖上要用的：那張圖和原圖的像素寬高。"""
    img = element(board(q), image_id)
    if img.get("type") != "image":
        raise ValueError(f"{image_id} 不是圖")
    size = image_size(img.get("path") or "")
    if not size:
        raise ValueError(f"讀不到 {img.get('path')} 的像素尺寸（檔案不見了？），沒辦法換算座標")
    return size


def add_label(q, image_id, text, at=None, color="orange"):
    """在圖上加標籤。at=(x, y) 是泡泡左上角，單位是原圖的像素；沒給是正中間。回傳標籤（編號是 n）。"""
    text = text.strip()
    if not text or color not in COLORS:
        raise ValueError(f"標籤文字不能是空的，顏色只能是 {'、'.join(COLORS)}")
    w, h = on_image(q, image_id)
    x, y = (at[0] / w, at[1] / h) if at else (0.5, 0.5)

    def change(b):
        if element(b, image_id).get("type") != "image":
            raise ValueError(f"{image_id} 不是圖")
        used = [e.get("n") for e in b["elements"] if e.get("type") == "label"]
        used += [t.get("n") for t in b["removed"] if isinstance(t, dict)]
        n = max((u for u in used if type(u) is int), default=0) + 1  # 拿掉的編號不重用
        e = new_element(b, "label", n=n, text=text, x=num(x, True), y=num(y, True), on=image_id,
                        color=color, by=AGENT, replies=[])
        b["elements"].append(e)
        return e
    return update(q, change)


def add_line(q, image_id, points, arrow=False, color="orange"):
    """在圖上畫線。points=[(x, y), …] 是原圖的像素：兩點是直線，多點是折線。arrow=True 最後一點畫箭頭。"""
    if not points or color not in COLORS:
        raise ValueError(f"線至少要一個點，顏色只能是 {'、'.join(COLORS)}")
    w, h = on_image(q, image_id)

    def change(b):
        if element(b, image_id).get("type") != "image":
            raise ValueError(f"{image_id} 不是圖")
        e = new_element(b, "stroke", points=[[num(x / w, True), num(y / h, True)] for x, y in points],
                        arrow=arrow, color=color, on=image_id)
        b["elements"].append(e)
        return e
    return update(q, change)


def remove(q, *ids):
    """從 board 拿掉元素，檔案留在圖庫。綁在上面的線和標籤一起拿掉，每個都留墓碑（標籤留編號）。"""
    def change(b):
        gone = []
        for eid in ids:
            element(b, eid)
            todo = [eid]
            while todo:
                cur = todo.pop()
                if cur not in gone:
                    gone.append(cur)
                    todo += [e.get("id") for e in b["elements"] if e.get("on") == cur]
        out = [e for e in b["elements"] if e.get("id") in gone]
        b["elements"] = [e for e in b["elements"] if e.get("id") not in gone]
        b["removed"] = [t for t in b["removed"] if not (isinstance(t, dict) and t.get("id") in gone)]
        t = now()
        b["removed"] += [{"id": e["id"], "n": e["n"], "at": t} if e.get("type") == "label" and type(e.get("n")) is int
                         else {"id": e["id"], "at": t} for e in out]
        return out
    return update(q, change)


def check_boards():
    """每個 board 讀一次並檢查格式。印「沒有問題」才算完成；圖檔不見、綁的元素不在只是提醒。"""
    bad, reminders = [], []
    files = sorted(BOARDS.glob("*.json")) if BOARDS.is_dir() else []
    for p in files:
        if not re.fullmatch(r"b[0-9a-f]{12}\.json", p.name):
            continue
        shown = p.relative_to(VAULT).as_posix()
        try:
            b = json.loads(p.read_bytes())
        except (OSError, ValueError) as e:
            bad.append(f"{shown}：讀不了（{e}）")
            continue
        if not isinstance(b, dict):
            bad.append(f"{shown}：不是 JSON 物件")
            continue
        if type(b.get("format")) not in (int, float) or b["format"] != 1:
            bad.append(f"{shown}：format 要是 1")
        if b.get("id") != p.stem:
            bad.append(f"{shown}：id 和檔名對不上")
        els = b.get("elements") or []
        if not (isinstance(els, list) and all(isinstance(e, dict) for e in els)):
            bad.append(f"{shown}：elements 要是物件陣列")
            continue
        ids = {e.get("id") for e in els}
        nums = [e.get("n") for e in els if e.get("type") == "label"]
        if len(nums) != len(set(nums)):
            reminders.append(f"{shown}：標籤編號重複（同步衝突合併時 Patina 會重新編號）")
        for e in els:
            if e.get("type") == "image" and not (e.get("path") and exact(e["path"]) and (VAULT / e["path"]).is_file()):
                reminders.append(f"{shown}：{e.get('id')} 的圖不在圖庫裡：{e.get('path')}")
            if e.get("on") and e["on"] not in ids:
                reminders.append(f"{shown}：{e.get('id')} 綁的元素不在了：{e['on']}")
    for title, rows in (("格式不對的 board", bad), ("提醒", reminders)):
        if rows:
            print(f"{title}：{len(rows)}")
            for row in rows:
                print(f"  {row}")
    if not bad:
        print("沒有問題")
    return not bad
```

在放 `patina_board.py` 的資料夾執行：

```sh
python3 - <<'EOF'
from patina_board import boards, board, notes, reply, add_image, add_text, add_label, add_line, remove, check_boards
print(boards())                                   # 先看有哪些 board、各有幾個沒回覆的標籤
for n in notes("旅行參考"):                         # 名稱或 id
    print(n["n"], n["text"], n["by"], n["path"], n["px"])
reply("旅行參考", 1, "done", "改成 2:3 了")          # done 已改、skip 沒改、note 只回話
img = add_image("旅行參考", "2025/旅行/IMG_0002.jpg")
add_text("旅行參考", "夜景的光線")
add_label("旅行參考", img["id"], "這盞燈太亮", at=(1200, 300))   # 原圖像素
add_line("旅行參考", img["id"], [(900, 800), (1200, 340)], arrow=True, color="blue")
# remove("旅行參考", img["id"])                     # 圖上的線和標籤一起拿掉
check_boards()
EOF
```

### 回覆使用者在 board 上標的東西

1. 使用者在 board 的圖上加了標籤，例如「拱的比例」「這盞燈太亮」。
2. `boards()` 看哪個 board 有沒回覆的標籤（`open`），`notes(<board>)` 列出來：編號、文字、誰標的、在哪張圖（`path`）、位置（`px`，原圖像素，是泡泡的左上角；沒綁在圖上的標籤 `px` 是 `None`，看 `world`）。
3. 看那張圖：直接讀圖庫裡的檔案 `~/Patina/<path>`，標籤指的是原圖像素 `px` 附近。不要拿 board 上的 `w`、`h` 換算。
4. 做事：例如用側檔工具的 `edit()` 加標籤、寫描述，或在圖上用 `add_line()`、`add_label()` 指出位置。不改照片本身。
5. 回覆每個處理過的標籤：做了寫 `done`，沒做寫 `skip` 加原因，只是回答問題寫 `note`。回覆寫了就不會再出現在 `notes()`。
6. 跑 `check_boards()`（改了側檔也跑 `check()`），印「沒有問題」才算完成。告訴使用者回覆了哪幾個標籤。

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

圖庫用 Syncthing 在多台電腦同步時，兩邊離線時改了同一份檔案，會多一份名稱含 `.sync-conflict-` 的副本（例如 `.patina/meta/2025/IMG_0001.jpg.sync-conflict-20261104-083000-ABCDEFG.json`）。

- **側檔的副本**（`.patina/meta/`、`.patina/removed/` 底下）：Patina 0.2.4 起打開圖庫、偵測到變動時會自動合併：標籤取聯集，描述、網址、評分取 `updated_at` 比較新的那份（沒有 `updated_at` 就看側檔的修改時間），被蓋掉的值寫進 `.patina/log/`。agent 不要自己合併或刪掉。`check()` 列出來時，請使用者打開 Patina（0.2.4 以上）讓它合併，再跑一次 `check()`；還在就告訴使用者是哪幾份。
- **board 的副本**（`.patina/boards/` 底下）：Patina 0.3.0 起也會自動合併，逐個元素取 `updated_at` 比較新的，回覆取聯集。agent 不要自己合併或刪掉；`.patina/removed/boards/` 的副本 Patina 不動，告訴使用者。
- **照片、影片本身的副本**：Patina 不會自動處理（不知道哪個才對）。告訴使用者是哪幾份，讓使用者決定留哪個。

## iPhone 配對

Patina 的 iPhone 版內建 Syncthing，從 Mac 的 Syncthing 收圖庫。Mac 上要先有 Syncthing 在跑，圖庫已經是 Syncthing 資料夾（整個圖庫、包括 `.patina/`，見「Syncthing 設定」）。流程：手機掃 Mac 的 QR → Mac 接受手機、分享圖庫 → 手機勾要放的圖庫。

### 1. 產生 Mac 的 QR

QR 的內容就是 Mac 的 Syncthing 裝置 ID（純文字，不是網址）。

- 使用者自己做：Syncthing 網頁介面（預設 `http://127.0.0.1:8384`）右上「動作 › 顯示 ID」。
- Agent 產生圖檔：用 Syncthing 網頁介面自己的 QR 端點，不用另外裝工具。API key 從標準輸入交給 curl，不放進指令參數。

```sh
ID=$(syncthing device-id)
GUI=$(syncthing cli config gui raw-address get)
syncthing cli config gui apikey get | sed 's/^/X-API-Key: /' \
  | curl -sf -H @- -o ~/Desktop/patina-pair.png "http://$GUI/qr/?text=$ID"
```

把 `patina-pair.png` 給使用者看，讓手機掃。網頁介面開了 HTTPS 時把 `http://` 換成 `https://` 並加 `-k`（自簽憑證）。找不到 `syncthing` 指令（例如用的是 Syncthing 的 macOS app）時，請使用者用網頁介面的「顯示 ID」。

### 2. 手機掃 QR

使用者在 iPhone 打開 Patina，「連到 Mac」畫面掃 QR，手機顯示「等待 Mac 接受」。

### 3. Mac 接受手機、分享圖庫

```sh
syncthing cli show pending devices        # 等待加入的裝置：手機的 ID 和名稱
syncthing cli config devices add --device-id <手機 ID> --name "<手機名稱>"
for f in $(syncthing cli config folders list); do echo "$f  $(syncthing cli config folders "$f" path get)"; done   # 找圖庫的資料夾 ID
syncthing cli config folders <資料夾 ID> devices add --device-id <手機 ID>
```

- pending 裡的名稱要是使用者的手機才加；看不出來就問使用者。
- 網頁介面也可以：跳出「新裝置」→ 加入裝置 → 在圖庫資料夾的「共用」勾這支手機。
- 手機上刪掉的檔案 Mac 會跟著刪。建議 Mac 這邊的圖庫資料夾開「垃圾桶」版本保留，刪錯可以從 `.stversions` 救回：`syncthing cli config folders <資料夾 ID> versioning type set trashcan`。

### 4. 手機選圖庫

手機列出 Mac 分享的圖庫，使用者勾要放的（會顯示大小和手機剩下的空間，放不下不能開始）再按開始。之後在 Patina 的同步頁可以「加入圖庫」、切換圖庫、「從手機移除」（只刪手機上的，Mac 不動）。

### 卡住時

- 手機一直「等待 Mac 接受」：Mac 還沒加這支手機，或還沒分享圖庫。`syncthing cli show pending devices` 還看得到手機就是還沒加。
- 兩邊的 Syncthing 都要開著。不在同一個網路時靠 Syncthing 的全球探索與中繼，比較慢。
- 掃不了 QR 時：「連到 Mac」畫面改用「改用這支手機的 ID」，Mac 用那個 ID 加裝置、分享圖庫，手機再接受。

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
