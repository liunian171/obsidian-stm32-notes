# Jetson Orin Nano 推理端深度分析

## 项目定位
Jetson 端是系统的"大脑"，负责实时图像采集、神经网络推理、控制指令下发。

## 技术栈

| 技术 | 用途 | 版本 |
|------|------|------|
| Python | 运行时语言 | 3.8+ |
| PyTorch | 深度学习框架 | >=2.0 |
| TorchVision | ResNet18模型、预处理 | >=0.15 |
| OpenCV | 图像采集、编解码 | - |
| GStreamer | CSI摄像头流水线 | Jetson平台专用 |
| PySerial | UART串口通信 | - |
| Pillow | 图像格式转换 | >=9.0 |

## 系统架构

### 三线程模型

```
┌─────────────────────────────────────────────────────────────┐
│                      main.py                                │
│  ┌──────────────┐  轮询get_mode()   ┌──────────────────┐    │
│  │UartControlRdr│←──────────────────│  模式调度         │    │
│  │ (后台线程)   │   training/follow │  0x01→Recorder   │    │
│  │  串口读取    │   idle            │  0x02→Infer      │    │
│  └──────────────┘                   └──────────────────┘    │
│         │                                    │              │
│         │ 共享Camera                         │              │
│         ▼                                    ▼              │
│  ┌──────────────────┐              ┌──────────────────┐     │
│  │    Recorder      │              │  RealTimeInfer   │     │
│  │  (数据录制)      │              │  ┌─────────────┐ │     │
│  │  10fps保存       │              │  │camera_loop  │ │     │
│  └──────────────────┘              │  │infer_loop   │ │     │
│                                    │  │send_loop    │ │     │
│                                    │  └─────────────┘ │     │
│                                    └──────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### 数据录制 (Recorder)

```
状态机: idle → recording → idle

录制阶段(10fps):
  1. Camera.read() → frame
  2. uart.get_at_time(frame_ts) → 时间对齐的控制值
  3. cv2.imwrite(frame, "frames/%06d.jpg")
  4. CSV写入: steer, throttle, brake, gear, speed, target_valid

特性:
  - 控制去重: 值变化小于阈值跳过
  - 强制保存: 至少每秒1帧防长直路空白
  - 空闲超时: 3秒无数据自动停止
```

### 实时推理 (RealTimeInfer)

```
3个线程并行工作:

Camera Thread (30fps):
  读取摄像头 → 每3帧预处理1帧(BGR→RGB→224×224→Tensor→归一化)
  → 追加到 frame_buffer(maxlen=5)

Infer Thread (10Hz):
  取5帧 → AutofollowModel → [steer, throttle, brake_logit, target_valid_logit]
  → 后处理 → 更新latest_pred

Send Thread (20Hz):
  CommandSender.pack() → serial.write() → MCU(TYPE=0x10, 22字节)

后处理管线:
  target_valid_logit ──sigmoid──→ target_valid_prob
  brake_logit ──sigmoid──→ brake_prob
  ├─ target_valid < 0.45 → 强制刹车(无人)
  ├─ brake_prob > 0.65 → 刹车, 限制油门
  └─ 低通滤波: steer_alpha=0.3, throttle_alpha=0.2
  └─ 死区: steer_deadband=0.03
  └─ 刹车迟滞: on_frames=3, off_frames=5
```

## 通信协议

### 帧类型定义
| 方向 | TYPE | 说明 |
|------|------|------|
| MCU→Jetson | 0x01 | 训练数据 (50Hz遥测) |
| MCU→Jetson | 0x02 | 跟随模式就绪 |
| MCU→Jetson | 0x05 | 切换手动模式 |
| MCU→Jetson | 0x06 | 切换遥控模式 |
| Jetson→MCU | 0x10 | 主机控制指令(20Hz) |

### 归一化映射
```
MCU→Jetson (解析):
  steer_norm = steer_01 / (37.0 * 10)  →  [-1, 1]
  throttle_norm = thr_ref / 1000        →  [0, 1]

Jetson→MCU (打包):
  steer_01 = int(steer * 37.0 * 10)
  thr_ref = int(throttle * 1000)
  brake_u8 = 1 if brake > 0.5 else 0
```

## 学习要点

1. **多线程架构设计** — 摄像头/推理/发送三线程解耦，各自独立频率
2. **UART背景线程解析** — 后台持续读取+帧提取+CRC校验
3. **时间同步机制** — MCU端打时间戳，Jetson端按帧时间戳插值对齐控制值
4. **安全后处理** — 低通滤波、死区、刹车迟滞、target_valid门控
5. **YAML配置驱动** — autodrive.yaml集中管理所有参数
6. **Jetson专用摄像头流水线** — GStreamer nvarguscamerasrc + nvvidconv
7. **数据去重+强制保存** — 平衡数据集质量与数量

---

## 相关笔记

- [[00_项目总览]] — Jetson在整体架构中的位置
- [[03_模型训练端]] — 模型架构细节与训练流程
- [[04_通信协议详解]] — 22字节帧协议解析
- [[06_端到端流程与架构设计哲学]] — 决策层详解
- [[07_关键技术点详解]] — 线程安全、后处理管线原理
