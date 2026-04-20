# WhatsApp MCP Server — Architecture & Flow

## Overview

This document explains the full implementation of the WhatsApp MCP Server integrated with GitHub Copilot. It allows you to send WhatsApp messages directly from Copilot Chat using natural language prompts.

---

## Components

| Component | Location | Role |
|---|---|---|
| **mcp.json** | `%APPDATA%\Code\User\mcp.json` | Registers the MCP server with GitHub Copilot |
| **whatsapp-mcp-server** (Python) | `whatsapp-mcp-server/main.py` | MCP protocol layer — exposes tools like `send_message` to Copilot |
| **whatsapp-bridge** (Go) | `whatsapp-bridge/whatsapp-bridge.exe` | HTTP server on `localhost:8080` — talks directly to WhatsApp Web API |
| **SQLite DB** | `whatsapp-bridge/store/whatsapp.db` | Stores the WhatsApp session/device credentials |

---

## Request Flow

```
You (Copilot Chat)
       │
       │  "send 'build and run project' to 9412303645"
       ▼
GitHub Copilot (reads mcp.json)
       │
       │  spawns process via stdio
       ▼
whatsapp-mcp-server/main.py  (Python MCP Server)
       │
       │  HTTP POST http://localhost:8080/api/send
       │  { "recipient": "9412303645", "message": "build and run project" }
       ▼
whatsapp-bridge.exe  (Go HTTP Server on :8080)
       │
       │  uses whatsmeow library
       ▼
WhatsApp Web Servers (wss://web.whatsapp.com)
       │
       ▼
Recipient's WhatsApp Phone (9412303645)
```

---

## MCP Configuration (`mcp.json`)

```json
{
  "servers": {
    "whatsapp-mcp": {
      "type": "stdio",
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "C:\\Users\\jamod\\whatsapp-mcp\\whatsapp-mcp-server",
        "main.py"
      ]
    }
  }
}
```

- **type: stdio** — Copilot launches the Python server as a child process and communicates via stdin/stdout using the MCP protocol.
- **uv run** — Uses the `uv` Python package manager to run `main.py` inside the project's virtual environment.

---

## What We Built — Step by Step

### Step 1: MCP Configuration
`mcp.json` was already configured. GitHub Copilot reads this file and automatically spawns the Python MCP server when a WhatsApp tool is invoked.

### Step 2: Install GCC (required for CGO/sqlite3)
The Go bridge depends on `go-sqlite3` which requires CGO (C bindings). GCC was not installed, so we installed **WinLibs MinGW GCC** via winget:

```powershell
winget install --id BrechtSanders.WinLibs.POSIX.UCRT
```

GCC installed to:
```
C:\Users\jamod\AppData\Local\Microsoft\WinGet\Packages\
  BrechtSanders.WinLibs.POSIX.UCRT_Microsoft.Winget.Source_8wekyb3d8bbwe\
  mingw64\bin\gcc.exe
```

### Step 3: Build the WhatsApp Bridge with CGO

```powershell
$env:PATH = "<gcc-bin-path>;$env:PATH"
$env:CGO_ENABLED = "1"
Set-Location "C:\Users\jamod\whatsapp-mcp\whatsapp-bridge"
go build -o whatsapp-bridge.exe .
```

### Step 4: Update whatsmeow Library
The bundled `whatsmeow` version (March 2025) was rejected by WhatsApp with a `405 Client Outdated` error. Updated to the latest version:

```powershell
go get go.mau.fi/whatsmeow@latest
go get go.mau.fi/util/dbutil@v0.9.8
go mod tidy
```

### Step 5: Fix API Breaking Changes
The new whatsmeow API requires `context.Context` as the first argument in several calls. Five fixes were applied to `main.go`:

| Original Call | Fixed Call |
|---|---|
| `client.Download(downloader)` | `client.Download(context.Background(), downloader)` |
| `sqlstore.New("sqlite3", ...)` | `sqlstore.New(context.Background(), "sqlite3", ...)` |
| `container.GetFirstDevice()` | `container.GetFirstDevice(context.Background())` |
| `client.GetGroupInfo(jid)` | `client.GetGroupInfo(context.Background(), jid)` |
| `client.Store.Contacts.GetContact(jid)` | `client.Store.Contacts.GetContact(context.Background(), jid)` |

### Step 6: Start the Bridge & Scan QR Code

```powershell
& "C:\Users\jamod\whatsapp-mcp\whatsapp-bridge\whatsapp-bridge.exe"
```

On first run, a QR code is printed in the terminal.

---

## Authentication Flow

```
First Run (One-Time Setup):
  ┌─────────────────────────────────────────────────┐
  │  1. Start whatsapp-bridge.exe                   │
  │  2. QR code printed in terminal                 │
  │  3. Open WhatsApp on phone                      │
  │     Settings → Linked Devices → Link a Device   │
  │  4. Scan the QR code                            │
  │  5. Session saved to store/whatsapp.db          │
  └─────────────────────────────────────────────────┘

Subsequent Runs:
  ┌─────────────────────────────────────────────────┐
  │  1. Start whatsapp-bridge.exe                   │
  │  2. Reads session from store/whatsapp.db        │
  │  3. Connects directly — no QR needed            │
  └─────────────────────────────────────────────────┘
```

---

## Sending a Message via Copilot

Once the bridge is running and authenticated, you can prompt Copilot:

> *"Send 'build and run project' to 9412303645 via WhatsApp"*

Copilot calls the `send_message` MCP tool → Python server → Go bridge → WhatsApp.

---

## Prerequisites Summary

| Requirement | Version / Notes |
|---|---|
| Go | 1.24+ |
| Python + uv | For the MCP server |
| GCC (WinLibs) | Required for CGO/sqlite3 on Windows |
| whatsmeow | `v0.0.0-20260416104156` or later |
| WhatsApp account | For QR code scan (one-time) |
