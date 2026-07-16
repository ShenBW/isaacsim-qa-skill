# 实用工具、USD 与参考资料

## 各类浏览器

Isaac Sim 提供五种浏览器管理资产与场景：Content Browser（项目文件与资产管理）、Isaac Sim Asset Browser [Beta]、NVIDIA Asset Browser（更广泛的 NVIDIA 资产库）、Material Browser（材质资源）、SimReady Explorer（针对仿真优化的高质量资产），均通过菜单或侧面板访问。

来源：utilities/browsers.html

**Content Browser**：经 Window > Browsers > Content 打开，是主要的内容浏览工具，支持目录导航、搜索与加载 Isaac Sim Asset Cards；完整功能参见 Omniverse 官方文档。
来源：utilities/content_browser.html

**Asset Browser [Beta]**：经 Window > Browser 打开，用于浏览/加载 USD 资产。单击打开选项面板，双击打开原文件，拖入视口以 payload 方式加载；提供分类菜单、Load as Reference、Variant 预选、搜索栏等。已知限制：仅搜索当前分类（切到 "All" 才能全局搜）、默认只显示 USD 文件、非 USD 资产需先下载到本地、有缓存机制；多数任务官方仍推荐 Content Browser。
来源：utilities/asset_browser.html

## 扩展开发模板

**模板索引**：可选路径包括——直接修改现有交互示例；Extension Template Generator（面向机器人应用、利用 Isaac Sim 库）；Custom Extensions: C++（通用 C++ 模板）；VS Code 高级模板生成器（支持 C++/Python/OmniGraph/GUI 组合）；独立（standalone）项目可从 `standalone_examples` 文件夹起步。
来源：utilities/templates_index.html

**Extension Template Generator**：经 Utilities > Generate Extension Templates 打开，填写扩展路径、名称、描述后生成，再到 Extensions Manager 中启用（或用 `--enable` 参数）。四种模板：Load Scenario（Load/Reset/Run 三按钮，适合入门）、Scripting（将 Run 实现为脚本框架）、Configuration Tooling（动态查找 Articulation 并用 UI 控制关节）、UI Component Library（演示各 UIElementWrapper 及回调签名）。生成物含 README，建议先读。
来源：utilities/extension_template_generator.html

**自定义 C++ 扩展**：权威入口是 Omniverse Kit C++ Extension Template 文档（kit-extension-template-cpp），涵盖环境搭建、模板结构、构建与部署，适用于性能敏感或复杂功能。
来源：utilities/custom_cpp_extensions.html

## 调试与性能分析

**调试概览**：支持 Debug Drawing 扩展 API（可视化调试）、Omniverse Commands Tool、VS Code Python 调试，以及 Tracy 性能分析，并链接到性能优化手册。
来源：utilities/debugging/index.html

**VS Code Python 调试**三种方式：

1. 独立脚本：用 VS Code 打开 Isaac Sim 目录直接设断点调试，`launch.json` 的 `args` 传参；
2. Docker 容器：容器内安装 `debugpy`，以 `./python.sh -m debugpy --wait-for-client --listen 0.0.0.0:5678` 启动（`--wait-for-client` 使脚本等待调试器接入），VS Code 用 remote attach 并正确配置路径映射；
3. 附加到运行中的 Isaac Sim：启用 `omni.kit.debug.vscode` 扩展，界面显示 Attached/Unattached 状态，host/port 可用 `--/exts/omni.kit.debug.python/host` 类参数配置，须与 launch 配置一致。

来源：utilities/debugging/tutorial_advanced_python_debugging.html

**Tracy 性能分析**：启用 `omni.kit.profiler.tracy` 扩展后经 Profiler 菜单启动/连接（也可直接运行 extscache 下的 Tracy 二进制）。GUI 流程为连接后 Stop 再 Save trace；standalone 流程用 `python.sh 脚本 --enable omni.kit.profiler.tracy`，并在 `SimulationApp` 参数加 `"profiler_backend": ["tracy"]`。自定义 zone：Python 用 `@carb.profiler.profile` 装饰器或 `carb.profiler.begin/end`；C++ 用 `CARB_PROFILE_ZONE("zone title")` 宏。
来源：utilities/debugging/profiling_performance.html

## 扩展更新指南

经 Window > Extensions 添加/更新扩展：可在 Extension Search Paths 用绿色 + 添加本地路径或注册表 URL；搜索到扩展后点 UPDATE 更新。自定义扩展显示在 THIRD PARTY 标签下；部分扩展更新后需重启 Isaac Sim 才能正确加载新版。
来源：utilities/updating_extensions.html

## Python API 文档入口

该页为 API 参考的入口页，提供两个外链：Isaac Sim API（Python API 主参考）与 Omniverse API Documentation（平台级 API）。它位于 Python Scripting and Tutorials 章节内，衔接代码片段与 Core API 教程系列。命名空间组织详见下文"扩展重命名"（`isaacsim.core.api/prims/utils`、`isaacsim.sensors.*` 等分层结构）。
来源：reference_python_api.html

## 术语表精选

- **USD**：Pixar 开发的可扩展开源 3D 场景描述格式；**Stage**：场景中全部资产的层级表示；**Prim**：USD 基本容器对象，可嵌套并携带属性；**Layer**：代表可相互覆盖的独立"opinion"。
- 组合机制：**Reference**（轻量链接，橙色箭头）、**Payload**（可按需加载的引用，蓝色箭头）、**Instance**（轻量但可操作性低的副本，蓝色 "I"）。
- 机器人相关：**Articulation**（带关节的机器人：腿式、机械臂、轮式）、**World**（管理时间事件与物理步进的核心类）、**Scene**（World 内的资产管理类）、**Task**（模块化场景创建与逻辑）、**RMP**（黎曼运动策略，生成平滑避碰轨迹）。
- 数据：**Replicator**（合成数据生成工具）、**Ground Truth**（模拟真实世界的合成标注数据）。

来源：reference_material/reference_glossary.html

## Isaac Sim 约定（单位、坐标系）

- 默认单位：米、千克、秒、牛顿；物理步长默认 1/60 s（可配置）。
- 世界坐标：右手系，**+Z 向上，+X 向前**。
- 四元数顺序：Isaac Sim Core 与 USD 为 (QW, QX, QY, QZ)；PhysX 与 Dynamic Control 为 (QX, QY, QZ, QW)。物理计算内角度为弧度，USD 属性显示为度。
- 相机坐标：+Y 向上、-Z 向前；转 ROS 相机坐标需绕 X 轴旋转 180°。
- 图像像素原点 (0,0) 在左上角。

来源：reference_material/reference_conventions.html

## 性能优化手册要点

**常见瓶颈**：物理计算、场景复杂度、渲染管线、CPU/GPU 资源、传感器处理、内存管理。诊断：Windows 用任务管理器，Linux 用 `nvidia-smi`。

- **物理**：调整物理步长（`world.set_physics_step_size()`，大步长快但精度低）、设置最小仿真帧率、启用 GPU dynamics（GPU 未饱和时提速）。
- **机器人资产**：网格合并减少 mesh 数；场景图实例化（instancing）降内存；用球/圆柱等简单几何体替代网格碰撞体；不需要时禁用自碰撞。
- **渲染**：LOD 与隐藏不可见对象；RTX 实时模式；经 `/rtx/debugMaterialType` 禁用材质、关闭灯光；DLSS 性能模式(0)最快；headless 时 `disable_viewport_updates=True` 跳过视口更新。
- **CPU**：限制线程数（`carb.tasking.plugin/threadCount`、`persistent/physics/numThreads`、`omni.tbb.globalcontrol/maxThreadCount`，建议 ≤32）；Linux 将 CPU governor 设为 performance。
- **GPU/显存**：相机数与 GPU 数匹配最佳，SDG 通常 2 GPU 最优，GPU 物理仅用单卡；降低纹理流预算（默认 60%）、减少渲染像素、禁用多余视口；Linux 裸机可禁用 IOMMU 改善多 GPU 性能。
- **内存**：关闭纹理流可省显存（增内存）；长期运行可实验性调整 `GLIBC_TUNABLES` 分配器参数缓解泄漏。

来源：reference_material/sim_performance_optimization_handbook.html

## 基准测试

分两类 KPI：GPU 无关（包体大小 Windows 7.37GB/Ubuntu 8.17GB、启动时间、Docker 镜像大小）与 GPU 相关（Full Warehouse 场景帧率与加载时间、多机器人物理仿真速度、ROS2 渲染发布性能、SDG 帧率）。测量扩展为 `isaacsim.benchmark.services`；运行方式如 `./python.sh standalone_examples/benchmarks/benchmark_robots_o3dyn.py --num-robots 10 --num-gpus 1`。参考硬件覆盖 RTX 4080 Super/5080、A40、L40、RTX PRO 6000 Blackwell（i9-14900K + 32GB DDR5）。
来源：reference_material/benchmarks.html

## 4.5 扩展重命名背景

Isaac Sim 4.5 将扩展从 `omni.isaac.*` 全面改为 `isaacsim.*`，目的：品牌标准化 + 提高构建自定义 App 的模块化程度。命名从扁平变为分层：`isaacsim.[类别].[子类别].[功能]`，如 `omni.isaac.sensor` 拆为 `isaacsim.sensors.camera/physics/rtx`，`omni.isaac.core` 拆为 `isaacsim.core.api/prims/utils`。影响三方面：设置路径（如 `/exts/isaacsim.ros2.bridge/`）、OmniGraph 节点类型、Python import。`isaacsim.core.deprecation_manager` 提供向后兼容（自动映射旧设置、打开场景时更新节点类型），但 Python import 需手动改，场景需重新保存。
来源：overview/extensions_renaming.html

## OpenUSD 基础与 USD 使用

**OpenUSD**：Isaac Sim 描述机器人与环境的语言，支持多应用间 3D 内容互换。格式有文本 `.usda`（可直接编辑）与二进制 `.usd`（性能优）。核心概念：Prim（带类型的基本元素，如 Xform、Sphere）、组合/嵌套层级、属性继承、非破坏性 Layer 协作。默认单位米/秒/千克，角度为度，Metrics Assembler 自动转换不同标准的资产。官方 OpenUSD 教程可在 Script Editor 中运行。
来源：omniverse_usd/open_usd.html

**USD 使用指南**：保存方式包括普通保存、Save Flattened As（合并所有组件）、存为 .usda、Collect Assets（将外部依赖收集到单一文件夹）；加载可直接打开（可编辑）或 Add Reference（只读引用）。构建可复用资产的最佳实践：用 Xform 建层级、语义化命名（body、wheel_left）、设置 Default Prim 控制被引用时的导出内容、unparent 移除不需要引用的元素（如 PhysicsScene）。
来源：omniverse_usd/intro_to_usd.html

**USD 工具**：USD Paths（编辑/查找替换失效的资产路径引用）、Variant Presenter（查看和管理 variant 及 variant 组）、Variant Editor（添加、删除、修改 variant），详情均链接至 Omniverse 扩展文档。
来源：omniverse_usd/usd_tools.html

## Robot Schema

对 OpenUSD 的实验性扩展，基于 USD Common 与 Physics Schema 标准化描述机器人运动学树。四个核心 API：**IsaacRobotAPI**（根节点：描述、命名空间、从 base link 起的有序 link 列表、joint 列表）、**IsaacLinkAPI**（机器人本体，可用于刚体/形变体，支持名称覆盖）、**IsaacJointAPI**（关节，含名称覆盖与多自由度 DOF 偏移）、**IsaacReferencePointAPI**（工具/传感器附着点，含描述与前向轴）。应用方式：GUI 属性面板"编辑 API Schema"，或用 `usd.schema.isaac.robot_schema` 模块以代码应用（推荐写入独立 layer）；另提供获取关节/链接、生成并打印机器人树结构的解析工具函数。
来源：omniverse_usd/robot_schema.html

## 数据收集与许可证

**数据收集**：Omniverse 匿名收集系统/硬件信息（OS、CPU、GPU、分辨率）、网络速度、功能使用模式、错误崩溃日志；不收集邮箱姓名等个人身份信息。关闭方式：Kit 容器设 `OMNI_TELEMETRY_DISABLE_ANONYMOUS_DATA=1`，Launcher 1.85+ 在用户设置中关闭，Kit 模板将 `enableAnonymousData` 设为 false；可经 NVIDIA 隐私中心请求查看或删除数据。
来源：common/data-collection.html

**许可证**：该页为索引，含七类文档：Isaac Sim License（主协议）、Additional Software and Materials License、WebRTC Streaming Client License、Omniverse License、Licensing Disclaimer、Other Licenses（第三方）、Redistributable Omniverse Software（再分发条款）。版权归 NVIDIA（2023–2026）；页面注明 5.1.0 已停止支持，修复与新功能仅在新版本提供。
来源：common/legal.html
