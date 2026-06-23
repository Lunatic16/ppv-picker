<p align="center">
  <img src="ppv.png" alt="Project Screenshot" width="350">
</p>

# PPV Picker

Browse the event index straight from your terminal,
pick an event and a substream, and get back the shareable **p...to URL** plus
the **`<iframe>` embed snippet** the web UI's "Embed this stream" button
would have copied for you — all without opening a browser.

`ppv_picker.py` is a single-file Python 3.10+ script with **one runtime
dependency** (`httpx`). It re-implements the API calls the Nuxt SPA at
`p...to` makes under the hood (`GET /api/streams` and `GET /api/streams/<uri>`)
and renders the result as a filterable, arrow-key-driven TUI picker that
degrades to a numbered-list prompt when piped.

---

## Table of Contents

- [Key Features](#key-features)
- [Demo](#demo)
- [How It Works](#how-it-works)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Usage](#usage)
  - [Interactive Pickers](#interactive-pickers)
  - [Output Format](#output-format)
  - [CLI Reference](#cli-reference)
  - [Environment Variables](#environment-variables)
- [Architecture](#architecture)
  - [Request Lifecycle](#request-lifecycle)
  - [Module Map](#module-map)
  - [Data Flow](#data-flow)
- [The API Behind p...to](#the-api-behind-ppvto)
- [Troubleshooting](#troubleshooting)
- [Limitations & Caveats](#limitations--caveats)
- [Contributing](#contributing)
- [License](#license)

---

## Key Features

- **No browser required.** Pure-stdlib TUI with a single third-party HTTP
  client (`httpx`). No Electron, no headless Chrome, no scraping HTML.
- **Filterable arrow-key picker.** Type to live-filter the event list,
  `↑/↓` to move, `Enter` to select, `Esc` to cancel — implemented from
  scratch in raw `termios` mode (no `prompt_toolkit`, no `curses`).
- **Gracious TTY degradation.** When piped to a file or another command,
  colors disable automatically, the live spinner falls back to a one-line
  status, and the picker becomes a plain numbered list.
- **Substream awareness.** When an event has multiple language/quality
  feeds, the script automatically drills into the per-event detail endpoint
  and lets you pick which one to embed.
- **Copy-paste output.** For the stream you pick you get three things on
  stdout: the **p...to page URL**, the **raw iframe URL**, and the
  **`<iframe>` HTML snippet** in the exact shape the web UI copies.
- **Cross-domain support.** Default API is `api....`; pass
  `--api https://api..../api` (or any of `ppv.cx...`, `ppv.is...`, `ppv.lc...`)
  to use a mirror — the shareable URL host is derived automatically.
- **ANSI theme matched to dark terminals.** Uses a 256-color
  sky/amber/slate/sage palette and disables itself when `NO_COLOR` is set
  or `TERM=dumb`.

---

## Demo

```
$ python ppv_picker.py

  ╭────────────────────────────╮
  │  p...to  Stream Links      │
  │  https://api..../api       │
  ╰────────────────────────────╯

  ⠹  fetching event index

  Select an event  (24 on offer)

    filter ›
    STATUS  EVENT NAME            SOURCE         START TIME         CATEGORY
    ●LIVE   Austria vs. Türkiye    ORF            Jun 22 09:00 PM   UEFA
    ●LIVE   Real Madrid vs. …      Movistar+      Jun 22 09:00 PM   La Liga
    SOON    Arsenal vs. Brighton   Sky Sports     Jun 23 07:30 PM   Premier
    …
      ↑↓ navigate · type to filter · Enter select · Esc cancel  [24/24]

  ⠦  fetching detail for "Austria vs. Türkiye"

  Event: Austria vs. Türkiye  (austria-vs-turkiye)

  Select a source  (3 available)
   ★  ORF (default)
   ◦  Sky Sports UK  [en]
   ◦  DAZN Germany   [de]
```

After the source picker, you get:

```
  ┌── STREAM DETAILS ──────────────────────────────────────────
  │  Source:   ★ default  ORF
  │  PPV URL:  https://ppv.../live/austria-vs-turkiye/...
  │  Embed:    https://embedcentral.../embed/...../orf
  │
  │  Iframe Snippet:
  │    <iframe id="player"
  │            src="https://embedcentral.../embed/...../orf"
  │            marginheight="0" marginwidth="0"
  │            scrolling="no" allowfullscreen="yes"
  │            allow="encrypted-media; picture-in-picture;"
  │            width="100%" height="100%" frameborder="0"
  │            style="position:absolute;"></iframe>
  └────────────────────────────────────────────────────────────
```

---

## How It Works

```
                ┌───────────────────────┐
                │  ppv_picker.py (CLI)  │
                └──────────┬────────────┘
                           │
            ┌──────────────┼──────────────────┐
            │              │                  │
            ▼              ▼                  ▼
   GET /api/streams   GET /api/streams   build pickers
       (index)         /<uri> (detail)        │
            │              │                  │
            ▼              ▼                  ▼
       [Event, …]      [Embed, …]      arrow-key TUI
            │              │                  │
            └──────┬───────┴──────────────────┘
                   ▼
           render STREAM DETAILS box with
           p...to URL + iframe URL + snippet
```

1. **Index fetch.** The script calls `GET https://api..../api/streams`,
   which returns a list of categories. Each category contains a `streams`
   array of "flat" event rows (`id`, `name`, `tag`, `iframe`,
   `starts_at`, `always_live`, etc.).
2. **Event picker.** Those rows are flattened into `Event` dataclasses and
   rendered into the picker. Sorting prefers 24/7 feeds first, then by
   start time, then category.
3. **Detail fetch (only if needed).** When the chosen event has its own
   detail payload available (e.g. newer events only reachable on the
   per-event endpoint), a second `GET /api/streams/<uri>` call enriches it
   with the full substream list.
4. **Embed picker.** Each event resolves to a list of `Embed` objects:
   the *default* feed + any `substreams` returned by the API. A second
   picker narrows down which feed to emit.
5. **Render.** The chosen `Embed` is printed in the STREAM DETAILS box
   with the **p...to page URL**, the **iframe URL**, and a ready-to-paste
   `<iframe>` HTML snippet.

---

## Tech Stack

| Layer           | Choice                                              |
| --------------- | --------------------------------------------------- |
| Language        | Python **3.10+** (uses PEP 604 `X \| Y` type hints)  |
| HTTP            | [`httpx`](https://www.python-httpx.org/) (sync API) |
| Terminal I/O    | `termios` / `tty` in raw mode (no third-party TUI)  |
| Concurrency     | `threading` only (for the spinner)                  |
| Time formatting | `time.strftime` (no external date libs)             |
| Config          | `argparse` + `PPV_API_BASE` env var                 |
| Color theme     | 256-color palette (sky 110, amber 179, slate 246, sage 108) |
| Packaging       | Single file. No `pyproject.toml`, no `requirements.txt` |

---

## Prerequisites

- **Python 3.10 or newer** — uses PEP-604 union types (`str | None`)
  throughout. Tested on 3.11.
- **httpx** — the only runtime dependency.
- **A TTY (recommended)** — for the filterable picker. When stdout or
  stdin is piped, the script falls back to a numbered list automatically.
- **Linux / macOS** — the keypress reader uses BSD/POSIX `termios`. On
  Windows you'll need WSL or to swap in `msvcrt` (not bundled).

---

## Getting Started

### 1. Install `httpx`

If you haven't already:

```bash
pip install httpx
```

If you're inside a PEP 668 managed environment (system Python on Fedora,
Debian, etc.), use a venv or `uv`:

```bash
python -m venv .venv
source .venv/bin/activate
pip install httpx
```

### 2. (Optional) Make the script executable

```bash
chmod +x ppv_picker.py
```

After that you can run `./ppv_picker.py` directly. The shebang
(`#!/usr/bin/env python3`) is already in the file.

### 3. Run it

```bash
python ppv_picker.py
```

That's it. The first call will hit `https://api..../api/streams` and
present your event list.

### 4. (Optional) Pin the API domain

If `p...to` is blocked by your DNS or ISP, point the script at a mirror:

```bash
python ppv_picker.py --api https://api.p.../api
```

The script will derive the public ppv host from the API URL and emit
shareable URLs on that host.

---

## Usage

### Interactive Pickers

Once you launch the script, you'll see two picker phases:

**Phase 1 — Event picker**

| Key            | Action                                        |
| -------------- | --------------------------------------------- |
| `↓` / `↑`      | Move selection                                |
| Any character  | Append to filter (live substring match)       |
| `Backspace`    | Remove last char from filter (resets to row 0) |
| `Enter`        | Confirm event selection                       |
| `Esc` / `Ctrl-C` / `Ctrl-D` | Cancel cleanly (exit 0)         |

**Phase 2 — Source picker**

Same keybindings; this time the list is the friendly name + locale of
each embed (default + substreams).

### Output Format

For the chosen `Embed` the script emits a single STREAM DETAILS block
with three pieces of information:

| Field           | What it is                                              |
| --------------- | ------------------------------------------------------- |
| **PPV URL**     | `https://p...to/live/<event_uri>/<stream>` — the page a user would open in a browser |
| **Embed**       | The raw iframe `src` URL the player loads               |
| **Iframe Snippet** | A copy-pasteable `<iframe …>` HTML tag matching what the web UI's "Embed this stream" button produces |

You can pipe the output straight into a downstream tool:

```bash
python ppv_picker.py --raw | tee stream.txt
```

`--raw` forces color off regardless of whether stdout is a TTY (useful
for log files or when wrapping into a shell function).

### CLI Reference

```
python ppv_picker.py [OPTIONS]
```

| Flag               | Default                          | Description                                                                  |
| ------------------ | -------------------------------- | ---------------------------------------------------------------------------- |
| `--api URL`        | `https://api..../api` (or `$PPV_API_BASE`) | API base URL — switch to a mirror (`ppv.st`, `ppv.cx`, `ppv.is`, `ppv.lc`) |
| `--raw`            | auto-disabled when not a TTY     | Disable ANSI color output. Honors `NO_COLOR=1` automatically when off.        |
| `--show-default`   | off                              | Print the default feed's embed *before* the substream picker opens.         |
| `--list-only`      | off                              | Reserved for future use. Currently a no-op stub.                             |
| `-h` / `--help`    | —                                | Print the usage block above.                                                  |

Exit codes:

| Code | Meaning                                               |
| ---- | ----------------------------------------------------- |
| `0`  | Normal exit (including user cancellation)             |
| `1`  | Network / API error (also: zero playable sources)     |
| `2`  | Missing dependency (`httpx` not installed)            |

### Environment Variables

| Variable       | Used for                                                        | Default |
| -------------- | --------------------------------------------------------------- | ------- |
| `PPV_API_BASE` | Override the default `--api` URL without passing a flag         | `https://api..../api` |
| `NO_COLOR`     | When set (any non-empty value), disables ANSI color output      | unset   |
| `TERM=dumb`    | Treats the terminal as color-unaware                            | `xterm-256color` |

---

## Architecture

The script is intentionally one file, ~870 lines, organized in clearly
labelled sections. There is no framework, no MVC, no global state beyond
the `C` (color) instance passed by reference.

### Request Lifecycle

1. **`main(argv)`** — parses flags with `argparse`, then calls `run(...)`.
2. **`run(api_base, show_default, use_color)`** —
   - Prints the banner via `print_banner(api_base, c)`.
   - Constructs a `PPVClient(api_base=...)` (httpx client with custom
     `User-Agent`, `Origin`, `Referer`, `Accept` headers).
   - Fetches the index with `fetch_with_spinner(...)`. The spinner is a
     `threading.Thread`-driven braille animation that yields to the main
     thread every 80 ms until the request completes.
   - Normalizes the index into `list[Event]`. Each event is deserialized
     via `Event.from_index(category_name, raw)`.
   - Sorts events: **24/7 first**, then by `starts_at`, then by
     `category_name`. Builds rows with `event_row(...)` and hands them to
     `pick_from_list(...)`.
   - On Enter, fetches the per-event detail via `client.event(chosen.uri)`
     (wrapped in a `LookupError` → friendly warning). If the detail is
     richer (e.g. has additional substreams), it merges with the index
     record using `Event.from_event(detail)`.
   - Builds `Embed` rows via `embed_row(...)` and shows the source picker.
   - On source selection, prints `print_embed(...)` — the STREAM DETAILS
     block.
3. **`pick_from_list(...)`** — the picker itself. On a TTY it goes into
   `tty.setraw(...)`, hides the cursor (`\033[?25l`), and **redraws the
   whole block_height-line region in place** on every keystroke. All line
   advances use `\r\n` to stay safe in raw mode.
4. **`_read_key()`** — single-byte CSV reader. Handles arrow keys
   (`\x1b[A/B/C/D`, plus `H`/`F` for Home/End), `Esc`, `Enter`, `Backspace`,
   `Ctrl-C`, `Ctrl-D`, and printable text.
5. **`_pick_plain(...)`** — the non-TTY fallback. Just prints the list with
   `1.` `2.` `3.` numbering and asks for a numeric answer.

### Module Map

Even though it's a single file, the code is broken into commented
sections you can navigate to with grep:

| Section              | Lines (≈) | Responsibility                                              |
| -------------------- | --------- | ----------------------------------------------------------- |
| **Dataclasses**      | 70–100    | `Embed`, `Event` (with `from_index` / `from_event`)         |
| **Terminal helpers** | 100–190   | `enable_colors`, `C` palette, `hr`, terminal-size wrappers, `format_start` (live/soon/ended state detection) |
| **Banner + spinner** | 190–250   | `print_banner`, `fetch_with_spinner` (threaded animation)   |
| **API client**       | 250–300   | `PPVClient` (httpx-based with custom UA/Origin headers)     |
| **In-memory model**  | 300–390   | `Event` + `Embed` factories                                  |
| **Pretty printer**   | 390–430   | `print_embed` (STREAM DETAILS box)                           |
| **Picker**           | 430–670   | `ANSI_RE`, `strip_ansi`, `_read_key`, `_pick_plain`, `pick_from_list` |
| **Row formatting**   | 670–720   | `event_row` (badge + name + source + time + category), `embed_row` |
| **CLI flow**         | 720–870   | `derive_ppv_host`, `run`, `parse_args`, `main`             |

### Data Flow

```
┌─────────────────────┐      JSON       ┌──────────────────┐
│ api....             │ ───────────────▶│ PPVClient.index  │
│   /api/streams      │                 └────────┬─────────┘
└─────────────────────┘                          │
                                                 ▼
                                        ┌──────────────────┐
                                        │ list[dict]       │
                                        │ (raw categories) │
                                        └────────┬─────────┘
                                                 │ Event.from_index(...)
                                                 ▼
                                        ┌──────────────────┐
                                        │ list[Event]      │
                                        └────────┬─────────┘
                                                 │ event_row(...)
                                                 ▼
                                        ┌──────────────────┐
                                        │ list[str] (rows) │
                                        └────────┬─────────┘
                                                 │ pick_from_list(...)
                                                 ▼
                                        ┌──────────────────┐
                                        │ chosen index     │
                                        └────────┬─────────┘
                                                 │ client.event(uri) (if available)
                                                 ▼
                                        ┌──────────────────┐
                                        │ Event (merged)   │
                                        │   .embeds()      │
                                        └────────┬─────────┘
                                                 │ embed_row + pick_from_list
                                                 ▼
                                        ┌──────────────────┐
                                        │ chosen Embed     │
                                        └────────┬─────────┘
                                                 │ print_embed(...)
                                                 ▼
                                        STDOUT (STREAM DETAILS box)
```

### Key Components

- **`Embed.ppv_url(host, event_uri)`** — recovers a shareable
  `https://<host>/live/<tail>` URL. When the API only gives us the iframe
  URL (very common for substreams, which arrive with `uri=null`), the
  tail is reverse-engineered by stripping `https://<embed-host>/embed/`
  from the iframe URL. If the recovered tail already starts with the
  event uri, the script won't double-prefix it.

- **`format_start(unix_ts, ends_at)`** — returns a `(time_str, state)`
  tuple where state ∈ `{live, soon, ended, info}`. State drives the
  `●LIVE` / `SOON` / `DONE` / blank badge in the picker. A row whose
  `ends_at` already passed is marked `ended`, even if `starts_at` looks
  in the past.

- **`fetch_with_spinner(label, fn, c)`** — runs `fn()` in a daemon
  thread, animates a braille spinner (`⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏`) at ~12.5 fps
  until the request completes, then erases the spinner line and returns
  the result. Re-raises any exception swallowed by the worker thread.

- **`pick_from_list(title, rows, ...)`** — the picker heart. Computes a
  block height from the terminal size, reserves `list_room = block_height
  − 4` rows for the visible list, and on every keystroke redraws the
  whole region in place using ANSI cursor moves (`\033[<n>A`). A live
  filter (`substring, case-insensitive, ignoring ANSI codes`) reduces
  `rows` to a `list[int]` of indices into the original list.

- **`_clear_block(out, block_height)`** — erases the picker the moment
  the user hits Enter or Esc, without leaving stray ANSI codes or empty
  rows behind. Leaves the cursor at the top of the original block so the
  STREAM DETAILS banner can print below.

---

## The API Behind p...to

The Nuxt SPA at `https://ppv...` fetches its data from a small JSON API:

### `GET /api/streams`

Returns the index, grouped by category. Each category is a
`{ category: str, streams: [...] }` object. A typical stream entry looks
like:

```json
{
  "id": 12345,
  "name": "Austria vs. Türkiye",
  "tag": "FIFA World Cup",
  "source_tag": "ORF",
  "uri_name": "austria-vs-turkiye",
  "iframe": "https://embedcentral.../embed/...../orf",
  "starts_at": 1735003200,
  "ends_at": 1735006800,
  "always_live": false,
  "viewers": 14820,
  "substreams": [
    {
      "uri": "sky-uk",
      "source_tag": "Sky Sports UK",
      "locale": "en",
      "iframe": "https://embedcentral.../embed/...../sky-uk"
    }
  ]
}
```

### `GET /api/streams/<uri_name>`

Returns a single event with full detail (including fields the index
omits, like `start_timestamp`, `end_timestamp`, `poster`). New events
sometimes appear here before they reach the index.

---

## Troubleshooting

### "this script needs the httpx package"

`httpx` is the only third-party dep. Install it:

```bash
pip install httpx
```

If you're in a PEP 668 environment:

```bash
python -m venv .venv
source .venv/bin/activate
pip install httpx
```

### Script exits immediately without showing a picker

Possible causes:

- **Stdout is not a TTY** — the spinner suppressed itself, the picker
  fell back to numbered-list mode, and you'll see the list as plain
  text. Run from a real terminal, or pass `--raw`. To force the TUI use
  a real terminal emulator (Windows Terminal, iTerm2, GNOME Terminal,
  etc.).
- **Network blocked** — `p...to` may be blocked by your ISP or
  corporate firewall. Try a mirror:
  ```bash
  python ppv_picker.py --api https://api.p.../api
  ```

### Arrow keys don't move the selection

The script parses standard VT100 arrow-key escape sequences
(`\x1b[A/B/C/D`). If your terminal sends something different
(`\x1bOA`, `\x1bOH`, custom Kitty/iTerm sequences, etc.), the
`_read_key()` parser won't recognize them. Patches welcome — see
[Contributing](#contributing).

### Characters look washed-out / unreadable

The script uses a 256-color palette tuned for dark backgrounds (cornflower
blue, warm gold, muted grey, muted green). If you see neon-bright
foreground colors you're on a light background — set `NO_COLOR=1` to
disable the palette:

```bash
NO_COLOR=1 python ppv_picker.py
```

### Network error / "✗ failed to fetch"

`PPVClient.index()` raises whatever `httpx` throws on a timeout, DNS
failure, or HTTP error. Common quick fixes:

- **Verify connectivity:** `curl -I https://api..../api/streams`
- **Increase timeout:** search for `TIMEOUT = 15.0` in the file and bump it.
- **Try a mirror:** `--api https://api.p.../api`

### Substreams list looks empty for an event

If the event is "hot off the press" the index endpoint may have it but
the detail endpoint doesn't yet. The script handles this:

- If the detail fetch returns 404, the picker falls back to whatever
  `substreams` were embedded in the index payload.
- If both are empty, you'll see the message
  `✗  no playable sources found` and the exit code will be `1`.

---

## Limitations & Caveats

- **Windows is unsupported.** The keypress reader uses BSD/POSIX
  `termios`. WSL or Cygwin works; bare `cmd.exe` and PowerShell don't.
- **No CLI history or caching.** Every run hits the API fresh.
- **No tests, no type checker config.** It's a single-file tool — the
  types are inline hints only. Pull-requests adding `pytest` /
  `mypy` are welcome.
- **The script trusts the upstream API.** Field names, JSON shape, and
  behavior can change without notice. If `p...to` rotates the field
  `substreams` → `feeds` or similar, the script will need an update.
- **`<iframe>` embed snippets on shared pages may trigger CSP / X-Frame-Options**
  in the embedding site. That's not something the script can fix — it's
  a property of the embedding target + the embed host's response
  headers.
- **Streams are wrappers around third-party embeds.** The actual playback
  depends on the rights-holder's feed. If a stream shows a "geo-blocked"
  or "blackout" message in a real browser, the script can't help — you
  picked the right URL, the underlying source just isn't available in
  your region.

---

## Contributing

Patches welcome — this is a small, contained script that's easy to fork:

1. Fork or copy `ppv_picker.py`.
2. If you change behavior, update this README in the same PR.
3. Keep the public API stable (`run`, `main`, `parse_args`) so it
   stays embeddable.
4. Tested manually on Linux. macOS should work but isn't a CI target.
5. Run the script by hand before sending:

   ```bash
   python ppv_picker.py --raw | head -40        # non-TTY sanity check
   python ppv_picker.py                       # TTY sanity check
   ```

Things that would be genuinely useful additions:

- Unit tests for `Embed.ppv_url()` (lots of edge cases in URI recovery).
- A `--json` mode that emits the picked embed as JSON on stdout (instead
  of the human-readable STREAM DETAILS block) for piping into other tools.
- A `--refresh-interval` flag that re-polls the API and updates the LIVE
  badge without re-launching.
- A `Kitty`/`iTerm` cursor-key parser extension in `_read_key`.

---

## License

This script is provided as-is for personal use. The data it brokers
(live events, embed URLs) belongs to their respective broadcasters; the
script only fetches and reformats metadata that the public
`p...to` website already exposes.
