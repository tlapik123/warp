# Opencode Agent Session: Ctrl+C Conditional Behavior

## Context
This file tracks progress across opencode agent sessions on the Warp codebase.

## Active Task
Implement conditional Ctrl+C behavior in the terminal input editor:
1. **Text selected** → Copy selection to clipboard + clear selection (keep text)
2. **No selection, buffer has text** → Copy entire buffer to clipboard + clear buffer
3. **Buffer empty** → Send SIGINT to running process (double-tap Ctrl+C)

## Previous Exploration Results

### Key Code Locations
- `app/src/editor/view/mod.rs:4288` — `handle_ctrl_c()` (editor Ctrl+C handler)
- `app/src/terminal/view.rs:7845` — `handle_ctrl_c_input_event()` (terminal Ctrl+C handler)
- `app/src/terminal/view.rs:7888` — `ctrl_c()` (terminal copy-or-SIGINT logic)
- `app/src/editor/view/mod.rs:4154` — `copy()` — writes to system clipboard via `ctx.clipboard().write()`
- `app/src/editor/view/model/mod.rs:2637` — `shell_cut()` — writes to internal clipboard only

### Architecture Understanding
- `FixedBinding` — hardcoded, cannot be changed by user (e.g., `ctrl-y` → Yank)
- `EditableBinding` — customizable in Settings > Keyboard shortcuts (e.g., `ctrl-y` → Delete all left)
- Editable bindings always override fixed bindings
- `Ctrl+U` binding: `editor_view:clear_and_copy_lines` with description "Copy and clear selected lines"
  - Currently expands selection to full lines then uses `shell_cut()` → internal clipboard only
  - Does NOT copy to system clipboard
  - This is by design per code comment at line 1762
- `Ctrl+Y` on Linux/Windows: bound to "Delete all left" (editable binding overrides fixed Yank binding)
  - Internal clipboard paste is therefore inaccessible on Linux/Windows by default
- `Ctrl+C` on Windows: copies selected text if any, otherwise clears buffer
- `Ctrl+C` on Linux/macOS: always clears buffer, never copies selection

### Related Issues Found
- #61 "KB: Ctrl + U/K doesn't cut to clipboard" — closed as completed
- Users want Ctrl+U to write to system clipboard (it doesn't)
- Users want Ctrl+Y to paste internal clipboard on Linux (editable binding blocks it)

## Implementation Plan
1. **Modify `handle_ctrl_c()` in `app/src/editor/view/mod.rs`**
   - Add case for text selection → copy + clear selection
   - Add case for buffer with text, no selection → copy all + clear buffer  
   - Keep existing case for empty buffer → emit CtrlC event for SIGINT
   - Remove `#[cfg(windows)]` guard to make copy-on-selection universal

2. **Unify platform-specific Ctrl+C in terminal view**
   - Remove `#[cfg(windows)]` / `#[cfg(not(windows))]` split in `app/src/terminal/view.rs`
   - Make the copy-on-selection behavior universal across all platforms

3. **Update tests**
   - Find Ctrl+C tests in `app/src/editor/view/vim_handler_tests.rs`
   - Find Ctrl+C tests in `app/src/terminal/view_tests.rs`
   - Update expectations to match new behavior

4. **Build and verify**
   - Run relevant test suite
   - Build and test manually if possible

## TODOs (DONE)
- [x] Create AGENTS.md (this file)
- [x] Modify `handle_ctrl_c()` in editor view
- [x] Remove `#[cfg(windows)]` guard and unify behavior
- [x] Handle vim visual mode selection
- [x] Handle agent view edge cases
- [x] Commit changes

## Testing Required
- [ ] Build and run `cargo test -p warp editor::view::vim_handler_tests`
- [ ] Build and test manually

## Open Questions
1. Should this be universal (all platforms) or Linux-only?
2. Should visual mode in vim exit after copy?
3. Should Ctrl+C still exit agent view on first press?
