# DVD-MKV Digitizer
<img width="1254" height="1254" alt="DVD-MKV" src="https://github.com/user-attachments/assets/c80817d7-0a56-4a03-9b66-81d42cc4d5e8" />

**DVD-MKV Digitizer** is a desktop media extraction application written in Python with **PyQt6**. It scans optical-media sources, displays discovered titles and tracks, and extracts selected video titles to **MKV** or Audio CD tracks to **FLAC**.

The application is built around two major layers:

* a **scanner** that identifies the source type and collects title, video, audio, subtitle, and chapter information;
* an **extraction engine** that processes selected titles using **HandBrakeCLI and/or FFmpeg**.

## Features

### Supported sources

The source resolver can handle:

* DVD and Blu-ray media
* Optical devices
* DVD `VIDEO_TS` folders and VOB files
* Blu-ray `BDMV/STREAM` folders and M2TS files
* ISO images
* Audio CD sources
* `.cue` and `.cda` sources
* Regular media files supported by FFprobe

### Title scanning

The scanner uses several fallback methods:

1. HandBrakeCLI
2. `lsdvd` for DVDs
3. Direct DVD/VOB inspection
4. Direct Blu-ray/M2TS inspection
5. FFprobe
6. A brute-force media-file scan

HandBrake scan results can be read as JSON or parsed from text. FFprobe can then be used to enrich the discovered metadata.

### Track handling

The application keeps separate models for:

* Video titles
* Audio tracks
* Subtitle tracks
* Chapters
* Disc metadata
* Extraction progress

The GUI allows individual titles, audio tracks, and subtitle tracks to be selected before extraction.

### Output

| Source                       | Output  |
| ---------------------------- | ------- |
| DVD / Blu-ray / video source | `.mkv`  |
| Audio CD                     | `.flac` |

The normal FFmpeg path attempts to **copy streams without re-encoding**. When that path fails, the code can fall back to H.264 encoding using either CPU encoding (`libx264`) or NVIDIA NVENC (`h264_nvenc`).

## Architecture

```text
                    +----------------------+
                    |      PyQt6 GUI       |
                    | MainWindow / Tables  |
                    +----------+-----------+
                               |
                         Qt workers/signals
                               |
             +-----------------+------------------+
             |                                    |
     +-------v--------+                  +--------v---------+
     | FullDiscScanner|                  | ExtractionEngine  |
     | source detection|                 | output + progress |
     +-------+--------+                  +---------+---------+
             |                                      |
        metadata models                       external tools
             |                              /        |       \
      +------v------+                 HandBrakeCLI  FFmpeg  OS tools
      | DVDTitle    |                                      |
      | AudioTrack  |                              +-------v------+
      | Subtitle    |                              | MKV / FLAC   |
      | Chapter     |                              |   outputs    |
      | DiscInfo    |                              +--------------+
      +-------------+
```

## Core components

### `Settings`

Stores application preferences as JSON in the user's home directory.

Settings include:

* output directory
* log directory
* minimum title length
* temporary directory
* sound settings
* theme selection

The code also contains migration support for an older settings filename.

### `FullDiscScanner`

Responsible for:

* identifying the input type;
* mounting ISO images when possible;
* discovering available tools;
* scanning titles;
* collecting metadata;
* enriching metadata with FFprobe;
* cleaning up mounted paths.

### `ExtractionEngine`

Responsible for the actual extraction process.

```text
Selected title
     |
     +---- Audio CD ------> FFmpeg ------> FLAC
     |
     +---- HandBrakeCLI --> MKV
     |
     +---- FFmpeg copy --> MKV
     |
     +---- FFmpeg encode --> H.264 --> MKV
```

It also manages active subprocesses, cancellation, progress parsing, and output-file naming.

### Qt workers

The application uses `QThread` workers for long-running operations:

* `ScanWorker`
* `ExtractWorker`
* `BatchWorker`

This keeps media processing away from the main GUI thread.

### `MainWindow`

The main window handles:

* source selection
* drive detection
* scanning
* title selection
* audio/subtitle selection
* batch queue management
* extraction
* cancellation
* hardware testing
* themes
* progress display
* logging

## Data model

### `DVDTitle`

Represents a discovered title or Audio CD track.

It can contain:

* duration
* video codec
* resolution
* frame rate
* bitrate
* aspect ratio
* source file/VOB paths
* audio tracks
* subtitle tracks
* chapters
* VTS information
* angle information
* source type
* selection state

### `AudioTrack`

Stores:

* codec
* language
* channel count/layout
* sample rate
* bitrate
* title
* default state
* selection state

### `SubtitleTrack`

Stores:

* codec
* language
* title
* forced/default state
* selection state

### `Chapter`

Stores:

* chapter index
* start time
* end time
* chapter title

### `DiscInfo`

Stores source-level information and the list of discovered titles.

### `ExtractProgress`

Carries:

* title index
* total titles
* percentage
* speed
* ETA
* current size
* status
* message

## Dependencies

### Python packages

The source lists:

```bash
pip install PyQt6 PyQt6-Multimedia Pillow pygame-ce pefile
```

Some components are optional at runtime. For example, the program checks whether Pygame and Qt multimedia support are available before using them.

### External tools

The application checks for:

```text
ffmpeg
ffprobe
lsdvd
dvdbackup
HandBrakeCLI
mplayer
mkvmerge
blkid
isoinfo
udisksctl
```

Not every tool is needed for every workflow because the application selects different scanner/extraction paths depending on what is installed.

## Installation

A typical development setup is:

```bash
python3 -m venv .venv
source .venv/bin/activate

pip install PyQt6 PyQt6-Multimedia Pillow pygame-ce pefile

python3 DVD-MKV.py
```

On Windows, use the normal Windows virtual-environment activation command instead.

The program also contains resource-path handling for PyInstaller-style packaged applications.

## GUI workflow

### 1. Source selection

Choose or type a source, detect an optical drive, select a batch directory, or start scanning.

### 2. Global settings

Configure:

* output directory
* log directory
* minimum title duration
* CPU threads
* GPU/NVENC mode
* encoder preset
* sound options

### 3. Title selection

Discovered titles appear in the title list.

The GUI provides quick actions such as:

* Select All
* Deselect All
* Main Only

### 4. Track inspection

Tabs provide information about:

* title metadata
* audio tracks
* subtitles
* chapters
* application information

### 5. Extraction

Selected titles are processed in a background worker while the GUI displays progress, speed, ETA, and status messages.

## Batch processing

Batch mode processes queued sources without requiring every source to be handled manually.

The batch workflow is:

```text
Queue source
     |
     v
Scan source
     |
     v
Filter titles by minimum duration
     |
     v
Choose longest valid title
     |
     v
Extract
     |
     v
Record successful source in history
```

Sources already recorded in the history are skipped.

## Performance options

The application exposes CPU and GPU controls.

CPU encoding uses:

```text
libx264
```

GPU encoding uses:

```text
h264_nvenc
```

Available presets:

```text
fast
medium
slow
```

The source also includes a hardware-test mode that generates a synthetic 1920×1080, 60 FPS video stream and measures encoding performance.

## Configuration and history

The application stores information in the user's home directory.

```text
~/.dvd_mkv_settings.json
~/.dvd_ripper_settings.json
~/.dvd_mkv_history.json
```

The default output directory is:

```text
~/DVDRips
```

The temporary working directory is based on the system temp directory.

## Themes

The application includes a large collection of custom Qt themes, including:

* Dark Blue
* Dark
* Light Grey
* Light
* Grey
* Frutiger Aero
* Dark Aero
* Dark Aero Metallic Green
* Dark Aero Metallic Grey
* Abstract Tech
* DORFiC
* Neo-Aero
* Skeuomorphism
* Frutiger Eco
* Atomic Age
* Cassette Futurism
* Acid Trip

Themes are implemented using Qt palettes and style sheets.

## Platform behavior

### Windows

The code includes Windows-specific handling for:

* optical-drive detection
* ISO mounting
* hiding subprocess windows
* opening output directories
* packaged/application-window behavior

### Linux

The code includes Linux-specific handling for:

* common optical device paths
* ISO loop mounting
* output-folder opening with `xdg-open`

### macOS

macOS has limited generic support in the current source, such as opening output folders, but there is no dedicated optical-drive implementation comparable to the Windows/Linux branches.

## Logging

Errors are written to:

```text
DVD_MKV_Error.log
```

The log location normally follows the configured log directory or falls back to the output directory.

## Cancellation and cleanup

The application can cancel scans and extractions.

The extraction engine attempts to:

1. terminate the active process;
2. wait briefly;
3. force-kill it if necessary.

Mounted ISO/disc paths are also cleaned up after processing.

## Important implementation detail: stream copying

The normal FFmpeg extraction path uses stream copying:

```text
-c copy
```

This avoids unnecessary re-encoding when possible.

The fallback path instead re-encodes the video:

```text
libx264
```

or:

```text
h264_nvenc
```

while copying audio and subtitle streams.

## Audio CD mode

Audio CD tracks are represented internally using the same title model as video titles, but their source type is set to:

```text
audiocd
```

When extraction begins, Audio CD tracks use:

```text
.flac
```

and FFmpeg's FLAC encoder.

## Main-title selection

In batch mode, the program does not automatically extract every title.

It:

1. filters out titles below the configured minimum duration;
2. selects the longest remaining title;
3. extracts that title.

## Persistent completion history

Successful sources are written to:

```text
~/.dvd_mkv_history.json
```

The history is used to avoid repeating sources that have already completed successfully in batch processing.

### `Acid Trip` theme

The source also contains an intentionally disruptive theme with animated effects and sound effects. The application includes special handling around enabling this theme.

## Source structure

The current project is implemented as a single Python file.

Conceptually, it contains:

```text
Imports / constants
        |
Settings
        |
Custom dialogs
        |
Data classes
        |
Disc scanning
        |
Extraction engine
        |
Qt workers/signals
        |
Themes/widgets
        |
MainWindow
        |
Program entry point
```

For a larger project, this could be split into modules:

```text
app/
├── main.py
├── settings.py
├── models.py
├── scanner.py
├── extractor.py
├── workers.py
├── themes.py
└── ui/
    ├── main_window.py
    ├── dialogs.py
    └── widgets.py
```

That separation would make the media-processing code easier to test independently from the PyQt6 interface.

## Usage note

Only use the application with media you are legally permitted to copy or process. Media-copying rules vary by jurisdiction and by the material involved.

## Summary

At a high level, the application works like this:

```text
source
  ↓
detect / mount
  ↓
scan titles
  ↓
inspect tracks
  ↓
select content
  ↓
extract
  ↓
report progress
  ↓
clean up
```

The core technical ideas are:

**tool detection + multiple scan fallbacks + metadata models + Qt worker threads + FFmpeg/HandBrake integration + persistent settings/history + a heavily customized PyQt6 interface**
