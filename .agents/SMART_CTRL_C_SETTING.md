# Opencode Agent Session: Toggleable "Smart Ctrl+C" Setting

## Context  
This session builds on the completed "Conditional Ctrl+C Behavior" feature (commit 1b3fa903). The feature is now hardcoded. We want to make it a user-toggleable setting called "Smart Ctrl+C" that is enabled by default.

## Completed Work
- Conditional Ctrl+C behavior implemented in `app/src/editor/view/mod.rs` (commit 1b3fa903)
  - Text selected → copy to clipboard + clear selection
  - No selection, buffer has text → copy entire buffer + clear buffer
  - Buffer empty → emit CtrlC event → SIGINT to running process
- Platform-specific `#[cfg(windows)]` guards removed from terminal view
- AGENTS.md tracking file created

## Current Problem
The feature is currently **always enabled** (hardcoded). We need to add a toggle so users can disable it if they prefer the old behavior (Ctrl+C always clears buffer).

## Proposed Setting

### Setting Design
| Field | Value |
|---|---|
| **Internal name** | `smart_ctrl_c` |
| **Display name** | "Smart Ctrl+C" |
| **UI Description** | "When enabled, Ctrl+C copies selected text to the clipboard. If no text is selected, it copies the entire buffer and clears it. When the buffer is empty, it sends Ctrl+C to stop running commands." |
| **Default** | `true` (new behavior enabled by default) |
| **Settings group** | `AppEditorSettings` |
| **TOML path** | `terminal.input.smart_ctrl_c` |
| **Platforms** | All (`SupportedPlatforms::ALL`) |
| **Cloud sync** | Global (`SyncToCloud::Globally(RespectUserSyncSetting::Yes)`) |

### Behavior When Enabled (Default)
| Editor State | Ctrl+C Action |
|---|---|
| **Text selected** | Copy selection to system clipboard + clear selection (text stays) |
| **No selection, buffer has text** | Copy entire buffer to clipboard + clear buffer |
| **Buffer empty** | Emit CtrlC event → terminal sends SIGINT to running process |

### Behavior When Disabled
- Always clears the buffer (old behavior, universal across platforms)
- Never copies to clipboard
- If buffer is empty, sends SIGINT

## Implementation Plan

### Part 1: Define the Setting
**File:** `app/src/settings/editor.rs`
1. Add `smart_ctrl_c` entry to `define_settings_group!(AppEditorSettings, ...)`
2. Add `smart_ctrl_c_enabled()` helper method to `impl AppEditorSettings`

### Part 2: Add UI Toggle Widget
**File:** `app/src/settings_view/features_page.rs`
1. Import `SmartCtrlC` from settings module
2. Add `ToggleSmartCtrlC` variant to `FeaturesPageAction` enum
3. Add telemetry event handler in `features_page_action` match block
4. Add action handler in `handle_features_page_action`
5. Add `SmartCtrlCWidget` struct + `SettingsWidget` impl
6. Register widget in `render_body_content` (under Terminal Input section)

### Part 3: Wire Setting to Feature Code
**File:** `app/src/editor/view/mod.rs` — `handle_ctrl_c()`
- Wrap new conditional logic in `if AppEditorSettings::as_ref(ctx).smart_ctrl_c_enabled() { ... } else { ... }`
- Keep old behavior in `else` branch

**File:** `app/src/terminal/view.rs` — `ctrl_c_internal()`
- Same conditional wrapping for terminal output copy behavior

### Part 4: Update Documentation
- Update `AGENTS.md` with completion status
- Update `settings.md` or similar docs if available

## Files to Modify
1. `app/src/settings/editor.rs` — Add setting to define_settings_group!
2. `app/src/settings_view/features_page.rs` — Add UI toggle (~6 locations)
3. `app/src/editor/view/mod.rs` — Wrap behavior in conditional
4. `app/src/terminal/view.rs` — Wrap behavior in conditional

## Testing Plan
1. **Default enabled:** Verify new Ctrl+C behavior works
2. **Disabled via settings:** Verify Ctrl+C always clears buffer (old behavior)
3. **Settings persistence:** Toggle setting, restart app, verify state preserved
4. **Cloud sync:** Verify setting syncs across devices
5. **Platform consistency:** Verify works on Linux, macOS, Windows

## Open Questions
1. Should there be separate settings for editor vs terminal output selections?
2. Should the description mention the "double Ctrl+C" workflow explicitly?
3. Should we add a keyboard shortcut to toggle the setting on/off?

## Status: PAUSED — Ready to implement when requested
