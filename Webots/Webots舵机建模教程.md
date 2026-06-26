# Webots 舵机建模教程

## 一、舵机基础

### 1.1 什么是舵机

舵机（Servo/Servo Motor）是一种**位置（角度）可精确控制的电机**，广泛用于机器人关节、机械臂、云台等场合。核心特性：

- **位置闭环**：内部有电位计/编码器反馈，可将输出轴锁定在目标角度
- **PWM 控制**：标准舵机通过 50Hz PWM 的脉宽（1~2ms）控制角度
- **三线制**：电源 (VCC)、地 (GND)、信号线 (PWM)

### 1.2 常见型号参数

| 型号 | 扭矩 | 速度 (60°) | 工作电压 | 重量 | 应用 |
|------|------|-----------|---------|------|------|
| SG90 | 0.18 N·m | 0.12s | 4.8V | 9g | 小车云台、小机械臂 |
| MG996R | 1.0 N·m | 0.17s | 6V | 55g | 机器人关节 |
| MG995 | 1.2 N·m | 0.15s | 6V | 55g | 大扭矩场景 |
| AX-12A | 1.5 N·m | 0.14s | 12V | 55g | 总线舵机 (串口) |

### 1.3 在仿真中的意义

Webots 中不需要模拟 PWM 信号，而是通过 `RotationalMotor` 节点直接提供**位置控制接口**，仿真器内部求解物理方程得到实际运动。

## 二、Webots 节点体系

### 2.1 关节类继承关系

```
Device (抽象)
├── Motor (抽象)
│   ├── LinearMotor      # 直线运动舵机
│   └── RotationalMotor  # 旋转舵机 ◄── 本章重点
├── Brake                # 制动器
└── PositionSensor       # 位置传感器 ◄── 配合舵机使用

Joint (抽象)
├── HingeJoint           # 旋转关节 ◄── 配合 RotationalMotor
│   ├── Hinge2Joint      # 双轴旋转关节 (万向节)
│   └── BallJoint        # 球关节 (3自由度)
└── SliderJoint          # 滑动关节 ◄── 配合 LinearMotor
```

### 2.2 舵机建模所需的四个核心节点

```
Solid (基座)
  └── HingeJoint (关节)
        ├── JointParameters (物理参数)
        ├── Device (电机 + 编码器)
        │     ├── RotationalMotor (执行器)
        │     └── PositionSensor (传感器)
        └── EndPoint (终点)
              └── Solid (输出臂 + 负载)
```

## 三、逐节点详解

### 3.1 Solid（刚体基座）

舵机的固定部分（外壳），定义碰撞体和外观。

```
Solid {
  translation 0 0 0          # 位置
  rotation 0 0 1 0           # 朝向
  children [ Shape { ... } ]  # 视觉几何
  boundingObject ...          # 碰撞体
  physics Physics {           # 物理属性
    mass 0.009                # SG90 9g = 0.009kg
    inertiaMatrix [ ... ]     # 惯性张量
  }
}
```

> **坑点**：`mass` 和 `inertiaMatrix` 必须合理设置，否则物理仿真会抖动或穿透。惯性张量可通过 `inertiaMatrix [Ixx 0 0  0 Iyy 0  0 0 Izz]` 设置。

### 3.2 HingeJoint（旋转关节）

定义旋转轴、运动范围和约束。

```
HingeJoint {
  jointParameters HingeJointParameters {
    axis 0 0 1                # 绕 Z 轴旋转
    anchor 0 0 0              # 旋转中心 (相对基座)
    minPosition -1.57         # -90° (弧度)
    maxPosition 1.57          # +90° (弧度)
    dampingConstant 0.05      # 阻尼系数 (防震荡)
    stopErp 0.8               # 硬停止弹性恢复
    stopCfm 0.001             # 硬停止柔度
  }
  device [ ... ]              # 电机 + 传感器
  endPoint Solid { ... }      # 输出端
}
```

**关键字段说明**：

| 字段                        | 含义    | 建议值      | 注意事项               |
| ------------------------- | ----- | -------- | ------------------ |
| `axis`                    | 旋转轴向量 | `0 0 1`  | 必须单位长度，方向遵循右手定则    |
| `anchor`                  | 旋转中心  | 相对基座的偏移  | 通常是舵机输出轴位置         |
| `minPosition/maxPosition` | 角度限位  | 弧度       | SG90 一般为 ±1.57 rad |
| `dampingConstant`         | 阻尼    | 0.01~0.1 | 太小会震荡，太大会卡顿        |
| `stopErp`                 | 限位弹性  | 0.8      | 0=完全软 1=完全硬        |
| `stopCfm`                 | 限位柔度  | 0.001    | 越小限位越硬             |

### 3.3 RotationalMotor（电机）

定义扭矩、速度等驱动特性。

```
RotationalMotor {
  name "shoulder_pitch"       # 唯一名称，控制器通过此名称访问
  maxForce 0.18               # 最大扭矩 [N·m]
  maxVelocity 6.98            # 最大角速度 [rad/s] ≈ 60rpm
  controlPID 10 0 0           # PID 参数 [P, I, D]
}
```

> **关于 controlPID**：默认 `[10, 0, 0]` 即纯 P 控制。如果运动到位后持续震荡，可增大 D 项（如 `[10, 0, 0.05]`）；如果有稳态误差，加 I 项。

**SG90 参数换算**：

```
SG90 速度: 0.12s / 60°
→ 100° / s
→ 100 * π / 180 = 1.745 rad/s
→ 但 Webots maxVelocity 应设大一些 (仿真加速)
```

### 3.4 PositionSensor（位置传感器）

读取实际角度反馈。

```
PositionSensor {
  name "shoulder_pitch_sensor"
}
```

配合控制器 API：

```c
WbDeviceTag sensor = wb_robot_get_device("shoulder_pitch_sensor");
wb_position_sensor_enable(sensor, TIME_STEP);

// 循环中读取
double angle = wb_position_sensor_get_value(sensor);  // 弧度
```

### 3.5 EndPoint（输出端）

舵机臂和负载的刚体，是关节的**输出端**。

```
endPoint Solid {
  translation 0.05 0 0        # 臂长 5cm (相对 anchor)
  rotation 0 0 1 0            # 不旋转
  children [ Shape {          # 舵机臂视觉
    geometry Box { size 0.05 0.008 0.005 }
  } ]
  boundingObject Box { size 0.05 0.008 0.005 }  # 碰撞体
  physics Physics {
    mass 0.005                # 臂重 5g
  }
}
```

> **关键**：`EndPoint.Solid` 的 `translation` 方向决定了舵机臂的初始朝向，其正方向对应 0° 位置。

## 四、完整建模示例

### 4.1 单舵机 (.wbt 片段)

```
# 舵机基座
DEF SERVO_BASE Solid {
  translation 0 0 0
  children [
    Transform {
      translation 0 0.015 0
      children [
        Shape {
          appearance PBRAppearance { baseColor 0.3 0.3 0.3 roughness 0.5 metalness 0.2 }
          geometry Box { size 0.023 0.03 0.012 }
        }
      ]
    }
    # 旋转关节
    HingeJoint {
      jointParameters HingeJointParameters {
        axis 0 0 1
        anchor 0 0 0
        minPosition -1.57
        maxPosition 1.57
        dampingConstant 0.05
      }
      device [
        RotationalMotor {
          name "sv1"
          maxForce 0.18
          maxVelocity 6.98
        }
        PositionSensor {
          name "sv1_sensor"
        }
      ]
      endPoint Solid {
        translation 0.02 0 0
        children [
          Shape {
            appearance PBRAppearance { baseColor 0.8 0.2 0.2 roughness 0.3 }
            geometry Box { size 0.04 0.006 0.004 }
          }
        ]
        boundingObject Box { size 0.04 0.006 0.004 }
        physics Physics { mass 0.003 }
      }
    }
  ]
  boundingObject Box { size 0.023 0.03 0.012 }
  physics Physics { mass 0.009 }
}
```

### 4.2 双舵机关节臂

```
# 肩关节 (肘关节原理相同，嵌套在肩关节的 EndPoint 内)
DEF SHOULDER Solid {
  translation 0 0.5 0
  children [
    Shape { geometry Box { size 0.04 0.06 0.04 } }
    HingeJoint {
      jointParameters HingeJointParameters { axis 0 0 1 anchor 0 0.03 0 }
      device [ RotationalMotor { name "shoulder" } PositionSensor { name "shoulder_sensor" } ]
      endPoint Solid {
        translation 0 0.15 0
        children [
          Shape { geometry Box { size 0.03 0.3 0.03 } }
          # ★ 肘关节嵌套在此
          HingeJoint {
            jointParameters HingeJointParameters { axis 0 0 1 anchor 0 0.15 0 }
            device [ RotationalMotor { name "elbow" } PositionSensor { name "elbow_sensor" } ]
            endPoint Solid {
              translation 0 0.1 0
              children [ Shape { geometry Box { size 0.02 0.2 0.02 } } ]
            }
          }
        ]
      }
    }
  ]
}
```

## 五、控制器端编程

### 5.1 C API 完整示例

```c
#include <webots/robot.h>
#include <webots/motor.h>
#include <webots/position_sensor.h>
#include <math.h>

#define TIME_STEP 32

int main() {
  wb_robot_init();

  // 获取舵机设备
  WbDeviceTag servo = wb_robot_get_device("sv1");
  WbDeviceTag sensor = wb_robot_get_device("sv1_sensor");

  // 启用位置传感器
  wb_position_sensor_enable(sensor, TIME_STEP);

  // 角度 → 弧度 宏
  #define deg2rad(d) ((d) * M_PI / 180.0)

  double target = 0.0;
  bool dir = true;

  while (wb_robot_step(TIME_STEP) != -1) {
    // 方法1: 位置控制 (到达目标角度后保持)
    wb_motor_set_position(servo, target);

    // 方法2: 速度控制 (需要设 position = INFINITY)
    // wb_motor_set_position(servo, INFINITY);
    // wb_motor_set_velocity(servo, 2.0);

    // 读取当前角度
    double current = wb_position_sensor_get_value(sensor);
    printf("target: %.2f  current: %.2f\n", target, current);

    // 来回摆动
    if (dir) {
      target += deg2rad(1);
      if (target > deg2rad(90)) dir = false;
    } else {
      target -= deg2rad(1);
      if (target < deg2rad(-90)) dir = true;
    }
  }

  return 0;
}
```

### 5.2 Python API (控制器)

```python
from controller import Robot

robot = Robot()
timestep = 32

servo = robot.getDevice("sv1")
sensor = robot.getDevice("sv1_sensor")
sensor.enable(timestep)

import math

target = 0.0
direction = 1

while robot.step(timestep) != -1:
    servo.setPosition(target)
    current = sensor.getValue()
    print(f"target: {math.degrees(target):.1f} current: {math.degrees(current):.1f}")
    target += math.radians(1) * direction
    if abs(math.degrees(target)) > 90:
        direction *= -1
```

### 5.3 多点轨迹 (平滑运动)

```c
// 插值数组: {时间[s], 角度[°]}
const double trajectory[][2] = {
  {0.0,  0},
  {1.0,  90},
  {2.0,  -45},
  {3.0,  30},
  {4.0,  0}
};
const int TRAJ_LEN = 5;

void set_target_at_time(WbDeviceTag servo, double time) {
  int i;
  for (i = 0; i < TRAJ_LEN - 1; i++) {
    if (time >= trajectory[i][0] && time < trajectory[i+1][0]) {
      double t = (time - trajectory[i][0]) / (trajectory[i+1][0] - trajectory[i][0]);
      double angle = trajectory[i][1] + (trajectory[i+1][1] - trajectory[i][1]) * t;
      wb_motor_set_position(servo, angle * M_PI / 180.0);
      return;
    }
  }
}
```

## 六、高级话题

### 6.1 PID 参数整定

Webots 舵机的默认 PID = `[10, 0, 0]` (纯 P)。整定方法：

| 现象 | 方案 | 示例 |
|------|------|------|
| 到位后持续震荡 | 增大 D | `controlPID [10, 0, 0.05]` |
| 响应太慢、到位慢 | 增大 P | `controlPID [20, 0, 0]` |
| 有稳态误差（差几度） | 加 I | `controlPID [10, 0.1, 0]` |
| 超调过大 | 增大 D + 适当减小 P | `controlPID [8, 0, 0.08]` |

> **经验**：一般从 `[10, 0, 0]` 开始，逐步增大 P 到出现震荡，然后加入 D（约 P/200）消除震荡。

### 6.2 maxForce 对行为的影响

![maxForce 影响曲线]

| maxForce 值 | 效果                   |
| ---------- | -------------------- |
| 远大于负载      | 到位快，撞击硬，可能弹飞负载       |
| 等于负载       | 正常运动，需配合 PID         |
| 小于负载       | 无法到位，会保持当前位置（实际舵机堵转） |

仿真中可将 `maxForce` 设为**物理值的 2~3 倍**以补偿仿真摩擦损耗。

### 6.3 舵机云台 (2-DOF)

```
Solid (基座)
  └── HingeJoint (Yaw 偏航)
        ├── JointParameters { axis 0 1 0 }     # 绕 Y 轴水平旋转
        └── EndPoint
              └── Solid
                    └── HingeJoint (Pitch 俯仰)
                          ├── JointParameters { axis 1 0 0 }  # 绕 X 轴俯仰
                          └── EndPoint
                                └── Solid (装载相机/负载)
```

### 6.4 总线舵机仿真

AX-12A 等总线舵机（串口通信）在 Webots 中不需要模拟通信协议，仍然使用 `RotationalMotor`，只是需要在控制器端增加：
- 多舵机 ID 管理（用 name 区分）
- 更丰富的状态反馈（温度、电压、负载等可在仿真中额外添加 `Emitter/Receiver` 模拟）

### 6.5 舵机建模检查清单

```
[ ] 基座 Solid 的 physics 和 boundingObject 是否设置？
[ ] axis 方向是否正确（右手定则检验）？
[ ] minPosition / maxPosition 是否以弧度为单位？
[ ] RotationalMotor 的 name 在控制器中是否一致？
[ ] PositionSensor 是否 enable？
[ ] EndPoint 的 translation 是否对齐旋转中心？
[ ] 嵌套关节时，子关节是否放在上层 EndPoint 的 children 内？
[ ] controlPID 是否需要调整以避免震荡？
[ ] maxForce 是否大于实际负载重力矩？
```

## 七、常见问题 (FAQ)

**Q: 舵机运动时剧烈震荡？**
A: 增大 `dampingConstant` (0.05→0.1) 或调整 `controlPID` 增大 D 项。

**Q: 舵机臂穿模（穿透其他物体）？**
A: 确认 `boundingObject` 是否设置，`stopErp` 是否设得过低 (建议 0.8)。

**Q: 嵌套关节的 EndPoint 偏移怎么算？**
A: 每个关节的 `anchor` 相对于其所属的 `Solid`。外层 `EndPoint.translation` 控制臂长，内层 `HingeJoint.anchor` 控制子关节位置。

**Q: 舵机响应太慢，目标角度半天到不了？**
A: 检查 `maxVelocity` 是否够大，`maxForce` 是否大于负载需求，P 增益是否合理。

**Q: 多个舵机能同步运动吗？**
A: 可以。每个 `wb_robot_step()` 周期内分别设置每个舵机的目标位置即可。

## 八、参考

- [[e-puck_line循线控制器]] — e-puck 循线示例
- [[Webots节点继承关系]] — Webots 节点继承体系完整图谱
- [[e-puck_line包容架构.canvas]] — 包容架构可视化
- [Webots 官方文档: Motor](https://cyberbotics.com/doc/reference/motor)
- [Webots 官方文档: Joint](https://cyberbotics.com/doc/reference/joint)
