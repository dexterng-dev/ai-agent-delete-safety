[English](README.md) · [繁體中文](README.zh-hant.md) · **Cantonese**

# 唔好俾你個 AI 累 刪咗你啲檔案

### 俾 AI 用緊 terminal 嘅人，一個用 hook 做嘅安全網

*作者：Dexter Ng，同埋 Sonne T——我嘅 AI 家族辦公室——一齊寫*

---

## 問題喺邊度

你俾個 AI 寫程式嘅 agent 有埋 terminal 權限，遲早佢會走一句咁嘅指令：

```
rm -rf ./
Remove-Item -Recurse -Force C:\Users\you\Projects
```

可能佢估錯咗邊個資料夾先係「暫時嗰個」。又可能你叫佢「清理一下」，但佢理解闊過你本身諗嘅範圍。總之呢句嘢幾毫秒之內就跑完——冇彈窗、冇「你係咪肯定」、亦都冇得返轉頭。

## 點解 check 返 Trash 都救唔到你

Windows 嘅資源回收筒（Recycle Bin）同 macOS 嘅垃圾桶（Trash），淨係會兜截經過 *作業系統介面* 嘅刪除——即係 Explorer、Finder、或者 GUI 檔案總管嗰啲。用 terminal 指令（`rm`、`Remove-Item`）刪嘢，直情唔會經過呢層。啲檔案唔係匿咗喺 Trash 度等你救返轉頭——係真係冇咗；如果你部機仲開住雲端同步（OneDrive、Dropbox、Google Drive），嗰個刪除動作可能已經 sync 埋去雲端嗰邊。

所以「check 返 Trash 咪得囉」係一個錯得好緊要嘅諗法。真正嘅解法一定要 set 喺刪除動作 *之前*，唔係之後先諗辦法。

## 兩層防護

1. **預防（Prevention）**——整個 hook，每次執行指令 *之前* 都 check 一次，見到有殺傷力嘅 pattern 就直接兜截。
2. **翻生網（Recovery net）**——就算係啱嘅清理動作，都一律行「搬去 trash 資料夾＋log 低」呢條路，唔會真係刪除，咁樣就永遠有得救返。

淨係做第 1 點，你嘅 pattern-match 一有漏洞就大鑊。淨係做第 2 點，即係賭個 AI 自己肯行安全嗰條路。兩樣都做，一層就補到埋另一層嘅盲點。

---

## Step 1——喺指令執行之前兜截佢

Claude Code 支援 **執行前 hook（pre-execution hook）**：一段 script 會收到個 AI 就嚟要行嘅完整指令，喺佢未撞到 shell 之前就可以否決佢。

有樣嘢好易漏睇：hook 係跟 **工具名（tool name）** 去 register，唔係跟作業系統。`Bash` 同 `PowerShell` 喺 Claude Code 入面係兩個獨立嘅工具，睇你點 set，兩個都可以同時開住——如果你淨係 register 咗 `Bash` 嗰個 matcher，任何行 `PowerShell` 工具嘅指令就會直情繞過個 hook，你嗰個 `Remove-Item` 防護形同冇裝過。兩個 matcher 都要 register，先至冇漏洞。

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

同一支 script，兩份 entry——一個工具一份。下面嗰份 `DANGEROUS` 正則表達式清單已經同時 cover 咗 `rm -rf`（Bash）同 `Remove-Item -Recurse -Force`（PowerShell），所以其他嘢都唔使改。

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

Exit code `2` 就係 Claude Code 用嚟判斷「唔好行呢句」嘅信號。印落 stderr 嗰句嘢會傳返俾個 model 做理由——所以佢唔會齋齋 retry 返同一句指令，而係會見到指示，轉用安全嗰個做法。

## Step 2——俾佢一個有得翻生嘅代替做法

淨係兜截、冇代替方案，只會令個 agent 不斷 retry、唔同講法試過，或者索性擺爛個 task。俾佢一支 script，做嘅嘢同 `rm -rf` 一樣，但係有得翻生。

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

跟住，喺你個 project 嘅 instructions 度（`CLAUDE.md`、system prompt，或者你個 framework 會讀嗰個位）寫明呢條規矩：*永遠唔好直接 call `rm` / `Remove-Item`——一律 call trash-move 嗰支 script。* 配埋 Step 1 嗰個 hook，呢個就會自動強制執行：原始嘅刪除指令俾人兜截，個兜截訊息仲會準確話俾 agent 聽應該用邊樣代替。

## Step 3——第二重保障：都要人手確認一次

雙重保險。大部分 agent 工具都有一層獨立於 hook 嘅權限系統——用嚟將有殺傷力嘅指令標做「問一問（ask）」，唔係「照批（allow）」，等人喺指令行之前親眼睇到嗰句原始指令，唔係齋齋靠一支 script 嘅判斷：

```json
"permissions": {
  "ask": ["Bash(rm *)", "Bash(rm -rf *)", "PowerShell(Remove-Item * -Recurse -Force*)"]
}
```

## Step 4——先 test 過，先至信得過

1. 開一個用完即棄嘅 test 資料夾，入面擺個假檔案。
2. 叫個 agent 刪咗佢。
3. Check 清楚：原始嘅刪除指令俾人兜截，agent 轉用 trash-move script（或者先問你），個檔案最後落咗喺 `trash/` 度，`MANIFEST.md` 都多咗一行 record。
4. 試下講啲拗彎嘅講法——「permanent 移除呢個」、「將呢個資料夾徹底清乾淨」——check 清楚個 hook 仲係兜截到 *背後嗰句指令*，唔係淨係捉「刪除」呢兩個字。

---

## 唔只係 Claude Code——同一個原則邊度都用得

上面嗰個 hook API 係 Claude Code 專有嘅，但呢個原則適用於任何行得 terminal 指令嘅 agent framework（Cursor、Copilot agent mode，或者自己砌嘅 LangChain / agent-SDK loop）：

1. 搵出攔截點——任何俾 AI 掂到 shell 嘅 framework 都有一個：可能係一層 tool wrapper、一層 middleware，或者一個 approval callback。
2. 喺嗰個攔截點擺一份 deny-list 正則表達式，對嘅係指令嘅字面內容，唔係 AI 話自己想點做。
3. 導去一個有得翻生嘅 trash-move helper，唔好齋齋做一個死路式嘅硬兜截。
4. 每次搬動都 log 落一份只可以加、唔可以改嘅 manifest——就算係啱嘅清理動作，都應該留低書面紀錄。
5. 千祈唔好靠個 AI「記得」prompt 入面嗰條規矩。規矩一定要由 code 去強制執行，唔理 AI 肯唔肯合作。

---

## 重點摘要（TL;DR）

- CLI 刪除會繞過 Recycle Bin/Trash——冇原生嘅 Undo。
- 加一個 `PreToolUse` hook，用正則表達式兜截 `rm -rf`、`Remove-Item -Recurse -Force`、`git clean -f`、`git reset --hard`。
- 俾 agent 一支 trash-move script 做官方認可嘅代替方案——搬咗＋log 低，永遠唔好真係刪除。
- 喺同一組 pattern 度加多層「問一問（ask）」權限，做第二重、有人手把關嘅保障。
- 先攞個用完即棄嘅資料夾試過，先至擺去真正緊要嗰個度用。

---

## 授權（License）

CC BY 4.0——隨便share同改都得，記得credit返 Dexter Ng 就得。詳情睇 [LICENSE](LICENSE)。

有問題、想指正，或者見到錯處？歡迎開 Issue 或者 PR——但唔保證幾時得閒睇。
