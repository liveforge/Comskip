# Comskip Fork — Changes from Upstream

## FFmpeg 8.0 Compatibility (libavcodec 62+)

**Problem:** FFmpeg 8.0 fully removed the `AVCodecContext.ticks_per_frame` field, which was deprecated since FFmpeg 5.1. The upstream Comskip code references this field 11 times in `mpeg2dec.c`, making it impossible to compile against FFmpeg 8.0+.

**Goal:** Compile and run correctly against both FFmpeg 7.1 (libavcodec 61, used by Jellyfin) and FFmpeg 8.0+ (libavcodec 62+), while also improving frame rate detection.

### Changes in `mpeg2dec.c`

#### 1. Added `get_ticks_per_frame()` compatibility helper

Added `#include <libavcodec/codec_desc.h>` and a version-guarded static function that replaces all direct access to the removed field:

- **FFmpeg 8.0+** (`LIBAVCODEC_VERSION_MAJOR >= 62`): Uses `avcodec_descriptor_get()` and checks for `AV_CODEC_PROP_FIELDS` in the codec descriptor. Returns 2 for field-based codecs (MPEG-2) and 1 for everything else (MPEG-1, H.264). This is the official FFmpeg migration path.
- **Older FFmpeg**: Reads `ctx->ticks_per_frame` directly, preserving identical behavior to upstream.

All 11 call sites that previously accessed `->ticks_per_frame` now go through this helper.

#### 2. Improved frame rate detection with `av_guess_frame_rate()`

Three code paths that computed frame delay or FPS from `time_base * ticks_per_frame` as a fallback now use the modern `av_guess_frame_rate()` API first. This function combines information from the stream, codec, and container to produce a more accurate frame rate than raw `time_base` arithmetic, especially for containers with unusual timing (e.g., some MPEG-TS streams).

The cascade in each location is:

1. `dec_ctx->framerate` (or `r_frame_rate`) — primary, unchanged from upstream
2. `av_guess_frame_rate()` — new, better than raw time_base
3. `time_base * get_ticks_per_frame()` — last resort, equivalent to old behavior

Affected code paths:
- **Frame delay calculation** (per-frame decode loop)
- **Initial FPS calculation** (stream open)
- **Live TV retry frame delay** (reopen logic)

#### 3. Guarded MPEG-1 `ticks_per_frame` assignment

The line `is->dec_ctx->ticks_per_frame = 1` for MPEG-1 streams is wrapped in `#if LIBAVCODEC_VERSION_MAJOR < 62`. On FFmpeg 8.0+ the field doesn't exist, and the `get_ticks_per_frame()` helper already returns 1 for MPEG-1 (it lacks `AV_CODEC_PROP_FIELDS`).

#### 4. Removed dead commented-out code

Deleted stale commented-out blocks that referenced `ticks_per_frame` and the non-existent `dec_ctxpar` variable. These were unreachable and caused confusion.

### Impact

- **Build compatibility:** Comskip now compiles cleanly against FFmpeg 7.x and 8.0+ with zero new warnings.
- **Runtime behavior:** Identical output for all supported codecs (MPEG-1, MPEG-2, H.264) — the helper returns the same values the old field held.
- **Frame rate accuracy:** The `av_guess_frame_rate()` fallback is more robust than raw `time_base` arithmetic for edge-case streams where `framerate`/`r_frame_rate` are unset.

## Code Quality & Modernization

### Safety Improvements

- **`vsprintf` → `vsnprintf`** in `Debug()` — prevents buffer overflow in the debug formatting function.
- **`strcpy`/`strcat` → safe alternatives** — replaced 7 unsafe string operations across `comskip.c` and `ccextractor.c` with `memcpy`, `strncpy`, and `strncat` with proper size limits.
- **NULL checks on `malloc`** — added checks after `cc_memory`/`cc_screen` allocations and after `avformat_alloc_context()` in `mpeg2dec.c`.
- **`av_dict_free(&myoptions)`** — fixed AVDictionary memory leak in `mpeg2dec.c` cleanup path.

### Dead Code Removal

- Removed 232-line dead code block (`GetAvgBrightness`, `CheckFrameIsBlack`, `BuildBlackFrameCommList`) that was wrapped in `/* #ifdef notused ... #endif */`.
- Removed dangling forward declarations for the deleted functions.
- Cleaned up `set_fps()` — removed `#ifdef notused` block, commented-out variables, and dead `/* } } else ... */` block.
- Removed `register` storage class keywords (obsolete since C17).
- Removed unused-but-set variables: `sum_brightness2`, `lscore`, `b` (LoadCutScene), `count` (PrintLogoFrameGroups).

### Structural Improvements

- **Replaced `goto`-based thread loops with `for(;;)`** — all 4 scan functions (`ScanBottom`, `ScanTop`, `ScanLeft`, `ScanRight`) now use structured `for(;;)` loops instead of `goto again` for their semaphore-based worker pattern.
- **Hardware decoder selection refactored to lookup table** (`mpeg2dec.c`) — replaced 5 identical if-chains with a single `hw_decoders[]` table + loop.
- **Table-driven INI parsing** — replaced 130+ repetitive `if (FindNumber(...))` chains in `LoadIniFile()` with a static `ini_params[]` table and a 20-line processing loop. Adding a new INI parameter now requires one table entry instead of a copy-paste code block.

### EDL Output Improvements

- **Consolidated EDL output** — created `write_edl_entry()` helper that handles both `demux_pid`/`enable_mencoder_pts` variants. The duplicate `edl_file`/`live_file` blocks in `OutputCommercialBlock` are now a single block calling the shared helper.
- **Increased EDL precision from `%.2f` to `%.3f`** — all EDL/live file output paths (main output, BuildCommListAsYouGo, edlp_file) now use millisecond precision, improving commercial skip accuracy for Jellyfin LiveForge.

### Performance

- **Cached 10-bit conversion frame** (`mpeg2dec.c`) — the `AVFrame` used for YUV420P10LE → YUV420P conversion is now allocated once and reused per-frame instead of being allocated/freed every frame.
