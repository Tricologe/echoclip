# TODO — capture backends, encoder choice, and the performance work

Tracking the fork's work. Items are ordered; each should land as its own commit with measurements
where the item changes cost.

## Reproduce this build

```bash
# fork
gh repo fork Resister0/echoclip --clone=true
cd echoclip
git checkout -b <branch>

# or clone the fork directly
git clone https://github.com/Tricologe/echoclip.git
cd echoclip
```

Prerequisites on Windows (all already used for the commits below):

```bat
:: Rust (MSVC toolchain)
curl -L -o rustup-init.exe https://win.rustup.rs/x86_64
rustup-init.exe -y --default-toolchain stable-msvc --profile minimal

:: MSVC build tools (the VS 2026 install on this machine ships without the C++ compiler)
:: then, always build from the VS environment so rustc finds link.exe:
call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat"

:: libmpv-2.dll must sit next to the exe (prototype-native\libmpv-2.dll)
cd prototype-native
cargo build --release
copy /y target\release\echo-native-proto.exe EchoProto.exe
```

Note: `build.cmd` at the repo root builds `prototype-native` (Slint + mpv), **not** `src-tauri`.
The native app compiles `src-tauri/src/recorder.rs` unchanged through
`prototype-native/echo-core/src/lib.rs` (`#[path]`), so recorder changes are made there and apply
to the shipping app.

---

## Done

- [x] **Diagnose the original failure** — no desktop-attached DXGI output on this hybrid-GPU
      machine; `DuplicateOutput` → `DXGI_ERROR_UNSUPPORTED`, every `ddagrab` rung →
      *Selected output not supported*. See `docs/dxgi-no-output-report.md`.
- [x] **GDI fallback** — `gdi` / `gdi-x264` rungs appended after the DXGI ones.
      Buffer arms, Alt+F10 saves a 1080p60 clip with all 7 audio tracks.
- [x] **Encoder preference by display GPU** — `gdi` = AMF, `gdi-nvenc`, `gdi-x264`, ordered by
      `monitor_vendor()`. Measured: AMF 90.1 % of one core with no dGPU engine traffic;
      NVENC 98.3 % plus **37 % of the dGPU 3D engine**.
- [x] **Draft reports** — `docs/dxgi-no-output-report.md`, `docs/failsafe-and-performance-report.md`.

## Next — settings and UI

- [ ] **New settings keys** in `Settings` (`src-tauri/src/settings.rs`):
      - `capture_backend`: `"auto"` (default) | `"dxgi"` | `"gdi"` | `"wgc"` (once WGC exists)
      - `encoder`: `"auto"` (default) | `"amf"` | `"nvenc"` | `"x264"`
      - both need `#[serde(default)]` coverage and an entry in `recorder_changed()` so the recorder
        restarts when they change.
- [ ] **Recorder honours them** in `Recorder::start()`: build the mode list from the settings instead
      of the fixed `all` array. `"auto"` keeps today's ladder; an explicit choice tries only that
      family and reports a clear failure instead of silently falling back.
- [ ] **UI: Settings → Video / Quality**
      - a "Capture method" dropdown next to Resolution/FPS with the detected default shown
        (e.g. *Auto — GDI (no GPU capture output found)*)
      - an "Encoder" dropdown (Auto / AMD AMF / NVIDIA NVENC / CPU x264) with the reason the
        automatic choice picked what it did
      - a one-line health readout: active backend, encoder, measured cost
      - show an explanation when DXGI is unavailable, so the user is not left guessing why
        "Auto" chose GDI
- [ ] **Surface recorder failure in the UI** — today a failed start only appears in
      `%TEMP%\clipper_ffmpeg.log`; the toast should name the reason.

## Next — performance

- [ ] **Optimise the NVENC rung** (the "not like before" item):
      - currently every frame is uploaded from system memory, which parks the dGPU's **3D engine**
        at 37 % — the same engine games use
      - try `-init_hw_device d3d11` + `format=nv12,hwupload` (or `-hwaccel cuda
        -hwaccel_output_format cuda`) so the upload happens once and the encoder takes GPU frames
      - also try `-preset p1`, `-tune ll`, `-rc constqp`, `-delay 0`, `-bf 0`
      - acceptance: same clip quality, 3D-engine share at or near **0 %**, CPU not worse than AMF;
        record before/after with the protocol below
- [ ] **Multi-monitor crop for GDI** — `-i desktop` captures the whole virtual desktop; add
      `-offset_x/-offset_y/-video_size` from the selected monitor so cost does not scale with the
      number of displays
- [ ] **Cheaper defaults when GDI is active** — consider capping GDI capture at 30 fps unless the
      user raised it, and honour `capture_cursor=false` (skips per-frame cursor compositing)

## Next — WGC backend

- [ ] `Windows.Graphics.Capture` in `gpucap.rs`: `GraphicsCaptureItem` (monitor or window) +
      `Direct3D11CaptureFramePool`, frames stay as D3D11 textures
- [ ] Feed the existing named pipe (readback + write) as a first step — expected roughly 90 % → 35–45 %
      of one core
- [ ] Optional later: encode in-process with NVENC off those textures (no pipe, no upload) —
      the only variant that gets near 5 % CPU and frees the 3D engine entirely
- [ ] Handle the API's own limits: per-window opt-out (`WDA_EXCLUDEFROMCAPTURE`), protected content,
      border/highlight settings on newer Windows builds

## To verify on hardware

- [ ] **External monitor** — does plugging one into HDMI turn the NVIDIA stub output into a real
      output and revive the DXGI path? If yes, document that docking prefers `gpu` and only
      undocked use falls back to GDI.
- [ ] **Hybrid value of `encoder=auto`** when the game renders on the iGPU and the panel is on the
      dGPU (the inverse of this machine)
- [ ] **CPU-only machine** — confirm `gdi-x264` holds 60 fps at 1080p or document its ceiling

## Measurement protocol (keep consistent)

1. Sample the recorder's `ffmpeg` CPU time over a 60 s window; report **% of one core**
   (`core-seconds / 60`) and % of the machine (divide by logical cores).
2. Read bitrate from the **newest** 2-second segments (segment wrap makes whole-directory deltas
   wrong).
3. Sample `\GPU Engine(*)\Utilization Percentage` for the recorder's PID and report the top engine
   with its adapter LUID (AMD vs NVIDIA matters more than the total).
4. Record the scenario: idle desktop vs motion, resolution, fps, encoder.
5. Note confounders (other recorders running, player processes) in the entry.

## Open questions for upstream

1. Should the GDI/WGC fallbacks be automatic, or opt-in from Settings?
2. Is a WGC backend welcome in this repo, or would upstream prefer to stay on DXGI + a documented
   hardware requirement?
3. Should the recorder expose the chosen backend/encoder in the UI permanently (a small status chip)?
