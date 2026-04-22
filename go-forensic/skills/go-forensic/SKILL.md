---
name: go-forensic
description: Use when the user wants to run the local go-forensic.exe tool — including Android/iOS app data export, iOS device listing, USB port proxying, or SQLite keyword search. Triggers on keywords like "取证 / 导出 / go-forensic / 安卓导出 / iOS导出 / sqlite搜索 / forensic export", or any phrase combining "提取" with "数据 / app / 应用" (e.g. "提取微信数据", "提取这个app", "应用数据提取"). Target apps are identified by **bundle id / package name** (e.g. `com.tencent.mm`). On first use, asks the user for a default export root and persists it to the user's home dir. Every export goes into <root>/<platform><bundle_ids_underscored>/. The `rm` subcommand of go-forensic.exe is intentionally NOT exposed — refuse to invoke it and tell the user to run it manually.
---

# go-forensic skill

## Fixed environment (do not ask the user about these)

- **Binary:** `go-forensic` — assumed to be on the user's `PATH`. Always invoke as the bare command `go-forensic`, never with a hard-coded absolute path.
- **Config file:**
  - Bash: `$HOME/.go-forensic/config.json`
  - PowerShell: `$env:USERPROFILE\.go-forensic\config.json`
  Both paths point to the same file on Windows.
- If the command is not found, do **not** guess a path. Stop and tell the user that `go-forensic` is not on `PATH`; suggest they add it or reinstall the plugin's prerequisites.
- Never invent flags. The known flag set is documented below — if the user asks for something outside it, run `go-forensic <subcommand> --help` first to verify.

## Shell selection (decide ONCE per session, then reuse)

This skill must work whether Claude Code is configured to use Bash (Git Bash on Windows) or PowerShell. Detect once:

1. Try `echo "shell=$SHELL"` via the Bash tool. If output contains a non-empty `$SHELL` value (e.g. `shell=/usr/bin/bash`), use **Bash mode**.
2. Otherwise — including when `$SHELL` is empty / literal `$SHELL` (PowerShell does not expand it) — use **PowerShell mode**.
3. Remember the choice for the rest of the session. Every Bash/PowerShell tool call below has both a Bash and a PowerShell version; pick the matching one.

## Subcommand catalogue

| Intent | Command shape |
|---|---|
| Export Android app data | `android export -k <bundle_id> [-k <bundle_id> ...] -o <auto-derived> [-s <paths...>]` |
| Export iOS app data over SSH | `ios export -k <bundle_id> [-k <bundle_id> ...] -o <auto-derived> [-a user@host:port] [-p <pass>] [...]` |
| List connected iOS devices | `ios device list` |
| USB-proxy a TCP/UDP port from iOS | `ios proxy -l <local> -r <remote> [-p tcp\|udp] [-d <device-id>]` |
| Search keyword inside a SQLite DB | `sqlite -f <db.sqlite> search -k <keywords...> [--ignore-err]` |
| Show version | `version` |

> **Excluded on purpose:** the `rm` subcommand of go-forensic.exe is a destructive file-deletion command. This skill will NOT invoke it. If the user explicitly asks to delete files via go-forensic, refuse and tell them to run `go-forensic rm -p <path>` themselves in a terminal.

## App identification convention

The user always identifies target apps by **bundle id / package name** — dot-separated reverse-DNS form. Examples:

- iOS: `com.tencent.xin`, `com.burbn.instagram`, `com.temple`
- Android: `com.tencent.mm`, `com.xingin.xhs`, `com.example.MyApp`

Rules:

1. **Do not translate to pinyin, Chinese names, or any other identifier.** The bundle id goes straight into `-k` and into the subdirectory name.
2. If the user's message does not contain a bundle id (e.g. they just say "提取微信数据"), **ask them** for the bundle id — do not guess. Example prompt: "请提供要提取的应用的 bundle id / 包名（例如 `com.tencent.mm`）"
3. If the user lists multiple bundle ids (comma / 、 / space / "和" separated), pass **each as its own `-k` flag** in a single invocation. Never concatenate them into one `-k "a,b"` value.
4. Bundle id sanity check: must match the case-insensitive pattern `^[A-Za-z][A-Za-z0-9]*(\.[A-Za-z0-9_-]+)+$`. If the user-provided string fails this, ask for confirmation before using it.

## Initialization protocol (run on EVERY invocation, before anything else)

### Step 1 — Read existing config

**Bash mode:**
```bash
test -f "$HOME/.go-forensic/config.json" && cat "$HOME/.go-forensic/config.json" || echo "MISSING"
```

**PowerShell mode:**
```powershell
$cfg = Join-Path $env:USERPROFILE '.go-forensic\config.json'
if (Test-Path $cfg) { Get-Content -Raw $cfg } else { 'MISSING' }
```

### Step 2 — If MISSING or `output_root` empty

a. Use `AskUserQuestion` to ask: "首次使用 go-forensic 插件，请设置默认导出根目录的绝对路径（例如 `D:\forensic-data`）。所有导出会自动放进 `<根目录>/<平台><包名下划线形式>/` 子目录。"

b. Validate: must be absolute, must not contain `..`, must not be a system root like `C:\` / `D:\` alone.

c. Create the directory:
   - **Bash:** `mkdir -p "<root>"`
   - **PowerShell:** `New-Item -ItemType Directory -Force -Path "<root>" | Out-Null`

d. Write the config — **use a JSON-aware writer to avoid the backslash-escape bug**:
   - **Bash:** escape backslashes and double-quotes manually before writing:
     ```bash
     ROOT='<root>'
     mkdir -p "$HOME/.go-forensic"
     ESCAPED=$(printf '%s' "$ROOT" | sed -e 's/\\/\\\\/g' -e 's/"/\\"/g')
     printf '{\n  "output_root": "%s",\n  "version": 1\n}\n' "$ESCAPED" > "$HOME/.go-forensic/config.json"
     ```
   - **PowerShell:** let `ConvertTo-Json` handle escaping:
     ```powershell
     $dir = Join-Path $env:USERPROFILE '.go-forensic'
     New-Item -ItemType Directory -Force -Path $dir | Out-Null
     $obj = [ordered]@{ output_root = '<root>'; version = 1 }
     $obj | ConvertTo-Json | Set-Content -Path (Join-Path $dir 'config.json') -Encoding UTF8
     ```

e. Confirm to the user that the config has been saved, then proceed.

### Step 3 — Read the `output_root` field from the JSON

After reading the file content (Step 1), parse the JSON to extract `output_root`. **Important:** when interpreting the path string, treat the JSON `\\` escapes as single backslashes. So `"D:\\forensic-data"` in JSON means the actual path `D:\forensic-data`.

### Step 4 — Reset on user request

If the user says "改导出目录 / 修改默认路径 / reset config", treat it like first use — ask again and overwrite (steps 2a-e).

### Step 5 — Ensure the iOS export monitor wrapper script is installed

The iOS export uses a wrapper script that watches the tool's log and detects SSH auth failure by polling. Always (re)install it on every invocation — overwrites are cheap and guarantee the latest version. Pick the version matching the session's shell mode.

**Bash mode:**
```bash
mkdir -p "$HOME/.go-forensic/scripts"
cat > "$HOME/.go-forensic/scripts/ios-export-monitor.sh" <<'WRAPPER_EOF'
#!/usr/bin/env bash
# go-forensic ios export wrapper.
# Runs the export in background; polls last log line every 10 s, three times.
# If all three polls match "getting ssh connection", kills the process and
# exits 100 (auth failure marker). Otherwise the process runs to completion
# with NO upper time limit (large legitimate exports can take hours).
# Usage: ios-export-monitor.sh <output_dir> <bundle_id1> [<bundle_id2> ...] [-- -p PASSWORD]
set -u
OUTPUT_DIR="$1"; shift
BUNDLE_KS=()
EXTRA=()
SAW_DDASH=0
for arg in "$@"; do
  if [ "$arg" = "--" ]; then SAW_DDASH=1; continue; fi
  if [ "$SAW_DDASH" = "1" ]; then EXTRA+=("$arg"); else BUNDLE_KS+=(-k "$arg"); fi
done
LOG="$HOME/.go-forensic/last-ios-export.log"
mkdir -p "$(dirname "$LOG")"
: > "$LOG"
go-forensic ios export "${BUNDLE_KS[@]}" -o "$OUTPUT_DIR" -u "${EXTRA[@]}" > "$LOG" 2>&1 &
PID=$!
shopt -s nocasematch
for i in 1 2 3; do
  sleep 10
  if ! kill -0 "$PID" 2>/dev/null; then
    wait "$PID"; EC=$?; cat "$LOG"; exit $EC
  fi
  LAST=$(tail -1 "$LOG" 2>/dev/null)
  if [[ "$LAST" != *"getting ssh connection"* ]]; then
    wait "$PID"; EC=$?; cat "$LOG"; exit $EC
  fi
done
kill "$PID" 2>/dev/null; sleep 1; kill -9 "$PID" 2>/dev/null
cat "$LOG"
echo "__AUTH_FAILURE_DETECTED__"
exit 100
WRAPPER_EOF
```

**PowerShell mode:**
```powershell
$scriptDir = Join-Path $env:USERPROFILE '.go-forensic\scripts'
New-Item -ItemType Directory -Force -Path $scriptDir | Out-Null
$wrapperContent = @'
# go-forensic ios export wrapper. See Bash version for algorithm.
# Usage: ios-export-monitor.ps1 -OutputDir <dir> -BundleIds <id1>,<id2>,... [-Password <pass>]
param(
  [Parameter(Mandatory)][string]$OutputDir,
  [Parameter(Mandatory)][string[]]$BundleIds,
  [string]$Password
)
$logDir = Join-Path $env:USERPROFILE '.go-forensic'
New-Item -ItemType Directory -Force -Path $logDir | Out-Null
$log = Join-Path $logDir 'last-ios-export.log'
'' | Set-Content $log
$gfArgs = @('ios','export')
foreach ($id in $BundleIds) { $gfArgs += @('-k', $id) }
$gfArgs += @('-o', $OutputDir, '-u')
if ($Password) { $gfArgs += @('-p', $Password) }
$proc = Start-Process -FilePath 'go-forensic' -ArgumentList $gfArgs `
  -RedirectStandardOutput $log -RedirectStandardError $log -PassThru -NoNewWindow
for ($i = 0; $i -lt 3; $i++) {
  Start-Sleep -Seconds 10
  if ($proc.HasExited) { Get-Content $log; exit $proc.ExitCode }
  $last = Get-Content -Tail 1 $log -ErrorAction SilentlyContinue
  if ($last -inotmatch 'getting ssh connection') {
    $proc.WaitForExit(); Get-Content $log; exit $proc.ExitCode
  }
}
Stop-Process -Id $proc.Id -Force -ErrorAction SilentlyContinue
Get-Content $log
Write-Output '__AUTH_FAILURE_DETECTED__'
exit 100
'@
$wrapperContent | Set-Content -Path (Join-Path $scriptDir 'ios-export-monitor.ps1') -Encoding UTF8
```

## Output directory derivation (for android export / ios export only)

For every export invocation, the `-o` flag is auto-derived. **Do NOT ask the user for `--output`** — derive it deterministically from the bundle ids:

```
output_dir = <output_root>/<platform><bundle_token>
```

Where:

- `<platform>` is literal `android` or `ios`.
- `<bundle_token>` is derived from the bundle id(s):
  - **Single bundle id:** replace every `.` with `_` → `com.temple` becomes `com_temple`.
  - **Multiple bundle ids:** apply the single-id rule to each, then join with a single `-` separator in the order the user listed them. `com.temp1, com.temp2` becomes `com_temp1-com_temp2`.
- **Platform and token are concatenated with no separator.**
  - 1 bundle: `ios` + `com_temple` → `ioscom_temple`
  - 2 bundles: `ios` + `com_temp1-com_temp2` → `ioscom_temp1-com_temp2`
- If the joined `<bundle_token>` exceeds 120 characters, fall back to `<first_bundle_underscored>_and_<N>more` (e.g. `ioscom_temp1_and_4more`) to keep the path tractable.
- Bundle id case is **preserved** in both `-k` and the directory name (do not lowercase) — Windows filesystems are case-insensitive but case-preserving, so this is safe and avoids surprise renaming.
- Create the directory before invocation:
  - **Bash:** `mkdir -p "<output_dir>"`
  - **PowerShell:** `New-Item -ItemType Directory -Force -Path "<output_dir>" | Out-Null`

## Parameter-elicitation rules

Use the `AskUserQuestion` tool — never guess values, never silently fall back to defaults the user did not state.

1. **Identify the subcommand** from the user's message. If ambiguous (e.g. "导出数据" without saying Android or iOS), ask which platform first.
2. **Identify the target bundle id(s)** from the user's message. If the user did not give any bundle id, ask for it per the "App identification convention" section. Never substitute a Chinese name or pinyin for a bundle id.
3. **Required parameters per subcommand:**
   - `android export`: one or more `-k <bundle_id>`. `--specify-path` is optional — only ask if the user mentioned a specific path.
   - `ios export`: one or more `-k <bundle_id>`. Connection: default to USB-proxy (the tool's `-u` default). Do **not** ask for `--pass` upfront — try without it first, and only prompt the user for the password if the run fails with auth (see "iOS export hang recovery" below). Ask for `--device-id` only if `ios device list` shows >1 device.
   - `ios proxy`: `--local-port`, `--remote-port`. Protocol defaults to tcp; ask only if user mentions UDP.
   - `sqlite search`: `--file` (DB path), `--keywords`.
4. For multi-value `-k`: pass one `-k <value>` per bundle id. Never use `-k "a,b"`.
5. Never expose `--debug` unless the user explicitly asks for debug output.
6. **Never ask for `--output`/`-o`.** It is auto-derived per the section above.

## Execution protocol

1. Run the **Initialization protocol** first (config check / lazy create / wrapper-script install).
2. For export subcommands: derive `<output_dir>` per the convention above and create the directory using the shell-appropriate command.
3. Build the full command string. For N bundle ids:
   - **android export, sqlite, ios device list, ios proxy, version:** invoke `go-forensic` directly (single foreground call). Bash timeout 60000 ms is fine for non-export commands; for `android export` use 600000 ms (10 min) since these can be moderately long.
   - **ios export:** invoke the wrapper script — see "iOS export execution" below. The wrapper has NO upper time limit by design.
4. **Echo the resolved command back to the user** in a fenced block before running, so they can spot a wrong flag or bundle id. For exports, also print `Output directory: <output_dir>` directly above the command. **When echoing a command that contains `-p <password>`, mask the password as `-p ***`** — never print the literal password in chat output.
5. Run via the Bash or PowerShell tool. For `ios export` via the wrapper, set the Bash tool's own timeout to a very large value (e.g. 7200000 ms / 2 h) — the wrapper itself has no internal timeout, so this is just an upper safety bound.
6. After completion, report:
   - Exit code
   - Last ~20 lines of stdout
   - Full stderr if non-empty
   - For exports, the resolved output directory path so the user can navigate to it.
7. If exit code ≠ 0, do **not** silently retry (except for the iOS auth-failure recovery flow defined below). Surface the error and ask the user how to proceed.

### iOS export execution

Always go through the wrapper script installed in Init Step 5. Do **not** invoke `go-forensic ios export` directly.

**Bash mode:**
```bash
bash "$HOME/.go-forensic/scripts/ios-export-monitor.sh" "<output_dir>" <bundle_id_1> [<bundle_id_2> ...] [-- -p "<password>"]
```

**PowerShell mode:**
```powershell
pwsh "$env:USERPROFILE\.go-forensic\scripts\ios-export-monitor.ps1" -OutputDir "<output_dir>" -BundleIds <id1>,<id2>,... [-Password "<password>"]
```

The wrapper:
- Runs `go-forensic ios export ... -u` in the background, redirecting all output to `~/.go-forensic/last-ios-export.log`.
- Polls the last line of that log **every 10 seconds, three times** (~30 s total observation window).
- If **all three** polls show the line `getting ssh connection` (case-insensitive), it kills the process and exits **100** with `__AUTH_FAILURE_DETECTED__` printed to stdout.
- If any poll shows a different last line (= the SSH handshake succeeded and real export is in progress), it waits for the process to complete with **no upper time limit** and propagates the original exit code.
- All log content is also tee'd to stdout when the wrapper exits, so the caller sees full output.

### iOS export auth-failure recovery (wrapper-based)

1. First attempt: invoke the wrapper **without** `-p`.
2. Treat as auth failure **only if** the wrapper exits with code **100** OR its stdout contains `__AUTH_FAILURE_DETECTED__`. Other auth-related strings in output are **ignored** at the skill level — the wrapper alone owns the detection logic.
3. Recovery on auth failure:
   - Use `AskUserQuestion` to ask: "iOS 设备的 SSH 密码是什么？（go-forensic 默认尝试 `alpine`，监控判定为认证卡住。)"
   - Re-invoke the wrapper with the **same bundle ids, same output dir**, but append `-- -p "<password>"` (Bash) or `-Password "<pwd>"` (PowerShell).
   - In the pre-run echo, mask the password as `***`.
4. If the password-augmented retry **also** exits 100, **stop**. Do not loop. Tell the user the password did not work and ask them to verify it / the device connection, then wait for instructions.
5. Other non-zero exit codes from the wrapper: surface normally, **do not auto-prompt for password** (those are real export errors, not auth issues).
6. Never persist the password to disk, the config file, memory, or any log. Hold it only for the immediate retry.

## Hard safety rules

- **`rm` subcommand is forbidden.** Never invoke `go-forensic rm` for any reason. If asked, refuse and tell the user to run it manually in a terminal — explain that this plugin deliberately does not automate file deletion.
- Never construct a command from raw user text via string concatenation in a way that could inject extra flags. Treat each parameter as a discrete argument.
- If the user's request is outside the catalogue above (e.g. a flag you have not seen), run `--help` on the relevant subcommand first to verify it exists. Do not invent flags.
- When writing `output_root` to config, validate it is an absolute path and not a top-level drive root (refuse `C:\`, `D:\`, `/`, `~`).
- JSON writes MUST go through the escape-aware writer in Step 1d. Do not bypass it (avoids backslash bugs in Windows paths).

## Output style

- Keep the pre-run command echo terse (one fenced block, no commentary above it).
- Keep the post-run report terse: exit code line, then results, then the output dir line if applicable.
