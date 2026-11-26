# Reelo

> **Built by Bilal_tradess**
> Instagram — [@bilal_tradess](https://www.instagram.com/bilal_tradess)
>
> Reelo is my own project: a Windows 11 video player made to replace VLC.

**Reelo** is a Windows 11 Fluent Design video player built on VideoLAN's `libVLC` 3.0.21 engine.
No installer, no plugin hell, no ".NET Desktop Runtime" dialogs — one folder, one exe, and it plays.

**Version 2.3 — "Self-Healing"**

---

## Download

Grab `Reelo-Win64-v2.3-SelfHealing.zip` from the [Releases](../../releases/latest) page.

1. Right-click the zip → **Extract All…** (do *not* run `Reelo.exe` from inside the zip window,
   and do not copy `Reelo.exe` on its own — it needs the files beside it).
2. Open the extracted **`Reelo`** folder.
3. Double-click **`Reelo.exe`**.

No .NET download. No VLC install. No setup wizard. The complete .NET 8 + WPF runtime and the
complete VLC playback engine are already in the folder.

> **Upgrading from an older build?** Delete the *old* extracted folder first. Running an old
> `Reelo.exe` left in another folder is the most common cause of the ".NET Desktop Runtime" dialog.

---

## What Reelo fixes

### The ".NET Desktop Runtime" dialog is now impossible (v2.2)

**What you saw:** Windows shows *"You must install .NET Desktop Runtime to run this application.
Architecture: x64 · App host version 8.0.34"*.

**Root cause:** v2.1 embedded the runtime *inside* `Reelo.exe` as a compressed single-file bundle.
That design depends on a footer appended to the exe, which the .NET host must read to find the
embedded runtime. Anything that modifies the exe — antivirus scanning or quarantine, a partial
download, an interrupted "Extract All", or Windows launching an older `Reelo.exe` from a previous
extraction — breaks that footer. The host then silently falls back to "look for an installed
runtime", finds none, and shows the install prompt.

**v2.2 fix:** the runtime is no longer hidden inside the exe. `coreclr.dll`, `hostfxr.dll`,
`hostpolicy.dll`, `PresentationFramework.dll`, `wpfgfx_cor3.dll` and friends now sit as ordinary
files next to `Reelo.exe`, and `Reelo.runtimeconfig.json` declares
`includedFrameworks: .NETCore.App 8.0.34 + WindowsDesktop.App 8.0.34`.

The host therefore resolves everything from the folder — **there is no footer to corrupt and no
runtime to install.** It is also why VLC itself, and most portable Windows apps, ship this way.

Two extra safety nets:

- If the `libvlc` folder is ever missing, Reelo shows *"Reelo — playback engine not found"* and
  names the exact missing path instead of dying silently.
- Every launch writes **`reelo-startup.log`** next to the exe (falls back to `%TEMP%`), so a
  startup failure says exactly which stage broke.

### "Not responding" mid-video (v2.1)

- Playback runs on a dedicated background libVLC worker thread, so the UI thread never blocks on a
  native mutex. Seeks are coalesced to the newest target instead of flooding the decoder, the
  timeline comes from a lock-free interpolated clock, and video renders into a plain Win32
  `HwndHost` surface.
- **White bars above and below in `Fit`** — video sits on a pure `#000000` root and the Fluent HUD
  is a separate GPU-rendered layered window, DPI-locked with `PointToScreen` / `TransformFromDevice`.
  Letterbox is black from the first frame.

### "Playback just stops in the middle" is now self-healing (v2.3)

**What you saw:** the picture and timeline freeze partway through, with no error dialog.

**Why it happens:** mid-file freezes on 10-bit HEVC/x265 come from the *hardware* decoder, not from
the file. Long-GOP x265 streams occasionally push a GPU (or an aged driver) past what its D3D11VA
block can handle, so the decoder stops delivering frames while libVLC still reports "playing".
Nothing crashed — the video simply ran out of pictures.

**v2.3 fixes, in order of escalation:**

1. **Stall watchdog** — a dedicated thread watches the playback clock. If it stops advancing for
   6 seconds while the player claims to be playing, Reelo reacts immediately instead of sitting
   there frozen.
2. **Automatic hardware → software fallback** — on a stall (or any decoder error) Reelo rebuilds the
   entire pipeline with FFmpeg **software** decoding, using all CPU threads, and resumes at the
   *exact same* timestamp. You keep watching; you just get a one-line toast:
   *"Switching to software decoding so playback keeps going…"*.
3. **Manual override** — Video & Settings → **Decoding Engine → Software (compatible)** for files
   that misbehave on your GPU. Switch back to *Hardware (D3D11VA)* any time.
4. **Decoder-friendly playback options** — `--clock-jitter=0`, `--clock-synchro=0`, drop-late-frames
   and skip-frames enabled, a 3 s file cache, plus an explicit demux hint for MKV/TS. These remove
   the end-of-GOP stalls that long-GOP x265 produces with oversized keyframe intervals.
5. **`reelo-playback.log`** — written next to the exe (or `%TEMP%`). Every open, seek, stall,
   recovery and error is timestamped, so a freeze is never a mystery again.

**Bonus:** when an episode ends and the next one (`SxxEyy+1`) sits in the same folder, Reelo starts
it automatically with its own subtitles — no clicking between episodes.

---

## Features

**Playback** — MKV, MP4, AVI, MOV, WebM, TS/M2TS, FLV, WMV, MPEG; **10-bit HEVC/x265**, H.264, VP9,
AV1; D3D11VA hardware decoding with automatic software fallback; volume to 200 % (VLC boost); audio
delay; frame step; speed 0.25×–4×; **manual hardware/software decoding switch** with automatic
fallback if the GPU decoder ever stalls.

**Subtitles** — automatic companion discovery (`.srt`, `.ass`, `.ssa`, `.vtt`, `.sub`) beside the
episode and in `Subs\` / `Subtitles\` / `Subs\<EpisodeName>\`, matching exact names, prefixes **and
`S01E01` / `1x01` episode codes even when the release group differs**; automatic embedded MKV/MP4
track activation via `ESAdded`; Small/Medium/Large captions with Shadow/Acrylic/Cinema backdrops;
±100 ms live sync (`G`/`H`); drag and drop any subtitle onto the window.

> **Tip for files with embedded subtitles:** if a MKV carries its subtitles muxed inside and nothing
> shows up, put the matching `.srt` next to the episode (or in a `Subs\` folder beside it). Extract
> it once with `ffmpeg -i episode.mkv -map 0:s:0 episode.srt` and Reelo picks it up every time.

**Shell & HUD** — acrylic floating dock, chapter-segmented timeline, Chapters / Subtitles & Audio /
Speed / Video & Settings flyouts, Mini-Player PiP, Stats for Nerds telemetry, live taskbar progress,
true monitor-covering fullscreen.

---

## Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `Space` / `K` | Play / Pause |
| `F` or Double-Click | Fullscreen |
| `I` | Compact Overlay Mini-Player (always-on-top) |
| `←` / `J`, `→` / `L` | Back 10 s / Forward 30 s |
| `↑` / `↓`, Wheel | Volume ±5 % (to 200 %) |
| `M` | Mute |
| `C` / `V` | Cycle subtitle tracks (external + embedded + off) |
| `G` / `H` | Subtitle sync −100 / +100 ms |
| `[` / `]` | Speed −0.25× / +0.25× |
| `.` | Next frame |
| `S` | Stats for Nerds |
| `Ctrl+O` / `Ctrl+U` | Open file(s) / Open network stream |

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Windows still says "install .NET Desktop Runtime" | You are launching an **old** `Reelo.exe`. Delete every previous Reelo folder, extract this zip fresh with "Extract All", then run the exe inside. |
| Antivirus flags or quarantines files | Add the extracted folder to your AV exclusions, then re-extract. |
| "Reelo — playback engine not found" | The `libvlc` folder is missing — re-extract the whole zip, keeping the folder structure. |
| Black window, no video | Open `reelo-startup.log` next to the exe and check the last line. |
| Video freezes partway through | Already handled automatically (see v2.3). If it recurs, open `reelo-playback.log`, search for `stall:` and switch Video & Settings → Decoding Engine → **Software**. |
| No subtitles on a file that has them inside | Put a matching `.srt` beside the episode or in `Subs\` — see the tip above. |

---

## Built from source

```powershell
dotnet publish -c Release -r win-x64 --self-contained true
```

`PublishSingleFile` is intentionally **off** — see the v2.2 notes above. Copy the `libvlc\win-x64`
folder from the `VideoLAN.LibVLC.Windows` package next to the output, and you get exactly this
portable folder.

---

## Credits

**Reelo** — designed and built by **Bilal_tradess** ([Instagram](https://www.instagram.com/bilal_tradess)).

Playback engine: [VideoLAN VLC](https://www.videolan.org/) / `libVLC` (GPLv2+).
UI: WPF on .NET 8. Reelo's application source is not published in this repository yet — this
repository ships the portable build and its documentation.
