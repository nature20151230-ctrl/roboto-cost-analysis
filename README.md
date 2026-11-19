# ROBOTO_ORIGIN V2.0 复刻调研

社区调研结果，非 RoboParty 官方发布。数据截止 2026-11-19。

包含两条主线：**成本**（采购清单与降本方案）和**技术栈**（从仿真到真机的技能图谱与实施计划）。

## 内容

### 成本线

- `docs/ROBOTO_ORIGIN_V2_COST_ANALYSIS.md` — 完整调研报告（采购清单、方案对比、风险提示、行动清单）
- `docs/Roboto_Origin_V2.0_采购清单.xlsx` — 89 项料号逐项采购表，含淘宝链接、单价来源标注、成本汇总
- `docs/Roboto_Origin_V2.0_省钱方案对比.xlsx` — 6 档降本方案、电机价格证据、风险提示、行动清单

### 技术栈线

- `docs/ROBOTO_ORIGIN_TECH_STACK_AND_PLAN.md` — 技术栈分层梳理（建模→训练→导出→部署→硬件固件）、核实过的依赖版本、关键实操参数、六阶段实施计划

## 关键结论

**成本**

- V2.0 到手价估算 **约 ¥50,654**，与官方 V1.0 BOM 的 ¥49,743 基本吻合
- 执行器（¥31,337）+ 结构件（¥15,600）占 92.7% 成本
- 最大省钱点：DM10010L 官方 BOM 价 ¥1,989/台，渠道折扣价约 ¥1,199/台，9 台可省 ¥7,110
- 推荐方案 A（换采购渠道，省 ¥7,260）或方案 D（A + 非承力件 3D 打印，省 ¥10,260）

**技术栈**

- 聚合仓库 README 徽章标注的 Isaac Sim 4.5 / Lab 2.1 已过时，以训练子模块为准：**Isaac Lab 3.0 + Isaac Sim 6.2 + Python 3.12**
- 按每周 10-15 小时估算全程约 20 周；前 8 周零硬件成本，先仿真后采购
- 除 5 万 BOM 外的隐性成本：训练机需 Ubuntu 22.04 + NVIDIA 显卡（Isaac 生态不认 Windows/Mac）

## 来源

基于 [Roboparty/roboto_origin](https://github.com/Roboparty/roboto_origin) 及其 8 个子仓库、[Roboparty/rpo_hardware](https://github.com/Roboparty/rpo_hardware) 公开资料整理。
