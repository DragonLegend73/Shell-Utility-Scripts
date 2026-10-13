# Pinned progress bars, round 2

Covers: `image-to-avif`, `compress-mp4`, `pdf-to-png`, `benchmark-avif` (fish) and `translate-names` (Python).

Yes, all five can get the same treatment: output scrolls normally, one overall bar stays pinned to the bottom row (row 40 on your 150x40 terminal).

> **Status:** same caveat as round 1. I can't run fish in my sandbox, so none of this has been executed. Test each script on scratch data with `-k` (keep) first. See the checklist at the end.

---

## What the bar measures

| Script | Overall progress is based on | Inside a single item |
| --- | --- | --- |
| `image-to-avif` | source bytes (big images count more) | jumps per file (avifenc gives no progress) |
| `compress-mp4` | video duration in seconds, from `ffprobe` | **smooth**, via `ffmpeg -progress` (plus live speed, e.g. `1.4x`) |
| `pdf-to-png` | images per PDF, from `pdfimages -list` | **smooth**, by counting extracted files while `pdfimages` runs |
| `benchmark-avif` | both phases on one bar: encoding 60%, metrics 40% (tunable) | encode: per image; metrics: polls result files |
| `translate-names` | translation batches + 1 unit for the rename phase | countdown during the 10s pauses and 429 retry sleeps |

Two things also change in all of them: per-item log lines now scroll above the bar (several of these scripts printed nothing per file), and the old `\r\033[K` clutter goes away.

---

## Step 0 (fish): shared block

If you already pasted this block into `zip-dirs`, `pawchive-zip` and `unzip-all`, **make two small edits there too** (marked below with `# CHANGED`) and paste the block as is into the four new scripts.

Place it after `argparse` and the dependency checks, and **delete each script's old `progress_bar` function**.

```fish
# ───────────── pinned progress bar (bottom row) ─────────────
set -g __pb_on 0
set -g __pb_rows 0
set -g __pb_state 0 "" ""

function pb_start
    isatty stdout; or return
    set -g __pb_rows (tput lines 2>/dev/null)
    string match -qr '^\d+$' -- "$__pb_rows"; or return
    test $__pb_rows -ge 5; or return
    printf '\n\x1b7\x1b[1;%dr\x1b8\x1b[A' (math $__pb_rows - 1)
    set -g __pb_on 1
end

# usage: pb_draw FRACTION(0..1) LABEL INFO
function pb_draw --argument-names frac label info
    set -g __pb_state $frac "$label" "$info"
    test $__pb_on -eq 1; or return
    set -l cols (tput cols 2>/dev/null; or echo 80)
    set -l f (math "min(1, max(0, $frac))")
    set -l pct (math "floor(100 * $f)")
    set -l bw 40
    set -l filled (math "round($bw * $f)")
    set -l bar (string join '' (string repeat -n $filled '█') (string repeat -n (math "$bw - $filled") '░'))
    set -l line (printf '%-11s %s %3d%%  %s' "$label" "$bar" $pct "$info")      # CHANGED: %-10s -> %-11s ("Compressing" is 11 chars)
    set line (string shorten -m (math "$cols - 1") -- "$line")                  # CHANGED: was `string sub -l`; shorten is width-aware, so CJK file names (2 cells each) can't overflow the row
    printf '\x1b7\x1b[%d;1H\x1b[2K%s\x1b8' $__pb_rows "$line"
end

function pb_stop
    test $__pb_on -eq 1; or return
    set -g __pb_on 0
    printf '\x1b7\x1b[r\x1b8'
    printf '\x1b7\x1b[%d;1H\x1b[2K\x1b8' $__pb_rows
end

function __pb_on_exit --on-event fish_exit
    pb_stop
end

function __pb_on_int --on-signal INT
    pb_stop
    exit 130
end

function __pb_on_winch --on-signal WINCH
    test $__pb_on -eq 1; or return
    set -g __pb_rows (tput lines 2>/dev/null)
    printf '\x1b7\x1b[1;%dr\x1b8' (math $__pb_rows - 1)
    pb_draw $__pb_state
end
# ──────────────────────────────────────────────────────────
```

The `string shorten` change matters for you specifically: `zip-dirs`, `pawchive-zip` and `unzip-all` show archive and folder names that can be CJK, and `string sub -l` counts characters, not terminal cells.

(`stdbuf -oL` from round 1 is not needed here. `ffmpeg -progress` flushes on its own.)

Also run `sed -i 's/\\r\\033\[K//g' SCRIPT` on each fish script to strip the old carriage-return prefixes, then delete leftover empty `printf ""` lines.

---

## `image-to-avif`

**A. After `set total (count $files)`**, pre-count bytes:

```fish
set total_bytes 0
for f in $files
    set total_bytes (math $total_bytes + (stat -c %s -- "$f"))
end
test $total_bytes -gt 0; or set total_bytes 1
set done_bytes 0
```

**B. After the banner, before `for f in $files`:**

```fish
pb_start
pb_draw 0 Converting "0/$total"
```

**C. Top of the loop**, right after `set total_source_bytes (math $total_source_bytes + $source_size)`:

```fish
    pb_draw (math "$done_bytes / $total_bytes") Converting "$processed/$total  $f"
```

**D. Success branch.** After `set total_output_bytes (math $total_output_bytes + $output_size)`, add a log line:

```fish
        printf '[%d/%d] %s  %.2f -> %.2f MiB (%.1f%% saved, %.1fs)\n' \
            $processed $total "$f" \
            (math "$source_size / 1048576") (math "$output_size / 1048576") \
            (math "100 * (1 - $output_size / max(1, $source_size))") $file_elapsed
```

**E. Failure branch.** Replace

```fish
        echo ""
        echo "Error: failed to encode '$f'"
```

with

```fish
        echo "[$processed/$total] ERROR: failed to encode '$f'" >&2
```

**F. End of the loop body.** Replace the last three lines (`set running_saved_bytes`, `set running_saved_mib`, `progress_bar ...`) with:

```fish
    set done_bytes (math $done_bytes + $source_size)
    set running_saved_bytes (math "$total_source_bytes - $total_output_bytes")
    set running_saved_mib (printf "%.1f" (math "$running_saved_bytes / 1048576"))
    pb_draw (math "$done_bytes / $total_bytes") Converting "$processed/$total  $running_saved_mib MiB saved"
```

**G. After the loop**, replace the `printf ""` that sits before the SUMMARY box with:

```fish
pb_draw 1 Converting "$total/$total  done"
pb_stop
```

---

## `compress-mp4`

This one gets the biggest upgrade. Encodes are long, so the bar follows ffmpeg's real position instead of waiting for each file to finish.

**A. Helper + pre-pass.** Add after `set total (count $input_videos)`:

```fish
# duration in seconds; falls back to 1 when ffprobe can't tell (some GIFs report N/A)
function media_seconds --argument-names file
    set -l d (ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 -- "$file" 2>/dev/null | head -n1)
    if string match -qr '^\d+(\.\d+)?$' -- "$d"; and not string match -qr '^0+(\.0*)?$' -- "$d"
        echo $d
    else
        echo 1
    end
end

set durations
set total_sec 0
for v in $input_videos
    set d (media_seconds "$v")
    set -a durations $d
    set total_sec (math $total_sec + $d)
end
set done_sec 0
```

Make sure this comes *after* `set input_videos (... | sort -V)`, so `$durations[N]` lines up with `$input_videos[N]`.

**B. Before the loop** (next to `set running_saved_bytes 0`):

```fish
pb_start
pb_draw 0 Compressing "0/$total"
```

**C. Top of the loop.** Replace these lines:

```fish
    set running_saved_mib (printf "%.1f" (math "$running_saved_bytes / 1048576"))
    progress_bar (math $processed - 1) $total "Compressing" "$running_saved_mib MiB saved"
    echo ""
    echo "[$processed/$total] Encoding: $v"
```

with:

```fish
    set running_saved_mib (printf "%.1f" (math "$running_saved_bytes / 1048576"))
    echo "[$processed/$total] Encoding: $v"
```

**D. Replace the whole ffmpeg section**, from `# Detect if this is a GIF` through `set encode_status $status`. This also collapses your three near-identical ffmpeg calls into one with the same behaviour (GIF: no audio; video with audio: Opus 128k; silent video: no audio):

```fish
    # Detect if this is a GIF (always no audio)
    set is_gif 0
    if string match -qi '*.gif' -- "$v"
        set is_gif 1
    end

    set audio_args -an
    if test $is_gif -eq 0
        set has_audio (ffprobe -v error -select_streams a -show_entries stream=codec_type -of default=nw=1:nk=1 -- "$v" 2>/dev/null)
        if test -n "$has_audio"
            set audio_args -c:a libopus -b:a 128k
        end
    end

    set dur $durations[$processed]
    set speed_txt ""

    # -progress pipe:1 emits key=value blocks on stdout every ~0.5s; real errors still go to the terminal via stderr
    ffmpeg -y -hide_banner -nostdin -loglevel error -nostats -progress pipe:1 \
        -i "$v" \
        -c:v libsvtav1 -preset $preset -crf $crf \
        -pix_fmt yuv420p10le -svtav1-params tune=0 \
        $audio_args \
        -- "$out" | while read -l line
        switch "$line"
            case 'out_time_us=*' 'out_time_ms=*'      # (both are microseconds, despite the name)
                set -l us (string replace -r '^out_time_(us|ms)=' '' -- "$line")
                if string match -qr '^\d+$' -- "$us"
                    set -l cur (math "$us / 1000000")
                    pb_draw (math "($done_sec + min($cur, $dur)) / $total_sec") Compressing "$processed/$total  $running_saved_mib MiB saved  $speed_txt  $base"
                end
            case 'speed=*'
                set speed_txt (string trim -- (string replace 'speed=' '' -- "$line"))
        end
    end
    set encode_status $pipestatus[1]      # must be the very next line after `end`
```

**E. End of the loop body**, just before the loop's final `end`:

```fish
    set done_sec (math $done_sec + $dur)
    set running_saved_mib (printf "%.1f" (math "$running_saved_bytes / 1048576"))
    pb_draw (math "$done_sec / $total_sec") Compressing "$processed/$total  $running_saved_mib MiB saved"
```

(This runs for failed files too, so the bar never stalls.)

**F. After the loop.** Replace

```fish
set running_saved_mib (printf "%.1f" (math "$running_saved_bytes / 1048576"))
progress_bar $total $total "Compressing" "$running_saved_mib MiB saved"
```

with

```fish
pb_draw 1 Compressing "$total/$total  done"
pb_stop
```

and replace `printf "\n\r\033[K\n"` (just before the COMPRESSION SUMMARY) with `echo ""`.

---

## `pdf-to-png`

`pdfimages` prints nothing while it runs, so the script runs it in the background and counts the files appearing in the temp folder.

**A. After `set total (count $pdfs)`:**

```fish
# one row per extractable image (pdfimages -list prints 2 header lines first)
set pdf_units
set total_units 0
for p in $pdfs
    set c (pdfimages -list -- "$p" 2>/dev/null | tail -n +3 | count)
    test $c -ge 1; or set c 1
    set -a pdf_units $c
    set total_units (math $total_units + $c)
end
set done_units 0
```

**B. After the banner, before the loop:**

```fish
pb_start
pb_draw 0 Extracting "0/$total"
```

**C. Top of the loop.** Replace

```fish
    progress_bar $n $total "Extracting" ""
    echo ""
    echo "[$n/$total] Processing: $pdf"
```

with

```fish
    echo "[$n/$total] Processing: $pdf"
```

**D. Replace the single `pdfimages -png -- "$pdf" "$tmp_dir/img"` line** with:

```fish
    pdfimages -png -- "$pdf" "$tmp_dir/img" &
    set pdf_pid $last_pid
    while kill -0 $pdf_pid 2>/dev/null
        set k (find "$tmp_dir" -maxdepth 1 -type f | count)
        pb_draw (math "min($done_units + $k, $total_units) / $total_units") Extracting "$n/$total  $name"
        sleep 0.2
    end
    wait $pdf_pid
```

**E. End of the loop body**, just before `set n (math $n + 1)`:

```fish
    set done_units (math $done_units + $pdf_units[$n])
    pb_draw (math "$done_units / $total_units") Extracting "$n/$total  $name"
```

**F. Before the SUMMARY**, replace the lone `printf ""` (the old `\r\033[K` one) with:

```fish
pb_draw 1 Extracting "$total/$total  done"
pb_stop
```

---

## `benchmark-avif`

Both phases share one bar. The split is a guess; tune it after your first run.

**A. Delete the old `progress_bar` function and paste the Step 0 block** after the argument validation. Then add near the other settings at the top:

```fish
# share of the overall bar per phase (must add up to 1.0)
set w_encode 0.6
set w_metrics 0.4
```

**B. Phase 1.** After `set combo_num 0`:

```fish
pb_start
pb_draw 0 Encoding "0/$total_combos"
```

**C. Inside the encode loop.** Before `set file_start_ns (date +%s%N)`, add:

```fish
        pb_draw (math "$w_encode * ($combo_num - 1) / $total_combos") Encoding "$combo_num/$total_combos  q$quality  $filename"
```

Then replace the `if test $encode_status -ne 0; ... end` block and the `progress_bar $combo_num ...` line with:

```fish
        if test $encode_status -ne 0; or not test -f "$output"
            printf "  [%d/%d] FAILED: q%s %s\n" $combo_num $total_combos "$quality" "$filename"
            echo "FAIL $elapsed" >> "$encode_times_file"
        else
            echo "$elapsed" >> "$encode_times_file"
            printf "  [%d/%d] q%s  %s  %.2fs\n" $combo_num $total_combos "$quality" "$filename" $elapsed
        end

        pb_draw (math "$w_encode * $combo_num / $total_combos") Encoding "$combo_num/$total_combos  q$quality  $filename"
```

**D. After Phase 1.** Delete the `printf "\r\033[K"` line (keep `echo " Encoding complete."`). The bar stays pinned into Phase 2.

**E. Phase 2.** Replace the polling loop and what follows it (from `while kill -0 $parallel_pid ...` through `printf "\r\033[K"`) with:

```fish
set metric_total (math "max(1, $total_metric_jobs)")

while kill -0 $parallel_pid 2>/dev/null
    set completed (count (path filter -f $tmpdir/metric_q*_*.txt 2>/dev/null))
    pb_draw (math "$w_encode + $w_metrics * min($completed, $metric_total) / $metric_total") Metrics "$completed/$total_metric_jobs  $metric_jobs workers"
    sleep 0.5
end
wait $parallel_pid

pb_draw 1 Metrics "$total_metric_jobs/$total_metric_jobs  done"
pb_stop
```

(I left your `path filter` polling expression as it was. The bar is only as accurate as that count: if `benchmark-avif-metric-worker` creates its result file at the *start* of a job rather than the end, the bar runs slightly ahead.)

---

## `translate-names` (Python)

Same escape-code technique, as a small class. It also handles CJK widths (these file names are exactly the case where it matters).

**A. Imports.** Add:

```python
import atexit
import shutil
import signal
import unicodedata
```

**B. Replace the `progress_bar()` function** with:

```python
def _fit(text, width):
    """Truncate to a terminal *cell* width (CJK characters take 2 cells)."""
    out, used = [], 0
    for ch in text:
        w = 2 if unicodedata.east_asian_width(ch) in ("W", "F") else 1
        if used + w > width:
            break
        out.append(ch)
        used += w
    return "".join(out)


class PinnedBar:
    """Overall progress bar pinned to the bottom terminal row; output scrolls above it."""

    def __init__(self):
        self.on = False
        self.rows = 0
        self.state = (0.0, "", "")

    def start(self):
        if self.on or not sys.stdout.isatty():
            return
        self.rows = shutil.get_terminal_size().lines
        if self.rows < 5:
            return
        # newline (make room), save cursor, shrink scroll region, restore cursor, step back up
        sys.stdout.write(f"\n\x1b7\x1b[1;{self.rows - 1}r\x1b8\x1b[A")
        sys.stdout.flush()
        self.on = True
        atexit.register(self.stop)
        signal.signal(signal.SIGWINCH, self._on_winch)
        signal.signal(signal.SIGTERM, lambda *_: sys.exit(143))

    def draw(self, frac, label, info=""):
        self.state = (frac, label, info)
        if not self.on:
            return
        cols = shutil.get_terminal_size().columns
        f = min(1.0, max(0.0, frac))
        filled = round(40 * f)
        line = f"{label:<11s} {'█' * filled}{'░' * (40 - filled)} {int(100 * f):3d}%  {info}"
        sys.stdout.write(f"\x1b7\x1b[{self.rows};1H\x1b[2K{_fit(line, cols - 1)}\x1b8")
        sys.stdout.flush()

    def stop(self):
        if not self.on:
            return
        self.on = False
        sys.stdout.write(f"\x1b7\x1b[r\x1b8\x1b7\x1b[{self.rows};1H\x1b[2K\x1b8")
        sys.stdout.flush()

    def sleep(self, seconds, why):
        """time.sleep that keeps the bar alive with a countdown."""
        frac, label, info = self.state
        end = time.time() + seconds
        while True:
            left = end - time.time()
            if left <= 0:
                break
            self.draw(frac, label, f"{why}: resuming in {int(left) + 1}s")
            time.sleep(min(1, left))
        self.draw(frac, label, info)

    def _on_winch(self, *_):
        if not self.on:
            return
        self.rows = shutil.get_terminal_size().lines
        sys.stdout.write(f"\x1b7\x1b[1;{self.rows - 1}r\x1b8")
        self.draw(*self.state)


bar = PinnedBar()
```

`atexit` covers normal exit, `sys.exit(1)` and Ctrl+C (the traceback prints first, inside the scroll region, then the bar is cleaned up).

**C. In `translate_batch()`**, the 429 branch: replace `time.sleep(wait)` with the countdown version (the 2s retries can stay as they are):

```python
            if e.code == 429:
                wait = [30, 60, 90, 120][min(attempt, 3)]
                print(f"   ⚠ 429 rate limited, sleeping {wait}s (attempt {attempt+1})...")
                bar.sleep(wait, "429 rate limited")
```

**D. In `main()`**, replace the whole translation section, from `cache_hits = len(translations)` through the end of the `if uncached:` block (including the trailing `progress_bar(total_batches, ...)` and `sys.stdout.write("\r\033[K")`), with:

```python
    cache_hits = len(translations)
    total_batches = (len(uncached) + BATCH_SIZE - 1) // BATCH_SIZE
    total_units = total_batches + 1          # +1 = the renaming phase
    bar.start()
    bar.draw(0, "Translating", f"0/{total_batches} batches  cache: {cache_hits}")

    if uncached:
        if not api_key:
            print("Error: Gemini API Key not found. Pass --api-key, export GEMINI_API_KEY=..., or save to ~/.config/gemini-api-key", file=sys.stderr)
            sys.exit(1)
        for i in range(0, len(uncached), BATCH_SIZE):
            batch = uncached[i : i + BATCH_SIZE]
            batch_num = i // BATCH_SIZE + 1
            info = f"batch {batch_num}/{total_batches}  cache: {cache_hits}"
            bar.draw((batch_num - 1) / total_units, "Translating", info)
            try:
                result = translate_batch(batch, api_key)
                translations.update(result)
                cache.update(result)
                save_cache(cache)
                print(f"   ✓ Got {len(result)} translation(s)\n")
            except Exception as e:
                print(f"   ⚠ Batch error ({e}), falling back to item-by-item Gemini API calls...")
                fallback_success = 0
                for single_name in batch:
                    try:
                        res = translate_batch([single_name], api_key)
                        if res and single_name in res:
                            translations.update(res)
                            cache.update(res)
                            fallback_success += 1
                        else:
                            print(f"   ↳ Skipping '{single_name}' (API response missing key)")
                    except Exception as single_err:
                        print(f"   ↳ Skipping '{single_name}' ({single_err})")
                save_cache(cache)
                print(f"   ✓ Translated {fallback_success}/{len(batch)} item(s) via individual API calls\n")

            bar.draw(batch_num / total_units, "Translating", info)
            if batch_num < total_batches:
                bar.sleep(10, "rate-limit pause")   # respect free-tier rate limits
```

Behaviour is the same as before, except the missing-API-key check now happens once before the first batch instead of inside the `try`.

**E. Rename loop.** Replace the first line of the loop body:

```python
        progress_bar(idx, total_files, "Renaming", f"{skipped} skipped")
```

with:

```python
        bar.draw((total_batches + idx / max(1, total_files)) / total_units,
                 "Renaming", f"{idx}/{total_files}  {skipped} skipped")
```

and delete every `sys.stdout.write("\r\033[K")` / `sys.stdout.flush()` pair in the loop (dry-run branch, success branch, error branch).

**F. After the loop.** Replace

```python
    progress_bar(total_files, total_files, "Renaming", f"{skipped} skipped")
    sys.stdout.write("\r\033[K\n")
    sys.stdout.flush()
```

with:

```python
    bar.draw(1, "Renaming", f"{total_files}/{total_files}  {skipped} skipped")
    bar.stop()
```

---

## Gotchas

- **Capture `$pipestatus[1]` on the line right after `end`.** `compress-mp4` now runs ffmpeg inside a pipeline, so the old `set encode_status $status` would report the `while` loop's status instead of ffmpeg's.
- **Ctrl+C during `compress-mp4`** leaves a partial `*.av1.mp4` behind (same as before). The bar is still cleaned up properly.
- **`media_seconds` fallback of 1s** means GIFs without a reported duration jump the bar when they finish instead of moving smoothly. They're usually tiny, so it's not noticeable.
- **`pdfimages -list` pre-pass** parses every PDF once before extraction starts. Quick for normal PDFs, slower for huge scanned ones.
- **`benchmark-avif` weights** (`w_encode` + `w_metrics`) must add up to 1.0, otherwise the bar jumps at the Phase 2 to done transition.
- **`string shorten -m`** needs fish 3.1 or newer (you're on 4.x).
- When stdout isn't a terminal (`| cat`, redirect, cron), every bar function is a no-op, so logs stay clean.
- If a script is hard-killed (`kill -9`) the terminal may keep a shrunken scroll region. Run `reset`.

---

## Test checklist

Use a scratch folder and `-k` so nothing gets trashed.

1. **`image-to-avif -k`** on 5 to 10 mixed-size PNG/JPGs. Expect per-file log lines and a bar that moves in byte-weighted steps.
2. **`compress-mp4 -k`**. Make a quick clip: `ffmpeg -f lavfi -i testsrc=duration=20:size=1280x720:rate=30 test.mp4`. Expect smooth movement and a live `speed` readout. Also try one `.gif`.
3. **`pdf-to-png -k`** on a PDF with a few dozen images. Expect the bar to creep forward during each PDF.
4. **`benchmark-avif -n 3`**. Expect the bar to run through encoding (\~60%) then metrics.
5. **`python translate-names -n`** (dry run). Test with a mix of cached and uncached names. Watch the countdown during the 10s pause.
6. For each script: Ctrl+C mid-run (terminal returns to normal), resize the window mid-run (bar re-pins), and run with `| cat` (no escape junk).