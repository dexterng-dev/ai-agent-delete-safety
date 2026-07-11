[English](README.md) · **繁體中文** · [廣東話](README.cantonese.md)

# 別讓你的 AI 刪除你的檔案

### 為任何給予 AI 代理終端機（Shell）存取權的人而設的 Hook 安全網

*作者：Dexter Ng，與 Sonne T，我的 AI 家族辦公室，合著*

---

## 問題所在

只要給予 AI 編程代理終端機存取權限，遲早它會執行類似以下的指令：

```
rm -rf ./
Remove-Item -Recurse -Force C:\Users\you\Projects
```

也許它誤判了哪個資料夾才是「暫存的那個」。也許你叫它「清理一下」，而它把這句話理解得比你想像中更廣泛。無論如何，這個指令會在幾毫秒內執行——沒有對話框、沒有「你確定嗎」、也沒有復原（Undo）。

## 為何檢查資源回收筒救不了你

Windows 的資源回收筒（Recycle Bin）與 macOS 的垃圾桶（Trash），只會攔截經由 *作業系統介面* 進行的刪除——即 Explorer、Finder 或圖形介面檔案管理員。透過終端機指令（`rm`、`Remove-Item`）進行的刪除，會完全繞過這一層。檔案並不是躲在資源回收筒裡等你救回來——它們已經永久消失了；而如果你有雲端同步服務在運行（OneDrive、Dropbox、Google Drive），這次刪除可能已經同步到雲端副本上。

這正是為何「檢查一下資源回收筒就好」是個錯誤的教學方向。真正的解決方法必須設在刪除動作的 *上游*，而不是下游。

## 兩層防護

1. **預防（Prevention）**——設置一個 Hook，在每個指令 *執行前* 檢查一次，直接攔截具破壞性的模式。
2. **復原網（Recovery net）**——即使是合法的清理動作，也一律經由「移動到垃圾資料夾＋記錄一筆」的步驟處理，而非真正刪除，令一切永遠可復原。

只做第 1 點，一旦模式比對出現漏洞，後果就是災難性的。只做第 2 點，你就是在賭 AI 會自願走安全的那條路。兩者都做，一層就能補足另一層的盲點。

---

## 步驟一——在指令執行前先攔截它

Claude Code 支援 **執行前 Hook（pre-execution hook）**：一段腳本會接收 AI 即將執行的完整指令，並可以在它到達 shell 之前將其否決。

有一件事很容易被忽略：Hook 是按 **工具名稱（tool name）** 註冊的，而不是按作業系統註冊的。`Bash` 與 `PowerShell` 是 Claude Code 裡兩個獨立的工具，視乎你的設定，兩者可以同時啟用——如果你只註冊了 `Bash` 的 matcher，任何透過 `PowerShell` 工具執行的指令就會完全繞過這個 Hook，你對 `Remove-Item` 的防護形同虛設。兩個 matcher 都要註冊，才不會留下缺口。

**`.claude/settings.json`**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/scripts/block_destructive.py\"",
            "timeout": 5
          }
        ]
      },
      {
        "matcher": "PowerShell",
        "hooks": [
          {
            "type": "command",
            "command": "python \"$CLAUDE_PROJECT_DIR/scripts/block_destructive.py\"",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

同一支腳本，兩筆設定——每個工具各一筆。下面的 `DANGEROUS` 正則表達式清單已經同時涵蓋 `rm -rf`（Bash）與 `Remove-Item -Recurse -Force`（PowerShell），所以不用再改其他地方。

**`scripts/block_destructive.py`**

```python
import json, re, sys

DANGEROUS = [
    r"rm\s+-[a-z]*r[a-z]*f",        # rm -rf, rm -fr, rm -Rf...
    r"rm\s+-[a-z]*f[a-z]*r",
    r"Remove-Item.*-Recurse.*-Force",
    r"git\s+clean\s+-[a-z]*f",
    r"git\s+reset\s+--hard",
    r"DROP\s+TABLE",
    r"DROP\s+DATABASE",
]

payload = json.load(sys.stdin)
command = payload.get("tool_input", {}).get("command", "")

for pattern in DANGEROUS:
    if re.search(pattern, command, re.IGNORECASE):
        print(
            f"BLOCKED: '{command}' matches a destructive pattern.\n"
            "Use scripts/trash_move instead - it moves files to trash/ "
            "and logs the action, so it stays reversible.",
            file=sys.stderr,
        )
        sys.exit(2)   # exit code 2 = veto the tool call

sys.exit(0)
```

Exit code `2` 是 Claude Code 用來判斷「不要執行這個」的訊號。印到 stderr 的訊息會回傳給模型作為理由——所以它不會單純重試同一個指令，而是會看到指示，改用安全的替代方案。

## 步驟二——給它一個可復原的替代方案

只攔截、不給替代方案，只會讓代理不斷重試、換句話說，或索性放棄手上的任務。給它一支腳本，做和 `rm -rf` 一樣的事，但是可復原的。

**`scripts/trash_move.sh`**（macOS / Linux）

```bash
#!/usr/bin/env bash
set -euo pipefail
TARGET="$1"
TRASH_DIR="./trash"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p "$TRASH_DIR"
BASENAME=$(basename "$TARGET")
DEST="$TRASH_DIR/${TIMESTAMP}-${BASENAME}"
mv "$TARGET" "$DEST"
echo "$(date -Iseconds) | moved '$TARGET' -> '$DEST'" >> "$TRASH_DIR/MANIFEST.md"
echo "Moved to $DEST (recoverable)."
```

**`scripts/trash_move.ps1`**（Windows）

```powershell
param([Parameter(Mandatory=$true)][string]$Target)
$TrashDir = ".\trash"
$Timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
New-Item -ItemType Directory -Force -Path $TrashDir | Out-Null
$BaseName = Split-Path $Target -Leaf
$Dest = Join-Path $TrashDir "$Timestamp-$BaseName"
Move-Item -Path $Target -Destination $Dest
"$(Get-Date -Format o) | moved '$Target' -> '$Dest'" | Add-Content "$TrashDir\MANIFEST.md"
Write-Host "Moved to $Dest (recoverable)."
```

接著，在你的專案指示中（`CLAUDE.md`、system prompt，或任何你的框架會讀取的地方）明確寫下這條規則：*永遠不要直接呼叫 `rm` / `Remove-Item`——一律呼叫 trash-move 腳本。* 配合步驟一的 Hook，這就變成自我強制執行：原始的刪除指令被攔截，而攔截訊息會準確告訴代理該用什麼替代方案。

## 步驟三——第二層防護：同時要求人手確認

雙重保險。大部分代理工具都有一層獨立於 Hook 之外的權限系統——利用它把破壞性指令標記為「詢問（ask）」而非「允許（allow）」，讓人類在指令執行前親眼看到那行原始指令，而不是單憑一支腳本的判斷：

```json
"permissions": {
  "ask": ["Bash(rm *)", "Bash(rm -rf *)", "PowerShell(Remove-Item * -Recurse -Force*)"]
}
```

## 步驟四——先測試，再信任

1. 建立一個用完即棄的測試資料夾，裡面放一個假檔案。
2. 叫代理刪除它。
3. 確認：原始的刪除指令被攔截，代理改用 trash-move 腳本（或先詢問你），而該檔案最終落在 `trash/` 資料夾裡，`MANIFEST.md` 亦多了一行記錄。
4. 嘗試一些刻意誤導的說法——「永久移除這個」、「把這個資料夾徹底清乾淨」——確認 Hook 仍然能攔截 *底層的指令*，而不只是「刪除」這兩個字。

---

## 不只是 Claude Code——同樣原則放諸四海皆準

以上的 Hook API 是 Claude Code 特有的，但這個原則適用於任何能執行終端機指令的代理框架（Cursor、Copilot 代理模式，或自製的 LangChain / agent-SDK 迴圈）：

1. 找出攔截點——任何讓 AI 碰得到 shell 的框架，都會有一個：一層工具包裝、一層中介軟體，或一個核准回呼函式。
2. 在那個攔截點放一份拒絕清單（deny-list）的正則表達式，比對的是指令的字面內容，而不是 AI 聲稱的意圖。
3. 將指令導向可復原的 trash-move 輔助腳本，而不是一個死路式的硬性攔截。
4. 每次移動都記錄到一份只能新增、不能修改的日誌（append-only manifest）——即使是合法的清理動作，也應該留下書面紀錄。
5. 千萬別讓這個修正方案倚賴 AI「記得」提示裡的一條規則。規則必須由程式碼強制執行，不論 AI 願不願意配合。

---

## 重點摘要（TL;DR）

- CLI 刪除會繞過資源回收筒／垃圾桶——沒有原生的復原（Undo）機制。
- 加一個 `PreToolUse` Hook，用正則表達式攔截 `rm -rf`、`Remove-Item -Recurse -Force`、`git clean -f`、`git reset --hard`。
- 給代理一支 trash-move 腳本作為官方認可的替代方案——移動並記錄，絕不真正刪除。
- 在同一組模式上加一層「詢問（ask）」權限，作為第二重、由人手把關的防護。
- 先用一個用完即棄的測試資料夾試過，才把它用在真正重要的資料夾上。

---

## 授權條款（License）

CC BY 4.0——歡迎自由分享及改編，只需註明作者 Dexter Ng。詳見 [LICENSE](LICENSE)。
