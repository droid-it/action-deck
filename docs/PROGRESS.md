# Action Deck — Progress Tracker

**Last completed:** (none yet)  
**Next up:** Phase 1, Step 1 — Create `Package.swift` with four targets

---

## Phase 1: Project Skeleton

- [ ] Step 1.1: Create `Package.swift` with four targets: `ActionDeckCore` (library), `ActionDeckServer` (executable), `ActionDeckApp` (macOS app), `ActionDeckMCP` (executable).
- [ ] Step 1.2: Add a `Hello, World` in `ActionDeckServer/App.swift` — just `print("Hello")` and confirm `swift run ActionDeckServer` works.
- [ ] Step 1.3: Add Hummingbird 2 as a package dependency in `Package.swift`. Write a single route `GET /ping → "pong"` and confirm with `curl http://localhost:9900/ping`.
- [ ] Step 1.4: Add `default-config.json` to the `data/` directory with one profile and two buttons (no actions yet).
- [ ] Step 1.5: Create the Svelte client with `npm create vite@latest client -- --template svelte-ts`. Confirm `npm run dev` serves the default Svelte page.

---

## Phase 2: Data Model

- [ ] Step 2.1: Define `Action.swift` with all 6 action cases. Add `Codable` conformance with a `type` discriminator key.
- [ ] Step 2.2: **[test]** Write `ActionCodableTests.swift` — encode each action case, decode back, assert equality.
- [ ] Step 2.3: Define `Button.swift`, `Folder.swift`, `Profile.swift`, `AppConfig.swift`.
- [ ] Step 2.4: **[test]** Write a test that decodes `default-config.json` into `AppConfig` and confirms field values.
- [ ] Step 2.5: Create `ConfigStore.swift` — reads/writes `~/.action-deck/config.json`. Creates the file on first run by copying `default-config.json` from the bundle.
- [ ] Step 2.6: **[test]** `ConfigStoreTests.swift` — write config to a temp dir, read it back, assert round-trip fidelity.

---

## Phase 3: REST API — Config & Profiles

- [ ] Step 3.1: Wire `ConfigStore` into the Hummingbird app. Add `GET /api/config` — returns the full `AppConfig` as JSON.
- [ ] Step 3.2: Add `GET /api/profiles` — returns `config.profiles`.
- [ ] Step 3.3: Add `POST /api/profiles` — creates a new profile with a generated UUID, appends to config, saves.
- [ ] Step 3.4: Add `PUT /api/profiles/:id` — replaces an existing profile.
- [ ] Step 3.5: Add `DELETE /api/profiles/:id` — removes profile (error if it's the active one or the last one).
- [ ] Step 3.6: Confirm all five endpoints with `curl` commands.

---

## Phase 4: REST API — Buttons & Folders

- [ ] Step 4.1: Add `PUT /api/profiles/:pId/buttons/:btnId` — upsert a button (by position, if no ID collision).
- [ ] Step 4.2: Add `DELETE /api/profiles/:pId/buttons/:btnId` — remove a button slot.
- [ ] Step 4.3: Add `POST /api/profiles/:pId/folders` — create a folder.
- [ ] Step 4.4: Add `PUT` and `DELETE` for folders.
- [ ] Step 4.5: Add `PUT` and `DELETE` for buttons inside folders.

---

## Phase 5: Action Execution

- [ ] Step 5.1: Implement `ShellExecutor.swift`. **[test]** Execute `echo hello` and assert output.
- [ ] Step 5.2: Implement `AppleScriptExecutor.swift`. Manual test: run `display notification "test"`.
- [ ] Step 5.3: Implement `AppExecutor.swift` (open/close). Manual test: open `Calculator`.
- [ ] Step 5.4: Implement `KeyboardMaestroExecutor.swift`. Manual test: trigger a known macro by name.
- [ ] Step 5.5: Implement `OpenURLExecutor.swift`. Manual test: open `https://example.com`.
- [ ] Step 5.6: Implement `ActionExecutor.swift` dispatcher.
- [ ] Step 5.7: Add `POST /api/actions/execute` route. Test with `curl -d '{"action":{"type":"shell","command":"date"}}' -H 'Content-Type: application/json' http://localhost:9900/api/actions/execute`.

---

## Phase 6: WebSocket

- [ ] Step 6.1: Add `ConnectionManager.swift` actor. Register `/ws` WebSocket route in Hummingbird.
- [ ] Step 6.2: On new WebSocket connection: add to manager, send current profile state.
- [ ] Step 6.3: On disconnect: remove from manager.
- [ ] Step 6.4: Add `POST /api/profiles/:id/activate` route. Save, broadcast `profile:switched`.
- [ ] Step 6.5: Wire action execution to broadcast `action:start`, `action:complete`, `action:error`.

---

## Phase 7: Svelte Web Client

- [ ] Step 7.1: Replace default Svelte page with a dark-themed shell: top bar + empty grid area.
- [ ] Step 7.2: Implement `api.ts` — typed fetch wrappers for all REST endpoints. Load config on mount.
- [ ] Step 7.3: Implement `websocket.ts` — connect to `ws://localhost:9900/ws`, handle `profile:switched`.
- [ ] Step 7.4: Implement `ButtonGrid.svelte` — render a 5×3 CSS grid of empty cells.
- [ ] Step 7.5: Implement `ButtonCell.svelte` — show label, icon, background colour. Click triggers action via REST.
- [ ] Step 7.6: Add visual feedback to `ButtonCell` — spinner while action is running, flash green/red on result.
- [ ] Step 7.7: Implement `ProfileSwitcher.svelte` — dropdown in top bar. Switching calls `activate` endpoint.
- [ ] Step 7.8: Implement `ButtonEditor.svelte` — modal on right-click/long-press. Basic fields: label, icon, colour.
- [ ] Step 7.9: Implement `ActionEditor.svelte` — conditional form fields per action type.
- [ ] Step 7.10: Implement `EmptyCell.svelte` — click to open `ButtonEditor` in "create" mode.
- [ ] Step 7.11: Implement `FolderBreadcrumb.svelte` — render folder path; clicking an item navigates to it.
- [ ] Step 7.12: Wire folder navigation: clicking a folder button pushes its ID onto `currentFolderPath`.

---

## Phase 8: SwiftUI App

- [ ] Step 8.1: Create `ActionDeckApp.swift` with `MenuBarExtra`. Show a simple menu with "Open UI" and "Quit".
- [ ] Step 8.2: Create `ServerController.swift` — starts/stops the Hummingbird server in a `Task`.
- [ ] Step 8.3: Create `SettingsView.swift` — show server status, port, active profile. Accessible via `Settings` scene.
- [ ] Step 8.4: Add `LSUIElement = YES` to Info.plist so the app hides from the Dock.

---

## Phase 9: MCP Server

- [ ] Step 9.1: Create `ActionDeckMCP` executable target. Implement the stdio JSON-RPC read/write loop.
- [ ] Step 9.2: Implement `APIClient.swift` — thin URLSession wrapper talking to localhost:9900.
- [ ] Step 9.3: Implement `execute_action` tool. Test by running the MCP binary from the terminal and sending a raw JSON-RPC message.
- [ ] Step 9.4: Implement `list_profiles` and `switch_profile` tools.
- [ ] Step 9.5: Test end-to-end: configure Claude Code to use the MCP server; ask Claude to "run the date command".
