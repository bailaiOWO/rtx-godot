# rtx-godot

<p align="center">
  <a href="https://godotengine.org">
    <img src="misc/logo/logo_outlined.svg" width="300" alt="Godot Engine logo">
  </a>
</p>

**Godot 引擎的 NVIDIA RTX 分叉（硬件路径追踪 + DLSS），持续跟随上游最新开发版。**
[English](#english) below.

> 社区维护项目，与 NVIDIA、Godot 基金会均无关联。本仓库把 NVIDIA 官方分叉
> [NVIDIA-RTX/godot](https://github.com/NVIDIA-RTX/godot) 的路径追踪与 DLSS 实现合并到上游
> [godotengine/godot](https://github.com/godotengine/godot) 的最新快照上，并持续同步。

## 当前状态

| 项目 | 值 |
|---|---|
| 上游基线 | Godot **4.8-dev5**（提交 `9552dfb685`，2026-09-10） |
| NVIDIA 基线 | `nvidia-pt-dlss` 分支提交 `135dff3887`（2026-07-24，基于 2026-06-23 的上游 master） |
| Streamline SDK | 2.10.0（DLSS 超分辨率 / 光线重建 / 帧生成、Reflex、NIS） |
| 平台 | Windows x64（Vulkan 与 Direct3D 12）。Streamline 相关功能仅 Windows |
| 集成分支 | `rtx-main` |

## 功能（来自 NVIDIA 分叉）

- **硬件路径追踪**：Forward+ 渲染器的光追变体（`RenderForwardClusteredPT`），物理正确的全局光照、反射与阴影；支持蒙皮/形变网格、MultiMesh 合并 BLAS、NVIDIA Shader Execution Reordering。
- **DLSS**：超分辨率（Super Resolution）、光线重建（Ray Reconstruction，作为路径追踪降噪器）、帧生成（Frame Generation）；Reflex 低延迟与帧率限制；NIS。
- **RTProceduralInstance3D** 节点：以 AABB 向 TLAS 提交程序化几何，由自定义相交着色器处理。
- **编辑器简体中文**：上述新增设置项的显示名与类参考说明已翻译（`zh_Hans`），编辑器语言设为简体中文时可直接查看。
- **调试与工具**：路径追踪调试可视化通道、DLSS RR 输入缓冲调试绘制、光追统计（TLAS/BLAS 计数）、`--gpu-markers`（RenderDoc / Nsight / PIX 事件标记）、`--raytracing-validation`、可选的 Nsight Aftermath GPU 崩溃转储。

## 下载

到 [Releases](https://github.com/bailaiOWO/rtx-godot/releases) 下载 Windows x64 编辑器压缩包，解压即用。压缩包已包含：

- `rtx-godot_v<版本>_win64.exe` 与 `_console.exe` 编辑器
- Streamline 运行时 DLL（`sl.*.dll`、`nvngx_*.dll`），必须与 exe 同目录
- `D3D12Core.dll`（DirectX 12 Agility SDK）

**硬件要求**：Windows 10/11 x64。路径追踪需要支持光线追踪的 GPU（在 NVIDIA RTX 上测试）。DLSS 超分与光线重建需要 NVIDIA RTX 20 系及以上；帧生成需要 RTX 40 系及以上。请使用较新的 Game Ready 驱动。非 NVIDIA GPU 上 Streamline 功能不可用，引擎其余功能与上游一致。

**导出游戏**：导出模板需自行按下文编译 `target=template_release` / `template_debug`；导出后的游戏同样要把 Streamline DLL 放在游戏 exe 旁边。

## 使用与设置说明

### 开启路径追踪

1. 项目设置 → 渲染 → 渲染器：使用 **Forward+**。
2. 场景中的 `WorldEnvironment` → `Environment` 资源 → **Pathtracing** 分组：勾选 `pathtracing_enabled`。
3. **必须**把项目设置 `rendering/scaling_3d/mode` 设为 **DLSS**：DLSS 光线重建降噪是在 DLSS 缩放通道里执行的，缩放模式保持默认的 Bilinear 时，`pathtracing_denoiser` 选什么都不会跑，画面就是 1 spp 的原始噪点。`rendering/scaling_3d/scale` 设 1.0 为原生分辨率纯降噪（DLAA），0.5 到 0.67 则同时超分。

### Environment（`Environment` 资源，Pathtracing 分组）

| 属性 | 默认 | 说明 |
|---|---|---|
| `pathtracing_enabled` | 关 | 为该环境启用硬件加速路径追踪。与光栅化特性互斥：开启后 SDFGI、屏幕空间反射等基于光栅化的效果不再生效。 |
| `pathtracing_samples_per_pixel` | 1 | 每像素每帧发射的光线样本数。越高噪点越少，GPU 开销成比例增加。配合 DLSS 光线重建这类时域降噪器时 1 到 2 即可。 |
| `pathtracing_max_bounces` | 3 | 光线在被终止前允许的最大反弹次数。越高间接光与相互反射越准确，开销越大。0 表示只算直接光照。 |
| `pathtracing_denoiser` | DLSS 光线重建 | 降噪器。`PT_DENOISER_DLSS_RAY_RECONSTRUCTION` 使用 NVIDIA DLSS 光线重建做高质量时域降噪，需要 NVIDIA RTX GPU；`PT_DENOISER_NONE` 关闭降噪。 |
| `pathtracing_debug_mode` | 关闭 | 用调试通道替代最终画面，便于检查 G-buffer 输入与中间着色数据。可选：镜面反射、几何法线、最终法线、法线贴图、切线、副切线、UV、Albedo、ORM、漫反射 Albedo、高光 Albedo、法线+粗糙度、高光命中距离、金属度、粗糙度、视空间法线、漫反射/高光左右分屏、菲涅尔 F0、正面/背面（绿/红）、深度、自发光、BRDF 拒绝光线。 |

### 项目设置 `rendering/pathtracing/*`

| 设置 | 默认 | 说明 |
|---|---|---|
| `async_shader_compilation` | 开 | 路径追踪的命中着色器在后台异步编译，避免渲染中首次遇到新着色器变体时整帧卡顿。编译期间可能临时使用回退着色器而出现短暂瑕疵。若要求首帧就完全确定的画面，可关闭（代价是着色器预热时可能卡顿）。 |
| `deformed_mesh_cache_ttl_frames` | 60 | 蒙皮或顶点位移的形变网格 BLAS 在最后一次可见后于缓存中保留的帧数，超过即释放 BLAS 及其缓冲。形变网格短暂被遮挡后重新出现时若闪烁，可调大；想尽早回收显存则调小。需重启生效。 |
| `multimesh_blas_cache_ttl_frames` | 3600 | 合并 MultiMesh BLAS 在最后一次可见后保留的帧数（60 fps 下约一分钟），超过即释放 BLAS 与全部合并的顶点、属性、索引缓冲。MultiMesh 从场景消失后想尽快回收显存可调小。需重启生效。 |
| `multimesh_cache_cpu_transforms` | 关 | 开启后设置 `MultiMesh.buffer` 时会在 CPU 端同步保留一份实例变换镜像，路径追踪器首次读取每实例变换时无需 GPU 到 CPU 回读，消除一次性卡顿。代价是增加 CPU 内存占用。 |
| `multimesh_merged_blas_max_triangles` | 65536 | MultiMesh 表面使用单个合并 BLAS 的最大总三角数（实例数 × 每表面三角数）。超过则退回为每个实例一个 TLAS 条目、共用一个 BLAS。调小可降低大 MultiMesh 的峰值显存，调大则更倾向合并路径。 |
| `use_shader_execution_reordering` | 开 | 启用 NVIDIA Shader Execution Reordering（SER），运行时重排光追着色器调用，提高 GPU 占用率、减少不同材质线程间的分歧。在 Ada Lovelace（RTX 40）及之后的 GPU 上可显著提升路径追踪性能；不支持的硬件上无效果。 |
| `use_simple_shadows` | 关 | 阴影光线改用 Ray Query 而非完整追踪。Ray Query 便宜得多，但依赖快速的 alpha 贴图采样，对使用生成式或动画 alpha 贴图的材质可能产生错误阴影。仅在当前 Environment 开启路径追踪时有效。 |

### 项目设置 `rendering/streamline/*`

| 设置 | 默认 | 说明 |
|---|---|---|
| `dlss_preset` | `?` | DLSS 超分辨率默认预设。可选 `?`（自动）、`F` 到 `O`。 |
| `dlss_ray_reconstruction_preset` | `?` | DLSS 光线重建默认预设。可选 `?`（自动）、`D` 到 `O`。 |
| `reflex_mode` | 0 | 0 关闭；1 启用 NVIDIA Reflex，在垂直同步前恰好渲染以降低延迟；2 额外启用低延迟增强（Boost）。 |
| `reflex_frame_limit_us` | 0 | 不为 0 时启用 Reflex 帧率限制器，在尽量保持低延迟的同时限制帧率。单位微秒：60 FPS = 16667。 |
| `streamline_imgui` | 关 | 启用 Streamline 的 ImGui 调试面板。 |
| `streamline_log` | 关 | 启用 Streamline 日志，排错用。 |

### Viewport

| 项 | 说明 |
|---|---|
| `scaling_3d_mode = SCALING_3D_MODE_DLSS` | 3D 缓冲用 NVIDIA DLSS 放大。`scaling_3d_scale` 小于 1.0 时启用超分辨率；路径追踪开启时为光线重建。大于 1.0 不支持，会退回双线性降采样。 |
| `frame_generation` | 在该 Viewport 上启用 DLSS 帧生成（仅 NVIDIA，RTX 40 及以上）。 |
| 调试绘制 `DEBUG_DRAW_DLSS_RR_*` | 显示提供给 DLSS 光线重建的输入缓冲：漫反射 Albedo、高光 Albedo、法线+粗糙度、高光命中距离；`DEBUG_DRAW_RECONSTRUCTED_DEPTH` 显示降噪后重建的深度。仅光线重建激活时有效。 |
| 渲染信息 `RENDER_INFO_RT_*` | 本帧提交的 TLAS 实例数、完整 BLAS 构建数、BLAS 重构（refit）数、构建与重构处理的三角数。 |

### 脚本 API

- `Streamline` 单例：`get_capability(type)` 查询 `STREAMLINE_CAPABILITY_DLSS` / `DLSS_G`（帧生成）/ `DLSS_RR` / `NIS` / `REFLEX` / `PCL` 是否可用；`set_parameter(type, value)` 设置 `STREAMLINE_PARAM_REFLEX_MODE`、`REFLEX_FRAME_LIMIT_US`、`DLSS_PRESET`、`DLSS_RR_PRESET`。
- `RenderingServer`：`environment_set_pathtracing`、`viewport_set_frame_generation`、`instance_set_rt_procedural`、`instance_set_rt_procedural_bounds`。
- `RTProceduralInstance3D`：`bounds`（每图元 AABB 数组，相交着色器里用 `gl_PrimitiveID` 索引）、`size`（`bounds` 为空时的单个包围盒尺寸）、`custom_enclosing_aabb`（覆盖用于 TLAS 剔除的保守包围体）、`expose_aabb_bounds`（把每图元 AABB 暴露给命中着色器）。

### 命令行

`--gpu-markers`（Vulkan debug utils / D3D12 PIX 事件标记）、`--debug-shaders`（等同 `--generate-spirv-debug-info`，RenderDoc 源码级着色器调试）、`--raytracing-validation`（NVIDIA 光追校验层，需 `VK_NV_ray_tracing_validation`）。

## 从源码构建

前置：Visual Studio 2022（MSVC v143）、Python 3.8+、SCons 4.x。首次先下载 D3D12 与 AccessKit 依赖（进入仓库内 `bin/build_deps`）：

```bash
env -u LOCALAPPDATA python misc/scripts/install_d3d12_sdk_windows.py
env -u LOCALAPPDATA python misc/scripts/install_accesskit.py
```

编译编辑器（把 `dev5` 换成当前跟随的上游版本状态）：

```bash
GODOT_VERSION_STATUS=dev5 scons platform=windows target=editor arch=x86_64 d3d12=yes angle=no use_streamline=yes mesa_libs=<绝对路径>/bin/build_deps/mesa agility_sdk_path=<绝对路径>/bin/build_deps/agility_sdk pix_path=<绝对路径>/bin/build_deps/pix accesskit_sdk_path=<绝对路径>/bin/build_deps/accesskit
```

依赖路径必须是绝对路径。构建选项：`use_streamline`（默认开，仅 Windows）、`use_aftermath`（默认关，会从 NVIDIA 开发者站下载 Nsight Aftermath SDK）。运行前把 Streamline 2.10.0 的 `bin/x64/*.dll`（[NVIDIA-RTX/Streamline v2.10.0](https://github.com/NVIDIA-RTX/Streamline/releases/tag/v2.10.0)）复制到 exe 旁；头文件（`thirdparty/streamline/include`）与 DLL 版本必须一致。

## 分支与同步策略

- `rtx-main`：集成分支 = 上游快照 + NVIDIA 改动 + 本仓库的修复。用 **merge** 跟随上游，不 rebase、不强推。
- `4.8-devN` 等 tag 指向上游快照提交；`rtx-4.8-devN` 等 tag 对应本仓库的发布版。
- `.github/` 始终保持与上游一致，NVIDIA 的自定义 CI 未采用。
- NVIDIA 原始分支保留为 `nvidia-pt-dlss` 供对照。

| 日期 | 上游 | 说明 |
|---|---|---|
| 2026-09-13 | 4.8-dev5 (`9552dfb685`) | 首次同步。14 个文件冲突，解法见合并提交 `a158c2fca3` 的说明。 |
| 2026-09-13 | 4.8-dev5 (`9552dfb685`) | v4.8-dev5-2：修复路径追踪把正交相机渲染成透视锥的问题（光线生成改为按投影矩阵反投影构造主光线），DLSS 常量按相机投影报告正交。 |

## 致谢与许可

- Godot Engine：© Godot Engine contributors，[MIT](LICENSE.txt)。
- 路径追踪与 DLSS 集成：© NVIDIA Corporation，来自 [NVIDIA-RTX/godot](https://github.com/NVIDIA-RTX/godot)，MIT。
- Streamline SDK 头文件（`thirdparty/streamline`）与 `sl.*.dll`：© NVIDIA Corporation，[MIT](https://github.com/NVIDIA-RTX/Streamline/blob/main/license.txt)。
- `nvngx_dlss.dll`、`nvngx_dlssd.dll`、`nvngx_dlssg.dll` 等 DLSS 运行时为 NVIDIA 专有二进制，按 Streamline SDK 随附的 NVIDIA 许可条款再分发；发布包内附带许可文本。
- 本仓库自身的改动同样以 MIT 许可发布。

---

## English

**An NVIDIA RTX fork of Godot Engine (hardware path tracing + DLSS) that tracks upstream Godot's latest development snapshots.**

This is a community-maintained repository, not affiliated with NVIDIA or the Godot Foundation. It merges the path tracer and DLSS integration from NVIDIA's official fork, [NVIDIA-RTX/godot](https://github.com/NVIDIA-RTX/godot) (branch `nvidia-pt-dlss`), onto the newest [godotengine/godot](https://github.com/godotengine/godot) snapshot and keeps it in sync.

| Item | Value |
|---|---|
| Upstream base | Godot **4.8-dev5** (commit `9552dfb685`, 2026-09-10) |
| NVIDIA base | `nvidia-pt-dlss` at `135dff3887` (2026-07-24) |
| Streamline SDK | 2.10.0 (DLSS Super Resolution / Ray Reconstruction / Frame Generation, Reflex, NIS) |
| Platform | Windows x64, Vulkan and Direct3D 12. Streamline features are Windows-only |
| Integration branch | `rtx-main` |

**Simplified Chinese:** display names and class-reference descriptions of the new settings are translated (`zh_Hans`).

**Features (from NVIDIA's fork):** hardware path tracing as a variant of the Forward+ renderer (physically based GI, reflections and shadows, skinned and deformed meshes, merged MultiMesh BLAS, Shader Execution Reordering), DLSS Super Resolution, DLSS Ray Reconstruction as the path-tracing denoiser, DLSS Frame Generation, Reflex, NIS, the `RTProceduralInstance3D` node for AABB-based procedural geometry, path-tracing debug views, ray-tracing statistics, `--gpu-markers`, `--raytracing-validation`, and optional Nsight Aftermath crash dumps.

**Download:** grab the Windows x64 editor zip from [Releases](https://github.com/bailaiOWO/rtx-godot/releases). It ships the editor executables, the Streamline runtime DLLs (which must stay next to the executable) and `D3D12Core.dll`. Path tracing needs a ray-tracing capable GPU (tested on NVIDIA RTX); DLSS Super Resolution and Ray Reconstruction need an RTX 20 series or newer, Frame Generation an RTX 40 series or newer. Export templates are not shipped yet; build them with `target=template_release` / `template_debug` and keep the Streamline DLLs beside the exported executable.

**Usage:** use the Forward+ renderer, enable `pathtracing_enabled` on the `Environment` resource, and set `rendering/scaling_3d/mode` to DLSS (required: DLSS Ray Reconstruction denoising runs inside the DLSS scaling pass, so with the default Bilinear mode the denoiser setting has no effect). Scale 1.0 = native-resolution denoising (DLAA), 0.5 to 0.67 = upscaling as well. Path-tracing options live under `rendering/pathtracing/*`, Streamline options under `rendering/streamline/*`; `Viewport.frame_generation` toggles DLSS Frame Generation. The `Streamline` singleton exposes `get_capability()` and `set_parameter()`. See the Chinese section above for a description of every setting, or the built-in class reference.

**Building:** Visual Studio 2022, Python 3.8+, SCons 4.x. Install the D3D12 and AccessKit dependencies into `bin/build_deps` with `env -u LOCALAPPDATA python misc/scripts/install_d3d12_sdk_windows.py` and `install_accesskit.py`, then build with `GODOT_VERSION_STATUS=dev5 scons platform=windows target=editor arch=x86_64 d3d12=yes angle=no use_streamline=yes` plus absolute `mesa_libs`, `agility_sdk_path`, `pix_path` and `accesskit_sdk_path` pointing into `bin/build_deps`. Copy the Streamline 2.10.0 runtime DLLs next to the executable before running; header and DLL versions must match.

**Branching:** `rtx-main` = upstream snapshot + NVIDIA changes + our fixes, updated by merging upstream (no rebase). Tags `4.8-devN` mark upstream snapshot commits, tags `rtx-4.8-devN` mark our releases. `.github/` always mirrors upstream. The original NVIDIA branch is kept as `nvidia-pt-dlss` for reference.

**License:** Godot Engine and the NVIDIA integration are MIT licensed; Streamline SDK headers and `sl.*.dll` are MIT; the DLSS runtime binaries (`nvngx_*.dll`) are proprietary NVIDIA binaries redistributed under the NVIDIA license shipped with the Streamline SDK (the license text is included in release packages). Changes made in this repository are MIT as well.
