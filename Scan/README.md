
# 文档扫描与 OCR 识别工具

本项目包含两个 Python 脚本，分别用于**文档图像扫描校正**和**OCR 文字识别**，可组合使用实现"拍照 → 扫描 → 识别文字"的完整流程。

---

## 项目结构

| 文件 | 说明 |
|------|------|
| `scan.py` | 文档扫描器：自动检测文档轮廓、透视校正、二值化处理，输出扫描后的文档图像 |
| `test.py` | OCR 识别器：对文档图像进行预处理后，调用 Tesseract OCR 引擎提取文字 |

---

## 功能概述

### scan.py — 文档扫描校正

1. **读取图像**：通过命令行参数指定待扫描的文档图片路径
2. **预处理**：灰度化 → 高斯模糊 → Canny 边缘检测
3. **轮廓检测**：查找图像中面积最大的轮廓，并近似为四边形（文档的四个角）
4. **透视变换**：根据检测到的四个角点，进行四点透视变换，将文档校正为正视图
5. **二值化输出**：对校正后的图像进行二值化处理，保存为 `scan.jpg`

### test.py — OCR 文字识别

1. **读取图像**：读取 `scan.jpg`（可由 `scan.py` 生成）
2. **预处理**：支持两种预处理方式（均值模糊 / Otsu 阈值二值化），通过变量 `preprocess` 切换
3. **OCR 识别**：调用 Tesseract OCR 引擎提取图像中的文字
4. **结果输出**：将识别到的文字打印到控制台

---

## 环境依赖

### 系统级依赖

- **Tesseract OCR**（>= 4.0）：文字识别引擎

  下载地址：https://digi.bib.uni-mannheim.de/tesseract/

  安装后需将 Tesseract 安装路径加入系统环境变量 `PATH`，例如：

  ```
  E:\Program Files (x86)\Tesseract-OCR
  ```

  验证安装是否成功：

  ```bash
  tesseract -v
  ```

### Python 依赖

| 包名 | 用途 |
|------|------|
| `opencv-python` | 图像处理（边缘检测、透视变换、二值化等） |
| `pytesseract` | Python 封装的 Tesseract OCR 接口 |
| `Pillow` | 图像打开与处理（pytesseract 依赖） |
| `numpy` | 数组运算（scan.py 中使用） |

安装命令：

```bash
pip install opencv-python pytesseract Pillow numpy
```

---

## 使用方法

### 第一步：扫描文档（透视校正）

```bash
python scan.py -i your_image.jpg
```

- `-i` / `--image`：待扫描的文档图片路径（必填）

执行后会在当前目录生成 `scan.jpg`，即校正后的文档图像。

### 第二步：识别文字（OCR）

```bash
python test.py
```

脚本会自动读取当前目录下的 `scan.jpg` 并进行文字识别，识别结果将输出到控制台。

### 组合使用

```bash
python scan.py -i photo.jpg && python test.py
```

先扫描校正，再识别文字，一步到位。

---

## 参数配置

### test.py 预处理方式

修改 `test.py` 中的 `preprocess` 变量可切换预处理方式：

| 值 | 处理方式 | 适用场景 |
|----|---------|---------|
| `'blur'` | 中值模糊（`cv2.medianBlur`） | 含噪声的图像，去噪效果好 |
| `'thresh'` | Otsu 自动阈值二值化 | 对比度较好的清晰文档 |

```python
preprocess = 'blur'  # 可改为 'thresh'
```

### scan.py 关键参数

| 参数位置 | 参数名 | 默认值 | 说明 |
|---------|--------|--------|------|
| 命令行 | `--image` | 必填 | 输入图像路径 |
| `resize()` | `height` | 500 | 缩放后的高度（用于加速处理） |
| `cv2.GaussianBlur` | 核大小 | (5, 5) | 高斯模糊核 |
| `cv2.Canny` | 低阈值 | 75 | Canny 边缘检测低阈值 |
| `cv2.Canny` | 高阈值 | 200 | Canny 边缘检测高阈值 |
| `cv2.approxPolyDP` | epsilon | 0.02 × peri | 轮廓近似精度 |
| `cv2.threshold` | 阈值 | 100 | 二值化阈值 |

---

## 运行流程示意

```
原始照片 (photo.jpg)
    │
    ▼
scan.py ──→ 边缘检测 ──→ 轮廓检测 ──→ 透视校正 ──→ scan.jpg（扫描结果）
                                                    │
                                                    ▼
                                             test.py ──→ OCR 识别 ──→ 输出文字
```

---

## 注意事项

1. **Tesseract 环境变量**：确保 Tesseract 已安装并加入系统 PATH，否则 `pytesseract` 调用会报错
2. **输入图像质量**：文档扫描效果受原始照片质量影响较大，建议拍摄时保持光线均匀、文档尽量平整
3. **轮廓检测**：`scan.py` 默认取面积最大的四边形作为文档区域，如果图像中存在其他大面积四边形可能会误检
4. **中文识别**：如需识别中文，请安装 Tesseract 的中文语言包（`chi_sim` / `chi_tra`），并在 `test.py` 中指定语言参数

---

## 许可证

本项目仅供学习与参考。
