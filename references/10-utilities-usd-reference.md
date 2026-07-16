# 实用工具、USD 与参考资料（6.0.1）

## 浏览器变化

6.0 移除了旧的 `isaacsim.asset.browser`（5.1 的 Isaac Sim Asset Browser），资产改由以下入口获取：

- **Content Browser**（Window > Browsers > Content）：在目录树 **Isaac Sim** 文件夹下浏览/加载 Isaac Sim Asset Cards，是 6.0 的主资产入口；
- **NVIDIA Asset Browser / Material Browser / SimReady Explorer**：沿用 Omniverse 通用扩展；
- **SimReady Content Browser**（`omni.simready.content.browser`，新增教程页）：结合目录导航与 AI 搜索，支持三种模式——File Index（本地或 `Assets/Isaac/*` S3 路径名匹配）、AI Search（自然语言搜索 S3，返回 0–1 相关度分）、WSCache（按 profile/feature/tags 过滤 SimReady Workspace Cache）。

来源: utilities/browsers.html、utilities/content_browser.html、utilities/tutorial_search_extension.html

## 扩展模板（含 CLI，6.0 新增）

- **CLI Extension Templates（6.0 新增）**：终端脚手架 `./repo.sh template new`（Windows 用 `repo.bat`），交互式选择模板并填写名称/版本/类别；支持 `--generate-playback <file.toml>` 录制选择、`template replay` 非交互复放（CI 友好），生成后 `./build.sh` 重建。四种模板：Python 纯扩展、UI 扩展（接入 Examples Browser，含 Load/Reset、`SimulationManager` 物理回调、`BaseSampleUITemplate`）、C++ 扩展（Carbonite 插件 + pybind11 绑定）、OmniGraph 扩展（`.ogn` 定义 C++/Python 节点）。
- **GUI Extension Template Generator**（Utilities > Generate Extension Templates，提供 Loaded Scenario / Scripting / Configuration Tooling / UI Component Library 四类模板）**自 6.0.0 起已弃用**，官方建议迁移到 CLI 模板。
- 独立（standalone）项目请参考 `standalone_examples` 目录，不走扩展模板。

来源: utilities/templates_index.html、utilities/cli_extension_templates.html、utilities/extension_template_generator.html

## 调试与性能分析

- **Debug Draw**（`isaacsim.util.debug_draw`）：跨帧持久绘制点/线/样条，`_debug_draw.acquire_debug_draw_interface()` 后调用 `draw_points/draw_lines/draw_lines_spline`；
- **VS Code 调试**：`omni.kit.debug.vscode` 扩展，默认 127.0.0.1:3000，VS Code 用 "Python: Attach"；Linux 可直接 F5 调试 standalone 脚本（python 指向 `kit/python/bin/python3`）；容器内用 `./python.sh -m debugpy --wait-for-client --listen 0.0.0.0:5678` 并配置 pathMappings；
- **Tracy**：`omni.kit.profiler.tracy` 扩展（GUI 里 Launch and Connect），standalone 用 `SimulationApp({"profiler_backend": ["tracy"]})` + `--enable omni.kit.profiler.tracy`；物理瓶颈表现为 PhysXUpdateNonRender 下的 "Thread waiting"，渲染瓶颈为 `waitUntilDone` CPU 等待；Python 侧 `@carb.profiler.profile` + `CARB_PROFILING_PYTHON=1`，C++ 用 `CARB_PROFILE_ZONE`；
- **扩展安装/更新**：Window > Extensions，可加本地搜索路径与自定义 registry，搜索后 INSTALL/UPDATE，部分扩展更新后需重启。

来源: utilities/debugging/ext_isaacsim_util_debug_draw.html、tutorial_advanced_python_debugging.html、profiling_performance.html、utilities/updating_extensions.html

## 约定与渲染模式

- **约定**（与 5.1 一致）：SI 单位（米/千克/秒），物理步长默认 1/60 s；世界坐标右手系 +X 前、+Z 上；相机三套坐标——世界（+X 前 +Z 上）、USD（+Y 上 −Z 前）、ROS（−Y 上 +Z 前），Isaac→ROS 相机绕 X 轴转 180°；四元数：Isaac Sim Core 与 USD 用 (w,x,y,z)，PhysX/Dynamic Control 用 (x,y,z,w)；角度：Core/PhysX 弧度、USD 度；图像原点在左上角。
- **Rendering Modes（6.0 新增文档页）**：三种模式——**RTX Real-Time 2.0**（新默认，路径追踪 + DLSS 神经渲染，兼顾保真与实时）、**RTX Interactive (Path Tracing)**（累积采样，最高质量、最慢）、**RTX Minimal**（禁用间接光、只用首个远光源，面向 training-in-the-loop 吞吐）。切换：`SimulationApp({"renderer": "RealTimePathTracing"})`、`carb.settings` 的 `/rtx/rendermode`，或视口菜单。

来源: reference_material/reference_conventions.html、reference_material/rendering_modes.html

## 性能优化手册与基准要点

- 物理：调 `SimulationManager.set_physics_dt()`，启用 GPU dynamics；`--/persistent/simulation/minFrameRate` 是性能-精度旋钮；碰撞体性能排序：基本体（盒/球/胶囊/平面）> 凸包 > 复杂网格；关自碰撞、去冗余碰撞体、Merge Mesh、instancing；
- 渲染：默认已切 RTX Real-Time 2.0（相对 5.1 的变化），吞吐场景用 RTX Minimal；`--/app/renderer/skipMaterialLoading=true`、DLSS Performance 模式；headless 下 `disable_viewport_updates=True`；CPU 线程约 32 为宜；
- 多 GPU：GPU 数与相机数匹配收益最佳，物理仅用单 GPU；
- 基准：`isaacsim.benchmark.services` 扩展 + `standalone_examples/benchmarks` 脚本，覆盖启动/场景加载、物理吞吐、多相机渲染、ROS 2 桥、RTX LiDAR 发布、SDG 标注吞吐；指标含 Mean FPS、Real Time Factor、内存/显存、帧时间分解。

来源: reference_material/sim_performance_optimization_handbook.html、reference_material/benchmarks.html

## OpenUSD 与 Robot/Sensor Schema

- **基础**：open_usd.html 讲 prim/组合/图层/材质绑定等；intro_to_usd.html 讲保存/加载（Save Flattened、Collect Assets、default prim 控制引用导出）；usd_tools.html 介绍 USD Paths（路径查找替换）、Variant Presenter/Editor 三个 GUI 工具；
- **Building C++ USD Plugins（新增文档页）**：面向 standalone 安装版编译 ABI 兼容的 USD 插件（自定义 schema、file format、resolver）；要求 C++17、CMake ≥3.20、packman；须链接 `extscache/omni.usd.libs-*/bin/` 的运行时库而非 packman 开发库，并加 `-Wl,--no-as-needed`；注意 **6.0 的 OpenUSD 升到 25.11（5.1 为 24.05）**，插件须匹配版本；
- **Robot Schema**（生产可用，6.0 全部内置机器人已应用）：`isaacsim.robot.schema`（+UI 的 Robot Inspector）；`IsaacRobotAPI`（根 prim，有序 link/joint 列表）、`IsaacLinkAPI`、`IsaacJointAPI`（DOF 排序）、`IsaacSiteAPI`、`IsaacAttachmentPointAPI`、`IsaacNamedPose`、`IsaacSurfaceGripper`；内置纯 Python FK/雅可比/LM-IK；
- **Sensor Schema（新增）**：定义于 `isaacsim.robot.schema` 的 `SensorSchema.usda`，基类 `IsaacBaseSensor`，含 `IsaacContactSensor`（接触力阈值/半径/可视化）、`IsaacImuSensor`（线加速度/角速度/姿态）、`IsaacRaycastSensor`（逐射线原点偏移/方向/时间偏移）；`IsaacLightBeamSensor` 已弃用，改用 raycast。

来源: omniverse_usd/open_usd.html、intro_to_usd.html、usd_tools.html、building_cpp_usd_plugins.html、robot_schema.html、sensor_schema.html

## API 参考、数据收集与许可证

- **Python API**：reference_python_api.html 只是跳转页，实际 API 文档在独立站点（`py/index.html`），6.0 命名空间为 `isaacsim.*`；
- **数据收集**：匿名遥测（系统配置、会话/功能使用、崩溃日志），不含 PII；容器用 `OMNI_TELEMETRY_DISABLE_ANONYMOUS_DATA=1`、kit 文件 `enableAnonymousData = false` 关闭；
- **许可证**：文档列出多份协议——NVIDIA Isaac Sim License（主许可）、Additional Software and Materials License（第三方组件）、流客户端与 Omniverse 平台单独条款，另有 License FAQ；商用/再分发前应查阅原文。

来源: reference_python_api.html、common/data-collection.html、common/legal.html

## 相对 5.1 的主要变化汇总

移除 isaacsim.asset.browser（改用 Content Browser + SimReady Content Browser，新增其搜索扩展教程）；新增 CLI Extension Templates 并弃用 GUI 模板生成器；新增 Rendering Modes 页且默认渲染器改为 RTX Real-Time 2.0；新增 Building C++ USD Plugins 与 Sensor Schema 文档；OpenUSD 24.05 → 25.11；Robot Schema 转为生产可用并默认应用于内置机器人。
