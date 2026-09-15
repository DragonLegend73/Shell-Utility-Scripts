# Shell Utility Scripts

A collection of utility scripts for media conversion, benchmarking, archiving, and file management. The primary scripts are written for the Fish shell (along with one Python script and a Bash helper).

## Common Flags

Many scripts in this collection share a standard set of flags for consistency:
- `-r, --recursive`: Process files in subdirectories recursively.
- `-k, --keep`: Keep original files instead of moving them to the trash (by default, many scripts replace original files).
- `-q, --quality N`: Set the output quality level (e.g., `0-100` for AVIF compression).
- `-s, --speed N`: Set the encoder speed/effort (e.g., `0-10`).
- `-h, --help`: Show detailed help and usage instructions.

## Media Conversion & Compression

### `image-to-avif`
Converts all `.jpg`, `.jpeg`, and `.png` files in the current directory to `.avif`. Trashes original files upon successful conversion.
- **Options**:
  - `-q, --quality N`: Color quality 0-100 (default: 60)
  - `-s, --speed N`: Encoder speed 0-10 (default: 6)
  - `-l, --lossless`: Use lossless encoding
  - `-r, --recursive`: Process subfolders
  - `-k, --keep`: Keep originals

### `pdf-to-avif`
Extracts pages from `.pdf` files and converts them directly to `.avif` images without intermediate PNGs, then trashes the original PDF.
- **Options**: `-q, --quality N`, `-s, --speed N`, `-r, --recursive`, `-k, --keep`

### `compress-mp4`
Compresses `.mp4`, `.mkv`, `.mov`, `.avi` videos to AV1 format (`.av1.mp4`) using SVT-AV1 and Opus audio, optimizing for archival storage. Trashes original files after successful compression.
- **Options**:
  - `-c, --crf N`: Constant Rate Factor (default: 28)
  - `-p, --preset N`: Encoder preset (default: 6)
  - `-r, --recursive`: Process subfolders
  - `-k, --keep`: Keep originals

## Benchmarking

### `benchmark-avif`
Benchmarks AVIF quality by encoding a representative sample of images at various quality levels (default: q30, q40, q50, q60).
Calculates SSIM, PSNR, and SSIMULACRA 2 metrics, generating a detailed CSV report and matplotlib graphs.
- **Options**:
  - `-s, --speed N`: AVIF encoder speed (default: 6)
  - `-n, --sample N`: Maximum images to test (default: 5)
  - `-t, --threshold N`: SSIMULACRA 2 recommendation threshold (default: 70)
  - `-j, --jobs N`: Parallel metric jobs (default: 3)

*Note: Depends on `benchmark-avif-metric-worker` which is included in this repository.*

## Archiving & File Management

### `unzip-all`
Extracts `.zip`, `.rar`, and `.7z` files, automatically testing passwords from `~/.config/archive-passwords.txt`. Auto-detects and decodes Shift-JIS / CP932 / CP949 encodings to prevent garbled filenames.
- **Options**:
  - `-e, --encoding NAME`: Encoding for ZIP filenames (default: shift-jis)
  - `-r, --recursive`: Find archives in subfolders
  - `-f, --flatten`: Extract directly into the archive's root folder
  - `-k, --keep`: Keep archive files

### `zip-dirs`
Zips each subdirectory of the current folder into its own `<dirname>.zip` file, and optionally moves the original directory to the trash.

### `translate-names`
Translates Chinese, Japanese, and Korean (CJK) directory, zip, and image file names into English (or Romaji/Pinyin) using online translation services with caching support.
- **Options**:
  - `-r, --recursive`: Process subdirectories
  - `-n, --dry-run`: Show proposed renames without changing files
  - `--romaji`: Use phonetic Romanization instead of English translation
