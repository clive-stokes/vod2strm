# vod2strm – Dispatcharr Plugin

A high-performance Dispatcharr plugin that exports your VOD library (Movies and Series/Episodes) into a structured filesystem of `.strm` and `.nfo` files for Plex, Emby, Jellyfin, or Kodi.

> **Fork note** — this is a fork of [cmc0619/vod2strm](https://github.com/cmc0619/vod2strm). Changes from upstream are tracked under version `0.0.13-fork.*`. See the [Fork Changes](#fork-changes) section below for a full diff summary.

## Overview

vod2strm transforms Dispatcharr's VOD database into media server-compatible `.strm` files without duplicating content. It runs entirely inside Dispatcharr using the Django ORM for performance and Celery for background jobs. If Celery isn't available, it gracefully falls back to background threads.

## Features

### Core Functionality
- **Movies + Series Support**: Generates `.strm` files pointing to Dispatcharr proxy endpoints (`/proxy/vod/movie/{uuid}` and `/proxy/vod/episode/{uuid}`)
- **NFO Generation**: Creates `movie.nfo`, `season.nfo`, and `SxxExx.nfo` with TMDB/IMDB metadata
- **Season 00 Specials**: Automatically organizes season 0 episodes into "Season 00 (Specials)" folders
- **Manifest Caching**: Tracks generated files to skip unnecessary writes (protects SD cards/NAS from wear)
- **Cleanup**: Detects and removes `.strm` files for content no longer in the database (Preview or Apply modes)

### Performance & Protection
- **Adaptive Throttling**: Monitors NAS write performance and dynamically adjusts concurrency to prevent I/O overload
- **Smart Skipping**: Instantly skips entire series when filesystem already matches database
- **Batch Processing**: Processes files in batches with progress logging
- **Compare-Before-Write**: Only writes when content changes (hash-based comparison for NFO files)

### Automation
- **Auto-run After VOD Refresh**: Optionally trigger generation automatically when Dispatcharr refreshes VOD content (30-second debounce)

### Debugging & Reports
- **Dry Run Mode**: Simulate generation without touching filesystem (testing)
- **Database Statistics**: View content counts and provider breakdown
- **CSV Reports**: Detailed action logs for every run (`/data/plugins/vod2strm/reports/`)
- **Verbose Logging**: Optional debug logs (`/data/plugins/vod2strm/logs/vod2strm.log`)

## Output Structure

Movies and series are organised first by provider category (e.g. `EN - Action`), then by title. Folder names include a `{tmdb-XXXXX}` suffix when a TMDB ID is available — Jellyfin uses this for instant metadata matching without an API call.

```
/data/STRM/
├── Movies/
│   ├── EN - Action/
│   │   └── Movie Name (2023) {tmdb-12345}/
│   │       ├── Movie Name (2023) {tmdb-12345}.strm
│   │       └── movie.nfo
│   └── Uncategorised/
│       └── Another Movie (2022)/
│           ├── Another Movie (2022).strm
│           └── movie.nfo
└── TV/
    ├── EN - Drama/
    │   └── Series Name (2021) {tmdb-67890}/
    │       ├── tvshow.nfo
    │       ├── Season 01/
    │       │   ├── season.nfo
    │       │   ├── S01E01 - Episode Title.strm
    │       │   └── S01E01.nfo
    │       └── Season 00 (Specials)/
    │           └── S00E01 - Special Title.strm
    └── Uncategorised/
        └── ...
```

## Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| **Output Root Folder** | Text | `/data/STRM` | Destination for `.strm` and `.nfo` files |
| **Base URL (for .strm)** | Text | `http://dispatcharr:9191` | URL written inside each `.strm` file |
| **Use Direct URLs** | Boolean | ☐ | Write provider URLs directly instead of Dispatcharr proxy URLs |
| **Write NFO files** | Boolean | ✅ | Generate NFO metadata alongside `.strm` files |
| **Cleanup** | Select | Off | Off / Preview / Apply – removes stale files |
| **Max Filesystem Concurrency** | Number | 4 | Maximum concurrent file operations (adaptive throttling adjusts automatically) |
| **Adaptive Throttling** | Boolean | ✅ | Auto-adjust concurrency based on NAS performance |
| **Auto-run after VOD Refresh** | Boolean | ☐ | Automatically generate files when Dispatcharr refreshes VOD content |
| **Dry Run** | Boolean | ☐ | Simulate without writing (testing mode) |
| **Robust debug logging** | Boolean | ☐ | Enable verbose logging to `/data/plugins/vod2strm/logs/` |
| **Name Cleaning Regex** | Text | (empty) | Optional regex to strip patterns from names (e.g., `^(?:EN\|TOP)\s*-\s*`) |
| **Content Filter: Movie IDs** | Text | (empty) | Comma-separated database IDs to include (e.g., `123,456`). Empty = generate all |
| **Content Filter: Series IDs** | Text | (empty) | Comma-separated database IDs to include (e.g., `789,012`). Empty = generate all |

## Content Filtering

Filter specific movies or series by database ID instead of generating everything.

### Finding Content IDs

Content IDs are available in the Dispatcharr database. You can find them by:
- Browsing your Dispatcharr VOD library UI (IDs visible in URLs or via browser dev tools)
- Using database queries if you have direct access
- Using the Dispatcharr API

### Using Content Filters

1. **Get the database IDs** for content you want to include
2. **Add IDs to plugin settings**:
   - Navigate to **Plugins → vod2strm**
   - Enter comma-separated IDs in **Content Filter: Movie IDs**: `123,456,789`
   - Enter comma-separated IDs in **Content Filter: Series IDs**: `321,654,987`
3. **Generate .strm files**: Click **Generate All**

## Actions

| Action | Description |
|--------|-------------|
| **Database Statistics** | Shows content counts, provider breakdown, and series without episodes |
| **Stats (CSV)** | Writes summary CSV (counts from DB + filesystem) |
| **Generate Movies** | Builds `.strm` + NFO files for all movies |
| **Generate Series** | Builds `.strm` + NFO files for all series/episodes |
| **Generate All** | Runs Movies + Series generation plus optional Cleanup |

## Installation

### 1. Create plugin directory

```bash
mkdir -p /opt/dispatcharr/plugins/vod2strm
```

### 2. Copy files

```
vod2strm/
├── __init__.py
└── plugin.py
```

### 3. Restart Dispatcharr

Restart Dispatcharr or reload plugins via the UI.

### 4. Configure

Navigate to **Settings → Plugins → vod2strm** and configure your settings.

## Reports & Logs

### CSV Reports
`/data/plugins/vod2strm/reports/report_<mode>_<timestamp>.csv`

Columns: `type`, `series_name`, `season`, `title`, `year`, `db_uuid`, `strm_path`, `nfo_path`, `action`, `reason`

### Logs
`/data/plugins/vod2strm/logs/vod2strm.log`

Rotating log files with debug information (when enabled).

## Performance Tips

### First Run (100K+ files)
- **Adaptive throttling** starts at 4 workers and adjusts based on your NAS performance
- Slow NAS (>200ms writes): Automatically reduces to 3→2→1 workers
- Fast NAS (<50ms writes): Increases up to your max concurrency setting
- Progress logged every 100 files: `"Movies: processed 100 / 10000 (current workers: 3)"`

### Subsequent Runs
- **Manifest caching** skips writes for unchanged files (most files skip on 2nd+ runs)
- Only new content or changed metadata triggers writes
- Minimal SD card/NAS wear

## Safety Notes

- ✅ Read-only access to Dispatcharr database (no modifications)
- ✅ Writes only within configured output root
- ✅ Compare-before-write prevents redundant I/O
- ⚠️ **Cleanup → Apply** permanently deletes stale files — use Preview first!
- ⚠️ First run on large libraries may take hours (adaptive throttling helps)

## Troubleshooting

### NAS appears "stuck" on first run
**Expected behavior** - Processing 100K files takes time. Check logs for progress:
```
Movies: processed 100 / 10000 (current workers: 3)
Adaptive throttle: NAS slow (avg 0.350s), reducing workers 4 → 3
```

### Files not updating after VOD refresh
- Enable **Auto-run after VOD Refresh** for automatic generation
- Or manually click **Generate All** after refreshing content
- Check **Database Statistics** to verify content is in Dispatcharr

### Some series missing episodes
- Use **Database Statistics** to find series without episodes
- Plugin auto-refreshes series with 0 episodes on first encounter
- Check Dispatcharr VOD refresh to ensure provider has episode data

## Versioning

Semantic versioning: `MAJOR.MINOR.PATCH`

- Upstream: `0.0.13` ([cmc0619/vod2strm](https://github.com/cmc0619/vod2strm))
- This fork: `0.0.13-fork.1`

---

## Fork Changes

Everything below documents what this fork adds on top of upstream `0.0.13`. All changes are confined to `plugin.py`.

### Category directories in output paths

Movies and series are now grouped under their Dispatcharr provider category (the `VODCategory.name` field from `M3UMovieRelation` / `M3USeriesRelation`). The category name is filesystem-safe via `_norm_fs_name`. Content with no category falls into `Uncategorised/`.

### TMDB folder-name suffix

When a valid TMDB ID is available, folder names (and the `.strm` file inside) get a `{tmdb-XXXXX}` suffix. Jellyfin recognises this format and skips its own metadata lookup, which speeds up library scans and avoids mismatches on common titles.

### Full NFO metadata

NFO output was expanded well beyond the upstream title/plot/year stub:

| Element | Source |
| --- | --- |
| `<outline>` | First 200 characters of plot |
| `<runtime>` | `duration_secs // 60` |
| `<genre>` (×N) | `Movie.genre` / `Series.genre` split on `,` and `/` |
| `<director>` | `custom_properties.director` |
| `<actor><name>` (×N) | `custom_properties.actors` (movies) / `custom_properties.cast` (series), comma-separated |
| `<ratings><rating name="tmdb">` | `Movie.rating` / `Series.rating` (numeric) |
| `<uniqueid type="imdb">` | `Movie.imdb_id` / `Series.imdb_id` |
| `<thumb>` | `VODLogo.url` (fixed: upstream used `str(logo)` which returned the name, not the URL) |
| `<showtitle>` | Series name on episode NFOs |
| `<fileinfo><streamdetails>` | Video and audio stream info from `custom_properties.detailed_info` — codec, resolution, aspect ratio, bitrate, channels, sample rate |

### Stream-details key mapping

Dispatcharr stores ffprobe output in `detailed_info` using ffprobe key names (`codec_name`, `display_aspect_ratio`, `bit_rate` in bits/sec, `channel_layout`). The upstream code guessed simpler names. The fork maps correctly and converts `bit_rate` from bits/sec to kbps for the NFO.

### Query additions

- `duration_secs` and `custom_properties` added to `.only()` on Movie and Episode queries.
- `custom_properties` added to `.only()` on Series queries.
- `select_related('category')` added to relation prefetch querysets so category names are fetched in the same query, not N+1.

### Logo URL fix

`_logo_url()` helper extracts `VODLogo.url` correctly. Upstream called `str(logo)` which hit `VODLogo.__str__` and returned the logo name instead of its URL.

## License

MIT / Public Domain — use freely, attribution appreciated.

---

**Made for Dispatcharr** | Upstream: [cmc0619/vod2strm](https://github.com/cmc0619/vod2strm)
