# CubeMX + CubeIDE for VS Code：用户源文件不被编译的问题

## 问题场景

使用 STM32CubeMX 生成工程，配合 **CubeIDE for VS Code**（基于 `cube-cmake`）开发。在 `Core/Src/` 下手动添加了 `.c` / `.cpp` 文件（如 `pwm.c`、`pwm_platform_ops.c`），但编译时报错：

- `undefined reference to 'xxx'`
- 头文件中的声明提示 `undeclared` / `implicit declaration`

## 根本原因

CubeMX 生成的 `CMakeLists.txt` 只包含它自己创建的源文件列表，**不会自动发现**用户手动添加的文件。

```
CubeMX 首次生成 ──> CMakeLists.txt 写死源文件列表
       │
       ▼
用户手动加文件 ──> CMakeLists.txt 不变 ──> 新文件不参与编译
```

在完整版 CubeIDE（基于 Eclipse）中，Eclipse 的 CDT 维护一个文件数据库，新建文件会自动注册。但 **CubeIDE for VS Code** 直接调用 `cube-cmake` 构建，一切由 CMakeLists.txt 决定，没有"文件数据库"这一层。

> 注：CubeMX 确实会重新生成 `cmake/stm32cubemx/` 下的 CMake 片段，但顶层 `CMakeLists.txt` **不会被覆盖**——这是解决问题的关键前提。

## 解决方案

编辑项目根目录的顶层 `CMakeLists.txt`，找到 CubeMX 预留的 `# Add user sources here` 区域，加入以下内容：

```cmake
# Add user sources here
file(GLOB USER_SOURCES
    Core/Src/*.c
    Core/Src/*.cpp
    Core/Src/*.cc
    Core/Src/*.cxx
)
target_sources(${CMAKE_PROJECT_NAME} PRIVATE ${USER_SOURCES})
```

### 说明

| 部分 | 作用 |
|------|------|
| `file(GLOB ...)` | 收集 `Core/Src/` 下所有 `.c` / `.cpp` / `.cc` / `.cxx` 文件 |
| `target_sources(...)` | 将收集到的文件加入目标编译 |
| `PRIVATE` | 这些源文件仅用于当前目标，不传播给依赖方 |

### 优点

- 以后在 `Core/Src/` 下新增文件**无需再次修改** CMakeLists.txt
- 不依赖 CubeMX 后续版本是否支持此功能
- 只改顶层 CMakeLists.txt（CubeMX 不会覆盖），不影响 `cmake/stm32cubemx/` 下的自动生成文件

### 注意事项

- **启用 C++ 支持：** 如果工程混用 `.c` 和 `.cpp`，确保 `project()` 指令包含 `CXX`：
  ```cmake
  project(my_project C CXX ASM)
  ```
  CubeMX 生成的顶层 CMakeLists.txt 默认已经包含 `CXX`，但如果你重新创建了工程文件需要手动加上。
  
- **汇编文件（.s / .S）：** 通常 CubeMX 放在 `Core/Startup/` 下，已由它自动管理。如果用户自己添加汇编文件，可以额外 GLOB：
  ```cmake
  file(GLOB USER_ASM_SOURCES Core/Src/*.s Core/Src/*.S)
  target_sources(${CMAKE_PROJECT_NAME} PRIVATE ${USER_ASM_SOURCES})
  ```

- `file(GLOB)` 不会自动检测新文件——需要**重新运行 CMake configure**（或重新加载 VS Code CMake 插件）才能生效
- 如果文件分布在多个目录，可以添加多个 `GLOB` 路径或使用递归 `GLOB_RECURSE`（需谨慎，可能引入不需要的测试文件）
- 如果只想编译特定文件而非整个目录，可以用显式列表：

```cmake
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    Core/Src/pwm.c
    Core/Src/pwm_platform_ops.c
)
```

## CubeMX CMake 工程结构速览

```
my_project/
├── CMakeLists.txt           # 顶层（用户可修改，CubeMX 不覆盖）
├── cmake/
│   └── stm32cubemx/
│       ├── CMakeLists.txt   # CubeMX 自动生成（会被覆盖）
│       └── ...              # 工具链、链接脚本等
├── Core/
│   ├── Inc/                 # 头文件
│   └── Src/                 # 源文件（CubeMX 生成的 + 用户添加的）
└── Drivers/
    └── ...
```

顶层 `CMakeLists.txt` 大致结构：

```cmake
cmake_minimum_required(VERSION 3.22)
project(my_project C CXX ASM)

add_subdirectory(cmake/stm32cubemx)

# Add user sources here        <── 在这里插入 GLOB 代码
```

## 参考

- STM32CubeMX 用户手册（UM1718）
- cube-cmake 文档：https://github.com/STMicroelectronics/cube-cmake
