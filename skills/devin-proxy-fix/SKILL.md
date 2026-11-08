---
name: devin-proxy-fix
description: Fix "Unable to connect to Devin", a degraded/empty model picker, or `/model` returning "Unknown command" in Devin Desktop on Windows — usually caused by a local proxy (Clash/V2Ray etc.) that part of the app's traffic bypasses.
triggers:
  - user
  - model
---

# Devin Desktop connectivity & proxy fix (Windows)

Use this when any of these symptoms appear in Devin Desktop:

- Banner: **"Unable to connect to Devin. Some AI features may not work."**
- The model picker in a **Local** session shows only one model (e.g. `sonnet`) when the account should have the full catalog
- Typing `/model` replies `Unknown command: /model`
- The same account works normally on another machine

## Key facts (do not skip)

- Local sessions in Devin Desktop run through **ACP** (`devin acp`). `/model` is NOT in the ACP slash-command set — it only exists in the terminal REPL (`devin` CLI) and in cloud sessions before the first message. `Unknown command: /model` inside Desktop is expected; the model is changed via the client's own session-config controls.
- The account's real model catalog is server-side, cached at `%LOCALAPPDATA%\devin\cli\model_configs_v6.<id>.bin` (JSON envelope, `payload` = base64 protobuf). If the picker shows a subset, the catalog may be fine — the UI is just degraded.
- When parts of the app bypass the proxy, registry/model fetches time out and the UI falls back to a minimal state (one model, connect banner).

## Step 1 — Confirm a local proxy exists

```powershell
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" |
  Select-Object ProxyEnable, ProxyServer
```

`ProxyEnable=1` with something like `127.0.0.1:7897` means a local proxy (Clash/V2Ray) is active.

## Step 2 — Confirm the failing calls in the logs

Check the newest session log dir:

```
%APPDATA%\devin\logs\<timestamp>\window1\exthost\codeium.windsurf\Devin ACP.log
```

Signature lines:

- `Error fetching registry from server: ConnectError: [deadline_exceeded] the operation timed out`
- `acp feature flags (timed out after 5000ms waiting for auth state)`

Meanwhile `window1\devinWebapp.log` shows `wss://app.devin.ai/api/acp/live` connecting fine — that's the classic split: Electron/Chromium traffic honors the Windows proxy, the extension's ConnectRPC calls do not.

## Step 3 — Apply the fix (both parts)

A. Add proxy settings to Desktop user settings `%APPDATA%\devin\User\settings.json`:

```json
"http.proxy": "http://127.0.0.1:7897",
"http.proxyStrictSSL": false
```

B. Set user-level env vars so Node/gRPC clients pick up the proxy:

```powershell
[Environment]::SetEnvironmentVariable("HTTPS_PROXY","http://127.0.0.1:7897","User")
[Environment]::SetEnvironmentVariable("HTTP_PROXY","http://127.0.0.1:7897","User")
[Environment]::SetEnvironmentVariable("ALL_PROXY","socks5://127.0.0.1:7897","User")
```

Adjust the port if the user's proxy differs (check Step 1 output; Clash's mixed port serves both HTTP and SOCKS).

## Step 4 — Fully restart Devin Desktop

Closing the window is not enough: right-click the **system tray icon → Quit**, then relaunch. Env vars are only inherited at process start.

## Verify

- The "Unable to connect" banner is gone
- The model picker in a new Local session lists the full catalog (Opus, GPT, Gemini, SWE, Adaptive, …)
- `Devin ACP.log` no longer shows `deadline_exceeded`

## Alternatives and caveats

- **TUN/global mode** in the proxy client (Clash Verge/Mihomo etc.) transparently captures all traffic and needs no per-app config — prefer it when available.
- The env vars are global: if the user later stops running the proxy, `HTTPS_PROXY`/`HTTP_PROXY`/`ALL_PROXY` must be cleared or every CLI tool (git, npm, …) will time out.
- The bundled CLI lives at `%LOCALAPPDATA%\Programs\Devin\resources\app\extensions\windsurf\devin\bin\devin.exe`; add that dir to user `Path` to use `devin` in any terminal (`/model`, `devin models` work there — needs `devin auth login` first).
- `credentials.toml` in `%APPDATA%\devin` is written by Desktop sign-in; the standalone CLI may report "Not logged in" until its own `devin auth login` completes — this is not a fault.
