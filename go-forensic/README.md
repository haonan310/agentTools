# go-forensic plugin for Claude Code

Wraps the local `go-forensic.exe` binary (resolved from `PATH`) so Claude Code can invoke it via:

- a `/go-forensic` slash command, or
- natural-language keywords in chat: `go-forensic`, `取证`, `安卓导出`, `iOS导出`, `sqlite搜索`, `forensic export`, or phrases combining `提取` with `数据 / app / 应用` (e.g. `提取微信数据`, `提取这个app`).

## Layout

```
go-forensic/
├── .claude-plugin/plugin.json   # plugin manifest
├── skills/go-forensic/SKILL.md  # the actual "how to call it" logic
├── commands/go-forensic.md      # /go-forensic slash command entry
├── hooks/hooks.json             # keyword-trigger reminder hook
├── settings.json                # pre-authorize Bash to run the exe
└── README.md                    # this file
```

## Install

Once published as a plugin marketplace entry:

```
/plugin marketplace add <repo>
/plugin install go-forensic
```

For local testing without a marketplace, copy the `go-forensic/` folder into:

```
%USERPROFILE%\.claude\plugins\go-forensic\
```

Then merge `settings.json` into your `~/.claude/settings.json` (or accept the per-call permission prompt the first time).

## Prerequisites

`go-forensic.exe` must be available on the user's `PATH`. Verify with:

```
go-forensic version
```

If the command is not found, add the directory containing `go-forensic.exe` to your `PATH` environment variable and restart your shell / Claude Code. The plugin intentionally does **not** hard-code an install path — this keeps the same plugin shippable to any teammate regardless of where they put the binary.

### Shell support

The plugin works with both shells Claude Code commonly uses on Windows:

- **Bash** (e.g. Git Bash) — uses `mkdir -p`, `printf` + `sed`-escaped JSON writer
- **PowerShell** — uses `New-Item`, `ConvertTo-Json` (no escape pitfalls)

The skill auto-detects the active shell once per session via `echo "$SHELL"` and picks the matching command set. No manual configuration needed.

## First-run initialization

The first time you trigger any export, the plugin will ask for a **default export root directory** (an absolute path, e.g. `D:\forensic-data`). Your answer is saved to:

```
%USERPROFILE%\.go-forensic\config.json
```

After that, every export is automatically written to:

```
<your-export-root>\<platform><bundle_token>\
```

Where `<bundle_token>` is the app's **bundle id / package name** with every `.` replaced by `_`. For multiple bundle ids, each is underscored and joined by `-`. Examples:

- Single app, iOS: `com.tencent.xin` → `D:\forensic-data\ioscom_tencent_xin\`
- Single app, Android: `com.tencent.mm` → `D:\forensic-data\androidcom_tencent_mm\`
- Two apps at once, iOS: `com.temp1, com.temp2` → `D:\forensic-data\ioscom_temp1-com_temp2\`

You always identify target apps by their **bundle id / package name**, not by Chinese name or pinyin. If you say "提取微信数据" without giving the bundle id, the plugin will ask you for it (e.g. `com.tencent.mm`).

Multiple bundle ids in one request are passed as repeated `-k` flags in a single invocation — all their data lands in the same joined subdirectory.

To change the default root later, just say "改默认导出目录" or "reset go-forensic config" in chat.

## Safety notes

- The `go-forensic rm` subcommand is **deliberately not exposed** by this plugin. The skill will refuse to invoke it and ask the user to run it manually in a terminal if they really need to.
- The skill never invents flags — for unknown flags it runs `--help` first.
- `--debug` is never added unless the user asks.
