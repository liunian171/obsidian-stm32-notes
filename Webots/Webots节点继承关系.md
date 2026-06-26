# Webots 节点继承关系

Webots 的场景图（Scene Graph）基于继承体系组织。箭头方向表示"继承自"，子节点继承父节点的所有字段和 API。

## 核心继承链

```
Group
├── Pose              # 位置/朝向
│   ├── Transform     # 可缩放
│   ├── Solid         # 有物理属性（质量、碰撞等）
│   │   ├── Robot     # 可运行控制器
│   │   ├── Track     # 履带
│   │   ├── Charger   # 充电器
│   │   └── SolidDevice (abstract)
│   │       ├── Accelerometer / Altimeter / Camera / Compass
│   │       ├── Connector / Display / DistanceSensor
│   │       ├── Emitter / Receiver
│   │       ├── GPS / Gyro / InertialUnit
│   │       ├── LED / Lidar / LightSensor
│   │       ├── Pen / Propeller / Radar / RangeFinder
│   │       ├── Speaker / TouchSensor / VacuumGripper
│   │       └── ...
│   └── Fluid        # 流体（水等）
├── Billboard         # 始终面向相机
└── TrackWheel        # 履带轮
```

## 抽象节点体系

```
Device (abstract)
├── SolidDevice → 具体传感器/执行器（见上）
├── Skin
└── JointDevice (abstract)
    ├── Motor (abstract)
    │   ├── LinearMotor
    │   └── RotationalMotor
    ├── Brake
    └── PositionSensor

Geometry (abstract)
├── Box / Capsule / Cone / Cylinder
├── ElevationGrid / Plane / Sphere
├── IndexedFaceSet / IndexedLineSet / PointSet
└── Mesh

Light (abstract)
├── DirectionalLight
├── PointLight
└── SpotLight

Joint (abstract)
├── HingeJoint → Hinge2Joint → BallJoint
└── SliderJoint

JointParameters
├── HingeJointParameters
└── BallJointParameters
```

## 其他独立节点

Appearance、Background、CadShape、Color、ContactProperties、Coordinate、Damping、Focus、Fog、ImageTexture、ImmersionProperties、Lens、LensFlare、Material、Muscle、Normal、PBRAppearance、Physics、Recognition、Shape、Slot、SolidReference、TextureCoordinate、TextureTransform、Viewpoint、WorldInfo、Zoom

## 关键规则

- **Device** 节点必须放在 **Robot** 的 children 层级下（Connector 例外）
- **Solid** 节点通过 `boundingObject` 定义碰撞体
- **Geometry** 节点可用作 `boundingObject`（Box、Cylinder、Sphere 等）
- 虚线框节点为抽象节点，不能直接在 .wbt 文件中实例化

## 参考

- [Node Chart - Webots 官方文档](https://github.com/cyberbotics/webots/blob/master/docs/reference/node-chart.md)
