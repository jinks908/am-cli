# Architecture

# Table of Contents
[Architecture](#architecture)
    [1. Directory map](#1.-directory-map)
    [2. Layering](#2.-layering)
    [3. Core mechanisms](#3.-core-mechanisms)
        [3.1 AppleScript bridge — `src/music/mod.rs`](#3.1-applescript-bridge-—-`src/music/mod.rs`)
        [3.2 Simple playback commands — `src/main.rs`](#3.2-simple-playback-commands-—-`src/main.rs`)
        [3.3 `am now` — `src/cmd/now.rs`](#3.3-`am-now`-—-`src/cmd/now.rs`)
        [3.4 Metadata / Song.link — `src/music/metadata.rs`](#3.4-metadata-/-song.link-—-`src/music/metadata.rs`)
        [3.5 Discord presence — `src/cmd/discord/mod.rs` + `src/rich_presence/`](#3.5-discord-presence-—-`src/cmd/discord/mod.rs`-+-`src/rich_presence/`)
        [3.6 Formatting — `src/format.rs`](#3.6-formatting-—-`src/format.rs`)
        [3.7 Shell completions](#3.7-shell-completions)
    [4. Conventions](#4.-conventions)
    [5. Build, check, release](#5.-build,-check,-release)
    [6. Known quirks and likely bugs](#6.-known-quirks-and-likely-bugs)
    [7. How-to recipes](#7.-how-to-recipes)

---


`am` is a small, single-binary Rust CLI that controls the macOS **Music** app (Apple Music)
through AppleScript, shows "now playing" info, generates Song.link URLs, and runs a Discord
Rich Presence daemon.

- **Fork:** `origin` = `jinks908/am-cli`. Upstream is `ryanccn/am` (no `upstream` remote is
  configured; add one with `git remote add upstream https://github.com/ryanccn/am.git` to pull fixes).
- **Version:** 0.6.4, Rust edition 2024, macOS only (`compile_error!` on other targets, `src/main.rs:83`).
- **Size:** ~1,300 lines of Rust across 14 files. No tests (`cargo test` runs 0 tests).

---

## 1. Directory map

```
src/
├── main.rs                 CLI definition (clap), command dispatch, concise_now_playing()
├── format.rs               Duration formatting helpers ("3m25s", colored and plain)
├── http.rs                 Global reqwest Client (HTTPS-only, LazyLock)
├── cmd/
│   ├── mod.rs              Re-exports `now` and `discord`
│   ├── now.rs              `am now` / `am now --watch` (async state machine + renderer)
│   └── discord/
│       ├── mod.rs          `am discord` presence loop (polls Music every 5s)
│       └── agent.rs        `am discord install|uninstall` (launchd plist)
├── music/
│   ├── mod.rs              AppleScript bridge: tell/tell_raw, Track, Playlist, PlayerState
│   ├── metadata.rs         Apple Music web API: token scraping + catalog search
│   └── models.rs           serde structs for the catalog search response
└── rich_presence/          Vendored + async-ified fork of the `discord-rich-presence` crate
    ├── mod.rs              Re-exports
    ├── ipc_trait.rs        `DiscordIpc` trait: connect/handshake/send/recv/set_activity
    ├── ipc_impl.rs         `DiscordIpcClient` over a Unix socket (discord-ipc-0..9)
    ├── pack_unpack.rs      8-byte frame header (opcode u32 LE + length u32 LE)
    ├── activity.rs         Activity/Assets/Timestamps/Button builders (serde)
    └── errors.rs           `RichPresenceError` (thiserror)
flake.nix                   Nix flake via `ferrix`; exports package + Home Manager module
.github/workflows/          build (cargo auditable, x86_64 + aarch64), check (nix checks), release
```

Module tree is declared in `src/main.rs:14-18` (`cmd`, `format`, `http`, `music`, `rich_presence`).
Everything is a private module of the binary crate — there is no `lib.rs`.

---

## 2. Layering

```
          main.rs  (clap Cli / Commands enum → match dispatch)
             │
   ┌─────────┼───────────────────┐
   ▼         ▼                   ▼
cmd::now   cmd::discord      (simple playback commands inline in main.rs)
   │         │    └── agent.rs → launchctl + ~/Library/LaunchAgents
   │         │
   │         ├──────────► rich_presence  (Discord IPC over Unix socket)
   ▼         ▼
       music  ──────────► osascript (child process per call)
         │
         └── metadata ──► http::HTTP ──► music.apple.com / amp-api-edge.music.apple.com
format.rs  (used by main.rs and cmd::now)
```

Rules the code follows:
- **Only `music/` talks to the Music app.** Everything goes through `music::tell` / `music::tell_raw`.
- **Only `music/metadata.rs` talks to the network.** Through the shared `http::HTTP` client.
- `rich_presence` knows nothing about music; `cmd::discord` glues the two together.

---

## 3. Core mechanisms

### 3.1 AppleScript bridge — `src/music/mod.rs`

- `tell_raw(&[&str])` spawns `osascript` with one `-e` argument per line, returns trimmed stdout,
  and `bail!`s with stderr on non-zero exit (`mod.rs:40`).
- `tell(expr)` wraps a single expression in `tell application "Music" … end tell` (`mod.rs:58`).
- **Every call is a new process** (~tens to hundreds of ms). To fetch several fields cheaply, the code
  builds one multi-line script that concatenates values with `"\n"` and splits on the Rust side
  (see `get_current_track`, `concise_now_playing`, `update_state`). Follow this pattern rather than
  issuing many `tell` calls.
- **Decimal-comma normalization:** numeric results are parsed with `.replace(',', ".")` because
  non-English locales print `123,45`. Do this for any new float parsed from AppleScript (fixed
  upstream in #214).
- `is_running()` uses `pgrep ^Music$` — checking it before `tell` avoids launching Music by accident
  (any `tell application "Music"` will launch the app if it is closed).

Types:
| Type | Fields / notes |
|---|---|
| `Track` | `id` (Music **database id**, local, used for change detection), `name`, `album`, `artist`, `duration: f64` seconds |
| `Playlist` | `name`, `duration: i32` |
| `PlayerState` | `Stopped/Playing/Paused/Forwarding/Rewinding/Unknown`; `FromStr` never errors (unknown → `Unknown`); `to_icon()` returns Nerd Font glyphs; `Display` returns the AppleScript string |

### 3.2 Simple playback commands — `src/main.rs`

`play`, `pause`, `toggle` (alias `p`), `resume`, `back`, `forward`, `next`, `previous` (alias `prev`)
each do: `music::tell("<applescript verb>")` → print a colored status line → `concise_now_playing()`.

`concise_now_playing()` (`main.rs:87`) fetches name/album/artist/duration in one script and prints
two lines. Note it **does not check for the Stopped state**; if nothing is loaded, osascript errors.

### 3.3 `am now` — `src/cmd/now.rs`

Options (`NowOptions`): `--watch/-w`, `--no-nerd-fonts`, `--bar-width <i32>` (default 30).

It is a small actor system on tokio, connected by an `mpsc` channel of `PlaybackStateDelta`:

| Task | Cadence | Job |
|---|---|---|
| Poller | every 5 s | `update_state()`: reads player state, track id + position (one script), playlist name & duration; sends deltas |
| Ticker | every 250 ms | sends `PositionTick` (+0.25 s locally while Playing) and `Render` |
| Reducer | on message | owns `PlaybackState`, applies deltas, renders, decides shutdown |
| Ctrl-C watcher | `--watch` only | waits on `tokio::signal::ctrl_c()`, then sends on the shutdown channel |

Details worth knowing:
- **Track re-fetch handshake:** the poller sends `TrackIDRequestMoreInfo(id)`; the reducer replies on a
  second channel (`tx_request_track`) with `true` if the id changed. Only then is the full track fetched.
  This avoids an extra osascript call every 5 s.
- **Latency compensation:** the osascript round-trip time is added to the reported position (`now.rs:104`).
- **`mystery_counter`:** incremented by `State`, `Track`, `Playlist`, `Position` deltas. In one-shot mode
  (no `--watch`), the reducer only renders once the counter reaches 4 (i.e. a full poll has arrived),
  then sends on the `watch` shutdown channel so all tasks exit.
- **Rendering:** `update_display` collects the frame's lines, builds the whole frame into one buffer,
  and writes + flushes it once (separate writes let the terminal show half-drawn frames, i.e. flicker).
  Watch mode runs in the alternate screen with the cursor hidden (`EnterAlternateScreen` + `Hide` at
  the start of `now()`); each frame is `MoveTo(0, 0)`, then every line followed by
  `Clear(UntilNewLine)`, then `Clear(FromCursorDown)` to remove leftovers when the frame gets shorter.
  No raw mode.
  - Earlier approaches: `SavePosition`/`RestorePosition` saved a fixed screen row and appended frames
    endlessly when the prompt was near the bottom; counting lines and moving up by that count broke
    when a line wrapped. The alternate screen avoids both.
- **Teardown:** on Ctrl-C the watcher task triggers shutdown, `join_all` returns, and `now()` runs
  `Show` + `LeaveAlternateScreen`, which restores the terminal as it was. Task errors are discarded by
  `join_all`, so teardown still runs. A hard kill (SIGKILL) skips it and leaves the cursor hidden;
  run `reset` or `tput cnorm` to restore it.
- Despite the help text ("keyboard-interactive"), **there is no keyboard handling**; the
  `crossterm` `event-stream` feature and `futures` dependency are enabled but unused. Adding key
  controls would mean another task reading `crossterm::event::EventStream`.

### 3.4 Metadata / Song.link — `src/music/metadata.rs`

Used by `am song-link` and the Discord presence.

1. `fetch_token()` — scrapes a **developer JWT** from the Apple Music web player:
   GET `https://music.apple.com/` → `BUNDLE_REGEX` finds the main JS bundle path → GET bundle →
   `TOKEN_REGEX` extracts the token. Cached for process lifetime in a `OnceLock`.
2. `fetch_metadata(track)` — GET `amp-api-edge.music.apple.com/v1/catalog/us/search` with
   `term = "<name> <album> <artist>"`, `limit=1`, `include[songs]=artists`, Bearer token, spoofed
   browser `User-Agent` and `Origin`. Returns `Metadata { album_artwork, artist_artwork, share_url, song_link }`
   (artwork `{w}x{h}` → `512x512`; `song_link = https://song.link/i/<catalog id>`).

**Fragility (most likely thing to break):**
- Both regexes are tied to Apple's minified web bundle. The latest upstream fix (`5dbf923`) was exactly
  this: the bundle path gained `~` and the token regex anchor changed to the
  `"x-apple-jingle-correlation-key"` string. When song-link/artwork fails with
  "could not obtain bundle…" or "could not find … token in bundle", update these regexes first.
- Storefront hardcoded to `us`, locale `en-US`.
- Search is fuzzy and takes the first hit, so local-only tracks or ambiguous names can yield wrong
  links/artwork.
- The token is never refreshed. A long-running `am discord` daemon keeps the first token until restart;
  if it expires, every metadata call fails (presence still works, just without artwork/buttons).

### 3.5 Discord presence — `src/cmd/discord/mod.rs` + `src/rich_presence/`

`am discord` loop (every 5 s, exits on Ctrl-C):
1. If Music isn't running, or state ≠ Playing → `clear_activity()` and mark idle.
2. Read `{player position, player state}` (split on `", "`), then `get_current_track()`.
3. If same track id and position hasn't gone backwards → "ongoing", do nothing.
   Otherwise fetch metadata (failure is logged, non-fatal) and `set_activity` with details = track name,
   state = "artist · album", start/end timestamps, artwork assets, and two buttons
   ("Listen on Apple Music", "View on SongLink").
4. Connection errors (`CouldNotConnect`/`WriteSocketFailed`) are logged once as "Disconnected"
   and retried next tick; `write()` auto-reconnects.

Hardcoded identifiers:
- Discord application/client ID `861702238472241162` (upstream author's Discord app — controls the
  "Playing …" app name shown in Discord). Change at `discord/mod.rs:145` to use your own app.
- Launch agent label `dev.ryanccn.am.discord` (`agent.rs:12`).

IPC (`rich_presence/`):
- Socket path: first set env var of `XDG_RUNTIME_DIR`, `TMPDIR`, `TMP`, `TEMP`, then `discord-ipc-0..9`.
- Frame = 8-byte header (`opcode`, `len`, both u32 LE) + JSON. Opcodes: 0 handshake, 1 frame, 2 close.
- The trait (`ipc_trait.rs`) holds default methods; `ipc_impl.rs` supplies socket I/O. Uses `async_trait`.
- `Party`, `Secrets` and some `Activity` builders are unused (`#[expect(dead_code)]`).

Launch agent (`agent.rs`): writes `~/Library/LaunchAgents/dev.ryanccn.am.discord.plist` with
`ProgramArguments = [current_exe, "discord"]`, `KeepAlive`, `RunAtLoad`, logs to
`~/Library/Logs/am-discord.log`, then `launchctl load -w`. Reinstall if the binary moves.
Nix users get the equivalent via `homeModules.am-discord` in `flake.nix`.

### 3.6 Formatting — `src/format.rs`

- `format_duration(secs, cyan)` → colored `"1h2m3s"` style (used by the progress line).
- `format_duration_plain(secs)` → uncolored, used for concise output and playlist length.

### 3.7 Shell completions

`am completions <shell>` uses `clap_complete::generate` on the `Cli` definition — new subcommands are
picked up automatically.

---

## 4. Conventions

- **Errors:** `eyre::Result` everywhere at the app level, `color_eyre::install()` in `main`;
  `eyre!(...)`/`.ok_or_else(|| eyre!(...))` for missing fields. `rich_presence` uses its own
  `thiserror` enum, which `cmd::discord` downcasts to distinguish connection errors.
- **Output:** use `anstream::println!/eprintln!` (strips color when not a TTY) and `owo_colors`
  (`.green()`, `.dimmed()`, …). Status lines follow `"<Colored verb> rest of sentence"`.
- **Lints:** `clippy::all` + `clippy::pedantic` at warn; `unsafe_code = "forbid"`. Lint suppressions
  use `#[expect(...)]` (not `#[allow]`), scoped to the function.
- **Async:** tokio `full`; child processes via `tokio::process::Command`.
- **Licensing:** every source file starts with an SPDX header
  (`SPDX-FileCopyrightText` / `SPDX-License-Identifier: GPL-3.0-or-later`). Repo is REUSE-compliant
  (`REUSE.toml`); keep the header on new files.
- **TOML formatting:** taplo, column width 200 (`.taplo.toml`).

---

## 5. Build, check, release

| Task | Command |
|---|---|
| Build | `cargo build` (release: `cargo build --release`) |
| Lint | `cargo clippy` — currently 14 warnings (12 `double_must_use` from `async_trait`, 2 `redundant reference in format!`) |
| Format | `cargo fmt` |
| Tests | none exist |
| Nix | `nix build`, `nix flake check` (checks come from `ferrix.lib.mkFlake`) |

CI (`.github/workflows/`): `build.yml` (cargo auditable, both macOS arches, fat LTO),
`check.yml` (runs every `checks.*` flake attr, uploads clippy SARIF), `release.yml`
(on `v*.*.*` tag → build, `cargo publish` to crates.io, GitHub release). These are upstream's
workflows; in this fork, a tag push would try to publish to crates.io under the `am` name and fail
without credentials. Dependabot updates cargo and actions weekly.

**Testing locally:** `osascript` (Apple Events) is blocked inside Claude Code's sandbox, so `am` commands
that touch Music can't be verified from a sandboxed shell; the user has to run them
(e.g. `! cargo run -- now`). Build/clippy/fmt work in the sandbox.

---

## 6. Known quirks and likely bugs

Recorded for future bugfix work; none have been changed.

1. **`format_duration` hours color is inverted** (`format.rs:18-22`): the `cyan == true` branch prints
   hours uncolored and `false` prints cyan, the opposite of the minutes/seconds branches. Only visible
   for tracks ≥ 1 h.
2. **`format_duration_plain` uses `>` instead of `>=`** (`format.rs:55,63`): exactly 60 s shows `60s`,
   exactly 1 h shows `60m0s`.
3. **`am now` when playback is stopped is slow:** a stopped poll only sends `State`, so `mystery_counter`
   grows by 1 per 5 s poll and the one-shot output only appears after ~4 polls (~15 s).
4. **`concise_now_playing` errors when stopped** (no current track) instead of printing a friendly message,
   so `am pause` etc. can report an AppleScript error in that state.
5. **Ctrl-C on `am discord` without Discord running exits with an error:** shutdown calls
   `clear_activity()?` then `close()?` (`discord/mod.rs:187-188`), both of which try to reconnect and fail
   with `Could not connect to IPC socket`. (`close` also `unwrap`s the socket at `ipc_impl.rs:129`, but that
   line is only reached after a successful send.)
6. **`read()` retries on `WriteSocketFailed`, not `ReadSocketFailed`** (`ipc_impl.rs:114`) — read errors are
   never retried. Likely a copy-paste from `write()`.
7. **Newlines in track metadata** would misalign the `"\n"`-joined AppleScript parsing.
8. **Playlist duration parsed as `i32`** (`now.rs:121`); if Music ever returns a fractional/comma value
   this fails the whole poll.
9. **Metadata token never refreshes** in long-running processes (see §3.4).
10. `--watch` help text promises keyboard interactivity that doesn't exist (see §3.3).

---

## 7. How-to recipes

**Add a simple playback command** (e.g. `shuffle`):
1. Add a variant with a `///` doc comment to `Commands` in `src/main.rs` (add `#[command(visible_aliases = [...])]` if wanted).
2. Add a match arm: `music::tell("<AppleScript>").await?;` → colored `println!` → `concise_now_playing().await?;`.
3. Completions update automatically.

**Add a subcommand with options:** create `src/cmd/<name>.rs` with a `#[derive(Parser)] pub struct <Name>Options`
and `pub async fn <name>(opts) -> Result<()>`, register in `src/cmd/mod.rs` (`pub mod` + `pub use`),
then add `Commands::<Name>(cmd::<Name>Options)` in `main.rs` (mirrors `Now`).

**Read a new property from Music:** extend the multi-line script in `music::get_current_track`
(or write a similar function in `music/mod.rs`), append `& "\n" & <value>`, add a field to `Track`,
and parse it in order. Normalize decimal commas for numbers. Update every `Track { .. }` constructor.

**Use more Apple Music catalog data:** add fields to the serde structs in `music/models.rs`
(they mirror the JSON path `results.song.data[].attributes|relationships`) and expose them on `Metadata`.

**Change the Discord presence content:** edit the `Activity` builder in `cmd::discord::update_presence`.
Limits: max 2 buttons; label 1–32 chars; URL 1–512 chars (enforced in `activity.rs`).

**Useful AppleScript references:** the Raycast Music extension (credited in the README) and
Script Editor → File → Open Dictionary → Music.
