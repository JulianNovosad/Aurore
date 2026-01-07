---

# Aurore Mk VI — Full Plan v2.0 (single-repo, native-Pi, reproducible, C20)

---

## Repo layout (enhanced structure)

```
aurore-mk-vi/
├── build.sh                     # single entrypoint, orchestrates everything
├── README.md                    # now includes OS pinning & dependency list
├── docs/
│   ├── operator_manual.md       # now includes installation & runtime ops
│   ├── runbooks/
│   │   ├── rollback.md          # hardware rollback procedures
│   │   └── monitoring.md        # NEW: parsing unified.csv for latency spikes
│   └── kernel/                  # kernel patch docs & instructions
├── patches/                     # all patches (kernel, libcamera, drivers)
├── deps/                        # git submodules (not full clones) fetched by build.sh
│   ├── libedgetpu/
│   ├── tensorflow/
│   ├── libcamera/
│   ├── rpicam/
│   └── ffmpeg/
├── third_party/                 # small helper libs built in-repo if needed
├── kernel/                      # kernel source  pinned config  patches
│   ├── source/
│   └── config/                  # pinned .config & patches
├── src/
│   ├── camera/                  # libcamera wrapper with hardware failure handling
│   ├── inference/               # TF Lite loader, TPU delegate manager
│   ├── decision/                # safety & prediction logic with invariance checks
│   ├── servo/                   # PWM servo driver with liveness checks
│   ├── telemetry/               # CSV writer, unified.csv, run.json, rotation
│   ├── net/                     # minimal health API, optional stream server
│   └── tools/                   # small tools used by CI/tests
├── models/
│   ├── model.tflite
│   └── model_edgetpu.tflite
├── android_app/                 # expanded docs for RTSP/TLS setup
├── tests/
│   ├── test_harness/
│   └── fixtures/
├── packaging/
├── artifacts/
└── scripts/                     # NEW: auxiliary scripts
    ├── preflight.sh             # dependency & resource checker
    └── log_rotate.sh            # compress/archive logs after 3 runs
```

---

## High-level rules (enhanced)

* **Build concurrency**: always `-j2` or `-j3`. Default `JOBS=2`. `build.sh` CLI `--jobs=2|3`.
* **No Bazel** unless impossible. For TF Lite, prefer **CMake**. If Bazel unavoidable, use pinned binary from `deps/tools/` with checksum verification.
* **Native-only**: all compilation runs on Pi (no cross-compilation).
* **Reproducibility**: set deterministic env vars and packaging rules.
* **Single-command setup**: `./build.sh` clones, patches, builds, tests, collects artifacts, publishes, and exits.

**NEW**: `--skip-preflight` flag to bypass dependency checks if manually verified.

---

## Deterministic / reproducible build strategy (concrete, enhanced)

`build.sh` enforces these env settings before any compile:

```bash
# NEW: Default to repo commit timestamp for true reproducibility
export SOURCE_DATE_EPOCH="${SOURCE_DATE_EPOCH:-$(git log -1 --format=%ct 2>/dev/null || date %s)}"
export JOBS="${JOBS:-2}"              # default 2; allowed 2 or 3 only
export DEBFULLNAME="Aurore Mk VI Builder"
export DEBEMAIL="no-reply@example.invalid"
export TZ=UTC
export LC_ALL=C.UTF-8

# NEW: Pi safety - monitor temperature during builds
export THERMAL_THRESHOLD=75           # throttle builds if temp exceeds

# Compiler flags for reproducible builds:
export CFLAGS="-O2 -g0 -fdebug-prefix-map=${PWD}=. -fno-record-gcc-switches"
export CXXFLAGS="$CFLAGS -std=gnu20"
export LDFLAGS="--build-id=sha1"
export ARFLAGS="D"

# NEW: Static analysis tools
export CLANG_TIDY_CHECKS="bugprone-*,performance-*,modernize-*"
```

When creating tarballs:

```bash
tar --sort=name --mtime="@${SOURCE_DATE_EPOCH}" --owner=0 --group=0 --numeric-owner \
    -cf artifacts.tar .
```

For deterministic strip:

```bash
strip --strip-debug --preserve-date <binary> || true
```

When packaging, always record and publish:

* exact git commit SHAs for all deps in `deps/manifest.txt`
* toolchain versions in `toolchain.txt`
* produced artifact checksums (sha256) in `artifacts/checksums.txt`

Use `fakeroot` for packaging.

---

## Resource requirements & host dependencies (NEW section)

**Minimum Hardware**: Raspberry Pi 5 (4GB RAM minimum), 64GB SD card, active cooling.

**Resource Estimates per Milestone**:
* Kernel build: ~45-60 minutes with `-j2`
* TensorFlow Lite: ~90-120 minutes with `-j2`
* Full pipeline: ~4-6 hours total

**Host Dependencies** (`scripts/preflight.sh` verifies):
- `git`, `cmake` (≥3.20), `make`, `gcc` (≥11), `g`, `python3`
- `clang-tidy`, `cppcheck` for static analysis
- `libssl-dev`, `libprotobuf-dev`
- `fakeroot` for packaging

**Base OS**: Raspberry Pi OS Lite (64-bit). `build.sh` will by default run `sudo apt update && sudo apt upgrade -y` to pull the latest distribution/kernel packages and *then* record the post-upgrade kernel/package version in `artifacts/kernel/kernel_package_version.txt` and `deps/manifest.txt`. If you prefer to skip the upgrade, use `--skip-upgrade` (advanced use only).

---

## Kernel / DTB / firmware (pinned, reproducible)

* Kernel source stored in `kernel/source/`. The active kernel package/version is determined *after* the system upgrade step (see build flow); `build.sh` will capture the installed kernel package version and then pin or fetch matching kernel sources accordingly.
* Include **exact kernel config** under `kernel/config/` and all **patch files** under `patches/kernel/`.
* Required features: PREEMPT_RT, 4k page support, PCIe tweaks, MSI-X DTB patches, `pcie_aspm=off` in bootloader config.

Build script with thermal monitoring:

```bash

build_kernel() {
  scripts/preflight.sh --check-temp  # NEW: thermal check (and OS upgrade step by default)

  # Ensure latest kernel packages are installed and capture package version for pinning:
  # (build.sh already invoked scripts/preflight.sh which runs apt update/upgrade unless --skip-upgrade)
  KERNEL_PKG_VER=$(dpkg-query -W -f='${Package} ${Version}\n' raspberrypi-kernel 2>/dev/null || true)
  echo "$KERNEL_PKG_VER" > ${ROOT}/artifacts/kernel/kernel_package_version.txt || true

  cd kernel/source
  # If a corresponding KERNEL_COMMIT mapping exists for the installed kernel package,
  # use it; otherwise fall back to pinned commit in kernel/config/pinned.commit (if present).
  if [ -n "${KERNEL_COMMIT_MAPPING_DIR:-}" ] && [ -f "${KERNEL_COMMIT_MAPPING_DIR}/mapping.txt" ]; then
    # mapping.txt expected format: "<package-version> <git-commit>"
    map_commit=$(grep "$(echo "$KERNEL_PKG_VER" | awk '{print $2}')" "${KERNEL_COMMIT_MAPPING_DIR}/mapping.txt" | awk '{print $2}' || true)
  fi
  if [ -n "$map_commit" ]; then
    git checkout "$map_commit"
  else
    # fallback: use pinned commit/config (existing behavior)
    git checkout "${KERNEL_COMMIT}"
  fi

  cp ../../kernel/config/pinned.config .config
  for p in ../../patches/kernel/*.patch; do git apply "$p"; done
  make ARCH=arm64 O=build olddefconfig
  make ARCH=arm64 O=build -j${JOBS} Image dtbs modules

  # NEW: Verify PREEMPT_RT is enabled
  grep "PREEMPT_RT" build/.config || { echo "Missing PREEMPT_RT"; exit 1; }

  mkdir -p ${ROOT}/artifacts/kernel
  cp build/arch/arm64/boot/Image ${ROOT}/artifacts/kernel/
  cp build/arch/arm64/boot/dts/broadcom/*.dtb ${ROOT}/artifacts/kernel/
}


```

**Installation**: `docs/operator_manual.md` includes flashing procedure:
```bash
# Manually copy to /boot after build.sh completes
sudo cp artifacts/kernel/Image /boot/kernel8.img
sudo cp artifacts/kernel/*.dtb /boot/
```

---

## libedgetpu (Feranick c38ac3f6) — build & install

**Submodule approach** (reduces repo size):
```bash
# deps/libedgetpu is a git submodule at commit c38ac3f6
git submodule update --init deps/libedgetpu
```

Build with error handling:
```bash
build_libedgetpu() {
  cd deps/libedgetpu
  mkdir -p build && cd build
  cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr/local
  make -j${JOBS}
  
  # NEW: Verify delegate symbols
  nm -D libedgetpu.so | grep "edgetpu_create_delegate" || exit 1
  
  make DESTDIR=${ROOT}/artifacts/libedgetpu install
}
```

---

## TensorFlow / TensorFlow Lite 2.19.1 (CMake-first)

**Submodule**: `deps/tensorflow` is a shallow clone submodule.

Build with fallback protection:
```bash
build_tflite() {
  cd deps/tensorflow
  git checkout v2.19.1
  
  # NEW: Check CMake support for required components
  if ! cmake -L . | grep -q "TFLITE_ENABLE_DELEGATES"; then
    echo "CMake config missing delegates support - using Bazel fallback"
    exec build_tflite_bazel_fallback
  fi
  
  mkdir -p build && cd build
  cmake .. -Dtensorflow_ENABLE_XLA=OFF -Dtensorflow_ENABLE_GPU=OFF \
    -Dtensorflow_BUILD_SHARED_LIBS=ON \
    -DCMAKE_BUILD_TYPE=Release \
    -DTFLITE_ENABLE_DELEGATES=ON \
    -DTFLITE_ENABLE_XNNPACK=ON \
    -DCMAKE_INSTALL_PREFIX=/usr/local
  cmake --build . -- -j${JOBS}
  cmake --install . --prefix=${ROOT}/artifacts/tflite
}
```

**Bazel fallback** (isolated, documented as exception-only):
```bash
build_tflite_bazel_fallback() {
  mkdir -p deps/tools
  wget -O deps/tools/bazel-6.4.0 https://github.com/bazelbuild/bazel/releases/download/6.4.0/bazel-6.4.0-linux-arm64
  echo "<checksum>  deps/tools/bazel-6.4.0" | sha256sum -c
  # Build only missing component, then delete bazel binary
  rm deps/tools/bazel-6.4.0
}
```

---

## libcamera / rpicam / ISP / zero-copy pipeline (enhanced)

Goal: **zero-copy** from camera → IPA/ISP → inference.

**Hardware failure handling** (NEW):
```cpp
// src/camera/camera.cpp
if (!access("/dev/video0", F_OK)) {
  // Normal operation
} else {
  // Graceful degradation: use synthetic test pattern
  log_to_csv("camera_failure", "using_test_pattern");
  enable_test_pattern_mode();
}
```

**Thread affinity** (NEW):
```cpp
// Pin camera thread to CPU core 2
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(2, &cpuset);
pthread_setaffinity_np(camera_thread.native_handle(), sizeof(cpuset), &cpuset);
```

**Queue overflow handling** (NEW):
```cpp
// Drop oldest frame on full queue, log event
if (frame_queue.full()) {
  auto dropped = frame_queue.pop_front();
  telemetry.log_queue_overflow(dropped.timestamp);
}
```

Key runtime pattern:
1. libcamera `Request` provides `FrameBuffer` with DMABUF fd(s)
2. `camera` module exports buffer fd, width, height, pixel format, timestamp (monotonic)
3. `inference` module receives DMABUF fd; creates external tensor without copying
4. ISP produces 320x320 RGB for TPU and 1536x864 for encoder simultaneously

---

## Encoder / H.264 low-latency stream (<100ms)

Build ffmpeg with hardware encoder support:
```bash
build_ffmpeg() {
  cd deps/ffmpeg
  ./configure --enable-omx-rpi --enable-mmal \
    --enable-shared --disable-static \
    --extra-cflags="$CFLAGS" --extra-ldflags="$LDFLAGS"
  make -j${JOBS}
}
```

**Encoder tuning validation** (NEW integration test):
```bash
# tests/integration/test_stream_latency.sh
ffmpeg -re -f lavfi -i testsrc=size=1536x864:rate=60 -t 10 \
  -c:v h264_omx -preset fast -tune zerolatency -g 15 \
  -f rtsp rtsp://localhost:8554/test &
  
# Measure latency with ffprobe
LATENCY=$(ffprobe -v error rtsp://localhost:8554/test 2>&1 | grep "delay")
[ "$LATENCY" -lt 100 ] || exit 1
```

Command for production:
```bash
ffmpeg -f v4l2 -input_format yuv420p -framerate 60 -video_size 1536x864 -i /dev/video0 \
  -c:v h264_omx -preset fast -tune zerolatency -g 15 -bf 0 \
  -f rtsp rtsp://localhost:8554/stream
```

---

## Inference engine runtime (C20, enhanced)

**Threading model**:
- Single multi-threaded process with isolated threads:
  - Camera capture (pinned to CPU 2, SCHED_FIFO priority 50)
  - Inference scheduler (pinned to CPU 3, priority 60)
  - Decision/safety logic (priority 40)
  - Telemetry writer (normal priority)
  - Network streamer (normal priority)

**Error handling** (NEW):
```cpp
try {
  auto delegate = dlopen("libedgetpu.so", RTLD_NOW);
  if (!delegate) throw std::runtime_error("TPU delegate not found");
} catch (const std::exception& e) {
  telemetry.log_inference_fallback("cpu");
  switch_to_cpu_inference();
}
```

**Invariance monitoring** (NEW):
```cpp
// src/decision/safety_monitor.cpp
if (cpu_temp_celsius > 80 || cpu_percent > 95) {
  gate_actuation(false);
  telemetry.log_safety_invariant_violation("thermal_or_cpu");
}
```

**Latency tracking**:
```cpp
struct FrameTiming {
  uint64_t cam_produced_ns;
  uint64_t inference_start_ns;
  uint64_t inference_end_ns;
  uint64_t decision_end_ns;
  uint64_t actuation_command_ns;
};
```

---

## Servo / actuation (PWM 333 Hz, enhanced)

**Hardware liveness checks** (NEW):
```cpp
// src/servo/servo.cpp
if (!pwm_device_ready()) {
  telemetry.log_actuation_error("pwm_device_unavailable");
  enter_safe_state();
  return;
}
```

**Actuation timing monitoring** (NEW unified.csv fields):
```
..., servo_pwm_value, servo_write_latency_ms, servo_status, ...
```

**Safety gate**:
- All commands logged to `unified.csv` and `run.json`
- Hardware kill-switch is authoritative; software polls kill-switch GPIO before every command

---

## Telemetry / logging: unified.csv and run.json (enhanced)

Directory layout:
```
/var/log/aurore/run-<timestamp>/
  run.json
  unified.csv
  raw_frames/
  kernel_logs/
```

**Enhanced unified.csv header**:
```csv
produced_ts_epoch_ms,monotonic_ms,module,event,thread_id,thread_cpu,thread_priority,cam_frame_id,cam_exposure_ms,cam_width,cam_height,cam_pixel_format,cam_buffer_fd,queue_depth_before,queue_depth_after,queue_overflow_count,inference_start_ms,inference_end_ms,inference_latency_ms,inference_delegate_type,inference_confidence,model_name,actuation_decision_ms,actuation_command,servo_pwm_value,servo_write_latency_ms,servo_status,cpu_percent,cpu_temp_c,mem_mb,swap_mb,notes
```

**Log rotation & archiving** (NEW):
```bash
# scripts/log_rotate.sh
#!/bin/bash
RUN_DIRS=(/var/log/aurore/run-*)
if [ ${#RUN_DIRS[@]} -gt 3 ]; then
  # Compress oldest runs to archive
  tar -czf /var/log/aurore/archive/older_runs.tar.gz "${RUN_DIRS[@]:0:${#RUN_DIRS[@]}-3}"
  rm -rf "${RUN_DIRS[@]:0:${#RUN_DIRS[@]}-3}"
fi
```

---

## Tests & hardware-in-the-loop (enhanced)

`tests/` contains:
- Unit tests (C GoogleTest) with static analysis:
  ```bash
  # build.sh runs after compilation
  clang-tidy src/**/*.cpp --config-file=.clang-tidy
  cppcheck --enable=all --error-exitcode=1 src/
  ```
- Integration tests including **encoder latency validation**
- Hardware-in-the-loop test with manual confirmation and **automatic kill-switch polling**

**Test runner** (`build.sh`):
```bash
run_tests() {
  ctest --output-on-failure
  ./tests/integration/test_stream_latency.sh
  echo "HIL test requires manual confirmation. Kill-switch status:"
  ./scripts/check_kill_switch.sh
  read -p "Proceed? (y/N)" confirm
  [ "$confirm" = "y" ] && ./tests/hil/test_servo_movement
}
```

**Test results** saved to `artifacts/test_results.json` with checksums.

---

## `build.sh` behavior (enhanced orchestrating script)

Top-level behavior:
1. Parse CLI args: `--jobs=2|3`, `--publish-github`, `--skip-kernel`, `--force-clean`, `--skip-preflight`
2. Run `scripts/preflight.sh` (unless skipped) which by default will:
   - run `sudo apt update && sudo apt upgrade -y` to fetch and install the latest distro/kernel packages (unless `--skip-upgrade` is passed),
  - capture the resulting kernel package/version (written to `${ROOT}/artifacts/kernel/kernel_package_version.txt`) and use that value for subsequent kernel-source pinning or source selection,
   - verify all dependencies installed,
   - verify disk space ≥ 20GB free,
   - verify thermal headroom ≥ 15°C from threshold
3. Create `work/` sandbox. Export reproducible envs.
4. Initialize git submodules: `git submodule update --init --recursive --depth=1`
5. Apply patches from `patches/`.
6. Build kernel (unless skipped) with thermal monitoring.
7. Build libedgetpu, TFLite (CMake with fallback), libcamera, rpicam, ffmpeg.
8. Build `src/` with CMake and **static analysis**.
9. Run unit and integration tests.
10. Prompt for HIL test confirmation with kill-switch status.
11. Collect artifacts with deterministic tar.
12. Generate `artifacts/checksums.txt` and `deps/manifest.txt`.
13. Run `scripts/log_rotate.sh` to archive old runs.
14. If `--publish-github` provided, create release and upload.

**Important**: `build.sh` will **never** start more than `JOBS=3` jobs and monitors thermal throttling.

---

## CI / Publishing policy (enhanced)

* **Optional CI** on GitHub: syntax validation, doc builds, static analysis.
* **NEW**: QEMU-based Pi emulator for non-hardware integration tests (optional, documented in README).
* Publishing: `--publish-github` creates release with:
  - `aurore-mk-vi-${SOURCE_DATE_EPOCH}.tar.gz`
  - `checksums.txt`
  - `deps/manifest.txt`
  - `toolchain.txt`
  - `test_results.json`

Requires `GITHUB_TOKEN` env var.

---

## Android app (enhanced docs)

**Setup instructions** (NEW in `android_app/README.md`):
1. Connect to Pi's RTSP stream: `rtsp://<pi-ip>:8554/stream`
2. Configure TLS connection for telemetry API:
   ```bash
   # On Pi: copy server cert to android_app/res/raw/
   cp /etc/aurore/tls_cert.pem android_app/src/main/res/raw/pi_cert.crt
   ```
3. Build APK:
   ```bash
   cd android_app
   ./gradlew assembleRelease
   cp app/build/outputs/apk/release/app-release.apk ../artifacts/android/
   ```

**UX flows**:
- Dashboard shows queue depths, invariants, kill-switch status
- Live stream via ExoPlayer
- Manual controls: two-step confirmation  audit log entry in `run.json`

---

## Security & ops (enhanced)

- SSH-only remote access, manual updates only.
- **Audit trails**: every action logged to `run.json` with user identity (if SSH).
- Hardware kill-switch polling implemented in software:
  ```cpp
  while (true) {
    if (read_gpio(KILL_SWITCH_PIN) == 0) {
      emergency_stop();
      break;
    }
    std::this_thread::sleep_for(10ms);
  }
  ```
- **Kernel jitter mitigation**: PREEMPT_RT enabled and validated via `cyclictest` in HIL.

---

## Zero-copy and deterministic pipeline guarantees (practical checklist)

* libcamera configured for dual output via ISP
* Use `DMABUF` fds throughout; `mmap` only for metadata
* Pre-allocate fixed-size ring buffers (size=32 for frames, 128 for inferences)
* **Thread affinity**: camera/inference pinned to cores 2/3
* **Scheduler**: `SCHED_FIFO` for real-time threads (configured in systemd unit)
* **Clocks**: `CLOCK_MONOTONIC` for all deltas; `CLOCK_REALTIME` only for epoch timestamps

---

## Test milestones (ordered, with resource estimates)

1. **env & deps** (~10 min) — clone submodules, manifest generation, preflight checks pass
2. **kernel up** (~60 min) — apply patches, build kernel & dtbs, verify PREEMPT_RT, flash manually
3. **libcamera** (~20 min) — capture 1536x864 stream, verify ISP resizing, test hardware failure fallback
4. **libedgetpu** (~15 min) — build and verify delegate symbols load
5. **tflite runtime** (~120 min) — build via CMake, run CPU inference, validate fallback path
6. **encode & stream** (~30 min) — build ffmpeg, stream H.264, **run latency validation test**
7. **zero-copy pipeline** (~15 min) — camera→tpu path, measure <15 ms latency, verify no copies
8. **decision & safety** (~10 min) — integrate safety monitor, test invariance violations
9. **HIL** (~5 min  manual) — hardware test with kill-switch verification
10. **artifactization** (~5 min) — deterministic tarball, checksums, static analysis report included

Each milestone produces artifacts with checksums and can be run independently with `build.sh --milestone=<name>`.

---

## Edge cases & risk mitigation (enhanced)

* **libedgetpu ABI drift**: `dlopen()` with versioned symbols; CPU fallback with telemetry alert
* **Kernel PCIe breaks**: `docs/runbooks/rollback.md` includes one-liner restore:
  ```bash
  sudo cp /boot/kernel8.img.backup /boot/kernel8.img && sudo reboot
  ```
* **Camera failure**: automatic test pattern generation; `unified.csv` logs `camera_failure` event
* **TPU overheating**: safety monitor gates actuation above 80°C; logs `thermal_violation`
* **Queue overflow**: oldest frame dropped, counter incremented in `unified.csv`

---

## Useful exact command snippets (updated)

Kernel build snippet (with RT verification):
```bash
build_kernel() {
  scripts/preflight.sh --check-temp
  # ... build steps ...
  if ! grep -q "PREEMPT_RT=y" build/.config; then
    echo "ERROR: PREEMPT_RT not enabled"
    exit 1
  fi
}
```

TFLite build snippet (with CMake validation):
```bash
build_tflite() {
  if ! cmake -L . | grep -q "TFLITE_ENABLE_DELEGATES:BOOL=ON"; then
    echo "Falling back to Bazel for delegate support"
    build_tflite_bazel_fallback
    return
  fi
  # ... cmake build ...
}
```

**NEW**: Static analysis snippet:
```bash
run_static_analysis() {
  find src/ -name "*.cpp" -exec clang-tidy {} --config-file=.clang-tidy \;
  cppcheck --enable=all --error-exitcode=1 src/
}
```

**NEW**: Log rotation snippet:
```bash
scripts/log_rotate.sh  # Called by build.sh after tests
```

---

## Android app minimal notes (enhanced)

* App receives H.264 stream (RTSP) and connects to health API (HTTP/JSON  TLS)
* **Setup**: TLS cert must be copied from Pi to `res/raw/` before build
* **UX**: Dashboard includes kill-switch status indicator; manual controls require 2-step auth
* Build: Gradle; APK placed in `artifacts/android/` with checksum

---

## Deliverables at each milestone (enhanced)

`artifacts/` contains:
- `kernel/` (Image, dtbs, .config, RT validation log)
- `libs/` (libedgetpu, libtensorflow-lite)
- `bin/` (main binary)
- `models/`
- `tests/` (test reports, static analysis report)
- `android/` (APK)
- `checksums.txt`
- `deps/manifest.txt`
- `toolchain.txt`
- `static_analysis_report.txt`  # NEW
- `docs/` (installation guide, runbooks)

Single deterministic tarball with all of the above, plus `BUILD_INFO.json` containing:
- `build_timestamp`: SOURCE_DATE_EPOCH
- `os_version`: pinned OS version
- `thermal_throttle_events`: count during build
- `tests_passed`: count

---

## Build environment verification (NEW section)

Before first build, run:
```bash
./scripts/preflight.sh --full
```

This verifies (default flow):
- Run `sudo apt update && sudo apt upgrade -y` (unless `--skip-upgrade`); record resulting kernel/distro package versions to `${ROOT}/artifacts/kernel/kernel_package_version.txt` and `deps/manifest.txt`.
- All dependencies installed with correct versions
- Disk space ≥ 20GB
- CPU temperature < 60°C
- Git submodules properly configured


To install dependencies:
```bash
sudo apt update && sudo apt install -y git cmake make g-11 clang-tidy cppcheck \
  libssl-dev libprotobuf-dev fakeroot
```

---

## Runtime operations guide (NEW in docs/runbooks/monitoring.md)

**Monitoring latency**:
```bash
# Watch for end-to-end latency >15ms
tail -f /var/log/aurore/run-*/unified.csv | awk -F, '$9 > 15 {print "LATENCY WARNING:", $0}'
```

**Checking kill-switch**:
```bash
./scripts/check_kill_switch.sh  # Returns 0 if engaged, 1 if disarmed
```

**Updating without kernel changes**:
```bash
./build.sh --skip-kernel --skip-preflight
```

---

