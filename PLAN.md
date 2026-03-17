# Action Deck — Web-Based Stream Deck Clone

A localhost web application that replicates Elgato Stream Deck functionality for macOS software engineers.

## Tech Stack

| Layer | Choice | Why |
|---|---|---|
| **Frontend** | Svelte 5 + TypeScript + Vite | Lightweight, reactive, minimal boilerplate — perfect for an interactive button grid |
| **Backend** | Node.js + Express + TypeScript | Battle-tested, simple REST + WebSocket setup, huge ecosystem |
| **Storage** | JSON files on disk | Human-readable, easy to backup/version-control, config is <100KB |
| **Styling** | CSS (Svelte scoped) | No framework needed — dark theme, custom button styles |
| **Communication** | REST API + WebSocket | REST for CRUD config operations, WebSocket for action execution feedback |

---

## Core Features (by priority)

### Phase 1 — MVP: Button Grid + Action Execution
- Configurable grid of buttons (default 5×3, like Stream Deck MK.2)
- Click a button → execute an action on macOS
- Supported action types:
  - **Shell command** — run any terminal command (`git push`, `npm run dev`, etc.)
  - **Hotkey** — send keyboard shortcuts to active/target app via `osascript`
  - **Open App** — launch applications via `open -a AppName`
  - **Open URL** — open URLs in default browser via `open URL`
  - **Open File/Folder** — open files/directories via `open path`
  - **System controls** — volume, brightness, mute, lock screen, dark mode toggle
- Button editor modal: configure action, label, icon, background color
- Persistent config saved to `~/.action-deck/config.json`
- Dark theme UI

### Phase 2 — Organization
- **Profiles** — separate button layouts (e.g., "Coding", "Meetings", "DevOps")
- **Pages** — multiple pages within a profile, tab-style navigation
- **Folders** — a button that opens a nested sub-grid of buttons
- Profile/page switcher in sidebar

### Phase 3 — Multi-Actions & Advanced
- **Multi-action sequences** — chain actions with configurable delays (2ms–5000ms)
- **Key Logic** — press, double-press, long-press trigger different actions on same button
- **Smart Profiles** — auto-switch profile based on active macOS app (poll via `osascript`)
- **Pinned actions** — buttons that persist across all pages in a profile

### Phase 4 — Polish
- Drag-and-drop button reordering
- Icon picker (built-in icon set + custom upload)
- Import/export profiles as JSON
- Dynamic button states (e.g., show git branch, build status)
- Mobile-responsive layout (use from phone on same network)

---

## Architecture

```
Browser (localhost:5173)          Server (localhost:9900)          macOS
┌─────────────────────┐     ┌─────────────────────────┐     ┌──────────────┐
│  Svelte Frontend    │────▶│  Express REST API       │────▶│ child_process│
│  - Button Grid      │ HTTP│  - CRUD config          │     │ - osascript  │
│  - Button Editor    │     │  - Execute actions      │     │ - open       │
│  - Profile Manager  │     │                         │     │ - bash -c    │
│                     │◀───▶│  WebSocket Server       │     │ - defaults   │
│                     │  WS │  - Execution feedback   │     └──────────────┘
│                     │     │  - Live config reload   │
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
├── tsconfig.json                # Shared TS config
├── PLAN.md
│
├── shared/                      # Shared types
│   ├── package.json
│   └── src/
│       ├── types.ts             # Profile, Page, Button, Action types
│       └── index.ts
│
├── server/                      # Express backend
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts             # Entry: Express + WebSocket server
│       ├── routes/
│       │   ├── profiles.ts      # Profile CRUD
│       │   ├── pages.ts         # Page CRUD within profiles
│       │   ├── buttons.ts       # Button CRUD within pages
│       │   └── actions.ts       # POST /actions/execute
│       ├── services/
│       │   ├── config-store.ts  # Read/write config.json
│       │   └── executor/
│       │       ├── index.ts     # Action dispatch
│       │       ├── shell.ts     # child_process.exec wrapper
│       │       ├── hotkey.ts    # osascript keystroke
│       │       ├── apps.ts      # open -a wrapper
│       │       ├── url.ts       # open URL
│       │       └── system.ts    # volume, brightness, etc.
│       └── middleware/
│           └── error-handler.ts
│
├── client/                      # Svelte frontend
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
│       │       ├── config.ts    # Svelte store for profiles/pages
│       │       └── ui.ts        # UI state (selected profile, editing)
│       ├── components/
│       │   ├── ButtonGrid.svelte
│       │   ├── ButtonCell.svelte
│       │   ├── EmptyCell.svelte
│       │   ├── ButtonEditor.svelte
│       │   ├── ActionEditor.svelte
│       │   ├── Sidebar.svelte
│       │   ├── TopBar.svelte
│       │   └── FolderView.svelte
│       └── styles/
│           └── global.css
│
└── data/                        # Default data (copied on first run)
    └── default-config.json
```

---

## Data Model

```typescript
// === Action Types ===

type ActionType = 'shell' | 'hotkey' | 'open-app' | 'open-url' | 'open-file' | 'system' | 'multi-action'

interface ShellAction {
  type: 'shell'
  command: string
  workingDirectory?: string   // defaults to $HOME
  timeout?: number            // ms, default 30000
}

interface HotkeyAction {
  type: 'hotkey'
  key: string                 // e.g. "p", "space", "return"
  modifiers: Modifier[]       // ['command', 'shift', 'option', 'control']
  targetApp?: string          // bring app to front first
}

type Modifier = 'command' | 'shift' | 'option' | 'control'

interface OpenAppAction {
  type: 'open-app'
  appName: string             // e.g. "Visual Studio Code"
}

interface OpenUrlAction {
  type: 'open-url'
  url: string
}

interface OpenFileAction {
  type: 'open-file'
  path: string
}

interface SystemAction {
  type: 'system'
  command: 'volume-up' | 'volume-down' | 'volume-mute'
         | 'brightness-up' | 'brightness-down'
         | 'sleep' | 'lock-screen' | 'toggle-dark-mode'
}

interface MultiAction {
  type: 'multi-action'
  steps: { action: Action; delayAfter: number }[]
}

type Action = ShellAction | HotkeyAction | OpenAppAction
            | OpenUrlAction | OpenFileAction | SystemAction | MultiAction

// === Layout ===

interface Button {
  id: string                  // UUID
  position: number            // grid index (0-based, left-to-right, top-to-bottom)
  label?: string
  icon?: string               // emoji, URL, or data URI
  backgroundColor?: string    // hex color, default "#1a1a2e"
  textColor?: string          // hex color, default "#ffffff"
  action?: Action
  folder?: {                  // if set, button opens a nested sub-grid
    name: string
    buttons: Button[]
    columns?: number
    rows?: number
  }
}

interface Page {
  id: string
  name: string
  buttons: Button[]
  columns: number             // default 5
  rows: number                // default 3
}

interface Profile {
  id: string
  name: string
  pages: Page[]
  activePageId: string
  smartTrigger?: {            // Phase 3: auto-switch
    appBundleId: string       // e.g. "com.microsoft.VSCode"
  }
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
| `GET` | `/api/config` | Get full config |
| `PUT` | `/api/config` | Replace full config |
| **Profiles** | | |
| `GET` | `/api/profiles` | List all profiles |
| `POST` | `/api/profiles` | Create profile |
| `PUT` | `/api/profiles/:id` | Update profile |
| `DELETE` | `/api/profiles/:id` | Delete profile |
| `PUT` | `/api/profiles/:id/activate` | Set active profile |
| **Pages** | | |
| `POST` | `/api/profiles/:profileId/pages` | Create page |
| `PUT` | `/api/profiles/:profileId/pages/:pageId` | Update page |
| `DELETE` | `/api/profiles/:profileId/pages/:pageId` | Delete page |
| **Buttons** | | |
| `PUT` | `/api/profiles/:pId/pages/:pgId/buttons/:btnId` | Update button |
| `DELETE` | `/api/profiles/:pId/pages/:pgId/buttons/:btnId` | Delete/clear button |
| **Actions** | | |
| `POST` | `/api/actions/execute` | Execute an action `{ action: Action }` |

### WebSocket Messages

```typescript
// Server → Client
{ type: 'action:start', actionId: string }
{ type: 'action:output', actionId: string, data: string }
{ type: 'action:complete', actionId: string, exitCode: number }
{ type: 'action:error', actionId: string, error: string }
{ type: 'config:updated' }     // triggers client to re-fetch config

// Client → Server
{ type: 'action:execute', action: Action }
```

---

## macOS Action Execution

### Shell Commands
```typescript
import { exec } from 'child_process'
exec(command, { cwd, timeout, shell: '/bin/zsh' }, callback)
```

### Keyboard Shortcuts (osascript)
```bash
osascript -e 'tell application "System Events" to keystroke "p" using {command down, shift down}'
```
With target app:
```bash
osascript -e 'tell application "Visual Studio Code" to activate'
osascript -e 'tell application "System Events" to keystroke "p" using {command down, shift down}'
```

### Open App / URL / File
```bash
open -a "Visual Studio Code"
open "https://github.com"
open ~/Documents
```

### System Controls
```bash
# Volume
osascript -e 'set volume output volume ((output volume of (get volume settings)) + 10)'
osascript -e 'set volume output muted (not (output muted of (get volume settings)))'

# Brightness (requires brightness CLI or osascript)
# Lock screen
osascript -e 'tell application "System Events" to keystroke "q" using {command down, control down}'

# Dark mode toggle
osascript -e 'tell app "System Events" to tell appearance preferences to set dark mode to not dark mode'
```

---

## Security Considerations

- **No remote access by default** — binds to `127.0.0.1` only
- **Shell command sanitization** — no eval, use parameterized exec
- **Timeout on all commands** — default 30s, configurable
- **No arbitrary file writes** — config store only writes to `~/.action-deck/`

---

## Default Profile (ships with app)

A starter "Developer" profile with common actions:

| Button | Action |
|---|---|
| VS Code | Open app |
| Terminal | Open app |
| Browser | Open app |
| Git Status | Shell: `cd $(pwd) && git status` |
| Git Pull | Shell: `cd $(pwd) && git pull` |
| Git Push | Shell: `cd $(pwd) && git push` |
| npm Dev | Shell: `npm run dev` |
| Lock Screen | System: lock-screen |
| Mute | System: volume-mute |
| Dark Mode | System: toggle-dark-mode |
| Cmd+Space | Hotkey: Spotlight |
| Screenshot | Hotkey: Cmd+Shift+4 |
