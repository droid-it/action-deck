# Action Deck — Architecture Reference

> **Audience:** primarily the developer (an Android/Kotlin engineer learning Swift) and Claude Code sessions working in this repo.  
> **Goal:** be the single reference for how Action Deck is designed, why decisions were made, and how to implement it step-by-step.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Tech Stack Rationale](#2-tech-stack-rationale)
3. [Component Architecture](#3-component-architecture)
4. [Data Model](#4-data-model)
5. [REST API Design](#5-rest-api-design)
6. [WebSocket Protocol](#6-websocket-protocol)
7. [Web Client Design](#7-web-client-design)
8. [SwiftUI Admin App](#8-swiftui-admin-app)
9. [Action Execution Model](#9-action-execution-model)
10. [Profile & Folder System](#10-profile--folder-system)
11. [MCP Integration](#11-mcp-integration)
12. [File Organisation](#12-file-organisation)
13. [Swift for Android Developers](#13-swift-for-android-developers)
14. [Baby-Step Implementation Roadmap](#14-baby-step-implementation-roadmap)
15. [Swift Concepts by Phase](#15-swift-concepts-by-phase)

---

## 1. System Overview

Action Deck has three independent processes that work together:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         macOS Host                                   │
│                                                                       │
│  ┌─────────────────────┐     ┌──────────────────────────────────┐   │
│  │   ActionDeckApp     │     │      ActionDeckServer            │   │
│  │   (SwiftUI)         │     │   (Hummingbird 2, port 9900)     │   │
│  │                     │     │                                  │   │
│  │  • Menu bar icon    │────▶│  • REST API  /api/...            │   │
│  │  • Settings panel   │     │  • WebSocket /ws                 │   │
│  │  • Start/stop server│     │  • Action executor               │   │
│  │                     │     │  • Config store (JSON on disk)   │   │
│  └─────────────────────┘     └─────────────┬────────────────────┘   │
│                                             │ child_process           │
│                                             ▼                         │
│                                  ┌──────────────────┐               │
│                                  │   macOS Shell     │               │
│                                  │  osascript        │               │
│                                  │  open -a          │               │
│                                  │  /bin/zsh -c      │               │
│                                  └──────────────────┘               │
│                                                                       │
│  ┌─────────────────────┐                                             │
│  │  ActionDeckMCP      │                                             │
│  │  (stdio transport)  │◀──── LLM Agent (Claude, etc.)              │
│  │                     │                                             │
│  │  • execute_action   │────▶  calls ActionDeckServer REST          │
│  │  • list_profiles    │                                             │
│  │  • switch_profile   │                                             │
│  └─────────────────────┘                                             │
└─────────────────────────────────────────────────────────────────────┘

                    ▲ HTTP + WebSocket
                    │
          ┌─────────────────────┐
          │   Browser Tab(s)    │
          │   (Svelte 5 UI)     │
          │   localhost:5173    │
          └─────────────────────┘
```

**Key design decisions:**

- The HTTP server is embedded in/alongside the macOS app (not a separate daemon) so it starts and stops with the app.
- The web client is a separate Vite dev-server in development, and can be bundled into the Swift app as static files for distribution.
- The MCP server is a separate executable that calls the local REST API — this keeps it thin and avoids duplicating business logic.

---

## 2. Tech Stack Rationale

| Layer | Choice | Why not the alternative |
|---|---|---|
| **HTTP server** | Hummingbird 2 | Vapor adds too much boilerplate; Hummingbird 2 is async-native, lightweight, and simple to embed |
| **Language** | Swift 6 | Native macOS process execution; strong type system; this is the learning goal |
| **Web UI** | Svelte 5 | Minimal boilerplate; no virtual DOM overhead; runes are similar to Kotlin state flows |
| **Storage** | JSON file | Human-readable; easy to back up; `~100 KB` even with 10 profiles; no database overhead |
| **Concurrency** | Swift `async/await` | Natural mapping from Kotlin coroutines; actors replace manual locking |
| **MCP transport** | stdio | Standard for Claude Code; simplest to implement and debug |

---

## 3. Component Architecture

### 3.1 ActionDeckCore (Library Target)

The shared domain layer. Consumed by both the server and the MCP executable.

```
ActionDeckCore/
├── Models/
│   ├── Action.swift          # Action enum with associated values
│   ├── Button.swift          # Button struct
│   ├── Folder.swift          # Folder struct
│   ├── Profile.swift         # Profile struct
│   └── AppConfig.swift       # Root config type + Codable conformances
├── Executor/
│   ├── ActionExecutor.swift  # Protocol + dispatcher
│   ├── ShellExecutor.swift
│   ├── AppleScriptExecutor.swift
│   ├── AppExecutor.swift     # open / quit apps
│   ├── KeyboardMaestroExecutor.swift
│   └── OpenURLExecutor.swift
└── Storage/
    └── ConfigStore.swift     # Read/write ~/.action-deck/config.json
```

### 3.2 ActionDeckServer (Executable Target)

The Hummingbird 2 HTTP + WebSocket server. Depends on ActionDeckCore.

```
ActionDeckServer/
├── App.swift                 # Application entry, route registration
├── Routes/
│   ├── ConfigRoutes.swift    # GET /api/config
│   ├── ProfileRoutes.swift   # CRUD + activate webhook
│   ├── ButtonRoutes.swift    # Button CRUD within profiles/folders
│   ├── FolderRoutes.swift    # Folder CRUD
│   └── ActionRoutes.swift    # POST /api/actions/execute
└── WebSocket/
    └── ConnectionManager.swift  # Track open WS connections, broadcast
```

### 3.3 ActionDeckApp (SwiftUI macOS App)

Thin SwiftUI wrapper. Embeds or supervises the server process.

```
ActionDeckApp/
├── ActionDeckApp.swift       # @main entry, menu bar setup
├── MenuBarView.swift         # NSStatusItem / MenuBarExtra
└── SettingsView.swift        # Server port, theme, profile list
```

### 3.4 ActionDeckMCP (Executable Target)

Standalone MCP server. Reads its own binary path; talks to the local REST API.

```
ActionDeckMCP/
├── main.swift                # stdio MCP loop
├── Tools/
│   ├── ExecuteActionTool.swift
│   ├── ListProfilesTool.swift
│   └── SwitchProfileTool.swift
└── APIClient.swift           # Thin wrapper over URLSession → localhost:9900
```

### 3.5 Svelte Web Client

```
client/
├── src/
│   ├── App.svelte
│   ├── lib/
│   │   ├── types.ts          # Mirror of Swift data model
│   │   ├── api.ts            # REST fetch wrappers
│   │   └── websocket.ts      # WS connection + event emitter
│   └── components/
│       ├── ButtonGrid.svelte
│       ├── ButtonCell.svelte
│       ├── EmptyCell.svelte
│       ├── ButtonEditor.svelte   # Modal to create/edit a button
│       ├── ActionEditor.svelte   # Subform for each action type
│       ├── ProfileSwitcher.svelte
│       └── FolderBreadcrumb.svelte
└── vite.config.ts
```

---

## 4. Data Model

### 4.1 Swift Types

```swift
// ── Actions ──────────────────────────────────────────────────────────

// In Swift, an enum with associated values is the idiomatic way to model
// a "sealed class hierarchy" (the Kotlin equivalent of sealed class Action).
enum Action: Codable {
    case keyboardMaestro(macroName: String)
    case openApp(appName: String)
    case closeApp(appName: String)
    case shell(command: String, workingDirectory: String?, timeoutSeconds: Int?)
    case applescript(script: String)
    case openURL(url: String)
}

// ── Layout ────────────────────────────────────────────────────────────

// struct = value type, like a Kotlin data class
struct Button: Codable, Identifiable {
    var id: UUID
    var position: Int           // grid index 0-based
    var label: String?
    var icon: String?           // emoji, URL, or data URI
    var backgroundColor: String?
    var textColor: String?
    var action: Action?
    var folderId: UUID?         // if set, tapping opens this folder
}

struct Folder: Codable, Identifiable {
    var id: UUID
    var name: String
    var buttons: [Button]
    var columns: Int?
    var rows: Int?
}

struct Profile: Codable, Identifiable {
    var id: UUID
    var name: String
    var buttons: [Button]
    var folders: [Folder]
    var columns: Int            // default 5
    var rows: Int               // default 3
}

struct AppConfig: Codable {
    var version: Int            // = 1
    var activeProfileId: UUID
    var profiles: [Profile]
    var settings: Settings

    struct Settings: Codable {
        var port: Int           // default 9900
        var theme: Theme

        enum Theme: String, Codable { case dark, light }
    }
}
```

### 4.2 JSON Schema (on disk)

```json
{
  "version": 1,
  "activeProfileId": "11111111-0000-0000-0000-000000000001",
  "settings": { "port": 9900, "theme": "dark" },
  "profiles": [
    {
      "id": "11111111-0000-0000-0000-000000000001",
      "name": "Default",
      "columns": 5,
      "rows": 3,
      "folders": [],
      "buttons": [
        {
          "id": "22222222-0000-0000-0000-000000000001",
          "position": 0,
          "label": "Open Xcode",
          "icon": "🛠️",
          "action": {
            "type": "openApp",
            "appName": "Xcode"
          }
        },
        {
          "id": "22222222-0000-0000-0000-000000000002",
          "position": 1,
          "label": "Run tests",
          "icon": "🧪",
          "action": {
            "type": "shell",
            "command": "swift test",
            "workingDirectory": "~/PersonalProjects/action-deck"
          }
        }
      ]
    }
  ]
}
```

> **Codable note:** Swift's `Codable` for enums with associated values uses a `"type"` discriminator key by default when you implement custom `encode`/`decode`. Add a `CodingKeys` enum and a `type` string so the JSON is readable and matches the TypeScript client types.

---

## 5. REST API Design

All routes are under `/api`. The server binds to `127.0.0.1:9900` only.

### 5.1 Endpoint Reference

| Method | Path | Description |
|---|---|---|
| **Config** | | |
| `GET` | `/api/config` | Full `AppConfig` snapshot |
| **Profiles** | | |
| `GET` | `/api/profiles` | Array of `Profile` |
| `POST` | `/api/profiles` | Create profile; body: `{ name, columns?, rows? }` |
| `PUT` | `/api/profiles/:id` | Replace profile |
| `DELETE` | `/api/profiles/:id` | Delete profile (must not be the active one) |
| `POST` | `/api/profiles/:id/activate` | Switch active profile (webhook endpoint) |
| **Buttons (top-level)** | | |
| `PUT` | `/api/profiles/:pId/buttons/:btnId` | Upsert button in profile root grid |
| `DELETE` | `/api/profiles/:pId/buttons/:btnId` | Clear button slot |
| **Folders** | | |
| `POST` | `/api/profiles/:pId/folders` | Create folder |
| `PUT` | `/api/profiles/:pId/folders/:fId` | Update folder |
| `DELETE` | `/api/profiles/:pId/folders/:fId` | Delete folder |
| **Buttons (in folder)** | | |
| `PUT` | `/api/profiles/:pId/folders/:fId/buttons/:btnId` | Upsert button in folder |
| `DELETE` | `/api/profiles/:pId/folders/:fId/buttons/:btnId` | Clear button slot in folder |
| **Actions** | | |
| `POST` | `/api/actions/execute` | Execute action `{ "action": { "type": "...", ... } }` |

### 5.2 Response Envelope

```json
// Success
{ "ok": true, "data": { ... } }

// Error
{ "ok": false, "error": "Profile not found" }
```

### 5.3 Hummingbird Route Registration (example)

```swift
// In App.swift
let router = Router()
let api = router.group("api")

// Register all route groups
ProfileRoutes(configStore: configStore, wsManager: wsManager).register(on: api)
ActionRoutes(executor: executor).register(on: api)
// ...

let app = Application(router: router, configuration: .init(address: .hostname("127.0.0.1", port: 9900)))
try await app.runService()
```

```swift
// In ProfileRoutes.swift
struct ProfileRoutes {
    let configStore: ConfigStore
    let wsManager: ConnectionManager

    func register(on group: RouterGroup<some RequestContext>) {
        group.get("profiles") { request, context in
            let config = try await configStore.read()
            return Response.json(config.profiles)
        }

        group.post("profiles/:id/activate") { request, context in
            let id = try context.parameters.require("id", as: UUID.self)
            try await configStore.setActiveProfile(id: id)
            await wsManager.broadcast(.profileSwitched(profileId: id))
            return Response.json(["ok": true])
        }
    }
}
```

---

## 6. WebSocket Protocol

One WebSocket endpoint: `ws://localhost:9900/ws`

### 6.1 Server → Client Messages

```typescript
// Sent when a profile is switched (by webhook or UI)
{ "type": "profile:switched", "profileId": "uuid" }

// Sent when an action starts executing
{ "type": "action:start", "buttonId": "uuid" }

// Streamed output from shell/AppleScript actions
{ "type": "action:output", "buttonId": "uuid", "data": "..." }

// Sent on successful completion
{ "type": "action:complete", "buttonId": "uuid", "exitCode": 0 }

// Sent on failure
{ "type": "action:error", "buttonId": "uuid", "error": "..." }
```

### 6.2 Client → Server Messages

```typescript
// Request immediate action execution via WebSocket (alternative to REST)
{ "type": "action:execute", "buttonId": "uuid", "action": { ... } }
```

### 6.3 Swift WebSocket Manager

```swift
// actor ensures thread-safe access to the connection set
// (actor = Kotlin's StateFlow-protected ViewModel, but enforced by the compiler)
actor ConnectionManager {
    private var connections: Set<WebSocketConnection> = []

    func add(_ connection: WebSocketConnection) {
        connections.insert(connection)
    }

    func remove(_ connection: WebSocketConnection) {
        connections.remove(connection)
    }

    func broadcast(_ message: ServerMessage) async {
        let data = try! JSONEncoder().encode(message)
        for connection in connections {
            try? await connection.send(.text(String(data: data, encoding: .utf8)!))
        }
    }
}
```

---

## 7. Web Client Design

### 7.1 State Architecture

The Svelte client mirrors the config structure. Svelte 5 runes replace stores:

```typescript
// src/lib/state.svelte.ts

// $state() is like Kotlin's mutableStateOf() or MutableStateFlow
let config = $state<AppConfig | null>(null)
let currentFolderPath = $state<string[]>([])  // breadcrumb stack of folder IDs
let activeProfileId = $state<string | null>(null)

// $derived() is like Kotlin's derivedStateOf()
let activeProfile = $derived(
    config?.profiles.find(p => p.id === activeProfileId) ?? null
)
```

### 7.2 Component Hierarchy

```
App.svelte
├── TopBar.svelte
│   ├── ProfileSwitcher.svelte   (dropdown of profiles)
│   └── FolderBreadcrumb.svelte  (back navigation)
└── ButtonGrid.svelte
    └── ButtonCell.svelte        (one per grid slot)
        └── [ButtonEditor modal] (shown on long-press / right-click)
            └── ActionEditor.svelte (form fields per action type)
```

### 7.3 WebSocket Integration

```typescript
// src/lib/websocket.ts
export function createWSClient(url: string) {
    let ws = new WebSocket(url)

    ws.onmessage = (event) => {
        const message = JSON.parse(event.data)
        switch (message.type) {
            case 'profile:switched':
                activeProfileId = message.profileId
                break
            case 'action:start':
                setButtonState(message.buttonId, 'running')
                break
            case 'action:complete':
            case 'action:error':
                setButtonState(message.buttonId, 'idle')
                break
        }
    }

    ws.onclose = () => {
        // Reconnect after 2 seconds
        setTimeout(() => createWSClient(url), 2000)
    }
}
```

---

## 8. SwiftUI Admin App

The macOS app is a menu bar app (`LSUIElement = YES` in Info.plist so it doesn't appear in the Dock).

### 8.1 Structure

```swift
@main
struct ActionDeckApp: App {
    @State private var server = ServerController()

    var body: some Scene {
        // MenuBarExtra is a macOS 13+ API for menu bar icons
        MenuBarExtra("Action Deck", systemImage: "rectangle.grid.3x2") {
            MenuBarView(server: server)
        }
        Settings {
            SettingsView(server: server)
        }
    }
}
```

### 8.2 ServerController

```swift
// @Observable is the Swift 5.9+ equivalent of Kotlin's @ViewModel with
// StateFlow — changes automatically publish to any SwiftUI view that reads them.
@Observable
class ServerController {
    var isRunning = false
    var port = 9900
    private var serverTask: Task<Void, Never>?

    func start() {
        serverTask = Task {
            isRunning = true
            try? await runServer(port: port)
            isRunning = false
        }
    }

    func stop() {
        serverTask?.cancel()
        serverTask = nil
    }
}
```

---

## 9. Action Execution Model

### 9.1 Executor Protocol

```swift
// Protocol = Kotlin interface
protocol ActionExecutorProtocol {
    func execute(_ action: Action) async throws -> ActionResult
}

struct ActionResult {
    var output: String
    var exitCode: Int
}
```

### 9.2 Dispatcher

```swift
struct ActionExecutor: ActionExecutorProtocol {
    func execute(_ action: Action) async throws -> ActionResult {
        switch action {
        case .keyboardMaestro(let macroName):
            return try await KeyboardMaestroExecutor().execute(macroName: macroName)
        case .openApp(let appName):
            return try await AppExecutor().open(appName: appName)
        case .closeApp(let appName):
            return try await AppExecutor().close(appName: appName)
        case .shell(let command, let workingDirectory, let timeout):
            return try await ShellExecutor().execute(command: command, workingDirectory: workingDirectory, timeout: timeout)
        case .applescript(let script):
            return try await AppleScriptExecutor().execute(script: script)
        case .openURL(let url):
            return try await OpenURLExecutor().execute(url: url)
        }
    }
}
```

### 9.3 Shell Executor (annotated)

```swift
struct ShellExecutor {
    func execute(command: String, workingDirectory: String?, timeout: Int?) async throws -> ActionResult {
        // Process is the Swift equivalent of Java's ProcessBuilder
        let process = Process()
        process.executableURL = URL(fileURLWithPath: "/bin/zsh")
        process.arguments = ["-c", command]

        if let wd = workingDirectory {
            // (wd as NSString).expandingTildeInPath handles ~ expansion
            process.currentDirectoryURL = URL(fileURLWithPath: (wd as NSString).expandingTildeInPath)
        }

        // Pipe captures stdout/stderr — like reading from a process stream in Kotlin
        let pipe = Pipe()
        process.standardOutput = pipe
        process.standardError = pipe

        // withCheckedThrowingContinuation bridges a callback API into async/await
        // (similar to suspendCoroutine in Kotlin)
        return try await withCheckedThrowingContinuation { continuation in
            process.terminationHandler = { proc in
                let data = pipe.fileHandleForReading.readDataToEndOfFile()
                let output = String(data: data, encoding: .utf8) ?? ""
                continuation.resume(returning: ActionResult(output: output, exitCode: Int(proc.terminationStatus)))
            }
            do {
                try process.run()
            } catch {
                continuation.resume(throwing: error)
            }
        }
    }
}
```

### 9.4 Keyboard Maestro Executor

```swift
struct KeyboardMaestroExecutor {
    func execute(macroName: String) async throws -> ActionResult {
        // osascript invokes AppleScript; we shell out rather than using
        // the ScriptingBridge framework to keep dependencies minimal
        let script = """
            tell application "Keyboard Maestro Engine"
                do script "\(macroName)"
            end tell
            """
        return try await ShellExecutor().execute(
            command: "osascript -e '\(script)'",
            workingDirectory: nil,
            timeout: nil
        )
    }
}
```

### 9.5 Open URL Executor

```swift
struct OpenURLExecutor {
    func execute(url: String) async throws -> ActionResult {
        // NSWorkspace.shared.open() handles all URL schemes including custom deeplinks
        guard let parsedURL = URL(string: url) else {
            throw ActionError.invalidURL(url)
        }

        // @MainActor required because NSWorkspace is a UI-layer API
        await MainActor.run {
            NSWorkspace.shared.open(parsedURL)
        }
        return ActionResult(output: "Opened \(url)", exitCode: 0)
    }
}
```

---

## 10. Profile & Folder System

### 10.1 Profile Switching Flow

```
External tool (Alfred / Keyboard Maestro / curl)
    │
    │  POST /api/profiles/:id/activate
    ▼
ProfileRoutes.activate()
    │
    ├──▶ ConfigStore.setActiveProfile(id:)   ← writes JSON to disk
    │
    └──▶ ConnectionManager.broadcast(.profileSwitched(profileId:))
              │
              ▼ WebSocket
         All open browser tabs receive { type: "profile:switched", ... }
         and re-render their button grid
```

### 10.2 Folder Navigation (Client-Side)

Folders are a UI concept — the server stores them flat in the profile. The client maintains a `currentFolderPath: string[]` (stack of folder IDs). Clicking a folder button pushes its ID; the back button pops.

```typescript
// Push when user clicks a folder button
function enterFolder(folderId: string) {
    currentFolderPath = [...currentFolderPath, folderId]
}

// Pop when user clicks back
function goBack() {
    currentFolderPath = currentFolderPath.slice(0, -1)
}

// The displayed buttons are always the current folder's buttons
// (or the profile root if the path is empty)
let displayedButtons = $derived(() => {
    if (currentFolderPath.length === 0) return activeProfile?.buttons ?? []
    const folderId = currentFolderPath[currentFolderPath.length - 1]
    return activeProfile?.folders.find(f => f.id === folderId)?.buttons ?? []
})
```

---

## 11. MCP Integration

The MCP server (`ActionDeckMCP`) is a separate Swift executable that uses stdio transport to communicate with an LLM agent. It calls the local REST API to delegate work — it contains no business logic.

### 11.1 Exposed Tools

| Tool | Description | Parameters |
|---|---|---|
| `execute_action` | Execute any action type | `action: Action` (JSON) |
| `list_profiles` | List available profiles | none |
| `switch_profile` | Activate a profile by name or ID | `profile: string` |
| `list_buttons` | List buttons in the active profile | `folderId?: string` |

### 11.2 MCP Tool: execute_action

```swift
// Tool input schema (JSON Schema)
let executeActionSchema: [String: Any] = [
    "type": "object",
    "properties": [
        "action": [
            "type": "object",
            "description": "An Action Deck action to execute",
            "properties": [
                "type": ["type": "string", "enum": ["shell", "applescript", "openApp", "closeApp", "keyboardMaestro", "openURL"]],
                "command": ["type": "string"],       // for shell
                "script": ["type": "string"],        // for applescript
                "appName": ["type": "string"],       // for openApp / closeApp
                "macroName": ["type": "string"],     // for keyboardMaestro
                "url": ["type": "string"]            // for openURL
            ],
            "required": ["type"]
        ]
    ],
    "required": ["action"]
]
```

### 11.3 Stdio Loop (sketch)

```swift
// main.swift
// Read JSON-RPC messages from stdin, write responses to stdout
// This is the MCP stdio transport protocol
while let line = readLine() {
    guard let data = line.data(using: .utf8),
          let request = try? JSONDecoder().decode(MCPRequest.self, from: data) else { continue }

    let response = await handleRequest(request)
    let encoded = try! JSONEncoder().encode(response)
    print(String(data: encoded, encoding: .utf8)!)
    fflush(stdout)
}
```

---

## 12. File Organisation

### 12.1 Full Directory Tree

```
action-deck/
├── Package.swift
├── Sources/
│   ├── ActionDeckCore/
│   │   ├── Models/
│   │   │   ├── Action.swift
│   │   │   ├── Button.swift
│   │   │   ├── Folder.swift
│   │   │   ├── Profile.swift
│   │   │   └── AppConfig.swift
│   │   ├── Executor/
│   │   │   ├── ActionExecutor.swift
│   │   │   ├── ShellExecutor.swift
│   │   │   ├── AppleScriptExecutor.swift
│   │   │   ├── AppExecutor.swift
│   │   │   ├── KeyboardMaestroExecutor.swift
│   │   │   └── OpenURLExecutor.swift
│   │   └── Storage/
│   │       └── ConfigStore.swift
│   ├── ActionDeckServer/
│   │   ├── App.swift
│   │   ├── Routes/
│   │   │   ├── ConfigRoutes.swift
│   │   │   ├── ProfileRoutes.swift
│   │   │   ├── ButtonRoutes.swift
│   │   │   ├── FolderRoutes.swift
│   │   │   └── ActionRoutes.swift
│   │   └── WebSocket/
│   │       └── ConnectionManager.swift
│   ├── ActionDeckApp/
│   │   ├── ActionDeckApp.swift
│   │   ├── ServerController.swift
│   │   ├── MenuBarView.swift
│   │   └── SettingsView.swift
│   └── ActionDeckMCP/
│       ├── main.swift
│       ├── MCPProtocol.swift
│       ├── APIClient.swift
│       └── Tools/
│           ├── ExecuteActionTool.swift
│           ├── ListProfilesTool.swift
│           └── SwitchProfileTool.swift
├── Tests/
│   └── ActionDeckCoreTests/
│       ├── ActionCodableTests.swift
│       ├── ConfigStoreTests.swift
│       └── ExecutorTests.swift
├── client/
│   ├── package.json
│   ├── vite.config.ts
│   ├── index.html
│   └── src/
│       ├── App.svelte
│       ├── lib/
│       │   ├── types.ts
│       │   ├── state.svelte.ts
│       │   ├── api.ts
│       │   └── websocket.ts
│       └── components/
│           ├── ButtonGrid.svelte
│           ├── ButtonCell.svelte
│           ├── EmptyCell.svelte
│           ├── ButtonEditor.svelte
│           ├── ActionEditor.svelte
│           ├── ProfileSwitcher.svelte
│           └── FolderBreadcrumb.svelte
├── data/
│   └── default-config.json
└── docs/
    └── ARCHITECTURE.md
```

### 12.2 Config File Location

```
~/.action-deck/
└── config.json
```

Created on first run by `ConfigStore`. The app never writes to the repo directory at runtime.

---

## 13. Swift for Android Developers

This is a quick concept map. For each Android/Kotlin concept, here's the Swift equivalent used in this project.

### 13.1 Language Concepts

| Kotlin | Swift | Notes |
|---|---|---|
| `data class Foo(val x: Int)` | `struct Foo { var x: Int }` | Both are value types. Swift structs can have methods, extensions, and conform to protocols. |
| `class Foo` | `class Foo` | Reference type in both. Swift classes don't auto-generate `equals`/`hashCode`. |
| `sealed class Action` | `enum Action` with associated values | Swift enums are far more powerful than Kotlin enums. |
| `interface Executor` | `protocol ActionExecutorProtocol` | Protocols = interfaces, but with associated types and default implementations. |
| `when (action) { is Shell -> ... }` | `switch action { case .shell(let cmd): ... }` | Pattern matching. Swift is exhaustive by default (no `else` needed if all cases covered). |
| `val` / `var` | `let` / `var` | Same semantics. `let` in Swift is truly immutable. |
| `fun foo(): Int` | `func foo() -> Int` | Arrow instead of colon for return type. |
| `suspend fun foo()` | `func foo() async throws` | Swift separates async and throws. Both must be declared explicitly. |
| `launch { }` | `Task { }` | Unstructured concurrency. Prefer `async let` or task groups for structured. |
| `withContext(Dispatchers.Main) { }` | `await MainActor.run { }` | Run code on the main thread. |
| `StateFlow<T>` / `MutableStateFlow<T>` | `@Observable class` + `@State` / `@Binding` | SwiftUI's observation system. `@Observable` auto-publishes all stored properties. |
| `ViewModel` | `@Observable class` | Similar lifecycle. Swift's version is not lifecycle-aware by default. |
| `coroutineScope { }` | `async let` / `withTaskGroup { }` | Structured concurrency. |
| `Mutex` / `synchronized` | `actor` | Swift actors enforce mutual exclusion at the language level. |
| `companion object { }` | `static` properties/functions | `static` inside a `struct` or `class`. |
| `object Singleton` | `final class Singleton { static let shared = Singleton() }` | Swift has no language-level singletons. |
| `?.` safe call | `?.` optional chaining | Same syntax. |
| `?: Elvis` | `?? nil-coalescing` | Same concept, different syntax. |
| `lateinit var` | `var foo: Foo!` (implicitly unwrapped optional) | Avoid — prefer optionals or deferred initialisation via `lazy var`. |
| `List<T>` / `MutableList<T>` | `[T]` (Array) | Swift arrays are value types (copying is cheap due to COW). |
| `Map<K, V>` | `[K: V]` (Dictionary) | Also a value type. |

### 13.2 Codable vs Serializable

```swift
// Swift: Codable = Encodable + Decodable (synthesised automatically)
struct Profile: Codable {
    var id: UUID
    var name: String
    var columns: Int
}

// Encodes to JSON automatically — no @SerializedName needed unless key differs
let encoder = JSONEncoder()
encoder.keyEncodingStrategy = .convertToSnakeCase  // optional: use camelCase by default
let data = try encoder.encode(profile)
```

```kotlin
// Kotlin equivalent with kotlinx.serialization
@Serializable
data class Profile(
    val id: String,
    val name: String,
    val columns: Int
)
```

### 13.3 async/await vs Coroutines

```swift
// Swift
func fetchConfig() async throws -> AppConfig {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(AppConfig.self, from: data)
}

// Call it:
Task {
    do {
        let config = try await fetchConfig()
    } catch {
        print("Error: \(error)")
    }
}
```

```kotlin
// Kotlin equivalent
suspend fun fetchConfig(): AppConfig {
    val response = httpClient.get(url)
    return json.decodeFromString(response.bodyAsText())
}

// Call it:
viewModelScope.launch {
    try {
        val config = fetchConfig()
    } catch (e: Exception) {
        println("Error: $e")
    }
}
```

### 13.4 Actors vs synchronized blocks

```swift
// Swift actor — the compiler prevents data races
actor ConfigStore {
    private var config: AppConfig

    func read() -> AppConfig { config }

    func write(_ newConfig: AppConfig) {
        config = newConfig
        persist(newConfig)
    }
}

// Calling actor methods requires await — even from within the same module
let config = await store.read()
```

The key difference: in Kotlin you'd use a `Mutex` or `@Synchronized`. In Swift, the `actor` keyword does the same job but the compiler enforces it — you literally cannot call an actor method without `await` when crossing the actor boundary.

---

## 14. Baby-Step Implementation Roadmap

Each step should result in something you can run or test. Never spend more than ~1 hour on a single step. Steps marked **[test]** must include at least one test.

### Phase 1 — Project Skeleton

1. Create `Package.swift` with four targets: `ActionDeckCore` (library), `ActionDeckServer` (executable), `ActionDeckApp` (macOS app), `ActionDeckMCP` (executable).
2. Add a `Hello, World` in `ActionDeckServer/App.swift` — just `print("Hello")` and confirm `swift run ActionDeckServer` works.
3. Add Hummingbird 2 as a package dependency in `Package.swift`. Write a single route `GET /ping → "pong"` and confirm with `curl http://localhost:9900/ping`.
4. Add `default-config.json` to the `data/` directory with one profile and two buttons (no actions yet).
5. Create the Svelte client with `npm create vite@latest client -- --template svelte-ts`. Confirm `npm run dev` serves the default Svelte page.

### Phase 2 — Data Model **[test]**

6. Define `Action.swift` with all 6 action cases. Add `Codable` conformance with a `type` discriminator key.
7. **[test]** Write `ActionCodableTests.swift` — encode each action case, decode back, assert equality.
8. Define `Button.swift`, `Folder.swift`, `Profile.swift`, `AppConfig.swift`.
9. **[test]** Write a test that decodes `default-config.json` into `AppConfig` and confirms field values.
10. Create `ConfigStore.swift` — reads/writes `~/.action-deck/config.json`. Creates the file on first run by copying `default-config.json` from the bundle.
11. **[test]** `ConfigStoreTests.swift` — write config to a temp dir, read it back, assert round-trip fidelity.

### Phase 3 — REST API: Config & Profiles

12. Wire `ConfigStore` into the Hummingbird app. Add `GET /api/config` — returns the full `AppConfig` as JSON.
13. Add `GET /api/profiles` — returns `config.profiles`.
14. Add `POST /api/profiles` — creates a new profile with a generated UUID, appends to config, saves.
15. Add `PUT /api/profiles/:id` — replaces an existing profile.
16. Add `DELETE /api/profiles/:id` — removes profile (error if it's the active one or the last one).
17. Confirm all five endpoints with `curl` commands.

### Phase 4 — REST API: Buttons & Folders

18. Add `PUT /api/profiles/:pId/buttons/:btnId` — upsert a button (by position, if no ID collision).
19. Add `DELETE /api/profiles/:pId/buttons/:btnId` — remove a button slot.
20. Add `POST /api/profiles/:pId/folders` — create a folder.
21. Add `PUT` and `DELETE` for folders.
22. Add `PUT` and `DELETE` for buttons inside folders.

### Phase 5 — Action Execution

23. Implement `ShellExecutor.swift`. **[test]** Execute `echo hello` and assert output.
24. Implement `AppleScriptExecutor.swift`. Manual test: run `display notification "test"`.
25. Implement `AppExecutor.swift` (open/close). Manual test: open `Calculator`.
26. Implement `KeyboardMaestroExecutor.swift`. Manual test: trigger a known macro by name.
27. Implement `OpenURLExecutor.swift`. Manual test: open `https://example.com`.
28. Implement `ActionExecutor.swift` dispatcher.
29. Add `POST /api/actions/execute` route. Test with `curl -d '{"action":{"type":"shell","command":"date"}}' -H 'Content-Type: application/json' http://localhost:9900/api/actions/execute`.

### Phase 6 — WebSocket

30. Add `ConnectionManager.swift` actor. Register `/ws` WebSocket route in Hummingbird.
31. On new WebSocket connection: add to manager, send current profile state.
32. On disconnect: remove from manager.
33. Add `POST /api/profiles/:id/activate` route. Save, broadcast `profile:switched`.
34. Wire action execution to broadcast `action:start`, `action:complete`, `action:error`.

### Phase 7 — Svelte Web Client

35. Replace default Svelte page with a dark-themed shell: top bar + empty grid area.
36. Implement `api.ts` — typed fetch wrappers for all REST endpoints. Load config on mount.
37. Implement `websocket.ts` — connect to `ws://localhost:9900/ws`, handle `profile:switched`.
38. Implement `ButtonGrid.svelte` — render a 5×3 CSS grid of empty cells.
39. Implement `ButtonCell.svelte` — show label, icon, background colour. Click triggers action via REST.
40. Add visual feedback to `ButtonCell` — spinner while action is running, flash green/red on result.
41. Implement `ProfileSwitcher.svelte` — dropdown in top bar. Switching calls `activate` endpoint.
42. Implement `ButtonEditor.svelte` — modal on right-click/long-press. Basic fields: label, icon, colour.
43. Implement `ActionEditor.svelte` — conditional form fields per action type.
44. Implement `EmptyCell.svelte` — click to open `ButtonEditor` in "create" mode.
45. Implement `FolderBreadcrumb.svelte` — render folder path; clicking an item navigates to it.
46. Wire folder navigation: clicking a folder button pushes its ID onto `currentFolderPath`.

### Phase 8 — SwiftUI App

47. Create `ActionDeckApp.swift` with `MenuBarExtra`. Show a simple menu with "Open UI" and "Quit".
48. Create `ServerController.swift` — starts/stops the Hummingbird server in a `Task`.
49. Create `SettingsView.swift` — show server status, port, active profile. Accessible via `Settings` scene.
50. Add `LSUIElement = YES` to Info.plist so the app hides from the Dock.

### Phase 9 — MCP Server

51. Create `ActionDeckMCP` executable target. Implement the stdio JSON-RPC read/write loop.
52. Implement `APIClient.swift` — thin URLSession wrapper talking to localhost:9900.
53. Implement `execute_action` tool. Test by running the MCP binary from the terminal and sending a raw JSON-RPC message.
54. Implement `list_profiles` and `switch_profile` tools.
55. Test end-to-end: configure Claude Code to use the MCP server; ask Claude to "run the date command".

---

## 15. Swift Concepts by Phase

A map of which Swift concepts each phase introduces, so learning can be intentional.

| Phase | New Swift Concepts |
|---|---|
| 1 — Skeleton | `Package.swift`, executable targets, library targets, `@main`, basic `async/await`, `Task { }` |
| 2 — Data Model | `struct`, `enum` with associated values, `Codable`, `JSONEncoder`/`JSONDecoder`, custom `CodingKeys`, `UUID`, `FileManager`, `Bundle` |
| 3–4 — REST API | Hummingbird routing, `RequestContext`, `HTTPResponse`, generics (`Response<T>`), `throws`, `do/catch` |
| 5 — Execution | `Process`, `Pipe`, `withCheckedThrowingContinuation`, `URL`/`URLComponents`, `NSWorkspace`, `@MainActor` |
| 6 — WebSocket | `actor`, `async` isolation, `Set`, `for await` on `AsyncStream` |
| 7 — Svelte | Svelte 5 runes, TypeScript generics, `fetch` API |
| 8 — SwiftUI | `@Observable`, `@State`, `@Binding`, `Scene`, `MenuBarExtra`, `Settings`, `View`, `ViewBuilder` |
| 9 — MCP | stdio I/O, `readLine()`, `fflush`, `URLSession`, `JSONSerialization` |

---

*Last updated: April 2026. This document should be kept in sync with implementation decisions as they are made.*
