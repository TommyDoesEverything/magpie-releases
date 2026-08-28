# Magpie — releases

Magpie is a small Windows app for saving video, audio, and images from the web.
It works out where a link should go — yt-dlp for video and audio, gallery-dl
for image galleries, instaloader for Instagram archives — and wraps all three
in one window.

This repo carries the published versions and the manifest Magpie reads to keep
itself up to date. Development happens in a separate, private repo.

## Installing

1. Install [Python 3](https://www.python.org/downloads/), and **tick "Add
   python.exe to PATH"** during setup. This is the only prerequisite.
2. From the [latest release](../../releases/latest), download **`magpie.py`**
   and **`Magpie.bat`** into a folder of your own — somewhere you'll find
   again, like `Documents\Magpie`.
3. Double-click **`Magpie.bat`**.

On first run Magpie offers to fetch `yt-dlp.exe` for itself. To merge video
with audio, convert audio, or convert footage for editing, you'll also want
[ffmpeg](https://www.gyan.dev/ffmpeg/builds/) on your PATH.

## Updating

Magpie updates itself. It looks for a new version when it starts and shows a
quiet *"a new version is available"* button when it finds one. Nothing
installs without you clicking it, and nothing ever interrupts a download in
progress.

**Advanced → Updates** has the controls:

- **Look for a new version at startup** — turn it off and Magpie never phones
  home.
- **Versions…** — every release ever published, with dates and what changed.
  Install any of them, including going back to an older one if something new
  misbehaves.

Your settings, cookies, and downloaded files are never touched by an update.
The version being replaced is kept next to the app as `magpie.prev.py`; if a
release ever arrives broken, delete `magpie.py` and rename that file back.

## What's in a release

| File | What it is |
| --- | --- |
| `magpie.py` | the app itself — this is the file that updates |
| `Magpie.bat` | the double-click launcher; you only need it the first time |
| `manifest.json` | the version list Magpie reads when it checks for updates |

## Version numbers

`MAJOR.MINOR.PATCH`:

- **MAJOR** — something you rely on has changed shape; worth reading the notes
  before updating.
- **MINOR** — Magpie can do something it couldn't before, and nothing that
  worked has changed.
- **PATCH** — a fix, with nothing new added.
