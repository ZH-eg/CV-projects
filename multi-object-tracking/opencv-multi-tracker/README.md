
# OpenCV 多目标追踪工具

基于 OpenCV 实现的多目标实时追踪程序，支持多种内置追踪算法，可在视频流中框选并持续追踪多个目标物体。

## 功能特性

- 支持 **7 种** OpenCV 内置追踪算法切换（KCF、CSRT、Boosting、MIL、TLD、MedianFlow、MOSSE）
- 支持在视频帧中**多次框选**，同时追踪多个目标物体
- 实时显示追踪结果，绿色矩形框标注目标位置
- 支持任意视频文件输入（.mp4、.avi 等）
- 自动缩放帧至固定宽度，保证流畅播放

## 环境要求

- Python 3.6+
- OpenCV 3.4+（需包含 `contrib` 模块）
- NumPy

## 安装依赖

```bash
pip install opencv-python opencv-contrib-python numpy
```

> **注意**：追踪算法（如 CSRT、MIL、TLD 等）位于 `opencv-contrib-python` 包中，请确保安装的是 `opencv-contrib-python` 而非仅 `opencv-python`。

## 使用方法

### 基本用法

```bash
python multi_object_tracking.py --video <视频路径>
```

### 指定追踪算法

```bash
python multi_object_tracking.py --video <视频路径> --tracker kcf
```

### 参数说明

| 参数 | 简写 | 说明 | 默认值 |
|------|------|------|--------|
| `--video` | `-v` | 输入视频文件路径 | 必填 |
| `--tracker` | `-t` | 追踪算法类型 | `kcf` |

### 支持的追踪算法

| 算法名 | 类名 | 特点 |
|--------|------|------|
| `csrt` | TrackerCSRT | 精度高，速度中等，推荐首选 |
| `kcf` | TrackerKCF | 速度快，精度中等，默认算法 |
| `boosting` | TrackerBoosting | 经典算法，速度较慢 |
| `mil` | TrackerMIL | 精度较好，速度中等 |
| `tld` | TrackerTLD | 支持长时间追踪，可处理遮挡 |
| `medianflow` | TrackerMedianFlow | 适合运动平滑的目标 |
| `mosse` | TrackerMOSSE | 速度最快，精度较低 |

## 操作说明

运行程序后，在视频窗口中按以下键操作：

| 按键 | 功能 |
|------|------|
| `S` | 在当前帧框选一个目标区域，开始追踪（可多次框选，支持多目标） |
| `ESC` | 退出程序 |

**操作流程**：
1. 运行程序，视频开始播放
2. 在需要追踪的目标出现时，按 **S** 键
3. 用鼠标拖拽框选目标区域，释放后该目标即开始被追踪
4. 可重复步骤 2-3 框选多个目标
5. 按 **ESC** 退出

## 注意事项

- 框选目标时尽量在目标**清晰可见的第一帧**进行操作，框选越精确追踪效果越好
- 部分算法（如 TLD）支持目标遮挡后的重新识别，但整体追踪性能受目标运动速度、遮挡、光照变化等因素影响
- 程序会自动将每帧缩放到固定宽度（600px），以保证播放流畅度
- 若视频路径包含中文，建议使用 `cv2.VideoCapture` 兼容的路径写法

## 项目结构

```
.
├── multi_object_tracking.py    # 主程序
└── README.md                   # 项目说明文档
```
