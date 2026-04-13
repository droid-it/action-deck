# Action Deck — Claude Code Instructions

This document tells Claude how to work in this repository. Read it before writing any code.

---

## What This Project Is

A macOS-native Stream Deck clone consisting of three components:

1. **Swift macOS app** — SwiftUI settings/admin UI with an embedded Hummingbird 2 HTTP + WebSocket server that executes macOS actions.
2. **Svelte web client** — the button grid UI served locally; users open it in any browser tab.
3. **MCP server** — exposes the action/profile API as MCP tools so LLM agents can trigger automations.

This is also a **learning project for an Android/Kotlin developer picking up Swift**, so code must be clear, well-commented, and explicit. Favour readability over idiomatic cleverness.

---

## Project Structure

```
action-deck/
├── Package.swift                  # Swift Package Manager manifest
├── Sources/
│   ├── ActionDeckCore/            # Shared domain types, data model
│   │   ├── Models/                # Profile, Button, Action, AppConfig
│   │   └── Executor/              # Action execution (shell, AppleScript, etc.)
│   ├── ActionDeckServer/          # Hummingbird HTTP + WebSocket server
│   │   ├── App.swift              # Server entry point and route registration
│   │   ├── Routes/                # REST route handlers
│   │   └── WebSocket/             # WS connection manager + push logic
│   ├── ActionDeckApp/             # SwiftUI macOS app (menu bar + settings)
│   └── ActionDeckMCP/             # MCP server (stdio transport)
├── client/                        # Svelte 5 web client
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
│       ├── App.svelte
│       ├── components/
│       └── lib/
├── Tests/                         # Swift tests
│   └── ActionDeckCoreTests/
├── data/
│   └── default-config.json        # Seeded on first run
├── docs/
│   └── ARCHITECTURE.md            # Full architecture and roadmap
├── PLAN.md                        # Earlier planning notes (historical reference)
└── README.md
```

---

## How to Build and Run

### Requirements

- Xcode 16+ / Swift 6
- Node.js 20+ and npm (for the web client)
- macOS 14 Sonoma or later

### Swift backend

```bash
# Build
swift build

# Run the server (API on :9900)
swift run ActionDeckServer

# Run tests
swift test
```

### Svelte web client

```bash
cd client
npm install
npm run dev       # dev server on :5173 with HMR
npm run build     # production build into client/dist/
```

### Running both together

Open two terminal tabs: one for `swift run ActionDeckServer`, one for `cd client && npm run dev`. Then open `http://localhost:5173` in a browser.

---

## Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Swift | 6.0, strict concurrency | Primary backend language |
| Hummingbird 2 | HTTP + WebSocket | Async/await native, no Vapor overhead |
| SwiftUI | macOS 14+ | Admin/settings panel, menu bar app |
| Svelte 5 | TypeScript, Vite | Web UI |
| Storage | JSON on disk | `~/.action-deck/config.json` |
| MCP | Swift, stdio transport | LLM integration |

---

## Code Style

### Swift

- **Readability first.** This is a learning project. A longer, explicit version of code is always preferred over a shorter, clever one.
- **Comment non-obvious Swift patterns.** If something would confuse an Android/Kotlin developer, add a brief comment explaining why Swift works this way.
- **Avoid custom operators, complex generics, and protocol-heavy abstractions** unless they materially simplify something.
- Use `async/await` throughout; avoid completion handlers.
- Prefer value types (`struct`) over classes where practical; use `actor` for shared mutable state.
- Name things clearly: `executeAction(_ action: Action) async throws -> ActionResult` over `exec(_:)`.
- Keep files focused. One type or closely related set of functions per file.

### Svelte / TypeScript

- Use Svelte 5 runes syntax (`$state`, `$derived`, `$effect`).
- One component per file; keep components small.
- TypeScript strict mode. Shared types live in `src/lib/types.ts`.

### General

- Do not add docstrings or comments to code you did not touch.
- Do not add error handling for scenarios that cannot happen.
- Do not introduce abstractions "for future flexibility" — only add what is needed now.

---

## Testing

- Unit-test all action executors in `ActionDeckCoreTests`.
- Test JSON encoding/decoding for every model type.
- Use `XCTest` (no third-party test frameworks).
- Do not mock the file system in config-store tests — write to a temp directory and clean up.
- Swift tests run with `swift test`. No special setup required.

---

## Branch Naming

```
feature/<short-description>     # new functionality
fix/<short-description>         # bug fixes
docs/<short-description>        # documentation only
refactor/<short-description>    # code changes with no behaviour change
```

Branches target `master`. PRs should be small and focused.

---

## Key Files to Know

| File | What it does |
|---|---|
| `Sources/ActionDeckCore/Models/` | Single source of truth for all domain types |
| `Sources/ActionDeckServer/App.swift` | Server bootstrap and route registration |
| `Sources/ActionDeckServer/Routes/` | One file per REST resource group |
| `Sources/ActionDeckCore/Executor/` | Each action type has its own executor |
| `data/default-config.json` | Seeded on first run; edit for a good default experience |
| `docs/ARCHITECTURE.md` | Comprehensive architecture reference — read this first |

---

## Notes for Future Claude Sessions

- The tech stack is **Swift + Hummingbird 2**, not Node.js/Express. `PLAN.md` contains an earlier Node.js design that was superseded.
- The developer is an experienced **Android/Kotlin engineer** learning Swift. Frame explanations in those terms when helpful. Kotlin coroutines ≈ Swift async/await; `data class` ≈ `struct`; `sealed class` ≈ `enum` with associated values; `ViewModel` ≈ `@Observable` class.
- Prefer **baby steps**: small, compilable, testable commits over large scaffolding drops.
- When implementing Swift concurrency features, add an inline comment explaining the concurrency model decision (why `actor` vs `@MainActor` vs plain `async`).
