# Isaac Lab 与应用模板

## Isaac Lab 与 Isaac Sim 的关系及定位

Isaac Lab 是构建于 Isaac Sim 之上的**官方机器人学习框架**，专注强化学习与模仿学习。特性：模块化配置驱动的环境系统、预制学习环境套件、多 RL/IL 库兼容、支持手柄/键盘采集演示数据、自定义执行器模型用于 sim-to-real。旧框架 IsaacGymEnvs、OmniIsaacGymEnvs、Orbit 已弃用，官方提供迁移指南。教程覆盖：机器人导入（URDF/MJCF）、策略部署、ROS 2 集成、Cloner 与 Instanceable 资产等。

来源: https://docs.isaacsim.omniverse.nvidia.com/5.1.0/isaac_lab_tutorials/index.html

## 策略部署流程（RL 策略在 Isaac Sim 中部署）

1. **训练**: `./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task Isaac-Velocity-Flat-H1-v0 --headless`
2. **导出**: 运行 `play.py` 后在 `exported/` 生成 `policy_*.pt`；`logs/rsl_rl/<task>/<time>/params/` 下有 `agent.yaml`（网络参数）与 `env.yaml`（环境/机器人配置，如 `sim.dt: 0.005` 即 200Hz、初始状态、执行器刚度阻尼、观测与动作缩放）。
3. **编写策略控制器类**: 构造函数加载 USD 并创建 Articulation；`initialize()` 按 env.yaml 设置控制模式与关节增益；重写 `_compute_observation()` 严格按训练时观测顺序构造（H1 为 69 维：基座线速度/角速度/重力向量/命令/关节位置偏差/关节速度/上一步动作）；`forward()` 每物理步调用，按 decimation 周期推理，再 `robot.apply_action(ArticulationAction(joint_positions=default_pos + action*action_scale))`。
4. **注意**: 观测与动作都要乘缩放系数；勿用 `set_joint_position()` 直接传送关节；位置策略配扭矩控制机器人时用 `actuator_network.compute_torques()` 转换后 `set_joint_efforts()`。
5. **调试清单**: 先在 Isaac Lab 用 play.py 验证策略；比对 `dof_names` 关节顺序（顺序错→摔倒）；检查默认关节位置、刚度/阻尼（过高反应迟钝、过低震颤）、物理 Hz 是否等于 1/dt。

来源: https://docs.isaacsim.omniverse.nvidia.com/5.1.0/isaac_lab_tutorials/tutorial_policy_deployment.html

## 通过 ROS 2 运行 RL 策略

以 Unitree H1 人形机器人为例，ROS 2 作为策略与仿真器间的桥梁（需启用 `isaacsim.ros2.bridge` 扩展，骨盆装 IMU）。三个 ActionGraph：**ROS_Imu**（发布 `/imu`）、**ROS_Joint_States**（发布 `/joint_states`、订阅 `/joint_command` 并经 Articulation Controller 执行）、**ROS_Clock**（发布 `/clock` 同步时间）。物理 200 步/秒，单机器人禁用 GPU dynamics，broadphase 用 MBP。启动顺序：

1. `ros2 launch h1_fullbody_controller h1_fullbody_controller.launch.py`
2. Isaac Sim 播放场景
3. `ros2 run teleop_twist_keyboard teleop_twist_keyboard` 发 `/cmd_vel`（i 前进、j/l 转向、k 停止）

限制: 线/角速度超 0.75 会摔倒；平地策略不支持后退与侧移。成品资产: `Samples/ROS2/Robots/h1_ROS.usd`。

来源: https://docs.isaacsim.omniverse.nvidia.com/5.1.0/ros2_tutorials/tutorial_ros2_rl_controller.html

## Cloner 并行克隆机制

用于 RL 的向量化环境复制。**Cloner**: `generate_paths()` 生成目标路径，`clone()` 传入源 prim、目标路径及位置/朝向数组；**GridCloner**: 仅需 spacing 参数自动网格排布。克隆后用 `XFormPrimView("/World/Cube_*")` 通配符批量访问变换。`replicate_physics=True` 直接复制 PhysX 属性加速物理解析，但限制运行时修改属性；`copy_from_source=False`（默认）用 USD Inherits 更快但与源联动，`True` 为独立副本。

来源: https://docs.isaacsim.omniverse.nvidia.com/5.1.0/isaac_lab_tutorials/tutorial_cloner.html

## Instanceable 资产优化

利用 USD Scenegraph Instancing 让多个机器人副本共享同一网格，大幅降低大规模并行训练的内存占用。限制: USD 禁止修改实例化 prim 后代属性，故每个网格需父 Xform 承载引用与 instanceable 标记（Robot → Collisions → Sphere_Xform → Sphere）；材质/物理材质等关系应设在父 Xform 上。创建方式: URDF/MJCF 导入器勾选 "Create Instanceable Asset"（生成主 USD + 网格 USD）；或用工具函数 `create_parent_xforms()`、`convert_asset_instanceable()` 转换现有资产。

来源: https://docs.isaacsim.omniverse.nvidia.com/5.1.0/isaac_lab_tutorials/tutorial_instanceable_assets.html

## 常见问题

- **安装**: Python 3.9+；`pip install -e .` 装齐依赖；CUDA 版本需匹配。
- **性能**: 训练慢→减少环境数/场景复杂度；内存不足→调批大小；帧率低→减传感器或降分辨率。
- **环境配置**: 机器人初始化失败查 URDF/USD 路径；奖励项与观测须在配置中正确定义。
- **策略部署**: 部署观测空间必须与训练一致；性能下降需核对仿真参数；加载失败查文件格式。

来源: https://docs.isaacsim.omniverse.nvidia.com/5.1.0/isaac_lab_tutorials/troubleshooting.html

## Kit App Template 应用模板

提供轻量方式安装与自定义 Isaac Sim：直接从扩展注册表拉取扩展构建应用，无需完整二进制发行版。含两类模板——镜像官方 Isaac Sim 二进制配置的**默认应用**，以及可按需修改扩展组合的**可定制模板**。适合需要轻量或定制配置的开发者，是主应用的补充而非替代。仓库: https://github.com/isaac-sim/isaacsim-app-template

来源: https://docs.isaacsim.omniverse.nvidia.com/5.1.0/app_template/index.html
