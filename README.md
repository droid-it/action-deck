# Action Deck

A macOS Stream Deck clone you run locally. Press buttons in a browser tab to trigger macros, launch apps, run shell commands, execute AppleScripts, or open URLs — all driven by a native Swift backend running on your Mac.

This is also a **Swift learning project** built by an Android/Kotlin developer, so the code prioritises clarity and explicit patterns over idiomatic brevity.

---

## Features

- **Button grid UI** — 5×3 (configurable) grid of labelled, coloured buttons served as a Svelte web app
- **5 action types**
  - Keyboard Maestro macro trigger
  - Open / quit an application
  - Shell / zsh command
  - Inline AppleScript
  - Open URL or app deeplink (VS Code, Slack, Raycast, …)
- **Profiles** — separate button layouts per context (Coding, Meetings, DevOps, …)
- **Folder navigation** — group related buttons in a sub-grid; click to drill in, breadcrumb to go back
- **Webhook profile switching** — `POST /api/profiles/:id/activate` lets Alfred, Keyboard Maestro, macOS Shortcuts, or any HTTP client switch the active profile
- **WebSocket push** — all open browser tabs update instantly when a profile switches or an action completes
- **MCP server** — exposes actions and profiles as tools so LLM agents (Claude, etc.) can trigger automations

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Backend** | Swift 6, Swift Package Manager |
| **HTTP server** | [Hummingbird 2](https://github.com/hummingbird-project/hummingbird) (embedded in the macOS app) |
| **macOS UI** | SwiftUI (admin / settings panel) |
| **Web client** | Svelte 5, TypeScript, Vite |
| **Storage** | JSON file at `~/.action-deck/config.json` |
| **LLM integration** | MCP server (Swift, stdio transport) |

---

## Getting Started

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full architecture plan, data model, API reference, and a baby-step implementation roadmap.

Build requirements: Xcode 16+, Swift 6, Node.js 20+ (for the web client).

```bash
# Build and run the Swift backend (from repo root)
swift build
swift run ActionDeckServer

# In a second terminal — build and serve the web client
cd client
npm install
npm run dev
```

The web UI will be available at `http://localhost:5173` and the API at `http://localhost:9900`.
