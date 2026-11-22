# hermes-local-patches

A small, self-contained patch set for a local [Hermes Agent](https://github.com/NousResearch/hermes-agent)
checkout, plus the author's OpenCode helper scripts. This is **not** a fork and contains no
upstream source — only diffs you apply to your own clone.

## What's inside

### `patches/` — apply to your Hermes Agent checkout

Base: the checkout these were made against reports `v0.21.1` (`2237be3559 chore: release v0.21.1`).
On a newer base some hunks may need a rebase.

| Patch | What it does |
|---|---|
| `0001-fix-desktop-reveal-a-collapsed-terminal-pane-instead.patch` | `fix(desktop)`: `focus_pane('terminal')` used to flip the takeover atom, which is a no-op once the atom is already `true` and never undoes a collapsed tool pane — the agent got `success` while nothing painted. The reveal now clears the collapse (`setPaneCollapsed`) and then runs the tree-aware reveal. |
| `0002-feat-desktop-reveal_terminal_tab-tool-for-background.patch` | `feat(desktop)`: a `desktop_ui` tool `reveal_terminal_tab(process_id, title?)` that emits `terminal.reveal`, and the renderer handler that fronts the read-only mirror tab of a `terminal(background=true)` process. Active session only, so a background turn never steals the foreground; it never types into the shell. |
| `0003-docs-reference-list-reveal_terminal_tab-in-the-deskt.patch` | `docs(reference)`: lists the new tool in `tools-reference.md` and `toolsets-reference.md`. |

Apply:

```bash
cd /path/to/hermes-agent
git am /path/to/hermes-local-patches/patches/*.patch
```

The desktop UI is Electron code, so a rebuild is required for the renderer half:

```bash
cd apps/desktop && npm run pack     # or: npm run build && npm run dist
```

### `bin/` — OpenCode helpers for the Hermes desktop workflow

| Script | Purpose |
|---|---|
| `oc` | Run OpenCode headlessly with a short label: `oc <label> <prompt-file> [opencode run flags]`. The prompt comes from a file, so the mirrored terminal tab has a readable name instead of the whole command line. |
| `oc-tui` | Start the OpenCode TUI *with* a task and mark it: `oc-tui <label> <prompt-file> [dir]`. Uses `opencode --prompt`, exports `OC_TUI=1`, and spawns its own reaper for that PID. This is the "watch it work" mode — colours and expanded tool calls, mirrored read-only into the Hermes terminal pane. |
| `oc-window` | OpenCode TUI as a real tiled window in the bottom third of the screen (`oc-window <label> <prompt-file> [dir]`). Uses `oc-tui` inside `foot`, then aligns the window with `hl.dsp` dispatches, and lets the reaper close it when the run goes idle. |
| `oc-ptyfilter` | Runs a command under a fresh PTY and filters `?1003h` (any-motion mouse reporting) out of its output, keeping click/wheel reporting. `oc-tui` uses it; a background job's stdin is `/dev/null`, so input comes from `/dev/tty`. |
| `oc-tui-reap` | Kill agent-launched TUIs once their session has been idle: `--watch <pid> [--grace 120]` (steward for one TUI) or a one-shot sweep over every `OC_TUI=1` process. TUIs without the marker (your own) are never touched. |
| `oc-watch` | Follow another session's OpenCode run as text (reads `~/.local/share/opencode/opencode.db`), printing `… думает:` / `✓ shell:` lines and switching to a newer session by itself. |
| `oc-foot-guard` | Watches Hyprland's event socket and moves foreign OpenCode windows (`foot` with an `OpenCode…` title, e.g. from another Hermes session) to a hidden `special:opencode` workspace so they stop re-tiling the desktop. Own dock windows (`oc-tui-dock`) are ignored. Ships with a systemd user unit in `systemd/`. |
| `oc-open` | Open the real interactive TUI in a floating `foot` window (`-s <session-id>` to resume one), reused if already open. |

Install:

```bash
cp bin/* ~/.local/bin/ && chmod +x ~/.local/bin/oc*
```

## Getting expanded tool calls

`opencode run` drops ANSI when stdout is a pipe, so a mirrored `run` looks grey. Colour (and
click-free expansion) comes from the TUI plus this setting in `~/.config/opencode/cli.json`:

```json
{ "session": { "verbosity": "high" } }
```

Mouse capture stays on (wheel and clicks work), but `oc-tui` wraps the CLI in `oc-ptyfilter`,
which strips the TUI's `?1003h` request ("report every pointer motion") out of its output and
leaves `?1000h` / `?1002h` / `?1006h` intact. Result: the wheel scrolls, clicks work, and
hovering over the window no longer feeds SGR garbage into the prompt. `"mouse": false` in the
config is the sledgehammer if you prefer no mouse at all.

`low` collapses each run of tools and thoughts into one line; `high` opens the exploration and
instruction groups, which is what makes a read-only mirror readable.

## Desktop behaviour this expects

- `input:follow_mouse = 0` — focus on click only, so pointer movement over a mirrored or
  docked TUI does not hand it focus and mouse events.
- `SUPER + ALT + O` toggles the hidden `special:opencode` workspace where foreign OpenCode
  windows are parked.
- Install the guard: `cp systemd/oc-foot-guard.service ~/.config/systemd/user/ &&
  systemctl --user daemon-reload && systemctl --user enable --now oc-foot-guard`.

Exact Hyprland lines these scripts assume (Lua config, Hyprland 0.5x):

```lua
-- hyprland.lua — the dock must not grab focus when it opens; oc-window aligns it itself
o.window("oc-tui-dock", { no_initial_focus = true })

-- input.lua — hover no longer focuses, so a mirrored TUI stays passive until clicked
hl.config({ input = { follow_mouse = 0 } })

-- bindings.lua — peek at the parked foreign windows
o.bind("SUPER + ALT + O", "OpenCode windows (special workspace)",
       "hyprctl dispatch 'hl.dsp.workspace.toggle_special(\"opencode\")'")
```

Gotcha: a plain window rule cannot park the foreign windows —
`o.window({ class = "^foot$", title = "^OpenCode" }, { workspace = "special:opencode" })`
never matches, because `foot` sets its title *after* the window is mapped and `workspace`
rules are applied at map time. Hence `oc-foot-guard`, which listens on Hyprland's event
socket and moves the window once the title arrives (with `follow = false`, or the next new
window would open on the special workspace).

## Notes

- No credentials, tokens, or machine-specific secrets live in this repo; the scripts read the
  local OpenCode database and call the local CLI only.
- Verified against `opencode` 2.3.2 on Arch/Omarchy (Hyprland), Python 3.11, Node 22.

MIT.
