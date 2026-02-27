================================================================================
  TELEPHOTO MICROSCOPE - DEVELOPMENT LOG & TECHNICAL REFERENCE
  Giblets Creations | James (Founder & CTO)
  Build Session: 26 February 2026
  Doc Updated:   26 February 2026 (post stitching implementation)
================================================================================

  Platform:        Samsung Galaxy S24 Ultra
  Build Env:       Termux on-device (no PC)
  Language:        Kotlin
  Build System:    Gradle 8.4 + AGP 8.2
  Status:          STITCHING COMPOSITE - 688mm WIDE CALIBRATED OUTPUT ACHIEVED
  APK:             app-debug.apk (debug signed)

================================================================================
  HEADLINE RESULTS
================================================================================

  Best single-frame resolution: 0.01680 mm/px (16.8 microns/pixel)
  At: Telephoto ID:2 @8x zoom, 12.8cm distance, calibrated

  Best composite achieved: 4800x3160px | 688x453mm | 4 frames | [CALIBRATED]
  At: Telephoto ID:2 @1x, 13.9cm, NCC-registered stitch

  Full resolution capture: 4000x3000px (12MP) per frame via ImageReader
  Previously: 1920x1080 preview grabs only

  For context:
    Human hair (50-100 microns)  = 3-6 pixels wide. Clearly visible.
    Fine PCB trace (100 microns) = ~6 pixels. Measurable.
    0.4mm nozzle orifice         = ~24 pixels. E3D diagnostic grade.
    Wood tracheid cells (15-40u) = at resolution limit. Resolved at 8x.
    Bacteria (1-10 microns)      = below current limit

================================================================================
  1. THE CONCEPT
================================================================================

The TelephotoMicroscope exploits a gap between what modern smartphone camera
hardware can physically do versus what stock camera apps expose to users. The
S24 Ultra carries multiple rear cameras with different focal lengths. When a
telephoto lens is positioned close to a subject, it creates significant optical
magnification. Combined with the Laser AF sensor already built into the device,
this creates the foundation for a software-defined digital microscope.

The core insight: the Laser AF sensor is not just an autofocus mechanism. It
is a ToF (Time of Flight) distance sensor that continuously reports subject
distance in diopters. By polling this data in a Camera2 API preview callback,
the app detects when the phone is within microscopy range, automatically
switches to the telephoto camera, activates a measurement grid overlay,
calculates real-world scale in mm/px, and allows in-session zoom ratio control
to push magnification further without switching cameras.

TARGET CAPABILITIES
  [x] Automatic lens switching triggered by subject distance
  [x] Real-time mm/px scale calculation from laser AF readings
  [x] Measurement grid overlay with 1mm reference markers
  [x] Scientific-grade image capture with metadata burned in
  [x] Calibration mode - ground-truthed scale against physical reference
  [x] In-session zoom ratio control (1x, 2x, 3x, 5x, 8x)
  [x] Scale correctly compensates for zoom ratio
  [x] Dedicated debug button
  [x] Full resolution ImageReader capture (12MP)
  [x] Guided gyro stitching with NCC registration
  [ ] Tap-to-measure tool (outstanding)
  [ ] Seam blending feathered (outstanding)
  [ ] Per-zoom calibration storage (outstanding)

================================================================================
  2. ENVIRONMENT SETUP
================================================================================

The entire build chain was established on-device using Termux on the S24 Ultra
itself. No PC, no Android Studio, no USB cable.

2.1 WHAT WORKED
  - Termux pkg manager functional and up to date
  - openjdk-21 installed cleanly from Termux repositories
  - Android Command Line Tools downloaded from Google (146MB)
  - aapt2 available as native ARM binary via: pkg install aapt2
  - Android SDK Platform 34 + Build Tools 34.0.0 via sdkmanager
  - All 7 Google SDK licenses accepted via sdkmanager --licenses
  - Gradle 8.4 pinned via gradle-wrapper.properties (bypasses system Gradle)

2.2 PROBLEMS ENCOUNTERED & FIXED

  PROBLEM: AAPT2 Binary Incompatibility
  AGP bundles an x86 AAPT2 binary. Cannot run on ARM (S24 Ultra CPU).
  FIX: gradle.properties:
    android.aapt2FromMavenOverride=/data/data/com.termux/files/usr/bin/aapt2

  PROBLEM: Gradle / AGP Version Incompatibility
  Gradle 9.3.1 incompatible with AGP 8.2.0.
  FIX: gradle-wrapper.properties pinned to Gradle 8.4.

  PROBLEM: /tmp not available in Termux
  FIX: Use $TMPDIR instead. Always.

  PROBLEM: Termux not responding during builds
  Dual-surface ImageReader session heavy on heap.
  FIX: Clear Termux cache. Build with --no-daemon. Reduce heap to 768m.

  PROBLEM: Long-press debug trigger unreliable on TextureView
  FIX: Replaced with dedicated DEBUG button.

2.3 FINAL WORKING STACK
  Java:                  OpenJDK 21.0.10
  Gradle (wrapper):      8.4
  Android Gradle Plugin: 8.2.0
  Kotlin:                1.9.22
  compileSdk/targetSdk:  34
  minSdk:                26 (Android 8.0)
  Build command:         ./gradlew assembleDebug --no-daemon

================================================================================
  3. CAMERA ARCHITECTURE
================================================================================

3.1 THE S24 ULTRA CAMERA MAP (from on-device debug)

  ID | Facing | Focal  | Min Focus      | Max JPEG    | Zoom Range
  ---+--------+--------+----------------+-------------+------------
  0  | BACK   | 6.3mm  | 10.00 diopters | 4080x3060   | 0.6-10.0x
  1  | FRONT  | 3.3mm  |  5.00 diopters | 4000x3000   | 1.0-8.0x
  2  | BACK   | 2.2mm  | 20.00 diopters | 4000x3000   | 1.0-8.0x  <- MICROSCOPE
  3  | FRONT  | 3.3mm  |  5.00 diopters | 3392x2544   | 1.0-8.0x

  Camera ID:2 is the telephoto microscope lens.
  minFocus 20 diopters = 5cm physical minimum, ~10cm practical.
  Samsung does not expose 3x/5x as separate Camera2 cameras.
  They are accessed via CONTROL_ZOOM_RATIO within ID:2.

3.2 DUAL SURFACE SESSION (added Session 3)

  Surface 1: TextureView  - preview 1920x1080, continuous repeating request
  Surface 2: ImageReader  - 4000x3000 JPEG, fired on demand

  Both surfaces in SessionConfiguration from open.
  Still capture: TEMPLATE_STILL_CAPTURE to Surface 2.
  Preview continues uninterrupted during still capture.

3.3 ZOOM RATIO IMPLEMENTATION

  CONTROL_ZOOM_RATIO applied mid-session (API 30+).
  Levels: 1x, 2x, 3x, 5x, 8x. Confirmed optically valid to 8x.
  Scale compensates: mmPerPx = baseScale / currentZoomRatio

  Field widths at 10cm calibrated:
    1x: ~216mm | 2x: ~108mm | 3x: ~72mm | 5x: ~44mm | 8x: ~32mm

3.4 SCALE CALCULATION

  Calibrated: mmPerPx = calibration.mmPerPixelAt(distanceCm) / zoomRatio
  Estimated:  mmPerPx = (sensorWidthMm * distCm*10 / focalMm) / 1920 / zoomRatio

================================================================================
  4. CALIBRATION SYSTEM
================================================================================

  CalibrateActivity: draggable green rectangle overlay on camera ID:2.
  Presets: Credit Card (85.6x53.98mm), A4 (210x297mm), Custom.
  Calculates: average of width-axis and height-axis mm/px.
  Stores: mmPerPixel, distance, reference name, timestamp in SharedPreferences.
  Scales linearly to any distance: stored_mmPx * (currentDist / calDist)
  Displays: green [CAL] status on main screen, burned into captures.

================================================================================
  5. FULL RESOLUTION CAPTURE
================================================================================

  ImageReader at 4000x3000 JPEG added to session alongside preview.
  TEMPLATE_STILL_CAPTURE fires to ImageReader surface.
  Bytes decoded via BitmapFactory, metadata burned at full resolution.
  Grid redrawn for 4000px width. Bar height 7% of image height.
  Saved to Pictures/TelephotoMicroscope/ via MediaStore.
  Filename: Microscope_8x_20260226_141745.jpg
  Toast: "Saved 12.0MP: ..."

  Zoom progression series at 13.1cm (all 4000x3000):
    1x: 0.12755 mm/px | 510mm wide
    2x: 0.06378 mm/px | 255mm wide
    3x: 0.04585 mm/px | 183mm wide
    5x: 0.02751 mm/px | 110mm wide
    8x: 0.01719 mm/px |  69mm wide
  Scale halves as zoom doubles. Linear relationship confirmed 1x-8x.

================================================================================
  6. GYRO STITCHING SYSTEM
================================================================================

6.1 NEW FILES

  StitchSession.kt
    State machine: WAITING_START -> CAPTURING -> MOVING -> CAPTURING -> COMPLETE
    Grid sizes: 2x2, 3x3, 4x4. Snake capture order.
    Gyro (TYPE_ROTATION_VECTOR) for directional guidance only.
    Trigger: TAP-TO-CAPTURE. User taps when in position.
    Rationale: gyro rotation minimal when phone slides flat on surface.
    8-degree step target. Arrow guides direction.

  StitchOverlay.kt
    Top-left grid preview: green=captured, bright=current, dark=pending.
    Centre arrows: green=move, orange=arrived.
    Bottom message bar: current instruction.
    TAP ring indicator when awaiting input.
    Fixed DOWN/UP arrow path drawing (was rendering as triple arrow).

  FrameRegistrar.kt
    Pure Kotlin NCC. No OpenCV. No native libraries.
    Downsample both frames to 800px wide thumbnails.
    Extract 200px centre patch from frame 1.
    Search frame 2 within gyroEst +/- 120px window.
    Return (dx, dy, confidence). Scale back to full res.

  CompositeBuilder.kt
    Painter's algorithm, alpha=180 blend.
    Canvas: bounding box of all frame placements.
    MAX_DIM=12000px OOM guard.
    Metadata: "COMPOSITE 2x2 | WxHpx | AxBmm"
    Filename: Stitch_2x2_20260226_192227.jpg

6.2 STEP CALCULATION BUG (fixed)

  Original: stepDeg = atan(mmPerPx * 4000 * 0.8 / (distCm*10))
  Problem:  mmPerPx already zoom-compensated. At 1x: 275mm step = 57 degrees.
  Fix:      Fixed 8-degree steps. NCC handles actual alignment regardless.

6.3 STITCH RESULTS

  Attempt 1: 7200x5400px | no movement | pipeline confirmed
  Attempt 2: 4040x3040px | no movement | NCC zero displacement
  Attempt 3: 5680x3280px | movement, rotating | frames at angles
  Attempt 4: 4800x3160px | 688x453mm  *** FIRST SUCCESSFUL STITCH ***
    Wood grain continuous across seam. Floorboard joint unbroken.
    NCC registration working. Visible seam (blending next).

================================================================================
  7. OUTSTANDING WORK (PRIORITY ORDER)
================================================================================

  1. Seam blending          Feathered alpha ramp across overlap region  ~2h
  2. Per-zoom calibration   Store mmPerPx per camera+zoom combination   ~2h
  3. Capture rotation fix   Read EXIF, rotate bitmap before burn-in     ~1h
  4. Tap-to-measure         Two-pin distance tool on live preview        ~3h
  5. Torch illumination     CameraManager.setTorchMode() at close range  ~1h
  6. Release signing        keytool + signingConfigs in build.gradle     ~30m

================================================================================
  8. PROOF OF CONCEPT - FULL RESULTS LOG
================================================================================

  SESSION 1 - Proof of concept (estimated scale)
  13:04  10.0cm  ID:2  1x   0.1326mm/px [EST]   Wood grain, fibres visible

  SESSION 2 - Calibration + zoom
  13:46  10.0cm  ID:2  1x   0.11247mm/px [CAL]  Soil/mud, grain structures
  13:59  10.0cm  ID:2  1x   0.10500mm/px [CAL]  Wood grain
  14:15  10.9cm  ID:2  5x   0.02289mm/px [CAL]  *** 22.9 MICRONS/PX
  14:17  12.8cm  ID:2  8x   0.01680mm/px [CAL]  *** 16.8 MICRONS/PX
                                                  Mineral grains, resin deposits

  SESSION 3 - Full resolution ImageReader (4000x3000)
  17:09  12.8cm  ID:2  1x   0.12440mm/px [CAL]  12MP confirmed
  17:12  13.1cm  ID:2  1x   0.12755mm/px [CAL]  Zoom series start
  17:12  13.1cm  ID:2  2x   0.06378mm/px [CAL]
  17:12  13.1cm  ID:2  3x   0.04585mm/px [CAL]
  17:12  13.1cm  ID:2  5x   0.02751mm/px [CAL]
  17:13  13.1cm  ID:2  8x   0.01719mm/px [CAL]  Linear confirmed 1x-8x

  SESSION 4 - Gyro stitching
  19:22  13.9cm  ID:2  1x   0.1434mm/px  [CAL]  *** FIRST STITCH
                                                  4800x3160px | 688x453mm
                                                  Grain continuous, joint intact

================================================================================

  Session 1: proof of concept
  Session 2: 16.8 microns/px calibrated instrument
  Session 3: 12MP full resolution pipeline
  Session 4: 688mm wide NCC-registered composite
  Next:      seam blend + tap-to-measure = shippable product

  Giblets Creations 2026 | Built entirely on S24 Ultra in Termux

================================================================================
