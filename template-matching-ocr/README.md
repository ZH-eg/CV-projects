
# 信用卡卡号 OCR 识别（模板匹配版）

基于 OpenCV 传统图像处理 + 模板匹配（Template Matching）实现的信用卡卡号识别示例。
无需训练深度学习模型，只需一张 OCR-A 字体数字模板图，即可从信用卡照片中定位并识别出 16 位卡号，
并根据卡号首位数字自动判断发卡组织。

---

## 1. 项目简介

项目中 `ocr_template_match.py` 为可独立运行的命令行主程序，`myutils.py` 为其提供通用图像工具函数。
整体思路分两大阶段：

- **模板准备**：读取 OCR-A 模板图，二值化后提取 0~9 的数字轮廓，建立 `digits` 模板字典。
- **卡号定位与识别**：对输入信用卡图做礼帽运算、水平 Sobel 梯度、闭运算与 Otsu 二值化，
  把「四位一组」的卡号区域整体框出，再在每组内部切分单个数字，与模板逐一比对打分取最优。

## 2. 功能特性

- 通用工具：轮廓按方位排序（左到右 / 右到左 / 上到下 / 下到下）、等比例图像缩放
- 模板图自动切分，构建 0~9 数字模板字典（统一归一化到 57x88）
- 形态学礼帽（Top-Hat）+ 水平 Sobel 梯度 + 闭运算 + Otsu 自适应阈值，突出并连通凸起数字
- 按宽高比与宽高范围筛选出「四位一组」的数字块，并按 x 坐标从左到右排序
- `cv2.matchTemplate`（`TM_CCOEFF`）对每位数字逐模板打分，取最高分作为识别结果
- 依据卡号首位数字映射发卡组织：

  | 首位数字 | 发卡组织 |
  | --- | --- |
  | 3 | American Express |
  | 4 | Visa |
  | 5 | MasterCard |
  | 6 | Discover Card |

- 原图绘制识别框与识别文本，控制台打印卡组织与完整卡号

## 3. 环境依赖

- Python 3.x
- OpenCV 3.x（`opencv-python==3.4.x`）
- imutils
- NumPy

```bash
pip install opencv-python==3.4.2.17 imutils numpy
```

> **OpenCV 版本提示**：本代码按 OpenCV 3.x 编写，`cv2.findContours()` 采用「三值解包」
> （`ref_, refCnts, hierarchy = cv2.findContours(...)`）。若使用 OpenCV 4.x（返回值只有 2 个），
> 需把项目中三处解包改为 `refCnts, hierarchy = cv2.findContours(...)` 的形式，否则会抛
> `ValueError: not enough values to unpack`。

## 4. 目录结构

```
.
├── myutils.py                 # 通用工具函数：sort_contours / resize
├── ocr_template_match.py      # 主程序：卡号定位 + 模板匹配识别
├── ocr_a_templates.png         # 需自备：OCR-A 字体 0~9 数字模板图
└── credit_card.png             # 需自备：待识别的信用卡照片
```

> 模板图与信用卡照片需自行准备（可参考 PyImageSearch 信用卡 OCR 示例的公开素材）。

## 5. 快速开始

```bash
python ocr_template_match.py -i credit_card.png -t ocr_a_templates.png
```

运行过程中会按处理顺序逐步弹窗展示中间结果（灰度图、礼帽图、梯度图、二值图、轮廓图、各 ROI），
每个窗口按任意键继续；结束后展示带识别框与识别文本的最终结果图，并在控制台输出：

```
Credit Card Type: Visa
Credit Card #: 4453 1234 5678 9012
```

## 6. 命令行参数

| 参数 | 简写 | 是否必填 | 说明 |
| --- | --- | --- | --- |
| `--image` | `-i` | 是 | 待识别的信用卡图片路径 |
| `--template` | `-t` | 是 | OCR-A 数字模板图片路径 |

## 7. 处理流程

### 阶段一：模板图数字化（构建数字模板字典）

1. 读取模板图并转灰度；
2. `cv2.threshold(ref, 10, 255, THRESH_BINARY_INV)` 反色二值化，使数字为白、背景为黑；
3. `cv2.findContours(..., RETR_EXTERNAL, CHAIN_APPROX_SIMPLE)` 提取数字外轮廓；
4. `myutils.sort_contours(..., method="left-to-right")` 将轮廓按从左到右排序，保证轮廓顺序与 0~9 一致；
5. 逐个轮廓求外接矩形、裁剪 ROI 并 `resize` 到 57x88，存入 `digits[i]`。

### 阶段二：卡号定位

1. 读取输入图并用 `myutils.resize(image, width=300)` 统一分辨率，转灰度；
2. `MORPH_TOPHAT` 礼帽运算（卷积核 9x3），突出比背景更明亮的区域；
3. 水平方向 `cv2.Sobel(dx=1, dy=0, ksize=-1)` 求梯度，取绝对值并归一化到 0~255（弱化垂直边缘干扰）；
4. `MORPH_CLOSE` 闭运算把同一组内的数字粘连成整块，再用 Otsu 自动阈值二值化；
5. 再做一次 5x5 闭运算填补组内空洞；
6. 提取外轮廓，按 `2.5 < 宽高比 < 4.0` 且 `40 < w < 55`、`10 < h < 20` 筛出四位一组的候选框，并按 x 排序。

### 阶段三：逐位识别与输出

1. 每组区域四周外扩 5 像素裁剪，再次 Otsu 二值化；
2. 组内提取数字轮廓并按左到右排序，逐个 ROI 归一化到 57x88；
3. 与 `digits` 中每个模板做 `cv2.matchTemplate(roi, digitROI, TM_CCOEFF)`，`minMaxLoc` 取分数；
4. `argmax(scores)` 即为该位数字，拼接后写入 `output`；
5. 原图上绘制矩形框与识别文本，控制台输出卡组织与完整卡号。

## 8. myutils API

### `sort_contours(cnts, method="left-to-right")`

按外接矩形位置对轮廓排序，用于保证字符读取顺序。

| 参数 | 说明 |
| --- | --- |
| `cnts` | `cv2.findContours` 返回的轮廓列表 |
| `method` | `left-to-right` / `right-to-left` / `top-to-bottom` / `bottom-to-top` |

返回 `(cnts, boundingBoxes)` 二元组；只需排序结果时取 `[0]`。

### `resize(image, width=None, height=None, inter=cv2.INTER_AREA)`

按单边等比例缩放图像（给定宽或高之一，另一边按比例计算）；两者都为 `None` 时原样返回。
常用于把输入图统一到固定宽度，使后续硬编码的宽高阈值仍然适用。

## 9. 关键参数与调参建议

| 位置 | 参数/条件 | 默认值 | 调参建议 |
| --- | --- | --- | --- |
| 输入预处理 | `resize(image, width=...)` | 300 | 改动该值必须同步调整下方宽高阈值 |
| 形态学核 | `rectKernel` / `sqKernel` | 9x3 / 5x5 | 数字间距大则加长矩形核以完成粘连 |
| 模板 ROI | `cv2.resize(roi, (57, 88))` | 57x88 | 模板与待识别 ROI 必须使用同一尺寸 |
| 分组筛选 | 宽高比 `ar` | 2.5 ~ 4.0 | 按实际每组字符数调整区间 |
| 分组筛选 | `w` / `h` | 40~55 / 10~20 | 随 `width` 缩放比例同步缩放 |
| 分组二值化 | `cv2.threshold(..., THRESH_OTSU)` | 阈值 0 | 光照不均可先做 CLAHE 再二值化 |

## 10. 注意事项与已知限制

1. **弹窗阻塞**：`cv_show()` 会逐个中间结果弹窗，脚本为调试/演示用途设计；批量处理或服务器上运行时
   请注释掉 `cv_show` 调用（或改用 `cv2.imwrite` 落盘中间图）。
2. **阈值硬编码**：轮廓宽高与宽高比条件针对示例样本设定，换数据集需重调（见上表）。
3. **切片越界风险**：分组 ROI 使用 `gray[gY-5 : gY+gH+5, gX-5 : gX+gW+5]`，当数字贴边时负索引会取到
   图像另一侧内容，建议改为 `max(0, gY-5)` 形式做边界裁剪。
4. **字体与场景局限**：仅适配与 OCR-A 模板字形接近的卡号（凸印/标准卡号字体），
   倾斜、模糊、反光、非模板字体下识别率明显下降；生产环境建议改用 CRNN / PaddleOCR 等方案。
5. **卡组织判定仅看首位**：`FIRST_NUMBER[output[0]]` 只按首位粗判，未做 BIN 号段精查与 Luhn 校验；
   若未检出任何分组，`output` 为空会使 `output[0]` 抛出 `IndexError`，调用方需自行容错。
6. **敏感信息**：卡号属敏感个人信息，请勿将识别结果与原始卡照写入日志或上传外部服务。

## 11. 参考资料

- PyImageSearch：Credit card OCR with OpenCV and Python（本实现的算法脉络来源）
- OpenCV 官方文档：Morphology Transformations、Template Matching、`findContours`
- imutils：`imutils.contours.sort_contours`
