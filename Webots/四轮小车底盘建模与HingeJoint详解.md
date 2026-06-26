# 四轮小车底盘建模与 HingeJoint 详解

## 一、Robot 节点的设计思路

### 1.1 整体拓扑

```
Robot（根）
├── HingeJoint × 4（每个轮子一个）
│   ├── jointParameters HingeJointParameters（铰链轴心参数）
│   ├── device [ RotationalMotor ]（电机驱动）
│   └── endPoint Solid（轮子实体，含外观 + 碰撞体 + 物理）
└── DEF Body Group（车身，纯视觉）
    └── Shape → Box { size 0.2 0.1 0.1 }
```

### 1.2 关键设计要点

| 要素   | 方案                                      | 原因                             |
| ---- | --------------------------------------- | ------------------------------ |
| 轮子布局 | 四角对称，锚点在 `±0.06 (x) × ±0.06 (z) (Y轴朝上)` | 重心居中，转向稳定                      |
| 轮子轴  | Z 轴（`axis 0 0 1`）                       | 绕 Z 旋转 → 地面反作用力沿 X 方向驱动前进      |
| 轮子几何 | Cylinder 旋转 90° 绕 X                     | Cylinder 默认轴向 Y，旋转后轴向 Z 与铰链轴对齐 |
| 车身碰撞 | `boundingObject USE Body`               | 复用视觉 Group 作为碰撞体，物理质感一体        |
| 物理参数 | `density -1, mass 1`                    | density -1 表示按质量手动取值，不按体积计算    |
| 控制器  | `controller "go_forward"`               | C 语言控制器，四轮同速前进                 |

### 1.3 轮子布局

俯视图（XZ 平面）：

```
  wheel_1 ──── wheel_4   （前，x 正方向）
     │  车身     │   ———→ X 正方向
  wheel_2 ──── wheel_3   （后，x 负方向）
```

- `wheel_1`（前右）：`anchor 0.06 -0.05 0.06`
- `wheel_4`（前左）：`anchor -0.06 -0.05 0.06`
- `wheel_2`（后右）：`anchor 0.06 -0.05 -0.06`
- `wheel_3`（后左）：`anchor -0.06 -0.05 -0.06`

### 1.4 控制策略

```c
// 速度模式：wb_motor_set_position(motor, INFINITY) 切到速度环
// 前进：四轮同向同速
wb_motor_set_velocity(motor1,  FWD_SPEED);
wb_motor_set_velocity(motor2,  FWD_SPEED);
wb_motor_set_velocity(motor3,  FWD_SPEED);
wb_motor_set_velocity(motor4,  FWD_SPEED);

// 原地右转：左轮正转，右轮反转
wb_motor_set_velocity(motor1, -FWD_SPEED);  // 右侧
wb_motor_set_velocity(motor2, -FWD_SPEED);
wb_motor_set_velocity(motor3,  FWD_SPEED);  // 左侧
wb_motor_set_velocity(motor4,  FWD_SPEED);
```

---

## 二、HingeJoint 节点详解

### 2.1 HingeJoint 是什么

`HingeJoint`（铰链关节）是 Webots 中最常用的关节类型，提供一个**绕固定轴旋转**的自由度。它连接**父节点（机器人底盘）**和**子节点（轮子实体）**。

类比现实：**门铰链**——门绕门轴旋转。

### 2.2 节点结构

```
HingeJoint {
  jointParameters HingeJointParameters { ... }  ← 铰链的几何/物理属性
  device           [ RotationalMotor { ... } ]   ← 驱动装置（电机/位置传感器）
  endPoint         Solid { ... }                 ← 被连接的子实体（轮子）
}
```

### 2.3 jointParameters HingeJointParameters 字段详解

```vrml
jointParameters HingeJointParameters {
  anchor  0.06 -0.05  0.06    ← 旋转轴心在父系中的位置
  axis    0     0      1      ← 旋转轴方向
  position 0.0                ← 当前关节角（弧度，运行时自动更新）
  stop     -1.57  1.57        ← 机械限位（可选，默认无限）
}
```

#### `anchor` —— 轴心位置

- 类型：`SFVec3f`（三维坐标）
- 含义：旋转轴心在**父节点坐标系**中的位置
- 注意：它只定义轴心点，**不影响子实体的位置**。子实体的位置由 `endPoint.translation` **独立指定**，两者通常**设为相同值**以保证轮子在轴心处旋转

```
// 错误理解：anchor 会带动子实体移动
// 事实：anchor 只定义轴心，translation 才决定子实体在哪
```

#### `axis` —— 旋转轴方向

- 类型：`SFVec3f`（方向向量，自动归一化）
- 值域：任意方向，通常为单位向量
- 当前值：`0 0 1`（绕 Z 轴旋转）

不同轴对应的前进方向：

| axis | 车轮旋转平面 | 前进方向 |
|------|-------------|---------|
| `0 0 1` | XY 平面 | X 轴 |
| `1 0 0` | YZ 平面 | Z 轴 |
| `0 1 0` | XZ 平面 | 无（垂直滚动，一般不用）|

### 2.4 device —— 驱动装置

```vrml
device [
  RotationalMotor {
    name "motor_1"       ← 控制器通过此名称获取设备句柄
  }
]
```

### 2.5 endPoint —— 子实体

```vrml
endPoint DEF wheel_1 Solid {
  translation 0.06 -0.05 0.06    ← 子实体在父系的位置（独立于 anchor！）
  rotation 1 0 0 1.5708          ← 旋转几何体使 Cylinder 轴对齐 axis
  children [
    Shape {
      geometry DEF wheel Cylinder { height 0.01 radius 0.04 }
    }
  ]
  name "wheel_1"
  boundingObject USE wheel        ← 碰撞体复用几何体定义
  physics Physics {}               ← 赋予物理属性
}
```

**`translation` 为什么要和 `anchor` 一样？**
- 如果不等，轮子绕着空中一个点转 → 物理上表现为"偏心轮"，震动剧烈
- **理想情况：** 轮子质心 = 铰链轴心，这样旋转力矩不产生附加的离心力

**`rotation 1 0 0 1.5708` 的作用？**
- Cylinder 默认轴向 Y（竖直圆柱）
- 旋转 90° 绕 X → 轴向变为 Z
- 这样 Cylinder 的轴向 `Z` 和铰链 `axis 0 0 1` 完全对齐

### 2.6 DEF / USE 重用机制

Webots 通过 `DEF` + `USE` 实现节点复用：

```vrml
DEF wheel Cylinder { height 0.01 radius 0.04 }  ← 定义
...
geometry USE wheel    ← 引用（wheel_1 视觉）
boundingObject USE wheel  ← 引用（wheel_1 碰撞体）
```

优点：
- **一次定义，多处使用**，减少冗余
- boundingObject 和 Shape 共享同一个几何体
- 修改几何尺寸只需要改一处

---

## 三、常见问题

### 3.1 HingeJoint 不包含 Transform 功能

HingeJoint 的 `anchor` **不是** Transform 的 `translation`。它只定义旋转轴位置，子实体位置必须通过 `endPoint.translation` 单独设置。

### 3.2 ODE 引擎的粘滑现象

差速转向时，ODE 物理引擎对静摩擦→动摩擦的跳变处理不连续，导致"停→转→停"的卡顿伪像。现实中不会发生。

### 3.3 缺少 boundingObject 导致物理异常

车身 boundingObject 缺位时，轮子靠 HingeJoint 约束硬拽车身运动，物理行为不稳定，容易侧翻或抖动。
