**English** · [繁體中文](README.zh-hant.md) · [廣東話](README.cantonese.md)

# Stop Your AI From Deleting Your Files

### A hook-based safety net for anyone giving an AI agent shell access

*By Dexter Ng, with Sonne T, my AI Family Office*

---

## The problem

Give an AI coding agent terminal access, and sooner or later it will run a command like:

```
rm -rf ./
Remove-Item -Recurse -Force C:\Users\you\Projects
```

Maybe it misjudged which folder was "the temp one." Maybe you asked it to "clean this up" and it interpreted that more broadly than you meant. Either way, the command executes in milliseconds, and there's no dialog box, no "are you sure," no undo.

## Why checking the Trash doesn't save you

The Recycle Bin (Windows) and Trash (macOS) only catch deletions routed through the *OS shell* — Explorer, Finder, a GUI file manager. Deletions from a terminal command (`rm`, `Remove-Item`) bypass that layer entirely. The files aren't hiding in Trash waiting to be restored — they're gone, and if you have cloud sync running (OneDrive, Dropbox, Google Drive), the deletion may already have propagated to the cloud copy too.

This is why "just check the Trash" is the wrong instinct to teach people. The actual fix has to sit *upstream* of the delete, not downstream of it.

## The two-layer fix

1. **Prevention** — a hook that inspects every command *before* it runs and blocks destructive patterns outright.
2. **Recovery net** — even for legitimate cleanup, force it through a "move to a trash folder + log it" step instead of a real delete, so nothing is ever truly unrecoverable.

Do only #1 and a bug in your pattern-matching becomes catastrophic. Do only #2 and you're relying on the AI to voluntarily use the safe path. Do both, and one layer covers the other's blind spot.

---

## Step 1 — Intercept the command before it runs

Claude Code supports a **pre-execution hook**: a script that receives the exact command the AI is about to run, and can veto it before it ever reaches a shell.

One thing that's easy to miss: the hook is registered per **tool name**, not per operating system. `Bash` and `PowerShell` are two separate Claude Code tools, and depending on your setup either or both can be active at once — if you only register a `Bash` matcher, any command that comes through the `PowerShell` tool skips the hook entirely, and your `Remove-Item` protection is dead weight. Register both matchers so neither path is left open.

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

Same script, two entries — one per tool. The `DANGEROUS` regex list below already covers both `rm -rf` (Bash) and `Remove-Item -Recurse -Force` (PowerShell), so nothing else changes.

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

Exit code `2` is the signal Claude Code checks for "don't run this." The message printed to stderr goes back to the model as the reason — so it doesn't just retry the same command, it sees the instruction to use the safe alternative instead.

## Step 2 — Give the agent something reversible to use instead

A block with no alternative just makes the agent retry, rephrase, or give up mid-task. Give it a script that does the same job as `rm -rf`, but reversibly.

**`scripts/trash_move.sh`** (macOS/Linux)

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

**`scripts/trash_move.ps1`** (Windows)

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

Then state the rule plainly in your project instructions (`CLAUDE.md`, system prompt, whatever your framework reads): *never call `rm`/`Remove-Item` directly — always call the trash-move script.* Paired with Step 1's hook, this becomes self-enforcing: the raw delete is blocked, and the block message tells the agent exactly what to use instead.

## Step 3 — A second net: require human confirmation on the pattern too

Belt and braces. Most agent tools have a permission layer separate from hooks — use it to mark destructive patterns as "ask" rather than "allow," so a human sees the literal command before it runs, not just a script's judgment call:

```json
"permissions": {
  "ask": ["Bash(rm *)", "Bash(rm -rf *)", "PowerShell(Remove-Item * -Recurse -Force*)"]
}
```

## Step 4 — Test it before you trust it

1. Create a throwaway test folder with a dummy file.
2. Ask the agent to delete it.
3. Confirm: the raw delete gets blocked, the agent uses the trash-move script (or asks you first), and the file lands in `trash/` with a line in `MANIFEST.md`.
4. Try adversarial phrasing — "permanently remove this," "wipe this folder clean" — and confirm the hook still catches the *underlying command*, not just the word "delete."

---

## Beyond Claude Code — the same pattern works anywhere

The exact hook API is Claude Code's, but the principle transfers to any agent framework that can run shell commands (Cursor, Copilot agent mode, a custom LangChain/agent-SDK loop):

1. Find the interception point — every framework that lets AI touch a shell has one: a tool wrapper, a middleware layer, an approval callback.
2. Put a deny-list regex at that point, matched against the literal command, not the AI's stated intent.
3. Redirect to a reversible trash-move helper instead of a hard dead-end block.
4. Log every move to an append-only manifest — even legitimate cleanup should leave a paper trail.
5. Never let the fix depend on the AI remembering a rule in a prompt. The rule has to be enforced by code that runs whether or not the AI cooperates.

---

## TL;DR

- CLI deletes skip the Recycle Bin/Trash — there is no native undo.
- Add a `PreToolUse` hook that regex-blocks `rm -rf`, `Remove-Item -Recurse -Force`, `git clean -f`, `git reset --hard`.
- Give the agent a trash-move script as the sanctioned alternative — move + log, never truly delete.
- Add "ask" permissions on the same patterns as a second, human-in-the-loop net.
- Test with a throwaway folder before you trust it with a real one.

---

## License

CC BY 4.0 — share and adapt freely, just credit Dexter Ng. See [LICENSE](LICENSE).
