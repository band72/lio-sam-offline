# TODO: lio-sam-offline

Estimated effort: **3+ months**. Build platforms: Windows, Linux, macOS (develop on Linux first and keep Windows CI green).

## Phase 0: Project setup
- [ ] Create the public GitHub repo `band72/lio-sam-offline` and push this scaffolding
- [ ] CMake + vcpkg skeleton building an empty `liblio_sam` and `lio-sam-offline` on all three platforms in CI
- [ ] Capture reference outputs from LIO-SAM-V2 on Linux (trajectory, keyframe poses, map point count) for the 90 s and full Ouster sample bags; these are the regression baseline

## Phase 1: Architecture (~1–2 weeks)
- [ ] Map every ROS publisher/subscriber between the four nodes (`cloud_info`, `odometry/imu`, `odometry/imu_incremental`, `lio_sam/mapping/odometry`, …) to direct function calls or queues
- [ ] Decide the threading model: single-threaded deterministic replay first, parallel later
- [ ] Replace `rclcpp` parameters with a YAML config loader that accepts existing `params*.yaml` files unchanged
- [ ] Replace TF2 usage with Eigen transforms
- [ ] Write `docs/architecture.md`

## Phase 2: Core refactor (`src/`, ~4–6 weeks)
- [ ] `ImuPreintegration`: strip ROS, keep the GTSAM preintegration logic
- [ ] `ImageProjection`: ROS-free; keep Velodyne/Ouster point types and deskewing
- [ ] `FeatureExtraction`: ROS-free
- [ ] `MapOptimization`: ROS-free, including loop closure, GPS factor and landmark factor; replace `system("rm -rf")` with `std::filesystem`
- [ ] Port `targetDetector` (survey landmarks) or mark it out of scope for v1
- [ ] Replace custom messages (`cloud_info`, `SaveMap` service) with plain structs and functions
- [ ] Unit tests per stage

## Phase 3: I/O (~2 weeks)
- [ ] Bag reader: ROS 2 bags (sqlite3, then mcap) and ROS 1 `.bag`; evaluate the rosbag2 C++ API without ROS versus a small standalone reader
- [ ] Sensor adapters: Ouster (incl. clock re-stamping) and MicroStrain (NED→ENU, IMU-to-lidar calibration); fold in the logic from `upstream/scripts/convert_ros1_bag.py`
- [ ] Writers: PCD, PLY, LAS/LAZ (via PDAL or LASzip)
- [ ] Georeferencing (GNSS fit + UTM via PROJ), ported from `upstream/scripts/georeference_map.py`
- [ ] Dense map builder, ported from `upstream/scripts/densify_map.py`

## Phase 4: Apps (`apps/`, ~2 weeks)
- [ ] CLI: `process`, `export`, `densify`, with progress output and non-zero exit codes on failure
- [ ] GUI: reuse the LIO-SAM Studio layout (Tk via Python bindings, or Qt), driving the library
- [ ] Optional: lightweight point cloud preview

## Phase 5: Validation (~2 weeks)
- [ ] Regression: offline results within tolerance of the LIO-SAM-V2 baseline (path length within 1%, loop-closure gap within 0.5 m on the sample bag)
- [ ] Determinism: two runs produce identical maps
- [ ] Performance: faster than real time on the 11-minute sample bag on a typical laptop

## Phase 6: Packaging and release
- [ ] Windows: signed installer (WiX/MSIX) with vcpkg-built DLLs; Linux: AppImage or `.deb`; macOS: `.dmg`
- [ ] GitHub Actions release workflow attaching all three builds
- [ ] `THIRD_PARTY_NOTICES.md` (GTSAM BSD, PCL BSD, OpenCV Apache-2.0, Eigen MPL2, PROJ MIT, LASzip LGPL)
- [ ] User guide and a migration note for LIO-SAM-V2 users
