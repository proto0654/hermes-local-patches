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

| `0004-feat-config-opencode-dock-window-toggle.patch` | `feat(config)`: adds `opencode.dock_window` (boolean, default `false`) to `DEFAULT_CONFIG` and to the config schema, so the desktop Settings pane renders it as one toggle. |
| `0005-feat-desktop-label-the-opencode-dock-toggle.patch` | `feat(desktop)`: labels the row "OpenCode Dock Window" (RU: «Окно OpenCode (док)») and files it under *Delegation* instead of a bare "Dock Window" buried mid-list under *Agent*. Needs a desktop rebuild. |
| `0006-fix-kanban-stamp-evidence-on-human-done.patch` | `fix(kanban)`: closing a card by hand from the board was impossible — `complete_task()` accepted only running/ready/blocked/review (so a `todo`/`triage` drop failed with "status transition to 'done' not valid"), and the drawer has no field for a result (so the drop failed with "has no result or summary evidence"). Backlog states are closable now, and the dashboard's explicit operator action stamps `Закрыто вручную из борды — <time>, закрыл <profile>`. Worker completions still need real evidence. Backend-only, no desktop rebuild. |
| `0007-feat-desktop-copy-prompt-from-a-kanban-card.patch` | `feat(desktop)`: a copy affordance on every board card (hover, next to the short id) puts a paste-ready prompt on the clipboard — `<заголовок карточки> — выполняем карточку t_… из канбана «…»` (title first, so the new session gets a readable name) — so work can be handed to the agent without retyping ids. Board name and project come from the cached `/boards` query. Needs a desktop rebuild (`npm run pack`). |

Apply:

```bash
cd /path/to/hermes-agent
git am /path/to/hermes-local-patches/patches/*.patch
```

Backend-only patches (`0001`, `0006`) take effect after the gateway restarts: run
`systemctl --user restart hermes-gateway` from a shell **outside** the agent. The running
gateway refuses to stop itself (the restart command would die with it). Rebuild the desktop
only for the renderer patches (`0002`–`0005`).

The desktop UI is Electron code, so a rebuild is required for the renderer half:

```bash
cd apps/desktop && npm run pack     # or: npm run build && npm run dist
```

### `bin/kanban-done` — close a board card with generated evidence

`kd t_1234abcd "что сделано"` finds the card's board and project, stamps the time and the
operator into the result line and calls `hermes kanban complete`. The alias `kd` goes into
`~/.bashrc`. Needed because a completion without evidence is refused, and the board drawer
has no field for one (see patch `0006`).

### `bin/` — OpenCode helpers for the Hermes desktop workflow

| Script | Purpose |
|---|---|
| `oc` | Run OpenCode headlessly with a short label: `oc <label> <prompt-file> [opencode run flags]`. The prompt comes from a file, so the mirrored terminal tab has a readable name instead of the whole command line. |
| `oc-tui` | Start the OpenCode TUI *with* a task and mark it: `oc-tui <label> <prompt-file> [dir]`. Uses `opencode --prompt`, exports `OC_TUI=1`, and spawns its own reaper for that PID. This is the "watch it work" mode — colours and expanded tool calls, mirrored read-only into the Hermes terminal pane. |
| `oc-window` | OpenCode TUI as a real tiled window in the bottom third of the screen (`oc-window <label> <prompt-file> [dir]`). Uses `oc-tui` inside `foot`, hands the window to `oc-dock-align`, and lets `oc-ptyfilter` close it once the run has been silent for `OC_TUI_GRACE` seconds (the dock sets 25). |
| `oc-ptyfilter` | Runs a command under a fresh PTY and filters `?1003h` (any-motion mouse reporting) out of its output, keeping click/wheel reporting. `oc-tui` uses it. A background job's stdin is `/dev/null`, so both input and the window size come from `/dev/tty`; without that the TUI renders at a 120x40 fallback and the transcript looks cut off. |
| `oc-delegate` | Single entry point for a run: reads `opencode.dock_window` from `~/.hermes/config.yaml` and execs `oc-window` (own docked window) or `oc-tui` (terminal pane). `oc-delegate <label> <prompt-file> [dir]`. |
| `oc-dock-align` | Aligns the dock window into the bottom third (full width) with one `hyprctl --batch` per step — focus + `togglesplit`, then resize. Polls every 25 ms and targets only the *newly appeared* window, so a leftover dock from an earlier run cannot steal the alignment. Visible rearrange: ~0.1 s. |
| `oc-tui-reap` | Manual sweep over agent-launched TUIs (`OC_TUI=1`). Not needed in the normal path any more: `oc-ptyfilter` closes a TUI by itself once its child has been silent for `OC_TUI_GRACE` seconds — a DB-based idle check could see *another* run in the same directory as active and leave the window hanging. |
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

## One toggle in Settings

`opencode.dock_window` (Settings → Agent, off by default; `hermes config set opencode.dock_window true`
from the CLI) decides how a run is shown. `oc-delegate` is the only entry point that matters:

```bash
oc-delegate "<label>" <prompt-file> [dir]   # dock when the toggle is on, terminal pane when it is off
```

The flag is read at launch, so flipping it does not touch a run that is already going.

## Lifetime and timings

- `OC_TUI_GRACE` (seconds, default 120; `oc-window` exports 25) — how long the child may be
  silent before `oc-ptyfilter` closes the TUI. This is an output-idle check per process, not a
  lookup in the shared session DB: with two runs in one directory the DB always looked "active"
  and a finished window could hang.
- Silence alone is not enough: the filter also requires that the TUI has **no live descendants**.
  A long tool call (`sleep 40`, a build, a test suite) keeps its child process alive, so it is
  never killed mid-flight even if that tool prints nothing. Measured: during a real
  `sleep 40` step the TUI never went silent for 2 s (it animates the tool block), and with
  `OC_TUI_GRACE=15` the window stayed up for the whole run and closed 15 s after the last byte.
- Measured on the dock (`oc-window`, 1920x1080): the window appears at ~0.55 s, lands in the
  bottom third at ~0.68 s, and is gone 25 s after the last output. The 0.13 s rearrange is the
  floor for a tiled window — Hyprland cannot open one at a chosen position and ratio.

Commits that introduced these behaviours: `bace908` (batched alignment + idle close inside the
filter), `276e8d9` (window size taken from `/dev/tty`, otherwise the TUI renders at a 120x40
fallback and the transcript looks cut off).

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
