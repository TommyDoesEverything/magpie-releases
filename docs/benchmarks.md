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

## When adding to this file

1. Say what the measurement was *for* — which decision it was going to
   settle. A number with no question attached ages into trivia.
2. Keep the failures. "We tried this and it was slower" is the expensive
   half to rediscover.
3. Record the machine if it is not the one at the top.
4. Clean up what the run generated. Multi-gigabyte sources do not belong in
   a temp folder after the answer is in.
