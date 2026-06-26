# Webots 多几何体组合

## 核心原则：Transform 掌管相对位移和方向

**对。** 每个子几何体相对于 Solid 坐标原点的位置和朝向，统一由 `Transform` 处理。

```
Solid (原点 0,0,0)
  ├── Shape (不包 Transform)     → 几何体在 Solid 原点
  └── Transform {                → ★ 相对 Solid 原点的偏移 + 旋转
        translation 0.1 0 0      → 沿 X 移 0.1m
        rotation 0 0 1 1.57      → 绕 Z 转 90°
        children [ Shape { ... } ]
      }
```

> `Transform` = 局部坐标系变换。`translation` 设相对位置，`rotation` 设相对朝向。不包 Transform 的 Shape 默认在 Solid 原点。

## 单个 Solid 节点内组合多个几何体

一个 Solid 只能有一个 `boundingObject` 和一个 `physics`，但可以有**多个 Shape** 用于**视觉表现**。每个几何体通过 `Transform` 相对于 Solid 的原点定位。

```
Solid {
  children [                    # ★ 多个视觉几何放入 children
    Shape { geometry Box {} }                    # 在 Solid 原点
    Transform {                                  # 相对位移
      translation 0.1 0 0
      children [ Shape { geometry Cylinder {} } ]
    }
    Transform {                                  # 相对位移 + 旋转
      translation -0.05 0.02 0
      rotation 0 1 0 0.785
      children [ Shape { geometry Sphere {} } ]
    }
  ]
  boundingObject ...            # ★ 只有一个碰撞体
  physics Physics { ... }
}
```

## 视觉几何（children）

`children` 列表中可以放任意多个 `Shape`，每个 `Shape` 独立设置外观和几何类型。

```webots
children [
  # 主体 (位于 Solid 原点)
  Shape {
    appearance PBRAppearance { baseColor 0.2 0.2 0.8 }
    geometry Box { size 0.1 0.04 0.06 }
  }
  # 凸台：相对主体偏移 (+0.05, +0.02, 0)
  Transform {
    translation 0.05 0.02 0
    children [
      Shape {
        appearance PBRAppearance { baseColor 0.8 0.8 0.2 }
        geometry Cylinder { height 0.01 radius 0.015 }
      }
    ]
  }
  # 指示灯：相对主体偏移 (-0.04, +0.02, +0.03)
  Transform {
    translation -0.04 0.02 0.03
    children [
      Shape {
        appearance PBRAppearance { baseColor 1 0 0 emissiveColor 1 0 0 }
        geometry Sphere { radius 0.005 }
      }
    ]
  }
]
```

> **规则**:
> - 每个 Shape 可用不同 `appearance`（颜色/材质/纹理）
> - 用 `Transform` 调整子几何体的 **相对位移** (`translation`) 和 **相对朝向** (`rotation` / `scale`)
> - 视觉几何**不影响物理碰撞**，只决定渲染外观
> - 不包 Transform 的 Shape 直接在 Solid 原点

## 碰撞体三种方式

### 方式一：单个几何体（最简单）

```webots
boundingObject Box { size 0.1 0.04 0.06 }
```

### 方式二：Group 组合多几何体（★ 推荐）

用 `Group` 将多个几何体组合成一个碰撞体，每个几何体的相对位置和朝向同样用 `Transform` 设置。

```webots
boundingObject Group {
  children [
    # 主体碰撞体在原点
    Box { size 0.1 0.04 0.06 }
    # 凸台碰撞体：相对主体偏移
    Transform {
      translation 0.05 0.02 0
      children [ Cylinder { height 0.01 radius 0.015 } ]
    }
    # 指示灯碰撞体：相对主体偏移
    Transform {
      translation -0.04 0.02 0.03
      children [ Sphere { radius 0.005 } ]
    }
  ]
}
```

> **注意**:
> - `boundingObject` 中的 `Transform` 只支持 `translation` 和 `rotation`，**不支持 `scale`**
> - `children` 中只能放**几何体节点**（Box/Cylinder/Sphere/IndexedFaceSet 等），不能放 Shape
> - 无 Transform 包裹的几何体默认在原点，和视觉层规则一致

### 方式三：Transform 包裹单个几何体并偏移

```webots
boundingObject Transform {
  translation 0.05 0 0
  rotation 0 0 1 0.785
  children [ Box { size 0.08 0.02 0.02 } ]
}
```

适用于需要对单个碰撞体做平移/旋转偏移的场景。

## 典型示例：完整舵机模型

```
Solid {
  translation 0 0 0
  children [
    # ── 视觉层：多个 Shape ──
    Shape {                                          # 外壳主体
      appearance PBRAppearance { baseColor 0.1 0.1 0.1 roughness 0.5 }
      geometry Box { size 0.023 0.03 0.012 }
    }
    Transform {
      translation 0 0.015 0
      children [ Shape {                             # 上盖凸台
        appearance PBRAppearance { baseColor 0.3 0.3 0.3 }
        geometry Box { size 0.023 0.005 0.012 }
      } ]
    }
    Transform {
      translation 0 -0.015 0.006
      children [ Shape {                             # 底部出线口
        appearance PBRAppearance { baseColor 0.05 0.05 0.05 }
        geometry Cylinder { height 0.002 radius 0.004 }
      } ]
    }
    Transform {
      translation 0.01 0 0.005
      children [ Shape {                             # 轴指示标记
        appearance PBRAppearance { baseColor 1 0 0 }
        geometry Sphere { radius 0.002 }
      } ]
    }
  ]

  # ── 碰撞层：Group 组合 ──
  boundingObject Group {
    children [
      Box { size 0.023 0.03 0.012 }                 # 主体碰撞
      Transform {
        translation 0 0.015 0
        children [ Box { size 0.023 0.005 0.012 } ] # 凸台碰撞
      }
    ]
  }

  physics Physics {
    mass 0.009
    inertiaMatrix [ 1e-7 0 0  0 1e-7 0  0 0 1e-7 ]
  }
}
```

## 对比总结

| 目的   | 使用节点                       | 数量限制    | 支持 Transform             |
| ---- | -------------------------- | ------- | ------------------------ |
| 视觉外观 | `children` 下的 `Shape`      | 任意多个    | ✅                        |
| 碰撞检测 | `boundingObject` → `Group` | 多个几何体打包 | ✅ (translation/rotation) |
| 物理属性 | `physics`                  | 只能一个    | ❌                        |

### 常见误区

1. ❌ `boundingObject` 里直接放 `Shape` — **不支持**，放几何体节点
2. ❌ 期望 `boundingObject` 自动匹配视觉 — **不会**，必须显式定义
3. ❌ `boundingObject` 比视觉几何大/小 — 这常见且合理，例如用简化碰撞体提高性能
4. ✅ 视觉可以比碰撞体精细 — 推荐做法：视觉用复杂 Mesh，碰撞用简单 Box/Cylinder

## 参考

- [[Webots舵机建模教程]] — 完整舵机建模
- [[Webots节点继承关系]] — 节点体系
