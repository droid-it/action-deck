# Action Deck — Web-Based Stream Deck for macOS

A localhost web app providing a customizable button grid to trigger actions on macOS.

## Tech Stack

| Layer | Choice | Why |
|---|---|---|
| **Frontend** | Svelte 5 + TypeScript + Vite | Lightweight, reactive, minimal boilerplate |
| **Backend** | Node.js + Express + TypeScript | Simple REST + WebSocket, huge ecosystem |
| **Storage** | JSON file on disk | Human-readable, easy to backup, <100KB |
| **Styling** | CSS (Svelte scoped) | Dark theme, custom button styles |
| **Communication** | REST + WebSocket | REST for config CRUD, WebSocket for action feedback + profile switch events |

---

## Scope

### 4 Action Types

| Type | What it does | How it executes |
|---|---|---|
| **Keyboard Maestro** | Run a named KM macro | `osascript -e 'tell application "Keyboard Maestro Engine" to do script "MacroName"'` |
| **Open/Close App** | Launch or quit an app | `open -a "AppName"` / `osascript -e 'tell application "AppName" to quit'` |
| **Shell Command** | Run a terminal command | `child_process.exec(cmd, { shell: '/bin/zsh' })` |
| **AppleScript** | Run arbitrary AppleScript | `osascript -e '...'` or `osascript file.scpt` |
| **Open URL** | Open URL in default browser or trigger deeplinks | `open "https://..."` or `open "vscode://..."` |

### Profiles
- Separate button layouts per context (e.g., "Coding", "Meetings", "DevOps")
- **Manual switching** via UI (sidebar/dropdown)
- **Webhook switching** — `POST /api/profiles/:id/activate` to switch profiles programmatically (e.g., from Alfred, cron, Shortcuts.app, or Keyboard Maestro itself)
- Each profile has its own grid of buttons organized into folders

### Folders
- Group related actions within a profile (e.g., "Git", "Docker", "Slack")
- Click a folder button → drills into that folder's sub-grid
- Back button to return to parent
- Folders are a flat organizational layer (no deep nesting needed)

---

## Architecture

```
Browser (localhost:5173)          Server (localhost:9900)          macOS
┌─────────────────────┐     ┌─────────────────────────┐     ┌──────────────┐
│  Svelte Frontend    │────▶│  Express REST API       │────▶│ child_process│
│  - Button Grid      │ HTTP│  - Config CRUD          │     │ - osascript  │
│  - Folder Nav       │     │  - Execute actions      │     │ - open       │
│  - Profile Switcher │     │  - Webhook endpoints    │     │ - bash -c    │
│                     │◀───▶│  WebSocket Server       │     └──────────────┘
│                     │  WS │  - Action feedback      │
│                     │     │  - Profile change push  │
└─────────────────────┘     └─────────────────────────┘
                                      │
                                      ▼
                            ~/.action-deck/config.json
```

---

## Project Structure

```
action-deck/
├── package.json                 # Root workspace
├── tsconfig.json                # Shared TS base config
├── PLAN.md
│
├── shared/                      # Shared types between client & server
│   ├── package.json
│   └── src/
│       ├── types.ts
│       └── index.ts
│
├── server/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts             # Express + WebSocket server entry
│       ├── routes/
│       │   ├── profiles.ts      # Profile CRUD + activate webhook
│       │   ├── buttons.ts       # Button CRUD within profiles/folders
│       │   └── actions.ts       # POST /api/actions/execute
│       ├── services/
│       │   ├── config-store.ts  # Read/write ~/.action-deck/config.json
│       │   └── executor/
│       │       ├── index.ts     # Dispatch by action type
│       │       ├── keyboard-maestro.ts
│       │       ├── app.ts       # open/close apps
│       │       ├── shell.ts     # shell commands
│       │       └── applescript.ts
│       └── ws.ts                # WebSocket manager
│
├── client/
│   ├── package.json
│   ├── vite.config.ts
│   ├── index.html
│   └── src/
│       ├── main.ts
│       ├── App.svelte
│       ├── lib/
│       │   ├── api.ts           # REST client
│       │   ├── websocket.ts     # WebSocket client
│       │   └── stores/
│       │       ├── config.ts    # Profiles, buttons state
│       │       └── ui.ts        # Current folder path, editing state
│       ├── components/
│       │   ├── ButtonGrid.svelte
│       │   ├── ButtonCell.svelte
│       │   ├── EmptyCell.svelte
│       │   ├── ButtonEditor.svelte
│       │   ├── ActionEditor.svelte
│       │   ├── ProfileSwitcher.svelte
│       │   ├── FolderBreadcrumb.svelte
│       │   └── TopBar.svelte
│       └── styles/
│           └── global.css
│
└── data/
    └── default-config.json      # Starter config, copied on first run
```

---

## Data Model

```typescript
// === Actions ===

interface KeyboardMaestroAction {
  type: 'keyboard-maestro'
  macroName: string           // name or UUID of the KM macro
}

interface AppAction {
  type: 'open-app' | 'close-app'
  appName: string             // e.g. "Visual Studio Code", "Slack"
}

interface ShellAction {
  type: 'shell'
  command: string
  workingDirectory?: string   // defaults to $HOME
  timeout?: number            // ms, default 30000
}

interface AppleScriptAction {
  type: 'applescript'
  script: string              // inline AppleScript code
}

interface OpenUrlAction {
  type: 'open-url'
  url: string                 // https://, vscode://, slack://, etc.
}

type Action = KeyboardMaestroAction | AppAction | ShellAction | AppleScriptAction | OpenUrlAction

// === Layout ===

interface Button {
  id: string                  // UUID
  position: number            // grid index (0-based)
  label?: string
  icon?: string               // emoji, URL, or data URI
  backgroundColor?: string    // hex color
  textColor?: string          // hex color
  action?: Action             // if set, this is an action button
  folderId?: string           // if set, this button opens a folder
}

interface Folder {
  id: string
  name: string
  buttons: Button[]
  columns?: number            // override grid columns for this folder
  rows?: number               // override grid rows
}

interface Profile {
  id: string
  name: string
  buttons: Button[]           // top-level grid buttons
  folders: Folder[]           // folders referenced by buttons via folderId
  columns: number             // default 5
  rows: number                // default 3
}

interface AppConfig {
  version: 1
  activeProfileId: string
  profiles: Profile[]
  settings: {
    port: number              // default 9900
    theme: 'dark' | 'light'
  }
}
```

---

## API Design

### REST Endpoints

| Method | Path | Description |
|---|---|---|
| **Config** | | |
| `GET` | `/api/config` | Get full config |
| **Profiles** | | |
| `GET` | `/api/profiles` | List all profiles |
| `POST` | `/api/profiles` | Create profile |
| `PUT` | `/api/profiles/:id` | Update profile |
| `DELETE` | `/api/profiles/:id` | Delete profile |
| `POST` | `/api/profiles/:id/activate` | **Webhook**: switch active profile |
| **Buttons** | | |
| `PUT` | `/api/profiles/:pId/buttons/:btnId` | Update button |
| `DELETE` | `/api/profiles/:pId/buttons/:btnId` | Clear button slot |
| **Folders** | | |
| `POST` | `/api/profiles/:pId/folders` | Create folder |
| `PUT` | `/api/profiles/:pId/folders/:fId` | Update folder |
| `DELETE` | `/api/profiles/:pId/folders/:fId` | Delete folder |
| `PUT` | `/api/profiles/:pId/folders/:fId/buttons/:btnId` | Update button in folder |
| **Actions** | | |
| `POST` | `/api/actions/execute` | Execute an action `{ action: Action }` |

### WebSocket Messages

```typescript
// Server → Client
{ type: 'action:start', buttonId: string }
{ type: 'action:output', buttonId: string, data: string }
{ type: 'action:complete', buttonId: string, exitCode: number }
{ type: 'action:error', buttonId: string, error: string }
{ type: 'profile:switched', profileId: string }   // pushed when webhook activates a profile

// Client → Server
{ type: 'action:execute', buttonId: string, action: Action }
```

---

## Webhook: Profile Switching

External tools can switch the active profile via HTTP:

```bash
# Switch to "Meetings" profile from anywhere
curl -X POST http://localhost:9900/api/profiles/meetings-id/activate

# Example: Keyboard Maestro macro that switches Action Deck profile
# when you open Zoom → POST to the activate endpoint

# Example: macOS Shortcuts automation
# When Focus Mode changes → call webhook to switch profile
```

The server broadcasts `profile:switched` over WebSocket so all open browser tabs update instantly.

---

## macOS Execution Details

### Keyboard Maestro
```bash
osascript -e 'tell application "Keyboard Maestro Engine" to do script "Macro Name"'
# or by UUID:
osascript -e 'tell application "Keyboard Maestro Engine" to do script "DEADBEEF-1234-..."'
```

### Open/Close Apps
```bash
open -a "Visual Studio Code"
osascript -e 'tell application "Slack" to quit'
```

### Shell Commands
```typescript
exec(command, { cwd: workingDir, timeout, shell: '/bin/zsh' })
```

### Open URL
```bash
open "https://github.com/pulls"           # default browser
open "vscode://file/path/to/file"         # VS Code deeplink
open "slack://channel?team=T123&id=C456"  # Slack deeplink
open "raycast://extensions/..."           # Raycast deeplink
```

### AppleScript
```bash
osascript -e 'tell application "Finder" to empty trash'
osascript -e 'display notification "Done!" with title "Action Deck"'
```

---

## Security

- Binds to `127.0.0.1` only — no remote access
- Shell commands run with user's permissions (no elevation)
- Timeout on all commands (default 30s)
- Config writes limited to `~/.action-deck/`

---

## Build Phases

### Phase 1 — MVP
- Express server with config CRUD + action execution
- Svelte button grid (5×3)
- 5 action types working (KM, app, shell, AppleScript, open URL)
- Button editor modal
- Single profile, no folders
- JSON config persistence

### Phase 2 — Profiles & Folders
- Multiple profiles with manual switching
- Webhook endpoint for profile activation
- WebSocket push for profile changes
- Folder navigation (drill-in / breadcrumb back)

### Phase 3 — Polish
- Drag-and-drop button reordering
- Icon picker / emoji picker
- Visual feedback on action execution (loading spinner, success/error flash)
- Import/export profiles
- Configurable grid size per profile
