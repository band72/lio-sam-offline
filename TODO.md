# TODO: lio-sam-offline

Estimated effort: **3+ months**. Build platforms: Windows, Linux and macOS (develop on Linux first, keep Windows and macOS CI green from day one).

## Target platforms and versions (reviewed 2026-11)

| Platform | Target | Toolchain | CI runner | Package |
|---|---|---|---|---|
| Windows | **Windows 11 25H2/26H2** x64; Windows 10 22H2 x64 while ESU lasts (to 2027-10-12) | MSVC v143 (Visual Studio 2022), C++17 | `windows-2025` | Signed MSI (WiX v4+) or MSIX |
| Linux | **Ubuntu 22.04 and 24.04 LTS** x86_64 (other glibc ≥ 2.35 distros via AppImage) | GCC 11+ | `ubuntu-22.04` (build AppImage on the oldest glibc), `ubuntu-24.04` | AppImage + `.deb` |
| macOS | **macOS 14 Sonoma or later, Apple Silicon (arm64)** | Apple Clang (Xcode 16+) | `macos-15` (arm64) | Signed, notarized `.dmg` |

- Intel macOS is out of scope: GitHub retired the Intel `macos-13` runner in Dec 2025, and Intel support on GitHub runners ends with the macOS 15 image (fall 2027).
- Pin **GTSAM 4.2** to match LIO-SAM-V2 results; evaluate GTSAM 4.3 (Boost made optional) as a separate task.
- No ROS 2 at runtime. Bag formats: ROS 2 (sqlite3, mcap) and ROS 1 `.bag`.

## Phase 0: Project setup
- [ ] CMake ≥ 3.24 + **vcpkg manifest mode** (`vcpkg.json` with a `builtin-baseline`) and vcpkg binary caching in CI
- [ ] CI matrix (Windows / Linux / macOS) building an empty `liblio_sam` and `lio-sam-offline`; warnings as errors; `clang-format` and `clang-tidy` checks
- [ ] Turn on branch protection for `main` and require all three platforms to pass
- [ ] Capture reference outputs from LIO-SAM-V2 on Linux (trajectory, keyframe poses, map point count) for the 90 s and full Ouster sample bags; these are the regression baseline

## Phase 1: Architecture (~1–2 weeks)
- [ ] Map every ROS publisher/subscriber between the four nodes (`cloud_info`, `odometry/imu`, `odometry/imu_incremental`, `lio_sam/mapping/odometry`, …) to direct function calls or queues
- [ ] Threading model: single-threaded deterministic replay first, parallel later
- [ ] Replace `rclcpp` parameters with a YAML config loader that accepts existing `params*.yaml` files unchanged
- [ ] Replace TF2 with Eigen transforms
- [ ] Use only portable C++17 and `std::filesystem`; no POSIX-only calls (`system()`, `getenv("HOME")`)
- [ ] Write `docs/architecture.md`

## Phase 2: Core refactor (`src/`, ~4–6 weeks)
- [ ] `ImuPreintegration`: strip ROS, keep the GTSAM preintegration logic
- [ ] `ImageProjection`: ROS-free; keep Velodyne/Ouster point types (check MSVC and Apple Clang alignment) and deskewing
- [ ] `FeatureExtraction`: ROS-free
- [ ] `MapOptimization`: ROS-free, including loop closure, GPS factor and landmark factor
- [ ] Port `targetDetector` (survey landmarks) or mark it out of scope for v1
- [ ] Replace custom messages (`cloud_info`, `SaveMap` service) with plain structs and functions
- [ ] Unit tests per stage

## Phase 3: I/O (~2 weeks)
- [ ] Bag reader for ROS 2 (sqlite3, then mcap) and ROS 1 `.bag`, with no ROS dependency (small standalone reader, or mcap's C++ library plus SQLite)
- [ ] Sensor adapters: Ouster (incl. clock re-stamping) and MicroStrain (NED→ENU, IMU-to-lidar calibration), ported from `upstream/scripts/convert_ros1_bag.py`
- [ ] Writers: PCD, PLY, LAS/LAZ (LASzip or PDAL)
- [ ] Georeferencing (GNSS fit + UTM via PROJ, shipping `proj.db`), ported from `upstream/scripts/georeference_map.py`
- [ ] Dense map builder, ported from `upstream/scripts/densify_map.py`

## Phase 4: Apps (`apps/`, ~2 weeks)
- [ ] CLI: `process`, `export`, `densify`, with progress output and non-zero exit codes on failure
- [ ] GUI: one cross-platform toolkit for all three OSes (Qt 6 LTS, or Python/Tk on top of a Python binding); follow each platform's file-dialog and high-DPI behaviour
- [ ] Per-platform data folders: `%LOCALAPPDATA%` (Windows), XDG dirs (Linux), `~/Library/Application Support` (macOS)

## Phase 5: Validation (~2 weeks)
- [ ] Regression: offline results within tolerance of the LIO-SAM-V2 baseline (path length within 1%, loop-closure gap within 0.5 m on the sample bag), on all three platforms
- [ ] Determinism: two runs produce identical maps on the same platform; document expected cross-platform floating-point differences
- [ ] Performance: faster than real time on the 11-minute sample bag on a typical laptop

## Phase 6: Packaging and release
- [ ] **Windows:** WiX v4+ MSI (or MSIX) with the VC++ 2015–2022 redistributable; sign with **Azure Artifact Signing** (~$9.99/month) or OV; EV no longer gives instant SmartScreen trust
- [ ] **macOS:** Developer ID signing, hardened runtime, **notarization with `notarytool`** and stapling (Apple Developer Program, $99/year)
- [ ] **Linux:** AppImage built on Ubuntu 22.04 (oldest supported glibc) plus a `.deb` for 24.04; GPG-signed checksums
- [ ] GitHub Actions release workflow attaching all builds and SHA-256 checksums
- [ ] `THIRD_PARTY_NOTICES.md` (GTSAM BSD, PCL BSD, OpenCV Apache-2.0, Eigen MPL2, PROJ MIT, LASzip LGPL, Qt LGPL if used)
- [ ] User guide and a migration note for LIO-SAM-V2 users

## Ongoing
- [ ] Re-check the platform table each year: Windows 11 annual releases, GitHub runner retirements (macOS 14 arm64 listed as EOL around Nov 2026 by third-party trackers), Ubuntu LTS releases
