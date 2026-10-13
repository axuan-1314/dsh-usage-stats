# dsh-usage-stats

> **English | [中文](README.md)**

> A "usage statistics" plugin for the official DSH desktop build: it adds a **Usage statistics** section to the settings page and presents token usage the way the open-source ZCode build does — a five-cell lifetime summary bar, a GitHub-style heatmap, a trend line, and a model-usage donut chart. All data comes from the local machine (live event collection + historical session backfill), with zero uploads and zero new dependencies.

> Note: the plugin's in-app UI strings are Chinese; the English wording below is provided for clarity.

- **Plugin version**: 0.2.0 (see `package.json` for the source of truth; see `CHANGELOG.md` for the change history)
- **Host compatibility**: DSH 0.2.0-rc.2 (installable on all three profiles web / headless / desktop; the panel UI is rendered on web / desktop); event and file shapes are verified at the source level against that version
- **Distribution**: the GitHub repo tag tarball channel (plus a local clone for development); upgrading is covered in [Upgrade](#upgrade) (DSH plugins do not auto-update)
- **Data location**: everything stays local under `~/.dsh/usage-stats/` (the default, configurable via `storageRoot`); nothing is uploaded to any remote
- **Network**: local loopback only (the API goes through the kernel webServer's same-origin reverse proxy); no telemetry, zero uploads
- **Prerequisites**: DSH installed (any of the web / headless / desktop profiles); the host runs Node ≥ 22.15 (24.x on the reference environment; historical backfill uses `node:zlib` for zstd decompression); **zero runtime dependencies** (no `npm install`)
- **License**: MIT (see [LICENSE](LICENSE))

## Contents

- [Features](#features)
  - [Five-cell lifetime summary bar](#five-cell-lifetime-summary-bar)
  - [Token activity heatmap](#token-activity-heatmap)
  - [Trend line](#trend-line)
  - [Model usage donut chart](#model-usage-donut-chart)
  - [Historical backfill](#historical-backfill)
  - [Freshness](#freshness)
  - [Keyboard access](#keyboard-access)
  - [Resilience (the user-visible part)](#resilience-the-user-visible-part)
- [Quick start](#quick-start)
  - [Installation](#installation)
  - [Upgrade](#upgrade)
  - [Verification](#verification)
- [Data & privacy](#data--privacy)
- [Rollback & uninstall](#rollback--uninstall)
- [FAQ](#faq)
- [Development & testing](#development--testing)
- [Known limitations](#known-limitations)

## Features

The panel lives in the "Usage statistics" section of the settings page (`settings.section` order 13, after the other existing sections), titled "Usage statistics" with the subtitle "a rough view of session activity and model usage.". Every item below is something the user actually sees on the settings page.

### Five-cell lifetime summary bar

One row of five cells: **total tokens / peak tokens in a single day / longest chat duration / current streak / longest streak**. Large numbers use a compact format (≥ 10,000 is shown in "x.x wan" form and ≥ 100,000,000 in "x.x yi" form; wan and yi are the ten-thousand and hundred-million units the UI uses). Hovering a cell gives the exact value and the definition behind it (which day the peak was on, what day the total counts from, how the streak is computed). Definitions:

- Peak = the day with the largest sum of the four buckets;
- Streak = a day counts as active when its tokens > 0; if today has usage, count back from today, otherwise count back from yesterday.

### Token activity heatmap

GitHub-style cells covering the last **52 calendar weeks** (starting Sunday, last column aligned to the current week), with a row of month labels underneath. A pill in the top-right switches between three modes:

- **Daily**: one cell per day, five shade steps; hovering (or the keyboard cursor) shows "Oct 3, 2026 / 1,234 tokens · 5 turns" (tokens with thousands separators).
- **Weekly / cumulative**: the whole column becomes a bottom-aligned bar of color blocks (height = that week's usage relative to the max week, capped at 7 cells); hovering shows the week's last date plus **two same-basis** numbers — weekly mode shows "this week's total tokens · this week's turns", cumulative mode shows "cumulative tokens up to this week · cumulative turns".

### Trend line

A time-range pill (titled "Trend and model usage"): **Today / Last 7 days / Last 30 days** (default Last 30 days).

- Today: bucketed by the 24 whole hours (x-axis "0h" … "23h");
- Last 7 days / Last 30 days: one point per day, days with no data inside the window are zero-filled, so the line stays continuous with no axis breaks;
- One line per model (a six-color cycle, no cap and no merging), drawn with a hand-written monotone cubic interpolation curve (smooth without overshoot), Y axis hidden; **the legend is clickable to hide/restore** a line (hiding one does not change the colors or magnitudes of the others);
- Hovering brings up a follow card: a title "Oct 3 - 12,345 tokens" plus per-model detail rows (rows for models with 0 usage are omitted; with more than 8 models only the first 8 rows are listed, annotated "and N more models not listed").

### Model usage donut chart

The center shows the total tokens for the selected time range (compact format), and on the right a two-line legend: the top line = model name + share (a whole number at ≥ 10%, one decimal below 10%), the bottom line = that model's token count (hovering gives the exact value). The top 6 models by usage each get a color, and **more than 6 models are merged into a gray "other models"** (hovering that row lists the model names merged into it). Hovering a sector or legend row pops up a data card. Today mode works the same way (today's per-model usage is summed from today's trend data, giving equivalent numbers).

### Historical backfill

Loading the plugin automatically backfills **all historical sessions** (the multi-frame zstd session files under `~/.dsh/sessions` are decompressed frame by frame, and fork-inherited prefixes are trimmed by the kernel marker for dedup). It is incremental and idempotent: already-accounted sessions are skipped in seconds, and steady-state loading decompresses no historical sessions in the common case (a few "cursor but no mtime" sessions are the exception — see [Known limitations](#known-limitations)).

- The "Backfill history" button in the footer triggers it manually: an **HTTP 202 accept-style** flow (the POST returns immediately, the task runs in background shards, and the UI is not frozen);
- The button shows live progress: **"Backfilling… scanned X/Y sessions"**; on completion it reports "Backfill complete: N sessions added, R records" and auto-refreshes the panel; failures are reported explicitly and are never dressed up as success;
- The progress endpoint is `GET /api/dsh-usage-stats/backfill/status` (running / startedAt / scanned / total / backfilled / records / lostFrames / finishedAt / error).

### Freshness

- **Light 60-second polling**: while the panel is open, once a minute it silently pulls the summary plus the current range's trend and model shares (no flashing "Loading"; ticks are skipped in a background tab and one is immediately made up when you switch back to the foreground, and polling stops when the page closes); usage from a new conversation reaches the screen within about a minute;
- **On-demand heatmap refresh**: silent ticks only fetch the heatmap when "cumulative usage or the date has changed" — zero extra requests when idle, and cells catch up within 60 seconds at most when data changes (a failed fetch does not advance the detection baseline, so the next tick retries automatically, and the card header shows "failed to update activity data, showing the previous data");
- The left end of the footer always shows **"updated at HH:MM"**, which advances only when every data stream of the round was fetched; if one is missing it is marked "(some data not updated)" and freshness is never misreported. The "Refresh" button at the bottom-right re-fetches everything at any time.

### Keyboard access

The heatmap and the trend line each occupy **a single Tab stop**: once focused, move the cursor with ← → ↑ ↓ (for the heatmap, "a week / a day"; for the trend, "a point"); the cursor on the first keypress defaults to **today / the latest point** (not starting a year back); Home/End jump to the two ends; a data card identical to the mouse-hover one pops up at the cursor (on the heatmap the cursor cell is outlined, so the landing spot is visible even on absent days), and it collapses as soon as focus leaves the chart. Mode and range pills carry `aria-pressed`, and the chart containers carry `role="img"` with a one-line summary, so they are readable by screen readers. The trend legend is clickable to hide/restore a line (hidden items are grayed out and struck through, and the color assignment is not rearranged).

### Resilience (the user-visible part)

- Three states: loading / no usage data yet / load failed + retry; a failure in one data stream only degrades the corresponding card and shows an **inline "Retry" button**, without blowing up the whole panel; a render exception on the panel has a fallback banner and does not affect background collection;
- **The empty state self-heals**: while the panel sits at "no usage data yet", it turns into real data within 60 seconds of any data arriving in the background (no manual refresh needed);
- **The whole backfill is visible**: if history is being backfilled when you open the settings page, the empty slot shows "Backfilling historical sessions…" + "scanned X/Y sessions", and the button shows progress in sync; when it finishes, the panel auto-refreshes and reports the result;
- Single-writer lock: when two DSH instances are open at once, the one loaded later degrades to a **read-only mirror** (statistics remain viewable as usual, but it does not collect or backfill); once the lock is free it takes over automatically and catches up the increment;
- Light and dark themes follow the official setting and switch automatically.

## Quick start

### Installation

Pick one of the two channels (**zero runtime dependencies**, no `npm install` at any point):

**① GitHub tag tarball (recommended)** — install straight from a repo tag, pinned to a version:

```bash
dsh plugin --profile desktop add https://github.com/axuan-1314/dsh-usage-stats/archive/refs/tags/v0.2.0.tar.gz
# profile: web / headless / desktop
```

**② Local clone (development / self-use)** — source edits take effect live, suited to code changes:

```bash
git clone https://github.com/axuan-1314/dsh-usage-stats.git
dsh plugin --profile desktop add <path to your local clone>
```

After installing, verify the entry is in the tree: `dsh --profile desktop --dump-config`.

The plugin ships its own `cordis.patch.yml`, which takes effect automatically once loaded.

> To run new code after editing code / changing version: **the web profile reloads via HMR; desktop / headless need the instance restarted** (restart the desktop app; headless naturally picks up the latest on every start). The local-clone channel is a `link:` symlink, so source edits need no reinstall.

### Upgrade

DSH's plugin manager **does not support automatic updates**. So an upgrade = re-running the install command with the new version number:

```bash
# tag tarball channel: replace v0.2.0 in the URL with the new version (version numbers in Releases / CHANGELOG.md)
dsh plugin --profile desktop add https://github.com/axuan-1314/dsh-usage-stats/archive/refs/tags/v0.2.1.tar.gz
# local-clone channel: git pull, then no reinstall needed (the symlink takes effect)
```

- **Your data is unaffected by upgrades**: all statistics land in `~/.dsh/usage-stats/` — **outside the plugin package directory** — so uninstall / reinstall never touches it (unless you delete that directory yourself). Keeping data separate from code is deliberate.
- **Restart after upgrading**: after replacing the installed package, the process must restart to load the new JS modules (the web profile can reload via HMR; restart the desktop app; headless picks it up on next start).

### Verification

1. Restart DSH and open the settings page — a "Usage statistics" section appears in the section list;
2. On first load, all historical sessions are backfilled automatically in the background, so the panel may first show "no usage data yet", which is normal (the more history you have, the longer it takes; a large history needs a few minutes);
3. Click "Backfill history" in the footer to see progress: once accepted, the button shows "Backfilling… scanned X/Y sessions", and when it finishes it shows the result and auto-refreshes the panel (if an automatic backfill is still running, it says "a backfill is already in progress, tracking progress…" and switches to progress tracking);
4. Data on screen: the summary bar and the heatmap show the full history data;
5. You can also query the local loopback API directly: `GET /api/dsh-usage-stats/summary`.

## Data & privacy

- Data lives only in `~/.dsh/usage-stats/`: `records-YYYY-MM.jsonl` per-record incremental ledger (rotated by month, the single source of truth), `daily/YYYY-MM-DD.json` daily aggregate cache (rebuildable in full from the ledger at any time), `backfill-marker.json` the backfill cursor, and `lock` the single-writer lock.
- The only network path is local loopback (the API goes through the kernel webServer's same-origin reverse proxy, with a loopback + read-only method guard — read routes allow GET/HEAD, and only `/backfill` accepts POST); no telemetry, zero uploads.
- Collection copies the official tokenUsage projection fold semantics verbatim (`@deepseek-ai/dsh-token-meter`), so it can be reconciled item by item against the official trajectory "usage" tab.
- Definitions: the sum of the four buckets (uncachedInput + output + cacheRead + cacheWrite); `uncachedInputTokens` = the raw `inputTokens` (that is the official bucket name).
- Config keys: `enabled` (default true) / `storageRoot` / `sessionsRoot`; unknown keys are rejected with an error.

## Rollback & uninstall

1. **Disable without uninstalling**: set `enabled` to `false` in the profile's `cordis.patch.yml` (or bundle patch). How it takes effect depends on the profile: the web profile picks it up via HMR; desktop / headless need the instance restarted after the config change. There is no uniform "edit and it hot-applies" behavior.
2. **Uninstall the whole package**: `dsh plugin --profile desktop remove dsh-usage-stats`.
3. **Clear data**: delete `~/.dsh/usage-stats/` (a dedicated directory; other components are unaffected).

## FAQ

**Where does the data come from?**
Two paths feed one ledger: the live path comes from the host's `session/event` event stream (each event produces a net increment, written to disk following the official fold semantics); the historical path comes from backfilling the session files under `~/.dsh/sessions` (multi-frame zstd decompressed frame by frame; forked sessions have their inherited prefix trimmed by the kernel's `session/end-seed{inherited:true}` marker, so parent sessions are not double-counted). Both paths share the same fold state machine and accounting cursor, and never double-count each other.

**Why do the numbers match the official trajectory "usage" tab?**
The fold semantics copy the official tokenUsage projection bit for bit (replace-slot LWW + retry carry-over clearing), so the sum of the incremental stream is always equal to the official totals. A full reconciliation on the reference environment matched record by record, and 3000 rounds of random event streams showed zero mismatch against the official projection.

**After restarting / upgrading the plugin, does it recompute or double-count?**
No recomputation: the ledger is the single source of truth, backfill only adds increments based on each session's cursor, and sessions whose files are unchanged are skipped with zero decompression. No double counting: the v0.1.6 architecture refactor included an old-vs-new comparison across dozens of sessions, bit-for-bit identical; upgrading a snapshot from the old version appended 0).

**What about performance? Does it freeze the UI?**
Backfill is async and sharded (it yields a tick every 4 sessions), so decompression does not freeze the host; steady-state loading decompresses no historical sessions in the common case (an mtime sentinel; a few "cursor but no mtime" sessions are the exception). Only the first load needs to decompress all history; a large history takes a few minutes but the app stays usable throughout. While the panel is open it sends only two or three light requests every 60 seconds (Today mode: two streams — summary + trend; Last 7/30 days: three streams — summary + trend + model shares; the heatmap is not one of the per-tick base requests — it is refetched silently only when the cumulative total or the date changes).

**What happens if I open two DSH windows?**
The single-writer lock guarantees that only one instance collects; the other is a read-only mirror — the statistics panel remains viewable as usual, and once the lock is free it takes over automatically and catches up the increments from the mirror period. Clicking "Backfill history" during the mirror period says "this instance is currently a read-only mirror (another window holds the write lock); it will catch up automatically after taking over, so no manual backfill is needed".

**How do I clear the statistics?**
Quit DSH and delete `~/.dsh/usage-stats/` entirely; the next load backfills everything again as if it were a fresh install.

## Development & testing

```bash
npm test    # node --test, 110 tests (fold semantics / store & aggregation / API shapes / integration regression / client pure functions)
npm run check
```

- Five code files: `lib/index.js` (host wiring + backfill), `lib/fold.js` (fold state machine), `lib/store.js` (persistence + single-writer lock), `lib/aggregate.js` (pure aggregation functions), `lib/client.js` (settings-page UI).
- API: `GET /api/dsh-usage-stats/summary`, `/heatmap?days=` (default 366, 0 = everything), `/trend?days=1|7|30`, `/models?days=7|30`, `/backfill/status`, plus `POST /backfill` (202 accept-style).
- Debug log: written to stderr when `DSH_USAGE_STATS_DEBUG=1`.

Implementation details are in `SPEC.md`.

## Known limitations

The known limitations are listed below:

- Live fold slots live in memory and are discarded on restart: when a replacement window straddles a process restart, an old sample can very rarely be double-counted;
- Model attribution of backfill-path attempt samples depends on the in-session `request/header` being present; when it is missing the sample is recorded as unknown (0 unknown cases measured in the reference environment);
- Longest chat duration is bucketed by (session, day) and takes the range; sessions spanning days are truncated per day — this is an approximate definition;
- A few historical sessions that "have a cursor but no mtime" (a small fraction of sessions in production) are still decompressed on every load (about 0.7–4 seconds, tolerable);
- Multi-frame zstd frame boundaries are found by magic-number scanning, which in theory has a very low false-positive chance (0 false positives across a large frame sample; frames that fail to decompress are skipped, counted, and left on record);
- The heatmap always looks back 52 calendar weeks; usage older than that only counts toward full-period metrics such as the summary bar and does not appear in the cells;
- The heatmap's "weekly / cumulative" mode hover card takes the week's **last date** (Saturday) and the week total — a pre-existing definition (following ZCode);
- The trend hover card lists at most 8 model rows (beyond that it notes "and N more models not listed") — with a very large number of models in a single day the card no longer grows without bound.
