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
- The headless build (CLI and daemon, `--no-default-features`) must keep compiling;
  build it into a separate target directory so it never replaces the GUI test binary.
- There is no CI workflow: run the commands below yourself.

## Commands

Before sending any change:

```bash
cargo fmt --all -- --check
cargo clippy --locked --all-targets -- -D warnings
cargo test --locked --all-targets
cargo build --locked --no-default-features --target-dir target/headless
```

For one test, target the file (`tests/core.rs`, etc.) or the library:
`cargo test --locked --test core run_count_sort_and_page_counts`,
`cargo test --locked --lib <filter>`.

UI and desktop-integration changes additionally need the isolated UI tests; they
start their own Xvfb display, D-Bus session, tray host and portal fixtures. Build
the binary they run first — they default to `target/debug/file-minnow` and do not
build it (`--binary` overrides):

```bash
cargo build --locked
python3 scripts/test_ui.py --xvfb /path/to/Xvfb            # window, search, options, menus
python3 scripts/test_ui.py --xvfb /path/to/Xvfb --tray     # tray, global shortcut, close to tray
python3 scripts/test_ui.py --xvfb /path/to/Xvfb --tray --portal --daemon-first
python3 scripts/test_ui.py --xvfb /path/to/Xvfb --confined # confined: portals for open, show, trash
python3 scripts/test_focus.py --xvfb /path/to/Xvfb         # focus with Muffin as window manager
python3 scripts/test_search_provider.py                    # GNOME Shell / KRunner over D-Bus
```

They need `xdotool`, `xclip`, ImageMagick's `import`, and Python with `dbus` and
`gi`. After changing `packaging/file-minnow.svg`, run
`python3 scripts/render_icons.py`.

### Packaging

`.deb` is the release package; there is no snap. `scripts/package.py` packages an
existing build and does not run cargo. Build first, then package:

```bash
cargo build --release --locked
python3 scripts/package.py --format deb    # or 'tar' / 'all' -> dist/
```

It reads `target/release/file-minnow` (`--binary` overrides it), needs `objdump`
from binutils to compute the minimum glibc, and takes the version from
`Cargo.toml` (`--version` overrides). A fork carrying its own changes should bump
the version (e.g. `--version 0.9.0+runcount1`) so apt can order it against
upstream packages of the same number.

The `.deb` installs `/etc/xdg/autostart/file-minnow.desktop`, so the app starts in
the tray at login for every user. The app's **Options → Interface → Start File
Minnow at login** toggle writes a per-user entry, or a `Hidden=true` override when
the system entry exists. It never starts the app immediately and never enables the
systemd user unit. `scripts/test_deb.sh` only runs inside an ephemeral distro
container (`MINNOW_PACKAGE_TEST=1`); it refuses host installation.

## How the code fits together

- `src/index.rs` + `src/table.rs` — the path-ordered in-memory index; splices keep
  name/path order without re-sorting everything. Do not put mutable per-file
  bookkeeping (counts, flags) into index rows; keep side tables instead.
- `src/store.rs` — `index.bin`, the writer lock, socket paths and quarantine of
  unreadable indexes.
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

The default data directory is the XDG cache (`~/.cache/file-minnow`) and holds
`index.bin`; `settings.json`, `history.json` and `run_counts.json` live in XDG
config (`~/.config/file-minnow`) — `ConfigStore::for_data_dir` makes that split.
`--data-dir DIR` puts every file under `DIR`. Use it for manual checks.

## Conventions

- **Paths are bytes.** Keep raw Linux path bytes end to end; never convert paths
  lossily. Directory symlinks are not followed.
- **The daemon protocol stays compatible where it can.** GUI and daemon are
  separate processes that can outlive each other's version. New response
  fields need `#[serde(default)]` (like `SearchResponse.counts`); unknown
  requests from a newer GUI simply error out against an older daemon, so any
  `Request` or `Sort` addition means telling users to restart a running daemon
  after an upgrade (the deb never restarts a running daemon; it only installs
  the login entry above).
- **Speed and resource use come first.** Check idle CPU (the UI tests report it)
  after touching background loops. Sort expensive or non-index fields (like run
  counts) over the returned results, not during the index walk.
- **Work on any distribution.** No runtime behaviour tied to Debian, systemd or
  one desktop. Cinnamon is the main test desktop; document X11, Wayland, tray
  and portal differences honestly.
- **Confined installs.** `file_ops::confined()` gates portal-vs-direct behaviour;
  the `--confined` UI test sets `SNAP` to exercise it. eBPF or `/proc` process
  spying is not an option there. Anything that counts runs must keep working
  through the portal path.
- Tests use temporary folders and data directories, never a real home folder.
- Installation and global settings stay explicit user choices, except the
  package's login autostart described above. The background systemd user unit is
  never enabled by the package.
- On X11, window activation goes through `desktop::x11_activate` (a pager-style
  `_NET_ACTIVE_WINDOW` request). A plain `present()` is refused by
  focus-stealing prevention when the request comes from the global shortcut, the
  tray or a second launch.
- Do not claim full compatibility with Everything; document differences.

## Run counts

The counters are a small JSON side table beside the settings (default
`~/.config/file-minnow/run_counts.json`, at most 5,000 paths, least recently used
evicted), never index rows. The service owns them (`runtime::Handle::counts`); the
GUI records app-mediated opens through `Backend::record_runs` (`record_runs` over
the socket when remote) for Enter/double-click, `Open with`, AppChooser and portal
`Open`/`OpenWith`. Parent-folder opens are not counted. System-wide `/proc`
counting is not implemented and may not fit confined installs.

The window opens with `Sort::RunCount` descending on every launch and every return
from the tray; a column sort clicked during the session is not saved in
`window.json`. Counts are not in the index: the sort finds each counted path with
`Table::lower_bound`, then appends uncounted rows in path order (prepends them when
ascending). An older daemon rejects the sort, and the window keeps name order for
the session with a "restart the daemon" hint.

The D-Bus provider (`src/search_provider.rs`) and the `file-minnow search` CLI pass
`None` for counters, so they show blank counts and use path order for
`Sort::RunCount`. `query::search` takes the counter store as its fourth argument;
update every caller when that signature changes
(`tests/core.rs::run_count_sort_and_page_counts` covers the sort and page counts).

## Working on the user's machine

The person running you is probably using the same desktop. Do not take
screenshots, move focus, open windows or send input on their display — use
`scripts/test_ui.py` and `scripts/test_focus.py`, which run on a private Xvfb
display and D-Bus session. For manual CLI checks, pass `--data-dir` under
`/tmp/opencode` so you do not touch their real index or settings. Measure
performance at realistic scale (1M+ entries, long paths) with the commands in
`docs/performance.md` (`bench_index`, `bench_search`, `bench_update`,
`bench_startup`, `scripts/measure_gui.py`), not only on small fixtures. Do not
claim numbers you have not measured.

Note: `cargo test --lib` run as one parallel suite currently fails two
environment-sensitive tests (`file_ops::tests::guesses_types_from_names_only`,
`thumbnails::tests::cached_made_and_failed_thumbnails`) on some desktops; both
pass in isolation and fail identically on the pristine checkout. That is a
pre-existing interaction, not something your change introduced — verify with a
filtered run before chasing it.

Inotify tests can also fail with `ENOSPC` / `folder watch limit reached` when
other desktop apps have exhausted `fs.inotify.max_user_instances`; close some
apps, stop a running File Minnow, or raise the limit before rerunning them.
