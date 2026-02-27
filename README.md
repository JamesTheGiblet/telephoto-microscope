# TelephotoMicroscope
### A calibrated optical microscope, stitching engine, and measurement instrument — built on a phone, in Termux, in one day.

**Giblets Creations** | James | 26 February 2026

---

## What is this?

A full Android app that turns a Samsung Galaxy S24 Ultra into a calibrated scientific imaging instrument. Built entirely on-device using Termux — no PC, no Android Studio, no USB cable.

It exploits a gap between what modern smartphone hardware can physically do and what stock camera apps expose to users.

---

## Headline Results

| Metric | Value |
|--------|-------|
| Best resolution | **16.8 microns/pixel** @ 8x zoom, 12.8cm |
| Composite width | **688mm** from 4-frame NCC-registered stitch |
| Tap-to-measure accuracy | **1.7% error** vs known dimension |
| Capture resolution | **4000x3000px (12MP)** per frame |
| Build environment | Termux on S24 Ultra, no PC |
| Language | Kotlin (first time) |
| Time to build | One day |

For context:
- Human hair (50–100µm) = 3–6 pixels wide. Clearly visible.
- Fine PCB trace (100µm) = ~6 pixels. Measurable.
- E3D 0.4mm nozzle orifice = ~24 pixels at 5x. Diagnostic grade.
- Digital vernier calliper costs £15–30 and gives 0.1mm accuracy.
- This app costs £0 and gives 0.3mm accuracy on existing hardware.

---

## How It Works

### The Core Insight
The Laser AF sensor in the S24 Ultra is not just an autofocus mechanism. It's a Time-of-Flight distance sensor that continuously reports subject distance in diopters. By polling this in a Camera2 API preview callback, the app:

1. Detects when the phone is within microscopy range (<15cm)
2. Automatically switches to the telephoto camera (ID:2, 67mm, f/2.4)
3. Calculates real-world scale in mm/px from laser distance readings
4. Activates a calibrated measurement grid overlay
5. Allows in-session zoom ratio control (1x → 8x) without switching cameras

### The Camera Map (S24 Ultra)
```
ID:0  200MP  ISOCELL HP2   23mm   f/1.7   main wide
ID:1   12MP  Sony IMX563   13mm   f/2.2   ultrawide 120°
ID:2   10MP  Sony IMX754   67mm   f/2.4   3x telephoto  ← MICROSCOPE
ID:3   50MP  Sony IMX854  111mm   f/3.4   5x periscope
      Laser AF  ToF                       depth ground truth
```

---

## Features

### ✅ Automatic Lens Switching
- Enters telephoto mode at ≤15cm subject distance
- 5cm hysteresis prevents bouncing
- Laser AF polls continuously via CaptureResult

### ✅ Calibrated Scale
- CalibrateActivity with draggable overlay
- Reference presets: Credit Card (85.6×53.98mm), A4, Custom
- Scales linearly with distance: `mmPerPx * (currentDist / calDist)`
- Compensates automatically for zoom ratio
- Stored in SharedPreferences, persists across sessions

### ✅ Zoom Ratio Control
- CONTROL_ZOOM_RATIO mid-session (API 30+)
- Cycles: 1x → 2x → 3x → 5x → 8x → 1x
- No camera switch, focus continues uninterrupted
- Scale display auto-compensates

### ✅ 12MP Full Resolution Capture
- Dual-surface Camera2 session: preview + ImageReader
- 4000×3000 JPEG at JPEG_QUALITY=95
- TEMPLATE_STILL_CAPTURE for optimal HAL settings
- Metadata burned in at full resolution
- Saves to Pictures/TelephotoMicroscope/ via MediaStore

### ✅ Guided NCC Stitching
- Grid sizes: 2×2, 3×3, 4×4
- Snake capture order
- Tap-to-capture (gyro guides direction, user decides when to fire)
- Pure Kotlin NCC registration — no OpenCV, no native libs
- CompositeBuilder assembles frames onto canvas
- Outputs calibrated composite JPEG with physical dimensions in metadata

### ✅ Tap-to-Measure
- Long-press CALIB to enter measure mode
- Tap A → Tap B → live distance readout
- Auto-format: mm for large distances, µm below 1mm
- 1.7% accuracy confirmed against known dimension
- Updates live as distance/zoom changes

---

## File Structure

```
app/src/main/java/com/giblets/microscope/
  MainActivity.kt          Camera2, state machine, zoom, capture, stitch, measure
  CalibrateActivity.kt     Calibration screen with draggable reference box
  CalibrationOverlay.kt    Custom View — draggable corner handles
  CalibrationStore.kt      SharedPreferences + distance scaling
  OverlayView.kt           Live 1mm grid + crosshair overlay
  StitchSession.kt         Stitch state machine + gyro directional guidance
  StitchOverlay.kt         Direction arrows + grid progress UI
  FrameRegistrar.kt        Pure Kotlin NCC frame alignment
  CompositeBuilder.kt      Canvas assembly + composite JPEG output
  MeasureOverlay.kt        Two-pin tap-to-measure custom View
```

---

## Build Instructions

Everything runs on-device in Termux. No PC required.

```bash
# Install build tools
pkg install openjdk-21 aapt2 -y

# Download Android SDK (one time)
# sdkmanager platform-tools platforms;android-34 build-tools;34.0.0

# Build
cd TelephotoMicroscope
./gradlew assembleDebug --no-daemon

# Install
cp app/build/outputs/apk/debug/app-debug.apk /sdcard/Download/
# Then install from Files app
```

### Critical gradle.properties settings
```properties
android.aapt2FromMavenOverride=/data/data/com.termux/files/usr/bin/aapt2
org.gradle.jvmargs=-Xmx768m
```

### Critical gradle-wrapper.properties
```properties
distributionUrl=https\://services.gradle.org/distributions/gradle-8.4-bin.zip
```

---

## Results Gallery

### 16.8 Microns/Pixel — Wood Grain at 8x
Individual mineral grains, resin deposits, and tracheid cell colour variation resolved.

### 688mm Wide Composite
4-frame NCC-registered stitch. Wood grain continuous across seams. Floorboard joint unbroken.

### Tap-to-Measure — 17.7mm vs 18.0mm Known
Single keycap width measurement. 1.7% error. Live on preview.

### 12MP Zoom Series at 13.1cm
```
1x: 0.12755 mm/px  |  510mm field
2x: 0.06378 mm/px  |  255mm field
3x: 0.04585 mm/px  |  183mm field
5x: 0.02751 mm/px  |  110mm field
8x: 0.01719 mm/px  |   69mm field
```
Scale halves as zoom doubles. Linear relationship confirmed 1x–8x.

---

## What's Next

- [ ] Feathered seam blending for stitched composites
- [ ] Per-zoom-level calibration storage
- [ ] Stereo depth from multi-camera parallax → 3D point cloud
- [ ] Surface reconstruction (Poisson/Marching Cubes) — pure Kotlin
- [ ] STL export → direct to 3D printer
- [ ] Magnetic field sonifier (TYPE_MAGNETIC_FIELD → AudioTrack)
- [ ] GibletsTone — open beacon navigation standard for blind users

---

## Dev Log

Full session-by-session technical log: [DEVLOG.md](DEVLOG.md)

---

## Commercial Applications

- **3D printing diagnostics** — nozzle wear inspection at E3D/Prusa grade
- **Industrial QC** — surface inspection, crack detection, wear analysis
- **Field science** — entomology, geology, materials without lab equipment
- **Medical** — skin feature monitoring at calibrated, reproducible scale
- **Education** — microscopy without microscope hardware

---

## Built By

**Giblets Creations** | James  
Founder & CTO  
Upper Heyford, Oxfordshire  

*"I build what I want."*

One day. One phone. Termux. Kotlin (first time).  
No PC. No USB cable. No prior Kotlin experience.

---

*© 2026 Giblets Creations*
