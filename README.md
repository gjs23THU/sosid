# SoSID 野火灭火体系场景复现与环境（SoSID Wildfire SoS）

本项目面向 **System of Systems Inverse Design (SoSID)** 野火航空扑救体系场景（上游参考仓库：[ibrahimoguzc/wildfire-rl-new](https://github.com/ibrahimoguzc/wildfire-rl-new)），对标本地/云端 `APS` 项目组织规范，提供结构化场景输入表、复现技术校核笔记及完整参考源码。

## 1. 目录结构

```text
sosid/
├── AGENTS.md                        项目交付与同步约定
├── README.md                        项目说明
├── SoSID_Replication_Notes.md       上游代码库场景、状态/动作与 MoE 校核说明
├── requirements.txt                 云端无头仿真与强化学习依赖清单
├── sosid_inputs/                    结构化标准输入表与原始配置（对标 fang2025_inputs）
│   ├── config.json                  统一场景、RL 接口与 MoE 权重/基准配置
│   ├── scenarios.csv                3 个基准火场场景（Salamis / Pyrenees / Palisades）参数表
│   ├── aircraft_catalog.csv         22 型候选飞机（4 型基准机 + 18 型逆向设计候选机）性能参数表
│   ├── tactics.csv                  24 种离散灭火战术组合（4 × 3 × 2）编码表
│   ├── SoSID_Replication_Notes.md   输入表字段与上游源码映射说明
│   └── raw_configs/                 上游原始 JSON 配置备份（scenarios/ 与 aircraft/）
└── upstream/                        ibrahimoguzc/wildfire-rl-new 完整源码（src/sosid 与 examples/wildfire）
```

> **大型二进制数据说明**：上游仓库的栅格地形数组（`*.npy`）与预训练权重（`*.zip`）通过 Git LFS 管理；完整用户手册 `SoSID-Toolkit-User-Manual.docx` 已归档至同步的 Google Drive `LocalProject/SoSID/` 目录中。

## 2. 云项目与跨端查看方式

- **GitHub 仓库（已绑定 `sosid` 云项目）**：`https://github.com/gjs23THU/sosid`（后续在本云项目新开任何会话均自动克隆至 `/home/antigravity/gjs23THU/sosid`）
- **Google Drive 同步目录**：`我的计算机/LocalProject/SoSID/`（与 `APS/`、`SOAS/` 同级，包含本文档、结构化输入表与手册）
