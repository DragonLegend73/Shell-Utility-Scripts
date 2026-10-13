# Pinned progress bar for `zip-dirs`, `pawchive-zip`, `unzip-all` (fish)

Goal: command output scrolls normally, while one **overall** progress bar stays fixed on the bottom row of the terminal (row 40 of your 150x40 terminal).

> **Status:** I could not install fish in my sandbox, so none of this has been run in fish. Do **Step 1** (standalone test) before touching the real scripts. The snippets avoid the usual fish traps (listed at the end), but verify on your machine.

---

## How it works

Terminals support a *scroll region* (DECSTBM, escape `ESC[1;Nr`). If you set the region to rows `1..39`, everything printed scrolls inside those rows and row 40 is untouched. The bar is drawn by:

1. save cursor (`ESC 7`)
2. jump to row 40, clear it, print the bar
3. restore cursor (`ESC 8`)

so the cursor never leaves the scrolling area. On exit, the region is reset and the bar row is wiped. Works in Konsole, kitty, alacritty, tmux, etc.

When stdout is not a TTY (piped or redirected), all bar functions become no-ops and the scripts behave like plain logs.

**Overall progress definition**

| Script | Progress unit | Notes |
| --- | --- | --- |
| `zip-dirs` | one zip "adding:" line (each file and dir entry) | pre-counted with `find` |
| `pawchive-zip` | same | pre-counted: image files + entries in matching dirs |
| `unzip-all` | archive size in bytes | smooth inside `.zip` files (counts `inflating:` lines), jumps per archive for `.7z`/`.rar` |

---

## Step 0: shared block (paste into all three scripts)

Put it **after** `argparse` and the dependency checks, and **delete** each script's old `progress_bar` function.

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
    # newline (make room), save cursor, shrink scroll region, restore cursor, step back up
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
    # string join, NOT (cmd)(cmd): an empty substitution would blank the whole bar
    set -l bar (string join '' (string repeat -n $filled '█') (string repeat -n (math "$bw - $filled") '░'))
    set -l line (printf '%-10s %s %3d%%  %s' "$label" "$bar" $pct "$info")
    set line (string sub -l (math "$cols - 1") -- "$line")
    printf '\x1b7\x1b[%d;1H\x1b[2K%s\x1b8' $__pb_rows "$line"
end

function pb_stop
    test $__pb_on -eq 1; or return
    set -g __pb_on 0
    printf '\x1b7\x1b[r\x1b8'                          # reset scroll region, keep cursor
    printf '\x1b7\x1b[%d;1H\x1b[2K\x1b8' $__pb_rows    # wipe the bar row
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

# Line-buffer zip/unzip output so lines appear live instead of in big chunks
set unbuf
command -q stdbuf; and set unbuf stdbuf -oL
```

---

## Step 1: standalone test

Save the block above as `pb-test.fish`, append this, run `fish pb-test.fish`:

```fish
pb_start
for i in (seq 1 200)
    echo "line $i: some scrolling output"
    pb_draw (math "$i / 200") Testing "$i/200  demo"
    sleep 0.02
end
pb_stop
echo finished
```

Expected: lines scroll, bar stays on the last row and fills, then disappears, and `finished` prints right under the last line. Also test:

- Press **Ctrl+C** mid-run: terminal should return to normal (if not, run `reset`).
- Resize the window mid-run: bar should re-pin to the new bottom row.
- `fish pb-test.fish | cat`: no escape junk, just plain lines.

---

## Step 2: first, strip the old carriage-return clutter

The old `\r\033[K` prefixes are no longer needed. In each script's folder:

```fish
sed -i 's/\\r\\033\[K//g' zip-dirs pawchive-zip unzip-all
```

Then delete leftover empty `printf ""` lines and every remaining `progress_bar ...` call (the sections below replace them).

---

## `zip-dirs`

**A. After `set dirs ...` has been filled and the "No subdirectories" check**, add the pre-count (replaces the old `progress_bar` function):

```fish
set total_dirs (count $dirs)

# zip prints one "adding:" line per file AND per directory entry; find lists the same set
set unit_counts
set total_units 0
for d in $dirs
    set n (find "$d" | count)
    set -a unit_counts $n
    set total_units (math $total_units + $n)
end
set done_units 0
```

**B. After the banner `echo`s, before the loop:**

```fish
pb_start
pb_draw 0 Zipping "0/$total_dirs"
```

**C. Replace the top of the loop** (from `for d in $dirs` through the `if test $pipestatus[1] -ne 0` line):

```fish
for d in $dirs
    set current_dir (math $current_dir + 1)
    set dir_units $unit_counts[$current_dir]
    echo "[$current_dir/$total_dirs] Zipping directory: $d"

    set -l k 0
    $unbuf zip -r "$d.zip" "$d" 2>&1 | while read -l line
        printf '%s\n' "$line"
        set k (math $k + 1)
        pb_draw (math "($done_units + $k) / $total_units") Zipping "$current_dir/$total_dirs  $d"
    end
    set zip_status $pipestatus[1]      # must be the very next line after `end`
    set done_units (math $done_units + $dir_units)
    pb_draw (math "$done_units / $total_units") Zipping "$current_dir/$total_dirs  $d"

    if test $zip_status -ne 0
```

The rest of the loop body stays as is.

**D. At the end**, replace

```fish
progress_bar $total_dirs $total_dirs "Zipping" "Completed"
printf ""
```

with

```fish
pb_draw 1 Zipping "$total_dirs/$total_dirs  done"
pb_stop
```

(`pb_stop` must run before the `SUMMARY` echoes.)

---

## `pawchive-zip`

**A. After `set keys (...)`** (replaces the old `progress_bar` function):

```fish
set total_units (count $image_files)
for d in $dirs
    set total_units (math $total_units + (find "$d" | count))
end
set done_units 0
```

**B. Before the loop**, after the banner:

```fish
pb_start
pb_draw 0 Zipping "0/$total_keys"
```

**C. Inside the loop**, right after `set zip_items (printf '%s\n' $zip_items | sort -V)`, add:

```fish
    set group_units (count $files)
    for d in $matching_dirs
        set group_units (math $group_units + (find "$d" | count))
    end
```

Then replace everything from the `printf "[%d/%d] Creating..."` line through `if test $pipestatus[1] -ne 0` with:

```fish
    echo "[$current_key/$total_keys] Creating: $key.zip"

    set -l k 0
    $unbuf zip -r "$key.zip" -- $zip_items 2>&1 | while read -l line
        printf '%s\n' "$line"
        set k (math $k + 1)
        pb_draw (math "($done_units + $k) / $total_units") Zipping "$current_key/$total_keys  $key"
    end
    set zip_status $pipestatus[1]
    set done_units (math $done_units + $group_units)
    pb_draw (math "$done_units / $total_units") Zipping "$current_key/$total_keys  $key"

    if test $zip_status -ne 0
```

**D. At the end**, same as `zip-dirs`:

```fish
pb_draw 1 Zipping "$total_keys/$total_keys  done"
pb_stop
```

---

## `unzip-all`

**A. Delete the old `progress_bar` function at the top.** Paste the shared block (Step 0) after the `set password_file ...` / `show_help` / `argparse` section, and add this helper right after it:

```fish
# prints one line of tool output; strips 7z's \r progress chatter, skips empties
function show_line --argument-names line
    set -l clean (string replace -r '.*\r' '' -- "$line")
    test -n "$clean"; or return
    printf '%s\n' "$clean"
    pb_draw $cur_frac Extracting "$cur_info"
end
```

**B. After `set total_archives (count $archives)`** (it must come *after* the `sort -V`, so the sizes line up):

```fish
set archive_sizes
set total_bytes 0
for a in $archives
    set s (stat -c %s -- "$a" 2>/dev/null; or echo 0)
    set -a archive_sizes $s
    set total_bytes (math $total_bytes + $s)
end
test $total_bytes -gt 0; or set total_bytes 1
set done_bytes 0
```

**C. After the banner, before the loop:**

```fish
pb_start
pb_draw 0 Extracting "0/$total_archives"
```

**D. Top of the loop.** Replace the `printf "[%d/%d] Extracting..."` and `progress_bar` lines with:

```fish
    set arc_bytes $archive_sizes[$current_archive_num]
    set -g cur_info "$current_archive_num/$total_archives  $base"
    set -g cur_frac (math "$done_bytes / $total_bytes")
    echo "[$current_archive_num/$total_archives] Extracting: $archive"
    pb_draw $cur_frac Extracting "$cur_info"
```

**E. Step 1 (unzip with encoding).** Replace the whole `unzip ... | while ... end` block (keep the surrounding `if` and the `set extract_status $pipestatus[1]` line):

```fish
    if test -n "$is_zip"; and command -q unzip
        set -l entries (unzip -Z1 -- "$archive" 2>/dev/null | count)
        set -l k 0
        $unbuf unzip -O "$zip_encoding" -o -- "$archive" -d "$output_dir" 2>&1 | while read -l line
            printf '%s\n' "$line"
            if string match -qr '^\s*(inflating|extracting|creating|linking):' -- "$line"
                set k (math $k + 1)
            end
            pb_draw (math "($done_bytes + $arc_bytes * min(1, $k / max(1, $entries))) / $total_bytes") Extracting "$cur_info"
        end
        set extract_status $pipestatus[1]
    end
```

**F. Steps 2 to 5 (the four `7z` calls).** In each, replace the multi-line `| while read -l line ... end` with a one-liner and add `$unbuf`:

```fish
        $unbuf 7z x -y -- "$archive" "-o$output_dir" 2>&1 | while read -l line; show_line "$line"; end
        set extract_status $pipestatus[1]
```

Same pattern for the `"-p$password"` and `"-p$manual_password"` variants; keep each one's own `set extract_status $pipestatus[1]` immediately after its `end`. (The `read -s -P "Enter password: "` prompt works fine inside the scroll region.)

**G. End of each loop iteration**, just before `set current_archive_num (math $current_archive_num + 1)`:

```fish
    set done_bytes (math $done_bytes + $arc_bytes)
    pb_draw (math "$done_bytes / $total_bytes") Extracting "$current_archive_num/$total_archives"
```

**H. At the end**, replace the lone `printf ""` before `SUMMARY` with:

```fish
pb_draw 1 Extracting "$total_archives/$total_archives  done"
pb_stop
```

---

## Gotchas (why the code looks the way it does)

- **Use `\x1b7`, not `\0337`.** printf reads `\0337` as one octal number (wrong byte). `\x1b7` is ESC followed by `7`.
- **`(cmd)(cmd)` with an empty result gives nothing.** Fish makes a cartesian product, so an empty `string repeat` would erase the whole bar. Hence `string join ''`.
- **Don't name a variable `status`.** It's a fish builtin. The helper uses `info`.
- **Capture `$pipestatus[1]` on the line right after `end`.** Any command in between overwrites it.
- **`stdbuf -oL`** is needed because `zip`/`unzip` fully buffer when piped, so without it lines arrive in large late chunks and the bar looks frozen.
- **Pre-count cost:** `find | count` walks each tree once before starting. Fast on cached/SSD, noticeable on huge trees.
- **If the terminal is left with a tiny scrolling area** (hard kill, `kill -9`), run `reset`.
- Requires fish 3.x or newer (`fish_exit` event). CachyOS ships 4.x, so you're fine.

## Final checks

1. Step 1 standalone test passes (Ctrl+C and resize included).
2. `zip-dirs` in a scratch folder with 2 or 3 subdirs, using `-k`.
3. `pawchive-zip -k` on a few `ID_Title_001_name.png` files.
4. `unzip-all -k` on a mix of one `.zip`, one `.7z`, and one password-protected archive.
5. `zip-dirs -k | cat` produces clean plain output with no escape codes.