
# CV-Projects

> 一个计算机视觉（Computer Vision）学习与实践项目集合。
>
> 每个子项目均提供独立、可运行的脚本与各自的 README 说明，适合作为 CV 入门与算法实践的参考。

## 项目总览

| # | 项目 | 方向 | 核心能力 | 主要依赖 |
|---|------|------|---------|---------|
| 1 | [答题卡自动阅卷系统](#1-答题卡自动阅卷系统) | 图像处理 / OCR | 透视校正 + 气泡检测 + 自动评分 | OpenCV, imutils |
| 2 | [blink-detector 眨眼检测](#2-blink-detector实时眨眼检测工具) | 面部关键点分析 | EAR 算法实时眨眼检测与计数 | OpenCV, dlib |
| 3 | [多目标实时追踪系统](#3-多目标实时追踪系统) | 目标检测 + 追踪 | SSD 检测 + 多进程并行追踪 | OpenCV, dlib |
| 4 | [OpenCV 多目标追踪工具](#4-opencv-多目标追踪工具) | 单目标/多目标追踪 | 7 种内置追踪算法 + 手动框选 | OpenCV (contrib) |
| 5 | [文档扫描与 OCR 识别工具](#5-文档扫描与-ocr-识别工具) | 文档处理 / OCR | 透视校正 + Tesseract 文字识别 | OpenCV, pytesseract |
| 6 | [信用卡卡号识别](#6-信用卡卡号识别) | 图像处理 / OCR | 模板匹配 + 卡号定位 + 自动判读发卡行 | OpenCV, imutils |

## 技术栈

- **核心库**：OpenCV（图像处理 / 视频处理）、NumPy
- **检测与关键点**：dlib（68 点面部关键点、相关滤波追踪器）、MobileNet SSD (Caffe)
- **文字识别**：Tesseract OCR（pytesseract）
- **运行环境**：Python 3.x

## 项目介绍

### 1. （answer_sheet）答题卡自动阅卷系统

基于 OpenCV 的答题卡（Bubble Sheet）自动识别与评分程序。对扫描后的答题卡图像进行**四点透视校正**，再通过 **Otsu 阈值分割 + 轮廓分析**定位每个选项气泡，最后与内置答案键（Answer Key）逐题比对并计算得分，用绿色/红色标注对错并叠加显示总分。

- **关键技术**：Canny 边缘检测、轮廓近似、四点透视变换、Otsu 二值化、气泡填充量计算
- **运行方式**：`python get_answer.py -i exam_image.jpg`

### 2. blink-detector（实时眨眼检测工具）

轻量级、高精度的实时眨眼检测工具，结合 dlib 面部关键点检测与 OpenCV 视频处理能力。通过计算**眼睛纵横比（EAR, Eye Aspect Ratio）**量化眼睛开合程度，配合"阈值 + 连续帧防抖"机制精准识别闭眼动作并统计眨眼次数，支持本地视频与摄像头实时分析。

- **关键技术**：dlib 68 点关键点、EAR 几何算法、双眼协同判定、实时可视化
- **运行方式**：`python detect_blinks.py -p shape_predictor_68_face_landmarks.dat -v video.mp4`

### 3. dlib_multi_tracker（多目标实时追踪系统）

采用「**先检测、后追踪**」两阶段方案的多目标实时追踪系统：用 MobileNet SSD (Caffe) 模型检测行人，再用 dlib 相关滤波追踪器逐帧追踪。提供 **Slow（单线程串行）** 与 **Fast（每个目标独立进程并行）** 两种实现，便于对比多核并行带来的性能提升。

- **关键技术**：MobileNet SSD 目标检测、dlib correlation_tracker、多进程并行 + 队列 IPC、FPS 统计
- **运行方式**：`python multi_object_tracking_fast.py -p ... -m ... -v race.mp4`

### 4. opencv-multi-tracker（OpenCV 多目标追踪）

基于 OpenCV 内置追踪器的多目标实时追踪程序，支持 **7 种追踪算法**（KCF、CSRT、Boosting、MIL、TLD、MedianFlow、MOSSE）自由切换。可在视频流中多次按 `S` 键手动框选目标进行持续追踪，并用绿色矩形框实时标注目标位置。

- **关键技术**：OpenCV 多追踪算法封装、交互式框选初始化、多目标并行追踪
- **运行方式**：`python multi_object_tracking.py --video <视频路径> --tracker kcf`
- **依赖提示**：需安装 `opencv-contrib-python`

### 5. Scan（文档扫描与 OCR 识别）

包含两个可组合使用的脚本，实现「**拍照 -> 扫描校正 -> 识别文字**」的完整流程：`scan.py` 自动检测文档轮廓、做透视校正与二值化，生成扫描图；`test.py` 对扫描图预处理后调用 **Tesseract OCR** 提取文字并输出到控制台。

- **关键技术**：边缘/轮廓检测、四点透视变换、二值化、Tesseract OCR
- **运行方式**：`python scan.py -i photo.jpg && python test.py`
- **依赖提示**：需安装系统级 Tesseract OCR 引擎并配置环境变量

### 6. template-matching-ocr(信用卡卡号识别)
基于 OpenCV 传统图像处理与模板匹配（TM_CCOEFF）的信用卡卡号识别程序，无需训练模型，仅需一张 OCR-A 字体模板图即可完成数字识别，并根据首位数字自动判断发卡组织（Visa / MasterCard / Amex / Discover）。
- **关键技术**：轮廓检测与排序、形态学操作（礼帽 + 闭运算）、Sobel 梯度、Otsu 二值化、模板匹配
- **运行方式**：python ocr_template_match.py -i <信用卡图片路径> -t <模板图片路径>
- **依赖提示**：需安装 opencv-python 和 imutils；注意 OpenCV 3.x 与 4.x 的 findContours 返回值差异，代码按 3.x 格式编写

## 免责声明

本项目集合仅供学习与研究使用。各子项目的详细参数、算法流程与注意事项，请参阅其目录下的独立 README。
