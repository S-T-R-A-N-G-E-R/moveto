# moveto

**A terminal utility for moving files to FAT32 drives — without the 4 GB headache.**

FAT32 is still the most compatible format for external drives, USB sticks, and SD cards. But it comes with a hard limit: **no single file can exceed 4 GB**. `moveto` handles this automatically — it detects the destination filesystem, splits oversized files into individually playable segments, verifies everything landed correctly, and only then removes the originals.

No data loss. No manual splitting. No leftovers.

---

## Features

- **Auto-detects FAT32** — no flags needed; the constraint is applied only when the destination requires it
- **Safe deletion** — source files are removed only after size and playability are confirmed on the destination
- **Smart splitting** — oversized video files are split into 60-second stream-copied segments (no re-encoding, lossless)
- **Organised output** — each split file gets its own subfolder named after the original
- **Works with any file** — non-video files are verified by size; video files are also verified by `ffprobe`
- **Interactive or scripted** — pass paths as arguments or let it prompt you
- **Coloured output** — progress and status at a glance (degrades gracefully in non-TTY contexts)

---

## Requirements

- macOS (uses `diskutil` for filesystem detection)
- [ffmpeg](https://ffmpeg.org) (includes `ffprobe`)

```bash
brew install ffmpeg
```

---

## Installation

```bash
# 1. Download the script
curl -o ~/.local/bin/moveto https://raw.githubusercontent.com/S-T-R-A-N-G-E-R/moveto/main/moveto

# 2. Make it executable
chmod +x ~/.local/bin/moveto

# 3. Ensure ~/.local/bin is in your PATH (add to ~/.zshrc or ~/.bashrc if needed)
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Or clone and symlink if you prefer keeping it in a repo:

```bash
git clone https://github.com/S-T-R-A-N-G-E-R/moveto.git ~/Developer/moveto
ln -s ~/Developer/moveto/moveto ~/.local/bin/moveto
chmod +x ~/.local/bin/moveto
```

---

## Usage

### Interactive mode

Run `moveto` with no arguments and it will prompt you for both paths:

```
$ moveto
Source directory: ~/Videos/Recordings
Destination directory: /Volumes/MyDrive/Backups
```

Paths can be typed with or without surrounding quotes — both work.

### Non-interactive mode

Pass both paths directly as arguments:

```bash
moveto ~/Videos/Recordings /Volumes/MyDrive/Backups
```

### Help

```bash
moveto --help
```

---

## How It Works

```
moveto
  │
  ├── Detects destination filesystem (FAT32 or not)
  │
  ├── Scans source directory (top-level files only)
  │
  ├── For each file ≤ 4 GB
  │     └── cp  →  verify size + playability  →  delete source
  │
  └── For each file > 4 GB  (FAT32 destinations only)
        └── ffmpeg stream-copy split into 60 s segments
              └── Each segment verified  →  delete source
```

### Splitting behaviour

When a file exceeds the FAT32 limit, `moveto` creates a subfolder on the destination named after the original file and splits it into 60-second segments using `ffmpeg -c copy` (stream copy — no re-encoding, no quality loss, very fast). Each segment is a valid, independently playable video file.

**Example:**

```
Source:
  lecture.mov  (12 GB)

Destination:
  lecture/
    part000.mov  (~730 MB)
    part001.mov  (~730 MB)
    ...
    part019.mov  (~480 MB)
```

The 60-second window is conservative by design. Even videos with significant bitrate variation (e.g. screen recordings that include video playback) stay well under the 4 GB ceiling per segment.

### Verification

Before any source file is deleted, `moveto` checks:

| File type | Checks |
|-----------|--------|
| Any file | Exists at destination, size > 0, size matches source |
| Video files (`.mov`, `.mp4`, `.mkv`, `.avi`, `.m4v`, `.webm`, `.hevc`) | All of the above + `ffprobe` can read and report a valid duration |

If any check fails, the source file is **not** deleted and the issue is reported.

### Partial success

If some files transfer successfully but others fail:

- The successfully transferred files are offered for deletion (with a prompt)
- The failed ones are left untouched on the source
- The process exits with a non-zero status

---

## Configuration

The script has a small config block at the top you can edit:

```bash
MAX_FAT32_BYTES=4294967296   # 4 GiB hard limit (don't change this)
SPLIT_SECS=60                # Segment length in seconds — lower = smaller files
VIDEO_EXTS="mov mp4 mkv avi m4v webm hevc"  # Extensions treated as video
```

To change the segment length to 30 seconds:

```bash
# Open the script in your editor
nano ~/.local/bin/moveto

# Change:
SPLIT_SECS=60
# To:
SPLIT_SECS=30
```

> **When to lower `SPLIT_SECS`:** If you encounter a warning that a segment still exceeded the FAT32 limit (rare — only happens with extremely high and sustained bitrate), halving the segment length will fix it.

---

## Example output

```
=== moveto ===
[10:30:01] Source:      ~/Videos/Recordings
[10:30:01] Destination: /Volumes/MyDrive/Backups
  ⚠  FAT32 detected — files > 4 GB will be split into 60s segments.

Found 5 file(s): 4 direct-copy, 1 need splitting

── Copying files ──
[10:30:01] Copying (1.2 GB): recording-01.mov
  ✓ recording-01.mov
[10:30:34] Copying (2.8 GB): recording-02.mov
  ✓ recording-02.mov

── Splitting large files ──
[10:31:55] Large file (9.3 GB): recording-03.mov
  Duration: 2100.0s → 35 segment(s) of up to 60s
  Subfolder: recording-03/
[10:34:10]   ✓ segment: part000.mov (451 MB)
  ✓ segment: part001.mov (512 MB)
  ...
  ✓ recording-03.mov → 35 segment(s) in recording-03/

════════════════════════════
  Verified:  5 / 5
  Failed:    0 / 5

[10:34:11] Deleting verified source files...
  ✓ Deleted: recording-01.mov
  ✓ Deleted: recording-02.mov
  ✓ Deleted: recording-03.mov

Done. All 5 file(s) moved successfully.
```

---

## Limitations

- Processes **top-level files only** — subdirectories in the source are not traversed
- **macOS only** — uses `diskutil` for filesystem detection; Linux/Windows would need adapting
- Split segments must be **reassembled manually** if you need the original single file back (use `ffmpeg -f concat` or `cat` for stream-copied segments)
- The FAT32 split logic applies only when the **destination** is FAT32; it never splits for non-FAT32 destinations regardless of file size

---

## Reassembling split files

If you later move the drive to a system that supports large files (APFS, NTFS, ext4) and want to join the segments back:

```bash
# Generate a concat list
ls recording-03/*.mov | sed "s/^/file '/" | sed "s/$/'/" > concat.txt

# Merge (stream copy — lossless, fast)
ffmpeg -f concat -safe 0 -i concat.txt -c copy recording-03-full.mov

# Clean up
rm -rf recording-03/ concat.txt
```

---

## License

MIT — do whatever you want with it.
