# lio-sam-offline

**ROS-free offline LIO-SAM: one executable that turns a recorded bag into a survey map.**

A refactor of [LIO-SAM-V2](https://github.com/band72/LIO-SAM-V2) for post-processing. Its four SLAM nodes
(IMU preintegration, image projection, feature extraction, map optimization) become one library driven by a
single offline program. It reads ROS 1/ROS 2 bags directly and writes PCD/PLY/LAS maps, with no ROS 2, DDS,
RViz or Qt at runtime. It builds natively on **Windows, Linux and macOS**.

> **Status: planning / scaffolding.** Nothing is built yet. See [TODO.md](TODO.md).

This is **Route C** of the [Windows packaging analysis](https://github.com/band72/LIO-SAM-V2/blob/main/Enabling%20Windows%20Capable%20Solution.md):
estimated **3+ months**, medium risk, and the lowest long-term maintenance of the three Windows options.

## Target usage

```text
lio-sam-offline process  survey.bag  --config params_ouster64.yaml  --out maps/run1/
lio-sam-offline export   maps/run1/  --las --georef gnss --epsg auto
lio-sam-offline densify  maps/run1/  survey.bag  --voxel 0.02
```

Plus a desktop GUI (bag picker, config, progress, export) on top of the same library.

## Design

```
bag reader (rosbag2 / ROS 1)  →  time-ordered IMU + lidar stream
        →  liblio_sam: ImuPreintegration · ImageProjection · FeatureExtraction · MapOptimization
           (plain C++ classes, direct calls instead of ROS topics, deterministic and faster than real time)
        →  map + trajectory  →  PCD / PLY / LAS (georeferenced)
```

- Runtime dependencies: **GTSAM 4.2, PCL, OpenCV, Eigen**, and a bag reader. No ROS.
- Visualization: export to CloudCompare or similar, with an optional lightweight viewer later.
- Deterministic: the same bag and config always produce the same map, unlike real-time playback.

## Repository layout

| Path | Purpose |
|---|---|
| `upstream/` | Git submodule → `LIO-SAM-V2` (`ros2` branch). Reference for the algorithms; **not modified here**. |
| `src/` | `liblio_sam`: SLAM core refactored out of the ROS nodes |
| `apps/` | `lio-sam-offline` command-line tool and the GUI |
| `python/` | Ported processing tools: bag conversion, LAS export, georeferencing, densify |
| `tests/` | Regression tests against LIO-SAM-V2 results on sample bags |
| `docs/` | Architecture notes, user guide |

## Getting the source

```bash
git clone --recurse-submodules https://github.com/band72/lio-sam-offline.git
```

## Build platforms

Windows (MSVC 2022), Linux (GCC/Clang) and macOS (Clang), via CMake and vcpkg.

## License

MIT (see [LICENSE](LICENSE)), matching LIO-SAM-V2. Algorithms derive from LIO-SAM (Tixiao Shan et al.);
keep upstream attribution in the source headers.
