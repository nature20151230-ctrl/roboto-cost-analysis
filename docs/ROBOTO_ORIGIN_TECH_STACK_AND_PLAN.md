# ROBOTO_ORIGIN 技术栈梳理与实施计划

> 社区调研文档，非官方发布。数据截止 2026-11-19。
> 适用对象：准备复刻 Roboto Origin V2.0（RPO）的 DIY 玩家与研究者。
> 配套文档：`ROBOTO_ORIGIN_V2_COST_ANALYSIS.md`（采购清单与降本方案）。

---

## 0. 先说一个版本陷阱

聚合仓库 `roboto_origin/README_cn.md` 顶部的徽章标注的是 **Isaac Sim 4.5.0 / Isaac Lab 2.1.1 / Python 3.10**，但训练子模块 `roboparty_train/README.md` 的实际要求是 **Isaac Lab 3.0.0 / Isaac Sim 6.2.0 / Python 3.12**。徽章已过时，以子模块 README 为准，照旧徽章配环境会直接失败。

另一个易错点：`roboto_origin` 只是**快照聚合仓库**，模块级问题和代码贡献要去对应的 8 个子仓库；聚合仓库的 Issues/PR 仅处理快照同步、根目录文档与元数据。

---

## 1. 技术栈全景

整条链路是「建模 → 训练 → 导出 → 部署」，底层由硬件与固件支撑：

```
建模层 ──► 训练层 ──► 转换层 ──► 部署层
  │           │          │          │
  └───────────┴──────────┴──────────┘
               │
        硬件与固件底座
```

| 层 | 对应模块 | 核心内容 |
|---|---|---|
| 建模层 | `rpo_description` | URDF / MJCF、link-joint 树、视觉与碰撞网格、质量惯量参数 |
| 训练层 | `roboparty_train` | Isaac Lab 3.0 + Isaac Sim 6.2(PhysX) 或 MJWarp/Newton 1.6.1、RSL-RL 5.5.1 fork、4096 并行、域随机化、AMP / BeyondMimic / Parkour |
| 转换层 | train 的 export | ONNX 策略导出、Sim2Sim(MuJoCo) 验证、checkpoint 管理 |
| 部署层 | `roboparty_deploy` | ROS 2 Humble/Jazzy、C++17、ONNX Runtime 推理、观测组装、电机/IMU/深度相机驱动 |
| 固件底座 | `roboparty_firmware` | USB2CAN 固件、RK 板卡镜像构建、BMS 守护进程、systemd |
| 硬件底座 | `rpo_hardware` | CNC 图纸、PCB 打样、装配、CAN 接线、零位标定 |

实验性模块（非跑通必需）：`roboparty_navigation`（导航定位）、`roboparty_xr_teleop`（XR 遥操作）。
`rpo_appearance` 官方标注仅作静态展示，不建议打印。

---

## 2. 各模块技术细节（核实自代码）

### 2.1 训练层 roboparty_train

- **环境**：Python 3.12、Ubuntu 22.04 x64；用 `uv sync --locked` 管理两套独立环境——MJWarp 默认环境，PhysX 用 `UV_PROJECT_ENVIRONMENT=.venv-physx uv sync --locked --no-default-groups --extra physx`。
- **物理后端**：默认 MJWarp（`physics=newton_mjwarp`，Newton 1.6.1）；可选 Isaac Sim PhysX（`physics=isaacsim_physx`）。
- **算法**：RSL-RL 5.5.1 fork，支持 AMP、BeyondMimic、Parkour。
- **任务**：`RPO-` 与 `RP1-` 前缀，验证命令 `isaaclab list_envs --keyword RP`。
- **训练规模**：`--num_envs=4096`，默认 TensorBoard 日志，可选 `--extra wandb`。
- **数据**：动捕数据经 HHTools 重定向，`convert_motion.py` 转训练数据——AMP 120Hz（.pkl）、BeyondMimic 50Hz（.npz）。
- **导出**：play 时导出到 `exported/`；Parkour 额外导出 `0-depth_encoder.onnx` 与 `actor.onnx`，部署时必须连同 `.onnx.data` 一起保留。

### 2.2 部署层 roboparty_deploy

- **框架**：ROS 2 Humble 或 Jazzy，C++17，x86_64 / aarch64 双平台。
- **目录职责**：`src/motors`（DM/EVO/LRO/RP/XYN 电机驱动）、`src/imu`（HIPNUC/MCT7123）、`src/inference`（机器人接口 + ONNX 推理 + 观测组装）、`src/camera`（RealSense 深度）、`tools`（环境检测与启动）、`scripts`（Python SDK 与零位工具）。
- **控制器**：Orange Pi 5 Plus（Ubuntu 22.04，内核 5.10）或 RDK X5（Ubuntu 22.04，内核 6.1.83，官方镜像预装 RT 内核）。
- **依赖**：`ccache libfmt-dev libspdlog-dev libeigen3-dev screen zlib1g-dev`；手柄需 `ros-<distro>-joy`；脚本需 `python3-yaml python3-numpy`。
- **实时性**：`/etc/security/limits.conf` 配置 `rtprio 98` 与 `memlock unlimited`，重启后 `ulimit -r` 应输出 98。Orange Pi 需自行安装 5.10 RT 内核（deb 包在 `assets/`），RDK X5 跳过。
- **启动**：`./tools/start_robot.sh` 自动 colcon 编译并在两个 screen 会话（inference_session / joy_session）中启动；策略通过 `--policy` 选择：`default` / `amp` / `beyondmimic` / `getup` / `interrupt` / `parkour`。Parkour 需另开 `./tools/start_camera.sh --policy parkour`。

### 2.3 固件层 roboparty_firmware

三个子模块，镜像文件在各子模块的 release 中：

| 子模块 | 用途 |
|---|---|
| `roboto_usb2can` | USB 转 CAN 适配器固件 |
| `orangepi-build` | RK 系列板卡定制 Linux 镜像构建（含嵌套的 bms_daemon） |
| `x5-rdk-gen` | X5 平台固件镜像生成 |

克隆需 `--recursive`，或克隆后 `git submodule update --init --recursive`。

---

## 3. 关键实操参数（从部署文档提取）

| 项目 | 参数 |
|---|---|
| 电机型号与数量 | 23 个达妙 DM 关节：14×DM4340P + 9×DM10010L |
| CAN 波特率 | 1 Mbps，txqueuelen 1000 |
| 4 路 CAN 映射 | can0 左腿 / can1 右腿+腰 / can2 左手 / can3 右手 |
| USB 建议 | USB2CAN 接 USB 3.0 口；IMU 与手柄用 USB 2.0 |
| IMU | HiPNUC，波特率 921600，输出 500Hz（必须 > 200Hz） |
| 电机 ID | 用达妙配置工具设定，参照官方安装手册 |
| 零位标定 | 软件零位偏移见 `robot.yaml`；用腰部限位块标定时保留 `2.093`，用 3D 打印夹具时改为 `0.0` |
| AP 调试 | 可选，默认 SSID `atom` / 密码 `jujujuju`，单无线网卡 AP 模式下外网需用网线 |

---

## 4. 六阶段实施计划

参考投入：每周 10-15 小时，全程约 20 周（5 个月）。原则是**先仿真、后花钱**——前 8 周零硬件成本，先验证工具链能否跑通。

### 阶段 0（第 1-2 周）· 环境与算力

- 准备两台机器：
  - **训练机**：Ubuntu 22.04 x64 + NVIDIA 显卡（Isaac 生态不认 Windows/Mac，显存决定并行环境数）。
  - **控制器**：RDK X5（推荐，预装 RT 内核）或 Orange Pi 5 Plus（需自装 RT 内核）。
- 这是除 5 万 BOM 外最大的隐性成本，先把算力预算算进去。

### 阶段 1（第 3-6 周）· 仿真跑通

1. 克隆 `roboparty_train --recursive`，安装 Isaac Lab 3.0.0 源码到 `../IsaacLab-3.0.0`。
2. `uv sync --locked`，`isaaclab list_envs --keyword RP` 确认 RPO-/RP1- 任务可见。
3. 训练：`python robolab/scripts/rsl_rl/train.py --task=RP1-Flat --viz=none --logger=tensorboard --num_envs=4096`。
4. 回放：`play.py`；再跑 Sim2Sim 到 MuJoCo 验证。
5. 目标：能独立复现训练→导出→回放到真机前的全链路。若此处持续卡住，及时止损。

### 阶段 2（第 7-8 周）· 部署预演

- 在**没有真机**的情况下，把 `roboparty_deploy` colcon 编译通过。
- 读懂 `robot.yaml` 配置字段、手柄服务、启动脚本与 screen 会话管理。
- 真机到手后可直接跳过这段时间。

### 阶段 3（第 9-12 周）· 采购与制造

- 按配套成本报告的**方案 A（省 ¥7,260）或 D（省 ¥10,260）** 下单。
- 电机、CNC 结构件、PCB 打样三条线并行，CNC 周期最长先下单。
- 同步下载 Gerber 打样 RBE_Board_v2.0，用嘉立创报价器试算 CNC 件。

### 阶段 4（第 13-15 周）· 装配与接线

- 机械装配 → CAN 按映射接线 → udev 规则绑定设备到物理 USB 口 → 达妙工具配电机 ID → 零位标定。
- 标定后断电重启电机使零点生效。
- 每一步对照官方安全操作指南。

### 阶段 5（第 16-20 周）· 真机联调

- **先用官方现成策略**（`default` / `amp` / `beyondmimic` / `getup` / `parkour`）让它走起来，不要一上来就重训。
- 分级上电：先离地/吊起保护测试，再带载；确认零位与 CAN 映射无误。
- 最后才考虑重训策略、改 URDF 参数或做动捕模仿。
- 注意：**只换采购渠道无需任何软件改动**；只有改电机型号才需同步改 URDF 惯量/扭矩并重做域随机化，否则 Sim2Real 失败。

---

## 5. 三条核心原则

1. **先仿真、后花钱**：前 8 周零成本验证。这套栈劝退率主要在工具链搭建，不在 RL 算法本身。
2. **别一上来就重训**：官方权重可直接部署，能跑通就是阶段性胜利；重训是进阶目标。
3. **安全红线**：实时内核 + rtprio 98 是控制周期稳定的前提；零位标定错误或跳过安全指南，23 个关节同时上电可能直接掰断结构件。

---

## 6. 数据来源

| 来源 | 用途 |
|---|---|
| `Roboparty/roboto_origin` README_cn.md / README.md | 模块清单、贡献方向、总体说明 |
| `Roboparty/roboparty_train` README.md | 训练栈版本、依赖、命令、数据格式 |
| `Roboparty/roboparty_deploy` README.md + docs/rpo_deploy.md | ROS2 架构、环境、实时内核、接线与零位 |
| `Roboparty/roboparty_firmware` readme_cn.md | 固件子模块与镜像 |
| 官方文档站 roboparty.com/documents/party-os | 研发流程与 Know-How（外部链接） |

---

> **免责声明**：本文档为社区调研，非官方发布。软件版本与命令会随仓库更新变化，动手前请以各子仓库最新 README 为准。机器人属高危设备，操作风险由操作者自行承担。
