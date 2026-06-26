# e-puck_line 循线控制器

## 概述

Webots 中 e-puck 机器人的行为控制程序，基于 **包容架构（Subsumption Architecture, Brooks 1986）** 实现黑线循迹、避障绕行和自动回线。

代码路径：`D:\666WorkSpace\Webots2021a\2\controllers\go_foreward\go_foreward.c`

## 硬件配置

| 外设 | 数量 | 标识 | 用途 |
|------|------|------|------|
| IR 红外距离传感器 | 8 | ps0~ps7 | 避障检测 |
| 地面灰度传感器 | 3 | gs0~gs2 | 黑线检测 |
| LED | 8 | led0~led7 | 状态指示 |
| 轮式电机 | 2 | left/right wheel motor | 差速驱动 |

### 传感器布局

```
        前方
    PS0(R)    PS7(L)
    PS1(R)    PS6(L)
    PS2(R)    PS5(L)
  PS3(RR)  PS4(LR)
        后方
```

- 地面传感器：左(gs0)、中(gs1)、右(gs2)

## 包容架构（行为模块）

行为按优先级从低到高排列，高层激活时抑制低层输出。

### LFM — 循线模块（最低优先级）

基于 Braitenberg 车的差速控制，作为默认行为持续运行。

```
DeltaS = gs_value[GS_RIGHT] - gs_value[GS_LEFT]
Left  = LFM_FORWARD_SPEED - LFM_K_GS_SPEED * DeltaS
Right = LFM_FORWARD_SPEED + LFM_K_GS_SPEED * DeltaS
```

- `LFM_FORWARD_SPEED = 200`
- `LFM_K_GS_SPEED = 0.4`

### OAM — 避障模块

检测前方障碍物并转向避开。

1. 累加前方红外传感器值（左右各自求和）
2. 最大值超过 `OAM_OBST_THRESHOLD(100)` 则 `oam_active = TRUE`
3. 比较左右激活强度，确定障碍物侧别 `oam_side`
4. 加权差速转向（权重随传感器角度变化）：
   - 侧方 90°: ×0.2
   - 侧前方 45°: ×0.9
   - 正前方 0°: ×1.2
5. 差速限幅 ±600，防转向过猛
6. `oam_reset` 可复位模块

### OFM — 沿障行走模块

产生沿指定侧转向的倾向，与 OAM 竞争时可实现沿障碍物边缘行走。

| side | 左轮 | 右轮 | 效果 |
|------|------|------|------|
| LEFT | -150 | +150 | 左转 |
| RIGHT | +150 | -150 | 右转 |
| NO_SIDE | 0 | 0 | 停止 |

### LLM — 离线检测模块

监控机器人偏离黑线的时刻。检测 `side` 信号上升沿（-1 → 0/1）激活。

> ⚠️ 半成品框架，需要学生补充完整，当前仅检测离线事件，无实际控制输出。

### LEM — 重入线模块（最高优先级）

四状态有限状态机，在绕开障碍物后引导机器人重新回到黑线。

```
状态机流转:

                    ┌─────────────────────────────────────┐
                    │                                     │
                    v                                     │
  STANDBY → LOOKING_FOR_LINE → LINE_DETECTED → ON_LINE ──┘
  (待机)       (寻线)             (已检测)       (已回线)
```

| 状态 | 条件 | 动作 |
|------|------|------|
| STANDBY | 默认 | `lem_active = FALSE` |
| LOOKING_FOR_LINE | `gs_value[GS_Side] < 500` | 转 LINE_DETECTED，记录对侧黑白状态 |
| LINE_DETECTED | 对侧传感器黑→白下降沿 | 转 ON_LINE |
| ON_LINE | 进入即执行 | `oam_reset = TRUE`，返回 STANDBY |

- 根据障碍物侧别确定回线方向（左障右回，右障左回）
- `lem_black_counter` 计数连续检测到黑色的次数，用于判断黑线宽度

## 主循环流程

```
初始化:
  ├── wb_robot_init()
  ├── 获取 LED / 距离传感器 / 地面传感器 / 电机句柄
  ├── 启用所有传感器 (TIME_STEP)
  └── 电机设为速度模式 (位置=INFINITY)

主循环 (每 32ms):
  1. wb_robot_step(32ms)
  2. 检测仿真/真实模式切换 → 复位所有模块变量 + 切换偏移量
  3. 读取传感器值（距离传感器减去偏移量，负值截零）
  4. speed[LEFT] = speed[RIGHT] = 0
  5. 包容架构调度:
     ┌─────────────────────────────────────────────────┐
     │ 当前: 仅 LFM  → speed[] = lfm_speed[]           │
     │ 扩展: OAM/OFM 仲裁逻辑 + LEM 状态机              │
     └─────────────────────────────────────────────────┘
  6. 设置电机速度 (×0.00628 转换为 Webots 单位)
```

## 参数速查

| 宏 | 值 | 所属模块 | 说明 |
|----|----|---------|------|
| TIME_STEP | 32 ms | 全局 | 控制周期 |
| LFM_FORWARD_SPEED | 200 | LFM | 循线基础速度 |
| LFM_K_GS_SPEED | 0.4 | LFM | 循线转向增益系数 |
| OAM_OBST_THRESHOLD | 100 | OAM | 障碍物检测阈值 |
| OAM_FORWARD_SPEED | 150 | OAM | 避障基础速度 |
| OAM_K_PS_90 | 0.2 | OAM | 侧方传感器权重 |
| OAM_K_PS_45 | 0.9 | OAM | 侧前传感器权重 |
| OAM_K_PS_00 | 1.2 | OAM | 前方传感器权重 |
| OAM_K_MAX_DELTAS | 600 | OAM | 转向差速限幅 |
| OFM_DELTA_SPEED | 150 | OFM | 沿障转向差速 |
| LEM_FORWARD_SPEED | 100 | LEM | 回线基础速度 |
| LEM_K_GS_SPEED | 0.8 | LEM | 回线转向增益 |
| LEM_THRESHOLD | 500 | LEM | 黑白判断阈值 |
| LLM_THRESHOLD | 800 | LLM | 离线判定阈值 |
| GS_WHITE | 900 | 全局 | 白色地面参考值 |
| PS_OFFSET_SIMULATION | 300 | 全局 | 仿真模式距离偏移 |
| PS_OFFSET_REALITY | 各异 | 全局 | 真实模式距离偏移 |

## 传感器偏移量（真实模式）

| 传感器 | 偏移量 |
|--------|--------|
| PS0 (右前0°) | 480 |
| PS1 (右前45°) | 170 |
| PS2 (右90°) | 320 |
| PS3 (右后) | 500 |
| PS4 (左后) | 600 |
| PS5 (左90°) | 680 |
| PS6 (左前45°) | 210 |
| PS7 (左前0°) | 640 |

## 实验扩展方向

1. **避障循线**: 在 `*** START OF SUBSUMPTION ARCHITECTURE ***` 段加入 OAM 输出仲裁
2. **绕障回线**: 补充 LLM 逻辑 + 加入 LEM 状态机调度
3. **沿障行走**: OFM 与 OAM 竞争实现平滑绕障
4. **参数整定**: 根据实际机器人/环境调整各模块阈值和增益

## 相关文件

- [[e-puck_line包容架构.canvas]] — 架构可视化 Canvas
- [[Webots节点继承关系]] — Webots 场景图节点体系
- `D:\666WorkSpace\Webots2021a\2\controllers\go_foreward\go_foreward.c`
