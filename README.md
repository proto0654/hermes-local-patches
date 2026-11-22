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
{ "session": { "verbosity": "high" }, "mouse": false }
```

`mouse: false` turns off terminal mouse capture. With capture on (the default) the TUI
requests X10/SGR mouse reporting, and pointer movement over a mirrored or docked window gets
inserted into the prompt as escape-sequence garbage. The cost is that the wheel no longer
scrolls inside the TUI — use the keyboard there.

`low` collapses each run of tools and thoughts into one line; `high` opens the exploration and
instruction groups, which is what makes a read-only mirror readable.

## Desktop behaviour this expects

- `input:follow_mouse = 0` — focus on click only, so pointer movement over a mirrored or
  docked TUI does not hand it focus and mouse events.
- `SUPER + ALT + O` toggles the hidden `special:opencode` workspace where foreign OpenCode
  windows are parked.
- Install the guard: `cp systemd/oc-foot-guard.service ~/.config/systemd/user/ &&
  systemctl --user daemon-reload && systemctl --user enable --now oc-foot-guard`.

## Notes

- No credentials, tokens, or machine-specific secrets live in this repo; the scripts read the
  local OpenCode database and call the local CLI only.
- Verified against `opencode` 2.3.2 on Arch/Omarchy (Hyprland), Python 3.11, Node 22.

MIT.
