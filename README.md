# isaacsim-qa-skill — Isaac Sim 问答 Skill（多版本）

[Claude Code](https://claude.com/claude-code) 个人 skill：回答 NVIDIA Isaac Sim 使用过程中的各类问题（安装、URDF 导入、机器人搭建、ROS 2、Isaac Lab、Replicator 合成数据、传感器、物理、性能、版本迁移等）。

## 版本说明

**每个 Isaac Sim 版本对应一个独立 skill**。本仓库 `main` 分支始终存放**最新版本**，旧版本通过 git tag 留档：

| Isaac Sim 版本 | skill 名称 | 获取方式 |
|---|---|---|
| **6.0.1**（当前 main） | `isaacsim-601-qa` | `git clone` 默认即是 |
| 5.1.0 | `isaacsim-qa` | `git checkout v5.1.0` |

两个版本的 skill 可同时安装、互不影响：6.0.1 的 skill 声明"未指明版本的问题按最新版处理"，5.1 的 skill 只在明确提及 5.1 时触发。

## 工作方式

两级查询策略：

1. **本地优先**：先按主题速查表读取 `references/` 下的中文文档摘要（约 250 页官方文档的整理稿，保留关键命令、API 名、参数值与原文出处 URL；6.0.1 版含 8 篇 5.x→6.0 迁移指南详解）；
2. **网络兜底**：本地摘要不足以解决时（缺具体步骤、API 细节、报错匹配），自动带版本号与当前年份进行网络搜索，优先官方文档、NVIDIA 开发者论坛与 isaac-sim GitHub（6.0 起 Isaac Sim 开源，可直接核对源码）。

答案末尾会标注来源（本地摘要 / 官方原页 / 论坛）。

## 安装

安装最新版（6.0.1）：

```bash
git clone https://github.com/ShenBW/isaacsim-qa-skill.git
mkdir -p ~/.claude/skills
cp -r isaacsim-qa-skill ~/.claude/skills/isaacsim-601-qa
```

如需 5.1 版本（可与 6.0.1 并存）：

```bash
cd isaacsim-qa-skill && git checkout v5.1.0
cp -r . ~/.claude/skills/isaacsim-qa && git checkout main
```

安装后在 Claude Code 中提问 Isaac Sim 相关问题会自动触发，也可用 `/isaacsim-601-qa <问题>` 手动调用。

## 目录结构

```
isaacsim-qa-skill/
├── SKILL.md          # skill 主文件（查询流程与主题索引）
└── references/       # Isaac Sim 官方文档中文摘要（11 个文件）
    ├── README.md     # 摘要总索引、版本核心变化速览与关键速查
    ├── 01-overview-installation-migration.md
    ├── ...
    └── 10-utilities-usd-reference.md
```

## 来源与许可

- skill 本身（SKILL.md、README、仓库结构）以 [MIT License](LICENSE) 发布。
- `references/` 内容整理自 [NVIDIA Isaac Sim 官方文档](https://docs.isaacsim.omniverse.nvidia.com/)（6.0.1 版整理于 2026-07），各条目均附原文链接；文档版权归 NVIDIA 所有，摘要仅为学习用途，**不在 MIT 许可范围内**（详见 LICENSE 附注）。
