# Draft report — replay buffer cannot start when DXGI exposes no desktop output

**Status: DRAFT, not final.** Numbers below come from one machine; treat them as evidence, not benchmarks.

---

## Symptom

Echo 2.0.11 on Windows 11 never arms the replay buffer. The UI shows the recorder as stopped, and
`%TEMP%\clipper_ffmpeg.log` ends with:

```
gpu: failed: DuplicateOutput failed
[Parsed_ddagrab_0 @ ...] Selected output not supported
[Parsed_ddagrab_0 @ ...] Failed to configure output pad on Parsed_ddagrab_0
[fc#0 @ ...] Error configuring filter graph: Generic error in an external library
--- cpu mode failed, trying next ---
--- cpu-nvenc mode failed, trying next ---
--- cpu-qsv mode failed, trying next ---
Recorder failed to start - see the ffmpeg log
```

Audio initialises fine every time (mic, desktop loopback, per-app tracks), so the failure is purely
on the video capture side.

## Affected environment

| Item | Value |
|---|---|
| Machine | HP OMEN 15-en0xxx (2020, BIOS from launch) |
| CPU / iGPU | Ryzen 7 4800H, AMD Radeon(TM) Graphics (Renoir), driver 31.0.21925.1001 |
| dGPU | NVIDIA GeForce GTX 1660 Ti, driver 32.0.16.2124 |
| OS | Windows 11 25H2, build 26200 |
| Panel | eDP 1920x1080 @ 144 Hz, `DISPLAY\LGD05FE`, owned by the AMD iGPU |
| Also installed | two IDD virtual display drivers (MuMuPlayer, Meta Virtual Monitor) |

## Evidence

DXGI enumeration on the interactive desktop (`WinSta0\Default`, session 1):

| Adapter | Vendor | `EnumOutputs` |
|---|---|---|
| 0 — NVIDIA GTX 1660 Ti | 0x10DE | 1 output, but `DeviceName='\'`, `AttachedToDesktop=0`, 0x0 |
| 1 — AMD Radeon | 0x1002 | **0 outputs** (`DXGI_ERROR_NOT_FOUND`) |
| 2 — AMD Radeon | 0x1002 | 0 outputs |
| 3 — AMD Radeon | 0x1002 | 0 outputs |
| 4 — Microsoft Basic Render | 0x1414 | 0 outputs |

`DuplicateOutput` against the only enumerated output returns **`0x887A0004 DXGI_ERROR_UNSUPPORTED`** —
that is Echo's `DuplicateOutput failed`.

Meanwhile `QueryDisplayConfig` reports exactly one active path:

```
src adapter LUID 0x…  (the AMD adapter) → \\?\DISPLAY#LGD05FE#…  outputTechnology 0x80000000 (internal)
```

So win32k says the AMD adapter owns the internal panel, and DXGI says that same adapter has no
outputs. That contradiction is the defect.

Independent confirmation from a second codebase: `ffmpeg -f lavfi -i ddagrab=output_idx=N`
fails for every N (`Selected output not supported`, or `Failed to enumerate DXGI output N`),
because ddagrab binds the default (NVIDIA) adapter, which has no usable output.

Minor oddity observed while probing: the one stub output object answers `QueryInterface(IDXGIOutput1)`
but **not** `QueryInterface(IDXGIOutput)`, i.e. it is not a normal DXGI output object at all.

## Root cause

Echo's video path has exactly two families of capture, and both require a desktop-attached DXGI
output:

- `gpu` — own D3D11 zero-copy pipeline, calls `IDXGIOutput1::DuplicateOutput`
- `cpu` / `cpu-nvenc` / `cpu-qsv` — ffmpeg `ddagrab` (which internally uses Desktop Duplication)

On this system no adapter exposes such an output, so all four modes fail and the recorder gives up.
Echo itself is not at fault: the desktop is perfectly capturable through other APIs (WGC/GDI).

## Ruled out

| Hypothesis | Result |
|---|---|
| MuMuPlayer / Meta virtual display adapters | Disabled both, rebooted — no change |
| Stale AMD device state | `pnputil /restart-device`, then clean remove + rescan of the same driver package — no change |
| Reboot | Clean reboot — no change |
| MPO disable (`HKLM\SOFTWARE\Microsoft\Windows\Dwm\OverlayTestMode=5`) | Removed, DWM restarted — no change |
| Hardware GPU scheduling | `HwSchMode` unset (off) — not the cause |
| NVIDIA driver binding the stub output | Disabling the dGPU moved the stub to the AMD adapter — still no real output |
| Corrupt display topology store | Backed up and reset `GraphicsDrivers\{Configuration,Connectivity,ScaleFactors}`, rebooted — display numbering cleaned up (`DISPLAY25` → `DISPLAY1`), outputs still absent |
| Tampered/hooked DXGI | `dxgi.dll` is Microsoft-signed (10.0.26100.7019); no `AppInit_DLLs`; no display-class filter drivers |

Conclusion: a driver/WDDM-level fault on this machine, outside Echo's control.

## Fix in this branch

Two new recorder modes, appended **after** the existing ones so healthy systems never reach them:

| Mode | Capture | Encoder |
|---|---|---|
| `gdi` | `ffmpeg -f gdigrab -framerate <fps> -draw_mouse <0/1> -i desktop` | `h264_nvenc` (`p3`, `tune ll`, vbr) |
| `gdi-x264` | same capture | `libx264` (`ultrafast`, `zerolatency`) |

GDI hands over CPU BGRA frames, so this path skips `hwdownload` and runs the same
full-range → TV-range BT.709 conversion before `nv12`. Everything downstream is untouched: audio
tracks, 2 s mpegts segments, saving, hotkeys. The mode cache (`clipper_mode_v7.txt`) records the
winning mode, so a machine that once needs GDI returns to it directly after the 3 s GPU probe.

## Verification on the affected machine

```
start: 2026-11-20 06:51:25
--- cpu-qsv mode failed, trying next ---
recording: GDI desktop capture (48000 Hz, 2 ch)
save: 180s -> C:\Users\...\Videos\EchoClips\Clip 2026-11-20 06-53-29 (Echo).mp4 in 4.7s
```

- Rolling buffer: 26 segments, ~108 MB.
- Saved clip: 1920x1080 @ 60 fps H.264, 62 s, 20.6 MB, 7 AAC tracks (mix / desktop / mic / game / Discord / browsers).
- Encoder cost: ffmpeg ≈ 50 % of one core at 1080p60 on this CPU.

## Caveats

- GDI captures the **composited desktop**: fine for borderless/windowed games, not exclusive fullscreen.
- CPU-side capture costs more than the zero-copy path; on slower CPUs `gdi-x264` may not hold 60 fps at 1080p.
- First start on an affected machine wastes ~10 s probing the doomed DXGI modes before reaching GDI;
  after that the mode cache avoids the repeats.
- This is a fallback, not a replacement: **Windows.Graphics.Capture (WGC)** is the correct long-term
  backend for machines without DXGI outputs (GPU-side textures, no desktop-composition restriction).

## Open questions for upstream

1. Should `gdi` be enabled by default on Windows, or opt-in via Settings (since it is a visible
   behaviour change on exotic systems)?
2. Is a WGC backend acceptable as follow-up work in this repo?
3. Should the recorder surface the failure reason in the UI instead of only in the log?
