
# 多目标实时追踪系统

基于 OpenCV DNN 目标检测 + dlib 相关滤波追踪器的多目标实时追踪系统，提供单线程版（Slow）和多进程版（Fast）两种实现。

---

## 功能特性

- **目标检测**：使用 MobileNet SSD Caffe 模型进行实时目标检测，支持 20 类常见物体（重点关注 "person" 行人）
- **多目标追踪**：基于 dlib correlation_tracker 的相关滤波追踪器，可同时追踪多个目标
- **实时显示**：在视频帧上绘制检测框和类别标签
- **视频输出**：支持将追踪结果保存为视频文件
- **FPS 统计**：内置 FPS 计算工具，可评估追踪性能
- **双版本实现**：提供单线程版（slow）和多进程版（fast），便于对比性能差异

---

## 环境要求

| 依赖 | 版本要求 |
|------|---------|
| Python | >= 3.6 |
| OpenCV | >= 4.0 |
| dlib | >= 19.0 |
| NumPy | >= 1.18 |

安装命令：

```
pip install opencv-python dlib numpy
```

---

## 项目结构

```
.
├── multi_object_tracking_slow.py   # 单线程版本（基础版）
├── multi_object_tracking_fast.py   # 多进程版本（加速版）
├── utils.py                        # FPS 统计工具类
└── README.md                       # 项目说明文档
```

---

## 使用方法

两个版本使用相同的命令行参数：

| 参数 | 说明 |
|------|------|
| `-p, --prototxt` | Caffe 模型的 prototxt 文件路径（必填） |
| `-m, --model` | Caffe 预训练模型文件路径（必填） |
| `-v, --video` | 输入视频文件路径（必填） |
| `-o, --output` | 输出视频文件路径（可选） |
| `-c, --confidence` | 最小置信度阈值，过滤弱检测（默认 0.2） |

### Slow 版本（单线程）

```
python multi_object_tracking_slow.py -p mobilenet_ssd/MobileNetSSD_deploy.prototxt -m mobilenet_ssd/MobileNetSSD_deploy.caffemodel -v race.mp4
```

### Fast 版本（多进程）

```
python multi_object_tracking_fast.py -p mobilenet_ssd/MobileNetSSD_deploy.prototxt -m mobilenet_ssd/MobileNetSSD_deploy.caffemodel -v race.mp4
```

---

## 两个版本的区别

### 核心差异：追踪器的执行方式

| 对比维度 | Slow（单线程版） | Fast（多进程版） |
|---------|----------------|----------------|
| **追踪执行方式** | 所有追踪器在主线程中串行更新 | 每个追踪器运行在独立的操作系统进程中，并行更新 |
| **进程模型** | 单进程单线程 | 主进程负责检测和渲染，每个目标一个子进程负责追踪 |
| **进程间通信** | 无需 IPC | 使用 `multiprocessing.Queue` 在进程间传递帧数据和追踪结果 |
| **CPU 利用率** | 单核占用高，多核利用率低 | 多核并行，CPU 利用率更高 |
| **适用场景** | 目标数量少、调试开发 | 目标数量多、追求实时性能 |
| **启动开销** | 无进程创建开销 | 每个新目标需要创建新进程，有一定启动开销 |
| **内存隔离** | 所有追踪器共享同一进程内存 | 每个追踪器进程有独立内存空间 |

### Slow 版本工作流程

1. 读取视频帧，缩放并转换为 RGB
2. 如果没有活跃追踪器，使用 SSD 模型检测行人，为每个检测到的行人创建一个新的 `dlib.correlation_tracker`
3. 如果有活跃追踪器，在主线程中逐个调用 `t.update(rgb)` 更新所有追踪器
4. 获取追踪位置，绘制边界框和标签
5. 显示帧并计算 FPS

**特点**：代码简洁直观，所有操作（检测、追踪、绘图、显示）都在同一个循环中顺序执行。适合目标数量较少或用于调试理解算法流程。

### Fast 版本工作流程

1. 读取视频帧，缩放并转换为 RGB（与 Slow 版本相同）
2. 如果没有活跃追踪器，使用 SSD 模型检测行人，为每个检测到的行人：
   - 创建一对 `multiprocessing.Queue`（输入队列 + 输出队列）
   - 启动一个独立的 `multiprocessing.Process`，执行 `start_tracker` 函数
   - 将进程设为守护进程（`daemon = True`）
3. 如果有活跃追踪器，将所有追踪器的输入队列依次放入当前帧（`iq.put(rgb)`）
4. 从每个追踪器的输出队列获取追踪结果（`oq.get()`）
5. 绘制边界框和标签，显示帧并计算 FPS

**特点**：每个追踪器在独立的 OS 进程中运行，利用多核 CPU 并行处理追踪任务，显著提升多目标场景下的追踪速度。适合目标数量较多、对实时性要求较高的场景。

### 性能对比说明

- **Slow 版本**：所有追踪器的 `t.update(rgb)` 在主线程中串行执行，追踪器越多，单帧处理时间越长，FPS 越低
- **Fast 版本**：每个追踪器的 `t.update(rgb)` 在独立进程中并行执行，追踪器之间互不阻塞，整体 FPS 更高
- **Fast 版本的额外开销**：进程创建、帧数据序列化传输（通过 Queue）、结果收集等会引入一定开销，因此在目标数量很少时差异不明显，但随着目标数量增加，Fast 版本的优势会越来越显著

---

## 支持的检测类别

SSD 模型支持以下 20 类检测（本系统默认只追踪 "person" 行人）：

| 类别 | 类别 | 类别 | 类别 |
|------|------|------|------|
| aeroplane | bicycle | bird | boat |
| bottle | bus | car | cat |
| chair | cow | diningtable | dog |
| horse | motorbike | person | pottedplant |
| sheep | sofa | train | tvmonitor |

---

## 操作说明

| 操作 | 按键 |
|------|------|
| 退出程序 | ESC |

---

## 注意事项

1. **模型文件**：需提前下载 MobileNet SSD Caffe 模型文件（`MobileNetSSD_deploy.prototxt` 和 `MobileNetSSD_deploy.caffemodel`）
2. **置信度阈值**：`--confidence` 参数控制检测过滤灵敏度，值越低检测越多但误检也越多，建议根据实际场景调整
3. **缩放策略**：视频帧默认缩放到宽度 600px 进行处理，以平衡速度与精度
4. **仅追踪行人**：当前代码默认只保留 "person" 类别的检测框，如需追踪其他类别，修改 `CLASSES[idx] != "person"` 的判断条件即可
5. **Fast 版本进程管理**：子进程设置为守护进程，程序退出时会自动清理；但注意进程创建有一定开销，频繁新增目标时性能可能受影响
6. **多进程限制**：`multiprocessing.Queue` 在传递大图像数据时会涉及序列化开销，目标数量极多时需注意内存和 IPC 带宽瓶颈

---

## 算法原理简述

本系统采用 **"先检测、后追踪"** 的经典两阶段方案：

1. **检测阶段（Detection）**：使用 MobileNet SSD 深度学习模型对每一帧进行目标检测，仅在第一帧（无活跃追踪器时）执行检测，定位所有行人位置
2. **追踪阶段（Tracking）**：使用 dlib 相关滤波追踪器（Correlation Tracker）对每个已检测目标进行逐帧追踪，后续帧无需重复检测，大幅提升速度
3. **融合策略**：当追踪器数量为 0 时触发检测重新初始化；追踪过程中持续更新各追踪器状态，获取目标新位置

---

## 工具说明

### FPS 类（utils.py）

`FPS` 类用于计算和输出视频处理的帧率，提供以下方法：

| 方法 | 说明 |
|------|------|
| `start()` | 启动计时器 |
| `stop()` | 停止计时器 |
| `update()` | 帧数计数器 +1 |
| `elapsed()` | 返回已流逝的秒数 |
| `fps()` | 返回近似 FPS 值 |
