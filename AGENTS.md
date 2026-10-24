# AGENTS.md

Instructions for AI coding agents (opencode, Claude Code, Codex, and anything else
that reads `AGENTS.md`) working on this repository.

## What this project is

File Minnow is an instant file search app for Linux — an Everything-for-Windows
style tool: index file and folder names once, keep the index current with inotify,
answer queries as you type. Rust, GTK3, a background daemon (`src/service.rs`), a
D-Bus search provider, and a tray. The GUI is event-driven by principle: no timers,
no polling, no animations.

Read [README.md](README.md) and [CONTRIBUTING.md](CONTRIBUTING.md) first; their
principles apply on top of everything here.

## Environment setup

- Requires system GTK3 development packages for the default (GUI) build. Without
  them, prefix any cargo command with `scripts/with-gtk-sdk.sh` — it uses an
  extracted GTK SDK in `.dev/gtk-sdk` (or `$FILE_MINNOW_GTK_SDK`).
- The headless build (`--no-default-features`) must keep compiling; build it into a
  separate target directory so it never replaces the GUI test binary.

## Commands

Before sending any change:

```bash
cargo fmt --all -- --check
cargo clippy --locked --all-targets -- -D warnings
cargo test --locked --all-targets
cargo build --locked --no-default-features --target-dir target/headless
```

UI and desktop-integration changes additionally need the isolated UI tests; they
start their own Xvfb display, D-Bus session, tray host and portal fixtures:

```bash
python3 scripts/test_ui.py --xvfb /path/to/Xvfb            # window, search, options, menus
python3 scripts/test_ui.py --xvfb /path/to/Xvfb --tray     # tray, global shortcut, close to tray
python3 scripts/test_ui.py --xvfb /path/to/Xvfb --tray --portal --daemon-first
python3 scripts/test_ui.py --xvfb /path/to/Xvfb --confined # snap: portals for open, show, trash
python3 scripts/test_focus.py --xvfb /path/to/Xvfb         # focus with Muffin as window manager
python3 scripts/test_search_provider.py                    # GNOME Shell / KRunner over D-Bus
```

They need `xdotool`, `xclip`, ImageMagick's `import`, and Python with `dbus` and
`gi`. After changing `packaging/file-minnow.svg`, run
`python3 scripts/render_icons.py`.

### Packaging

`scripts/package.py` packages an existing build; it does not run cargo. Build
first, then package:

```bash
cargo build --release --locked
python3 scripts/package.py --format deb    # or 'tar' / 'all' -> dist/
```

It reads `target/release/file-minnow` (`--binary` overrides it), needs `objdump`
from binutils to compute the minimum glibc, and takes the version from
`Cargo.toml` (`--version` overrides). A fork carrying its own changes should bump
the version (e.g. `--version 0.9.0+runcount1`) so apt can order it against
upstream packages of the same number. `scripts/test_deb.sh` only runs inside an
ephemeral distro container (`MINNOW_PACKAGE_TEST=1`); it refuses host
installation.

## How the code fits together

- `src/index.rs` + `src/table.rs` — the path-ordered in-memory index; splices keep
  name/path order without re-sorting everything. Do not put mutable per-file
  bookkeeping (counts, flags) into index rows; keep side tables instead.
- `src/service.rs` — the daemon that owns the index; `src/client.rs` talks to it.
- `src/query.rs` — parser and bitset evaluator; they must stay in agreement
  (`query::tests`). OR binds tighter than the implicit AND.
- `src/ui.rs` — the whole window; result columns are declared in its `COLUMNS`
  table (title, width, sort key) and rendered by column number from the stored
  `SearchResponse` into a `gtk::ListStore`.
- `src/config.rs` — settings; `atomic_write` is the sanctioned way to persist
  side JSON files (history, run counts).
- `src/history.rs` / `src/run_counts.rs` — small JSON side tables beside the
  settings file, each keyed its own way, pruned by size. Follow this pattern for
  new user-frequency data.
- `src/watch.rs` — the folder watcher. **Never subscribe to open or access
  events**; our own scans would generate events and trigger endless rescans
  (guarded by `settings_runtime::own_scans_do_not_trigger_more_scans`).

## Conventions

- **Paths are bytes.** Keep raw Linux path bytes end to end; never convert paths
  lossily. Directory symlinks are not followed.
- **The daemon protocol stays compatible where it can.** GUI and daemon are
  separate processes that can outlive each other's version. New response
  fields need `#[serde(default)]` (like `SearchResponse.counts`); unknown
  requests from a newer GUI simply error out against an older daemon, so any
  `Request` or `Sort` addition means telling users to restart a running daemon
  after an upgrade (the deb/snap never restarts it: packages do not touch
  autostart or running services).
- **Speed and resource use come first.** Check idle CPU (the UI tests report it)
  after touching background loops. Sort expensive or non-index fields (like run
  counts) over the returned results, not during the index walk.
- **Work on any distribution.** No runtime behaviour tied to Debian, systemd or
  one desktop. Cinnamon is the main test desktop; document X11, Wayland, tray
  and portal differences honestly.
- **Snap confinement.** `file_ops::confined()` gates portal-vs-direct behaviour;
  eBPF or `/proc` process spying is not an option there. Anything that counts
  runs must keep working through the portal path.
- Tests use temporary folders and data directories, never a real home folder.
- Installation, autostart and global settings stay explicit user choices;
  packages never enable them.
- On X11, window activation goes through `desktop::x11_activate` (a pager-style
  `_NET_ACTIVE_WINDOW` request). A plain `present()` is refused by
  focus-stealing prevention when the request comes from the global shortcut, the
  tray or a second launch.
- Do not claim full compatibility with Everything; document differences.

## Current work: run counts

Run counts mirror Everything for Windows. The index service owns the counters
(`runtime::Handle::counts`, persisted to `run_counts.json` beside the settings);
the GUI records through `Backend::record_runs` (a `record_runs` socket request
remotely, direct locally) on every app-mediated open: Enter/double-click,
`Open with`, AppChooser, and portal `Open`/`OpenWith`. Folders opened as
"parent" are not counted. The file sits beside the settings file, which for the
default data directory means `~/.config/file-minnow/run_counts.json` (XDG
config, **not** `~/.cache/file-minnow/` where `index.bin` lives —
`ConfigStore::for_data_dir` makes that split); with `--data-dir <dir>` both go
under `<dir>`. Peek there while debugging, it is plain JSON. Created lazily on
the first recorded open.
`SearchResponse.counts` aligns with `entries` and
feeds the "Run count" column (hidden by default for new window states; first
click sorts most-used first via `Sort::RunCount`, which the query engine
post-sorts — the count is not in the index, so that sort allocates one path
lookup per match). System-wide counting (`/proc` polling in the daemon) is not
implemented and may not fit the snap sandbox.

Surfaces that search without a counter store (they show blank counts and fall
back to path order for `Sort::RunCount`): the D-Bus search provider
(`src/search_provider.rs`) and the `file-minnow search` CLI (`src/main.rs`)
pass `None`; wiring them is a three-line change each if wanted.
`query::search` takes the store as its fourth argument — every caller must be
updated when that signature changes (`tests/core.rs::run_count_sort_and_page_counts`
covers the sort and the page counts).

## Working on the user's machine

The person running you is probably using the same desktop. Do not take
screenshots, move focus, open windows or send input on their display — use
`scripts/test_ui.py` and `scripts/test_focus.py`, which run on a private Xvfb
display and D-Bus session. Measure performance at realistic scale (1M+ entries,
long paths) with `examples/bench_index.rs`, `examples/bench_update.rs` and
`scripts/measure_gui.py`, not only on small fixtures. Do not claim numbers you
have not measured.

Note: `cargo test --lib` run as one parallel suite currently fails two
environment-sensitive tests (`file_ops::tests::guesses_types_from_names_only`,
`thumbnails::tests::cached_made_and_failed_thumbnails`) on some desktops; both
pass in isolation and fail identically on the pristine checkout. That is a
pre-existing interaction, not something your change introduced — verify with a
filtered run before chasing it.
