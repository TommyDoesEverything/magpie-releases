# Magpie — releases

Magpie is a small Windows app for getting media off the web and into the shape
you need it in. Four tabs named after what you actually came to do, and a
fifth that puts them in an order:

| Tab | What it is for |
| --- | --- |
| **Download** | Paste links. Magpie works out where each one should go — yt-dlp for video and audio, gallery-dl for image galleries, instaloader for Instagram archives — and drives all three from one window |
| **Convert** | Turn files you already have into another codec or container, including the ProRes and DNxHR mezzanines an editor wants, and fifteen audio formats |
| **Resize** | Change the dimensions of video or pictures, or the sample rate of audio |
| **Shrink** | Fit a file inside a size limit — 10 MB for Discord, say — and sound or look as good as possible doing it |
| **Flow** | Do several of those in a row: download these links, resize what comes back, then get it all under ten megabytes. Saved under a name and run again whenever |

Convert, Resize and Shrink work on files from anywhere, not only on things
Magpie downloaded. Drag them onto the window and they land on the tab you
dropped them on.

Each of those three takes **video, images and audio** alike, and works out
which is which from the file itself — so a folder of clips, stills and songs
can go in together and each one is sent to the right engine. Audio is a kind
of file here rather than a checkbox: fifteen formats to write, loudness
targets for broadcast, podcast and streaming delivery, and a tick that takes
the soundtrack out of a video and leaves the picture behind.

A **flow** is how you stop driving the app once per step. Each step is one of
the four tabs and one of its saved profiles, so *download at full quality,
resize to 1080p, get under 10 MB* is one thing you set up once. Everything is
checked before it starts — every profile, every tool, the folder it writes to
— because a forty-minute download that fails at the last step is the failure
worth designing against.

This repo carries the published versions and the manifest Magpie reads to keep
itself up to date. Development happens in a separate, private repo.

**[What Magpie doesn't do yet, and why it might](docs/roadmap.md)** — the
list of problems worth solving, with the reasoning kept where it can be
argued with rather than remembered badly.

## Installing

1. Download **`Magpie.exe`** from the [latest release](../../releases/latest).
2. Put it in a folder of its own — `Documents\Magpie` is fine. Magpie writes
   its settings and downloaded tools next to itself, so it needs somewhere it
   can write. **Not** `Program Files`.
3. Double-click it.

There is nothing to install. Python, the image backends, and everything else
Magpie needs are inside the exe.

Windows will say **"Windows protected your PC"** the first time, because the
exe isn't signed by a paid-for certificate. Click **More info → Run anyway**.
This only happens on that first download — when Magpie updates itself, the new
copy doesn't carry the download marker that triggers the warning.

On first run Magpie offers to fetch `yt-dlp.exe`, and offers `ffmpeg` the
first time something needs it (about 73 MB — it's what joins the best video to
the best audio). Everything else it might want is offered the same way, when
and only when something asks for it: `pngquant` for shrinking pictures, and a
fuller `ffmpeg` build for one voice codec. Two of the fifteen audio formats
need something extra, and Magpie says so in the panel rather than at the end
of a batch.

## Updating

Magpie updates itself. It looks for a new version at startup and shows a quiet
*"a new version is available"* button when it finds one. Nothing interrupts a
download in progress.

By default nothing installs without you clicking it. If you would rather not
click, tick **Help → Install Updates Automatically** — or the box the versions
window offers when you install one by hand. Magpie then installs new versions
quietly in the background, and the new one runs the next time you open it:
nothing restarts underneath you, and a job in progress is never interrupted.

**Help → Versions and Release Notes…** lists every release, with dates and
what changed — newest first, opening on the newest. Install any of them,
including going back if something new misbehaves. **Help → Check for Updates at Startup** turns the check off
entirely.

Your settings, cookies, and downloaded files are never touched by an update.
The version being replaced is kept beside the app as `Magpie.prev.exe`; if a
release ever arrives broken, delete `Magpie.exe` and rename that file back.

## Two shapes of release

Releases **1.0.0 to 1.3.0** are a loose `magpie.py` script that needs Python
installed. **2.0.0 onwards** are the packaged `Magpie.exe`, which needs
nothing.

Magpie only offers versions matching the copy you're running — an exe can't
swap itself for a script, or the other way round. Moving between them means
downloading once by hand.

> **Avoid 2.0.0.** It shipped with a size limit left over from the script
> releases, so it refuses every exe update including its own replacement.
> [2.0.1](../../releases/latest) fixes it. If you already have 2.0.0, download
> 2.0.1 by hand — 2.0.0 can't fetch it for you.

## Version numbers

`MAJOR.MINOR.PATCH`:

- **MAJOR** — something you rely on has changed shape; worth reading the notes
  before updating.
- **MINOR** — Magpie can do something it couldn't before, and nothing that
  worked has changed.
- **PATCH** — a fix, with nothing new added.
