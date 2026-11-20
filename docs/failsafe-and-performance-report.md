# Draft report — failsafe capture path, laptop behaviour, and measured performance

**Status: DRAFT, not final.** All numbers below are from one machine (HP OMEN 15-en0xxx,
Ryzen 7 4800H 8C/16T + GTX 1660 Ti, Windows 11 25H2 26200, 1920x1080 @ 60 Hz capture).
They are measurements, not a benchmark suite. Predictions are labelled as predictions.

Companion to `docs/dxgi-no-output-report.md`, which explains why the normal DXGI paths fail
on this system and why `gdi` / `gdi-x264` were added.

---

## 1. The failsafe: how the mode ladder behaves

`Recorder::start()` tries capture modes in order and takes the first one that survives 3 seconds:

| # | Mode | Capture API | Encoder | Notes |
|---|---|---|---|---|
| 0 | `gpu` | own D3D11 zero-copy (`DuplicateOutput`) | GPU | fastest, needs a real DXGI output |
| 1 | `cpu` | ffmpeg `ddagrab` | `h264_amf` | needs a DXGI output |
| 2 | `cpu-nvenc` | ffmpeg `ddagrab` | `h264_nvenc` | needs a DXGI output |
| 3 | `cpu-qsv` | ffmpeg `ddagrab` | `h264_qsv` | needs a DXGI output |
| 4 | **`gdi`** | ffmpeg `gdigrab` | `h264_nvenc` | **no DXGI output required** |
| 5 | **`gdi-x264`** | ffmpeg `gdigrab` | `libx264` | **no DXGI output required, no NVENC required** |

Design properties that make this a failsafe rather than a replacement:

- **Additive only.** Modes 4 and 5 are appended after the existing ladder, so a machine whose
  zero-copy path works never reaches them. No behaviour change on healthy systems.
- **Two rungs, two independent failure modes covered.** `gdi` removes the DXGI dependency;
  `gdi-x264` additionally removes the NVENC dependency (old GPUs, driver trouble, other vendors).
- **Self-healing cache.** The winning mode index is written to `%TEMP%\clipper_mode_v7.txt` and
  tried second on the next start, so an affected machine pays the ~10 s probe once and then starts
  in ~5 s (3 s GPU probe + GDI).
- **Same downstream pipeline.** Audio tracks, 2 s mpegts segments, saving, thumbnails, hotkeys and
  the UI are untouched — the fallback is invisible apart from the log line
  `recording: GDI desktop capture`.

## 2. Behaviour on this laptop

### Verified

- Replay buffer arms on a machine where every DXGI mode fails (`DuplicateOutput` →
  `DXGI_ERROR_UNSUPPORTED`, `ddagrab` → *Selected output not supported*).
- The internal eDP panel is captured correctly at 1920x1080 @ 60 fps.
- Hotkey save works: `save: 180s -> …Clip 2026-11-20 06-53-29 (Echo).mp4 in 4.7s`
  (62 s of buffer available), filmstrip in 336 ms.
- Saved clip validated: H.264 1920x1080 @ 60 fps + 7 AAC tracks, 62 s, 20.6 MB.

### Predicted, not yet verified — external monitor

The NVIDIA adapter currently exposes one **unattached stub** output (`DeviceName='\'`,
`AttachedToDesktop=0`, 0x0). That is what a display head looks like with nothing plugged in, and
on this model the external port is expected to be wired to the dGPU.

- Plugging a monitor into HDMI should turn that stub into a real output, which would make the
  **`gpu` mode work again** (it already prefers the NVIDIA adapter); `cpu-nvenc` would likely work
  too. GDI would then simply never be reached.
- If the external port turns out to be routed through the AMD iGPU instead, nothing changes and
  GDI remains the working path.
- **Caveat for docking:** `gdigrab` with `-i desktop` captures the **whole virtual desktop**, so in
  extend mode the frame area (and therefore the CPU cost) grows with the total pixel count of all
  monitors. If the external-monitor path revives DXGI, that is the better option when docked.

To settle it: connect the monitor, then re-run the DXGI enumeration and read
`%TEMP%\clipper_ffmpeg.log` — the mode that starts is printed.

### Other laptop notes

- Capture is independent of AC/battery state; there is nothing power-aware in the path.
- With the lid closed and an external display attached, GDI captures the external desktop instead —
  correct behaviour, same code path.
- This machine has no battery installed, so no battery-mode measurement was possible here.

## 3. Measured performance

Method: sample the recorder's `ffmpeg` process CPU time over a 60 s window
(`charged core-seconds / 60`), and read buffer bitrate from the newest 2-second segments
(sizes divided by 2 s; segment wrap is 110 files = 180 s, so whole-directory deltas are unreliable).

| Scenario | Encoder | ffmpeg CPU (% of one core) | % of the 16-thread CPU | Buffer bitrate |
|---|---|---|---|---|
| Idle desktop, 1080p60 | `h264_nvenc` | **90.7 %** | 5.7 % | 0.93 Mbps |
| Video playing (motion), 1080p60 | `h264_nvenc` | **119.4 %** | 7.5 % | 16.3 Mbps |
| Motion, forced fallback | `libx264 ultrafast` | **155.1 %** | 9.7 % | target 33 Mbps (CBR) |

Supporting numbers:

- GPU during recording: `nvidia-smi` → GPU 4 %, **encoder 13 %**, memory 1 %. NVENC works on this
  render-only dGPU, which is why the `gdi` rung almost always succeeds and `gdi-x264` is rarely used.
- Recorder memory: 252 MB RSS.
- Save latency: 4.7 s to cut a 62 s clip (reads segments, stream-copies video, encodes AAC);
  filmstrip 336 ms.
- Disk: ~4.1 MB per 2 s segment under motion (≈ 16 Mbps), ~110 segments ≈ 180 s buffer.

### Reading the numbers

- On this 8C/16T CPU, GDI capture costs **roughly one core** at 1080p60 — noticeable in a
  frame-time-sensitive game, but far from saturating the machine (5.7–7.5 % of total capacity).
- The x264 fallback costs ~1.7x the NVENC path in CPU; it exists for machines without a usable
  hardware encoder, not as a preferred path.
- Cost scales with **pixels and motion**, not with the number of monitors per se: a 1440p or
  multi-monitor virtual desktop multiplies the capture surface.
- Confounders in these measurements: the x264 figure was taken while the NVENC recorder was also
  running, and the motion test used a video player that consumes CPU itself. Treat the direction and
  approximate magnitude as reliable, the exact percentages as ±10 %.

## 4. Open questions

1. Does the external-monitor path restore DXGI here? (needs the cable test above)
2. Should multi-monitor GDI capture be limited to a single monitor (offset/video_size) to keep the
   cost predictable?
3. Is WGC the right long-term backend, retiring both ddagrab and GDI?
