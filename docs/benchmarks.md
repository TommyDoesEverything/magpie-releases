# Benchmarks

> Copied from Magpie's source repo, which is private. References to files
> and functions are left as plain names rather than links, because the code
> they point at is not in this repo.

Measurements taken to settle a question the [roadmap](roadmap.md) was about
to answer from memory. Written down so they are not taken twice: a
benchmark that has to be re-run to be quoted is a benchmark nobody quotes.

Every entry says **what was measured**, **on what**, and **what it decided**.
A result that killed an idea is worth more than one that confirmed it, so
those are kept here rather than deleted with the branch.

## The machine

Numbers below come from one machine, and the ratios matter more than the
seconds. Anything read off a different one belongs in a new row, not an
edit of an old one.

| | |
| --- | --- |
| CPU | Intel Core i9-14900KS |
| GPU | NVIDIA GeForce RTX 4080 |
| OS | Windows 11 Pro 26200 |
| ffmpeg | 2024-04-18-git-35ae44c615-full_build (gyan.dev) |

---

## `ffmpeg -encoders` does not say what the machine can run

**Measured** — 2026-09-06, against
[roadmap #3](roadmap.md#3-hardware-acceleration-beyond-nvidia).

That entry proposes: *"Probe `ffmpeg -encoders` once and offer only what
this machine can actually do."* It cannot. `-encoders` lists what the
**build was compiled with**, which on a full build is everything:

```
h264_nvenc   h264_qsv   h264_amf   h264_vaapi   av1_qsv   av1_amf   ...
```

All of those are listed on the machine above. Only NVENC runs on it. A
five-frame trial encode to `null` separates the two cleanly:

```
ffmpeg -f lavfi -i testsrc=size=320x240:rate=10 -frames:v 5 -c:v <enc> -f null -
```

| Encoder | Exit | Why |
| --- | --- | --- |
| `h264_nvenc` | 0 | the card is there |
| `hevc_nvenc` | 0 | |
| `libsvtav1` | 0 | software, always available |
| `h264_qsv` | 171 | `Error creating a MFX session: -9` |
| `av1_qsv` | 171 | same |
| `h264_amf` | 171 | `DLL amfrt64.dll failed to open` |

**What it decided.** A `-encoders` probe would offer QSV and AMF to every
user of a full ffmpeg build and fail at run time for most of them — which
is the exact complaint #3 opens with about NVENC, reproduced three times
over. If that entry is built, the probe has to be a trial encode.

**What it costs.** 0.37 s for one that works, 0.60 s for one that does not
(the failing ones are slower — they time out looking for hardware). Six
encoders is roughly three seconds, so it belongs on a background thread with
the answer cached beside the settings, not on the startup path.

---

## `-hwaccel auto` on the decode side is not a free win

**Measured** — 2026-09-06, against the same entry, which claims decode
acceleration is *"the bigger miss"* because it *"speeds up every encode
Magpie does regardless of the codec it is writing."*

Source: 12 s of `testsrc2` at 3840×2160/30, H.264, 41 MB. Same encode both
times; the only difference is who decodes. Best of three.

| Work | Software decode | `-hwaccel auto` | |
| --- | --- | --- | --- |
| Decode only (`-f null`) | 0.52 s | 2.97 s | **5.7× slower** |
| → ProRes 422 HQ | 9.25 s | 9.02 s | 2.6% faster — noise |
| → H.264 `veryfast` | 2.35 s | 3.13 s | **33% slower** |

**Why.** Hardware decode lands frames in GPU memory, and a software filter
or encoder needs them back in system memory. That readback costs more than
the decode saved. It is worst where decode was never the bottleneck, which
on this hardware is most of the time.

**What it decided.** `-hwaccel auto` as a default is wrong on this evidence.
It is not the "an evening, speeds up everything" item the roadmap describes.
If it ships it should be off by default and offered for the case it suits —
a genuinely expensive decode feeding a *hardware* encoder, where the frames
never leave the GPU.

**What is still unmeasured.** `testsrc2` is synthetic and cheap to decode,
which stacks the deck against hwaccel. The fair test is real 4K camera
footage — high-bitrate H.264 and 10-bit HEVC — and the NVENC-to-NVENC path
with `-hwaccel_output_format cuda`, where readback never happens. That run
was started and abandoned: the synthetic "hard" sources it needed were
noise-based and ran to ~2 GB per 10 seconds, which is not a reasonable thing
to do to a working machine for a number this rough. **Do it with a real
clip, and ask first.**

---

## What flows cost, and what the new pictures path costs

**Measured** — 2026-09-06, before releasing 2.16.0. Twelve 1200x800 noise
PNGs, three runs, on the machine at the top - the spread is
quoted rather than the best, because these are all under a second and
noisy at that scale.

### The flow engine

| Twelve pictures, one resize | |
| --- | --- |
| Straight from the Resize tab | 0.54 – 0.57 s |
| The same work as a one-step flow | 0.62 – 0.68 s |
| | **+13 to +20%** |
| The same twelve through a three-step flow | 0.64 – 0.66 s |

**What it decided.** Nothing needs changing. The overhead is the working
folder and the delivery move at the end, and it is per *run* rather than
per step - three steps cost barely more than one, because the two after the
first are working on files a quarter the size. A flow is not the fast way to
do one thing; it is the way to do three without driving the window three
times, and 13% of half a second is the right price for that.

The delivery move is a rename, not a copy, because the working folder lives
inside the destination on purpose. That is why the number is this small
rather than doubling on large files - it does not scale with how big the
files are.

### The animation ladder

| | |
| --- | --- |
| 2.4 MB, 14 frames, under a 500 KB limit | 431 KB, **14 frames**, ~1.1 s |

Two passes of the ladder: full size at 256 colours, then one measured hop
down. Before this it took about a third as long and produced a
single-frame PNG, which is not a faster answer to the same question - it is
a fast answer to a different one.

### The settings form

| Page | Fields | Built and shown |
| --- | --- | --- |
| Shrink | 2 | 13 – 16 ms |
| Convert | 22 | 47 – 58 ms |
| Download | 33 | 64 – 78 ms |

**What it decided.** Nothing. Building a form from the field tables costs
about 2 ms a field, which nobody can see, and the Download form is the
worst case in the program.

---

## What a one-file exe costs, every time it starts

**Measured** — 2026-11-02, against the question of what would make Magpie
start faster and download faster short of a rewrite. The window was timed
from launch until it was visible, and until the whole process tree had used
no CPU for 300 ms ("settled"). Five runs each, warm; the same source for
every build. Python 3.14.5, PyInstaller 6.22.2, Nuitka 4.3.4 with MSVC 14.5.

### Magpie's own window

| Build | Visible | Settled | On disk |
| --- | --- | --- | --- |
| PyInstaller `--onefile` (what 2.27.1 ships) | 1.18 – 1.47 s | 2.5 – 3.3 s | 31 MB |
| PyInstaller `--onedir` | 0.26 – 0.40 s | 1.6 – 2.2 s | 63 MB folder |
| Nuitka `--onefile`, unpacked once per version | 0.41 – 0.43 s | 1.7 – 1.9 s | 38 MB, plus 195 MB unpacked |
| ...the first start of a version, unpacking | 0.72 s | 2.1 s | |
| From source, for scale | 0.26 – 0.34 s | 2.4 – 3.1 s | |

PyInstaller's one file unpacks itself into a fresh `%TEMP%` folder on
every start, and Defender looks at everything it unpacks: a second before
the window can appear. Nuitka's can unpack to a fixed folder
(`--onefile-tempdir-spec`) and reuse it, and re-unpacks by itself when the
payload changes - a rebuild under the same version number was picked up.

### Every link

Magpie starts one process per link, so this is paid per link, not per
batch. `--version`, which does no work, five runs:

| Started for each... | One file | Unpacked |
| --- | --- | --- |
| video link: yt-dlp | 0.85 – 0.88 s | 0.36 – 0.41 s (`yt-dlp_win.zip`) |
| picture link: `Magpie.exe --backend gallery_dl` | 1.32 – 1.56 s | 0.38 s through the Nuitka stub, **0.11 s** started directly |
| for scale: `python -m gallery_dl` | | 0.14 s |

**What it decided.** Magpie is built with Nuitka (`tools/release.py`;
`--pyinstaller` is kept for a machine without the C compiler), which keeps
the one file the updater and the portable copy depend on, and fetches
yt-dlp as its onedir build. yt-dlp's onedir build updates itself with `-U`
and `--update-to` like the exe - tried both ways - and carries curl_cffi
for `--impersonate`. A worker process kept alive to take link after link
was the other idea; with these, it would save a further 0.1 – 0.4 s a link
for a protocol, a respawn on Stop and global state shared between runs, so
it was not built.

**What it costs.** 195 MB unpacked per version, most of it gallery-dl's
extractors compiled to C (the exe inside is 146 MB); the folder an update
leaves behind is removed once the new version has started. And a release
build takes several minutes rather than one.

**What the second and a half after the window appears is.** The same in
every build, so it is the window and not the packaging. Profiled from
source: about 1.0 s laying out every page, hidden ones included (the first
`update_idletasks`, in `drops._attach` and `_lay_out`), 0.46 s drawing the
rounded controls pixel by pixel in Python (`cards._draw_round`), and 0.2 s
creating 700-odd widgets.

---

## Four toolkits, one page

**Measured** — 2026-11-02, against `docs/ui-toolkit.md`'s question of
which toolkit New should be drawn in. The Download page as Qt Quick
(PySide6 6.12), Slint (1.18.1b1, beta Python bindings) and WebView2
(pywebview 6.2.1), each built with Nuitka 4.3.4 as one file unpacked once,
against the real Tk Magpie built the same way. Timed as in the entry above;
memory summed over the process tree one second after it settled. Five runs,
medians; three for the renderer variants.

| | Visible | Settled | Private | Working set | Download | Unpacked |
| --- | --- | --- | --- | --- | --- | --- |
| Tk Magpie, whole app | 0.43 s | 1.99 s | 40 MB | 114 MB | 38 MB | 195 MB |
| Qt Quick | 1.69 s | 1.89 s | 182 MB | 355 MB | 121 MB | 412 MB |
| ...its software backend | 1.63 s | 1.74 s | 112 MB | 330 MB | | |
| Slint (default, GPU) | 0.59 s | 0.60 s | 274 MB | 241 MB | 18 MB | 50 MB |
| Slint, software renderer | 0.27 s | 0.31 s | 72 MB | 127 MB | | |
| WebView2, 7 processes | 0.66 s | 1.09 s | 287 MB | 443 MB | 10 MB | 31 MB |

The Tk build includes the engine (gallery-dl, instaloader, Pillow); the
trials do not.

**What it decided.** Qt Quick stopped being the recommendation: it starts
slowest of the four. Slint is the one to try harder, with its software
renderer. The reasoning is in `docs/ui-toolkit.md`.

**Watch for.** This machine sets `QT_SCALE_FACTOR_ROUNDING_POLICY=
RoundPreferFloor` for the user, which makes every Qt app draw 150% at 100%
unless it overrides the policy - the first Qt numbers were taken that way
and thrown out.

---

## The Slint window under load

**Measured** — 2026-11-07, for Part 4's Stress box in `docs/redesign.md`:
does the new window stay responsive with a lot to show? From source
(Python 3.14, slint 1.18.1b1, software renderer), the window on the
second monitor at 100%. "Worst frame" is the longest gap between ticks of
a 16 ms Slint timer, so the longest the window could not answer. Batches
ran stand-in work jobs through the real Runner (four progress steps and a
log line each, no Pillow or ffmpeg), so the window, not the work, is
what is timed.

| | Before | After |
| --- | --- | --- |
| 50,000 Activity lines at once: worst frame | 2,082 ms | 44 ms |
| 10 s stream, ~30,000 lines/s: p99 frame | 35 ms | 19 ms |
| 1,000 files, stand-in run: took / median frame | 11.5 s / 787 ms | 0.8 s / 16 ms |
| 3,000 files, stand-in run: took / worst frame | - | 3.1 s / 55 ms |
| 600 links, stand-in run: median frame | 180 ms | 16 ms |
| 2,000 links, stand-in run: took | over 10 min (gave up) | 5.6 s, worst 31 ms |
| Select all, 2,000 links | 803 ms | 2 ms |
| Flow, 2,000 links: worst frame on a change | 336 ms | 22 ms |
| Stop, real Resize, 3 lanes: worst frame | 427 ms | 187 ms |

Memory, private bytes: 150 MB with the window built, 155 MB shown, flat
through every page; a 10 s, 330,000-line stream took it to 205 MB (Copy
keeps every line). Not the 72 MB of the toolkit trial above, which was
one page and no engine.

**What it decided.** Activity is fed a pump at a time, the link and file
lists became ListViews, and Stop kills off the window's thread
(`redesign.md` has each fix). The before column is what the window did
until then.

**Watch for.** Import `magpie.gui.window` before `slint` in any script
that times the window: it sets the software renderer, which Slint reads
on import. A harness that imported slint first measured the GPU renderer
at 423 MB and was thrown out. And the first gap a ticker sees includes
whatever ran before it started - reset its clock on the first tick.

---

## When adding to this file

1. Say what the measurement was *for* — which decision it was going to
   settle. A number with no question attached ages into trivia.
2. Keep the failures. "We tried this and it was slower" is the expensive
   half to rediscover.
3. Record the machine if it is not the one at the top.
4. Clean up what the run generated. Multi-gigabyte sources do not belong in
   a temp folder after the answer is in.
