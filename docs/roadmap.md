# Roadmap

> Copied from Magpie's source repo, which is private. References to files
> and functions are left as plain names rather than links, because the code
> they point at is not in this repo.

Things Magpie could do and doesn't, written down so they stop being
remembered badly. Nothing here is a promise, and nothing here is scheduled —
it is a list of problems worth solving, sorted by how much solving them would
change the program.

Every entry leads with **the problem**, because a feature that cannot name the
failure it fixes does not belong in Magpie any more than it belongs in the
README. Anything that turns out not to be worth doing moves to
[Parked](#parked) with the reason, so it does not get re-argued from scratch
in six months.

## How to read an entry

| Field | Means |
| --- | --- |
| **The problem** | What someone has to do today instead, and why that is bad |
| **What to build** | The shape of the answer, not the implementation |
| **Where it lands** | The modules that would change |
| **Size** | Rough: an evening, a weekend, a week |

Tick a box when it ships. Move the entry into the README when it does, and
delete it from here — this file holds what Magpie *isn't*.

## The list

**Bump** is the part of the version number the entry would move — see
[Version numbers](../README.md#version-numbers).

| | | Bump |
| --- | --- | --- |
| | **The big two** | |
| 1 | [A "Send it to…" tab — route by destination](#1-a-send-it-to-tab--route-by-destination) | MINOR |
| 2 | [Trim and crop — the content Magpie cannot touch](#2-trim-and-crop--the-content-magpie-cannot-touch) | MINOR |
| | **Worth doing next** | |
| 3 | [Hardware acceleration beyond NVIDIA](#3-hardware-acceleration-beyond-nvidia) | MINOR |
| 4 | [A results ledger](#4-a-results-ledger) | MINOR |
| 5 | [Pipelines — the machinery is already written](#5-pipelines--the-machinery-is-already-written) | MINOR |
| 6 | [A CLI, and Explorer's right-click menu](#6-a-cli-and-explorers-right-click-menu) | MINOR |
| 7 | [AV1 on the Shrink tab](#7-av1-on-the-shrink-tab) | MINOR |
| | **Small wins** | |
| 8 | [Thumbnails in the queues](#8-thumbnails-in-the-queues) | MINOR |
| 9 | [Subtitle burn-in](#9-subtitle-burn-in) | MINOR |
| 10 | [The queue survives a restart](#10-the-queue-survives-a-restart) | PATCH |
| 11 | [Clipboard watcher on Download](#11-clipboard-watcher-on-download) | MINOR |
| 12 | [Dedupe by content, not by URL](#12-dedupe-by-content-not-by-url) | PATCH |
| 13 | [An audio path worth the name](#13-an-audio-path-worth-the-name) | MINOR |
| | [**Parked**](#parked) — considered, and not obviously worth it | |

---

## The big two

The ones that change what Magpie is, rather than what it has.

### 1. A "Send it to…" tab — route by destination

- [ ] **The problem.** The four tabs are verbs because that is what people
  arrive with. But the verb most people actually arrive with is *"I need to
  post this somewhere"*, and Magpie makes them translate that into a codec, a
  size cap, a resolution and an aspect ratio by hand — four lookups on four
  websites before they can touch a dropdown. Nobody thinks *"H.264 at 1080p
  in 9.8 MB"*. They think *"Discord"*.

  **What to build.** A fifth tab that takes files and a destination, and
  derives the settings from it. The constraints are public and stable:

  | Target | What it means |
  | --- | --- |
  | Discord | ≤10 MB, or 25 / 500 MB with Nitro; H.264 MP4 |
  | Instagram Reels / TikTok | 9:16, ≤90s, H.264 |
  | X | ≤512 MB, ≤140s, ≤1080p |
  | Email | ≤20 MB, and playable on anything |
  | Premiere | the H.264-or-ProRes logic the README already explains at length |

  It composes machinery that already exists — Shrink hits a size, Resize hits
  a resolution, Convert hits a codec — so most of the work is the panel and
  the table behind it. It should show what it derived, the way every other
  profile shows what it pinned, rather than being a black box with a logo on
  it.

  **Where it lands.** A new tab in `app.py`, a target table in `codecs.py`,
  reusing `convert_jobs_for` and the shrink job builders.
  Depends on [Trim and crop](#2-trim-and-crop--the-content-magpie-cannot-touch)
  for the aspect-ratio targets.

  **Size.** A weekend for the tab, longer to get the targets right and keep
  them right.

### 2. Trim and crop — the content Magpie cannot touch

- [ ] **The problem.** All four tabs change a file's format, its dimensions or
  its weight. None of them change what is *in* it. "I need thirty seconds of
  this" and "make this vertical" are two of the most common things anyone
  wants from a video, and both send you to another program — which is exactly
  the three-programs-and-six-tabs problem Magpie exists to end.

  **What to build.** A **Trim** tab: in and out points, and a crop or an
  aspect-ratio target. Two honest modes, chosen and explained rather than
  guessed:

  | Mode | What it costs |
  | --- | --- |
  | **Copy** (`-c copy`) | Instant and lossless, but the cut snaps to the nearest keyframe — so it lands near where you asked, not on it |
  | **Exact** | Re-encodes, so the cut is frame-accurate and the file is a generation down |

  This wants the same treatment *About two-pass*
  gets: say which one ran and why, rather than picking silently.

  **The Download-side payoff is larger than the editing one.** yt-dlp's
  `--download-sections "*1:30-2:00"` fetches only that range, so a trim set
  before the download saves the bandwidth and the disk as well as the
  scrubbing. That makes trim a control on the Download tab too, not only a
  tab of its own.

  **Where it lands.** New tab in `app.py`, `-ss`/`-to`/`-vf crop` in the
  job builders, `--download-sections` in `download_args`.

  **Size.** A week, mostly because setting in and out points without a
  preview is miserable — see
  [Thumbnails and preview](#8-thumbnails-in-the-queues).

---

## Worth doing next

High value, and each one contained enough to finish in a sitting or two.

### 3. Hardware acceleration beyond NVIDIA

- [ ] **The problem.** `codecs.py` offers NVENC and
  nothing else, so hardware encoding is for people with an NVIDIA card and
  software-only for everyone else. That excludes every Intel laptop — QSV has
  been on essentially every chip since 2015 — and every AMD card.

  **What to build.** Offer only what this machine can actually do — the same
  move `about_report` already makes for the tools it found.

  **Two things this entry used to say have been measured and were wrong**,
  so they are written down in [Benchmarks](benchmarks.md) rather than left
  here to be believed:

  - *"Probe `ffmpeg -encoders`."* That lists what the **build** carries, not
    what the machine can run — a full build offers QSV and AMF everywhere and
    fails at run time for most people, which is this entry's own complaint
    about NVENC, three times over. The probe has to be a five-frame trial
    encode to `null`, which separates them cleanly and costs about half a
    second each.
  - *"`-hwaccel auto` on the decode side by default."* Measured on a 4K
    H.264 source it was **5.7× slower** decoding, **33% slower** into
    `libx264`, and a wash into ProRes. Hardware decode puts frames in GPU
    memory and a software encoder needs them back, and that readback costs
    more than the decode saves. Not a default, and not the free win this
    entry claimed. It belongs with a *hardware* encoder, where the frames
    never leave the card — which makes it part of the QSV/AMF work rather
    than a cheap thing to do first.

  What survives is the original complaint: hardware encoding is NVIDIA-only,
  which excludes every Intel laptop — QSV has been on essentially every chip
  since 2015 — and every AMD card.

  **Where it lands.** `codecs.py`, `external.py` for the probe, About box.
  The probe wants a background thread and a cached answer, not the startup
  path.

  **Size.** A weekend to add and test QSV/AMF tiers, now that the evening
  that was supposed to come first turned out not to exist.

### 4. A results ledger

- [ ] **The problem.** The Output pane is the only record of what happened, and
  it dies with the window. There is no memory across sessions of what ran,
  what came out, where it went, or whether it worked. *"Where did that file
  go"* and *"did the third one of those forty fail"* both currently mean
  scrolling a log, if the log is even still there.

  **What to build.** One row per finished item — what it was, what came out,
  how big, how long, pass or fail. Double-click opens the file, right-click
  opens its folder, re-runs it, or sends its output to another tab.

  It costs almost nothing to collect:
  `_worker` already gathers the `produced` paths
  for every job and throws them away once clash detection has read them.

  **Where it lands.** `runner.py` reports it, a new pane or dialog in
  `app.py`, JSON beside the profiles.

  **Size.** A weekend.

### 5. Pipelines — the machinery is already written

- [ ] **The problem.** Every multi-step job means driving the app twice by
  hand: download it, then switch tabs, then add the file you just made, then
  shrink it. Meanwhile `Job.chain`, `Job.retry` and
  `Runner.start(expand=…)` already exist, and one
  pipeline is already hard-coded on top of them —
  download then convert for editing.

  **What to build.** Generalise that one case into a **Then…** row on every
  tab. Download → Shrink to 10 MB. Resize → Shrink. Convert → move to
  archive, which is already special-cased and would stop needing to be.

  **Where it lands.** `app.py`, mostly. The runner already does it.

  **Size.** An evening for the plumbing, a weekend to make the UI honest
  about what will run.

### 6. A CLI, and Explorer's right-click menu

- [ ] **The problem.** `cli.py` handles `--check` and
  `--backend` and nothing else, so Magpie cannot be scripted, scheduled, or
  pointed at a folder — which is a strange thing to be true of a program
  descended from three `.bat` files you dragged files onto.

  **What to build.**

  ```
  Magpie.exe shrink --mb 10 clip.mp4
  Magpie.exe convert --profile "Mezzanine (ProRes 422 HQ)" *.mov
  ```

  The vocabulary is already written: profiles are named and saved as JSON, so
  the CLI's arguments are the names people have already chosen.

  The real prize is what it unlocks — **Explorer's context menu**. Right-click
  any video, *Magpie → Shrink to 10 MB*. That is the whole app, minus the
  window, at the exact point where people meet their files.
  `installer/Magpie.iss` is where it would be
  registered, and the portable copy should be able to register and unregister
  itself from Tools.

  **Where it lands.** `cli.py`, `Magpie.iss`, a Tools menu entry.

  **Size.** A weekend for the CLI. The shell integration is a day and a lot
  of care about uninstalling cleanly.

### 7. AV1 on the Shrink tab

- [ ] **The problem.** Shrink's whole job is *fit inside N MB and look as good
  as possible*, and it does it with H.264 — a codec from 2003. SVT-AV1 at the
  same bitrate looks meaningfully better, often on the order of 30%. This is
  the one tab where the codec choice directly serves the tab's stated
  purpose, and it is the tab with the least codec choice.

  **What to build.** SVT-AV1 as an option on Shrink, with the honest warning
  attached: it is slower, and it will not play everywhere H.264 plays. Which
  makes it a choice rather than a default — and makes it wrong for the
  Discord target in
  [Send it to…](#1-a-send-it-to-tab--route-by-destination), which needs
  universal playback more than it needs the last 30%.

  **Where it lands.** `codecs.py`, the shrink job builder.

  **Size.** An evening.

---

## Small wins

### 8. Thumbnails in the queues

- [ ] **The problem.** It is a media tool whose queues are lists of filenames.
  Working out which of forty rows is the wrong one means reading forty names.

  **What to build.** A small thumbnail column. Pillow is already bundled and
  ffmpeg can pull a frame, so nothing new has to be fetched. A larger preview
  on hover or click.

  This is also what makes [Trim](#2-trim-and-crop--the-content-magpie-cannot-touch)
  bearable, so the two are worth doing in that order.

### 9. Subtitle burn-in

- [ ] **The problem.** Magpie downloads subtitles and then cannot burn them
  in. Soft subtitles vanish on Instagram, on TikTok, and on import to
  Premiere — which are three of the places the files are going.

  **What to build.** `-vf subtitles=` on the Convert tab, taking either an
  embedded track or a sidecar `.srt`/`.vtt`.

### 10. The queue survives a restart

- [ ] **The problem.** Convert rows each carry the settings that were showing
  when they went in — that is the whole point of the queue, and it is real
  work to assemble. A crash, an update, or a mis-clicked close throws all of
  it away.

  **What to build.** Write the queue to `profiles/` with everything else, and
  offer it back on the next launch rather than restoring it silently.

### 11. Clipboard watcher on Download

- [ ] **The problem.** Copy a link, switch to Magpie, click Paste. The middle
  step is the program asking to be told something it could have noticed.

  **What to build.** An opt-in *Watch the clipboard* tick: a copied URL
  appends itself to the box. Off by default — a program that reads your
  clipboard unasked is not a good neighbour.

### 12. Dedupe by content, not by URL

- [ ] **The problem.** `Skip already downloaded` keys on the URL, so two links
  to the same asset both land and you get it twice under two names.

  **What to build.** Hash what arrives, and say so when a new file matches
  one already in the destination. Say, not delete.

### 13. An audio path worth the name

- [ ] **The problem.** Audio is a checkbox on the Download tab and a
  loudness tick on Convert. There is no way to take a file and get sensible
  audio out of it — extract, normalise to a target, and write it as
  something.

  **What to build.** Broadcast and podcast loudness targets (−23 LUFS,
  −16 LUFS) rather than only `loudnorm` on or off, and audio extraction on
  the Convert tab rather than only at download time.

---

## Parked

Considered, and not obviously worth it. Recorded so the same argument does
not get had twice.

| Idea | Why it is parked |
| --- | --- |
| **Watch folders** | Real value, but it wants Magpie running all the time, and Magpie is a thing you open. Revisit once [the CLI](#6-a-cli-and-explorers-right-click-menu) exists — a scheduled task calling the CLI is the same feature without the daemon. |
| **Before/after compare** | Lovely, and a lot of window for a question that a thumbnail and a file size mostly answer. |
| **Resuming an interrupted encode** | ffmpeg has no honest way to resume a partial encode. Segment-and-concat is a different program. |
| **Cross-platform (macOS / Linux)** | Everything in `meta.py`, `theme.py` and `menus.py` is Windows to the bone, and the registry theme read and Inno installer are load-bearing. A port is a rewrite of the bottom half. |
| **Plugin system** | Nothing is asking for one. The custom-arguments escape hatches already cover the cases a plugin would. |
| **Keyboard accelerators** | Deliberately absent — see *The menus*. Worth revisiting *only* for shortcuts that cannot fire while someone is typing in the link box. |

---

## When something ships

1. Move the entry out of this file and into the README, where it is described
   as a thing Magpie does rather than a thing it might.
2. Bump the version part the entry names, and reset the ones to its right.
3. Add a `tools/selftest.py` check if it folded in behaviour that could go
   missing again.
4. Put the new file in the two lists that index it, if it added one: the
   module table and the layout tree in the README.
5. `python tools/publish_docs.py ../magpie-releases`, so the public copy of
   this page says the same thing as this one. `--check` answers whether it
   already does.
