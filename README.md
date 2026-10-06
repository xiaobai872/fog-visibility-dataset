# 雾天高速公路能见度图像数据集（FVEI 子集）

> 配套论文：融合物理先验与频域注意力的雾天高速公路能见度估计网络
> 数据版本：v1.0 

本数据集用于**雾天高速公路能见度等级分类与能见度数值回归的双任务**研究。
每幅雾天 RGB 图像都配有**等级标签、实测能见度值标签**以及一一对应的**16 位度量深度图**，
可直接用于多模态（RGB + 深度先验）与单模态方法的对比实验。

---

## 0. 发布包内容

| 文件              | 内容                    | 规格                             | 大小       |
| --------------- | --------------------- | ------------------------------ | -------- |
| `train.zip`     | 训练集 RGB 图像（含原图与水平翻转图） | 7 200 张，1920 × 1080 JPEG       | ≈ 609 MB |
| `test.zip`      | 测试集 RGB 图像（仅原图）       | 900 张，1920 × 1080 JPEG         | ≈ 85 MB  |
| `depth_224.zip` | 训练集与测试集的度量深度图         | 8 100 张，**224 × 224** 16 位 PNG | ≈ 0.4 GB |
| `README.md`     | 本说明                   | —                              | —        |

解压后目录结构为 `train/`、`test/`、`train_depth/`、`test_depth/`（各含 `0`~`4` 五个等级子目录）。

> **关于深度图的分辨率**：原始深度图为 1920 × 1080，共约 10.1 GB。本数据集发布的深度图为
> **224 × 224**，原因是训练与推理流程中 RGB 与深度都会被缩放到 224 × 224 后送入网络
> （见 4.2 节说明），因此 224 × 224 的深度图与原始流程**逐比特等价**，而体积缩小约 30 倍。
> 如需 1920 × 1080 的原始深度图，请联系作者获取。

## 1. 数据规模

| 划分     | 真实图像（原图）      | 水平翻转图         | RGB 图像合计  | 深度图合计     |
| ------ | ------------- | ------------- | --------- | --------- |
| train  | 3 600（每类 720） | 3 600（每类 720） | 7 200     | 7 200     |
| test   | 900（每类 180）   | **0**         | 900       | 900       |
| **总计** | **4 500**     | **3 600**     | **8 100** | **8 100** |

- **真实图像（原图）共 4 500 张**：训练 3 600 + 测试 900。
- **训练集的水平翻转增强已预先烘入数据集**，共 3 600 张，与 3 600 张原图一一对应，
  因此训练目录合计 7 200 张；**测试集不含任何增强图像**，以保证评估反映真实泛化性能。
- 图像与深度图**逐一对齐，配对完整**（0 缺失、0 多余），同名匹配（仅扩展名不同）。
- 类别**完全均衡**：训练集每类 720 原图 + 720 翻转，测试集每类 180 原图。

> ⚠️ **使用提醒**
>
> 1. 训练集里的翻转图是**已有原图的镜像**，不是独立样本。若您要自行做数据增强，
>    请先用 `a` 前缀把翻转图过滤掉，否则会造成增强叠加。
> 2. 划分训练/验证集时**必须把原图与其翻转图放在同一侧**，否则同一场景会跨越划分，
>    导致验证指标虚高（数据泄漏）。

## 2. 目录结构

```
train/                 训练集 RGB 图像（JPEG，含原图与翻转图）
├── 0/  1/  2/  3/  4/    按能见度等级分子目录
test/                  测试集 RGB 图像（JPEG，仅原图）
├── 0/  1/  2/  3/  4/
train_depth/           训练集深度图（PNG，与 train/ 同结构、同名）
├── 0/  1/  2/  3/  4/
test_depth/            测试集深度图（PNG，与 test/ 同结构、同名）
├── 0/  1/  2/  3/  4/
```

## 3. 文件命名规则与标签

文件名格式：**`<能见度等级>-<图像编号>-<能见度值>.<扩展名>`**

其中**图像编号带 `a` 前缀者表示该图是同一编号原图的水平翻转副本**。

| 文件名               | 含义                                       |
| ----------------- | ---------------------------------------- |
| `0-00012-44.jpg`  | 等级 0（浓雾）、图像编号 00012、实测能见度 44 m —— **原图** |
| `0-a00012-44.jpg` | 同上的**水平翻转图**（编号加 `a`），标签与原图完全相同          |
| `3-03928-259.png` | 等级 3（轻雾）、图像编号 03928、实测能见度 259 m 的深度图     |

- **第 1 段**：能见度等级（0–4），与所在子目录名一致；
- **第 2 段**：图像编号，可带 `a` 前缀（`a` = 翻转副本）；去掉 `a` 后与原图编号一一对应；
- **第 3 段**：能见度实测值，单位 **米（m）**，作为回归任务的标签。

因此**无需额外的标签文件**，标签可直接从路径解析：

```python
import os

name = os.path.splitext('0-a00012-44.jpg')[0]
grade, img_id, visibility = name.split('-')

grade      = int(grade)                  # 能见度等级 0~4
is_flipped = img_id.startswith('a')      # 是否为水平翻转副本
img_id     = img_id.lstrip('a')          # 还原原始图像编号
visibility = float(visibility)           # 能见度实测值 / m
```

## 4. 图像与深度图规格

### 4.1 规格表

| 项目    | RGB 图像      | 深度图               |
| ----- | ----------- | ----------------- |
| 格式    | JPEG        | PNG               |
| 分辨率   | 1920 × 1080 | **224 × 224**     |
| 色彩/位深 | RGB，8 位/通道  | **单通道 16 位无符号整型** |
| 物理含义  | 雾天高速公路监控图像  | 该像素对应的度量深度        |

### 4.2 深度图的单位与读取方式（重要）

16 位像素值为**度量深度的 1/256 米**，因此读取时须 `÷ 256.0`：

```python
import numpy as np
from PIL import Image

depth_16 = np.asarray(Image.open(depth_path)).astype(np.uint16)   # mode 'I;16'
depth_m = depth_16.astype(np.float32) / 256.0                     # 单位：m
```

用 OpenCV 读取时须保留位深（**注意：`IMREAD_ANYDEPTH`，不要用默认的 `IMREAD_COLOR`**）：

```python
import cv2
depth_16 = cv2.imread(depth_path, cv2.IMREAD_ANYDEPTH)
depth_m = depth_16.astype('float32') / 256.0
```

## 5. 能见度等级定义

等级划分参考《雾天高速公路交通安全控制条件》（GB/T 31445—2015），共 5 级：

| 等级  | 名称  | 论文中采用的区间                   |
| --- | --- | -------------------------- |
| 0   | 浓雾  | ≤ 50 m                     |
| 1   | 大雾  | 50 ~ 100 m                 |
| 2   | 中雾  | 100 ~ 200 m                |
| 3   | 轻雾  | 200 ~ 500 m                |
| 4   | 晴朗  | ≥ 500 m（超出量程上限者统一记为 500 m） |

## 6. 数据来源

数据源自公开的 **FVEI（Fog Visibility Estimation）数据集**（YANG W, ZHAO Y, LI Q, et al.
Expert Systems with Applications, 2023, 234: 121-151）中公开的真实高速公路雾天监控图像，
本文使用其中 4 500 张真实图像，并对训练集 3 600 张做水平翻转增强。

## 7. 加载示例（PyTorch）

```python
import os, cv2, numpy as np, torch
from PIL import Image
from torch.utils.data import Dataset


class FogVisibilityDataset(Dataset):
    """img_dir 与 depth_dir 同结构、同文件名（仅扩展名不同）。

    include_flipped=False 时只返回真实图像（不含预置的水平翻转副本），
    适合自行做数据增强或划分训练/验证集的场景。
    """

    def __init__(self, img_dir, depth_dir, include_flipped=True,
                 transform_rgb=None, transform_depth=None):
        self.records = []
        for root, _, files in os.walk(img_dir):
            for f in files:
                if not f.lower().endswith(('.jpg', '.jpeg', '.png')):
                    continue
                parts = os.path.splitext(f)[0].split('-')
                if len(parts) < 3:
                    continue
                is_flipped = parts[1].startswith('a')
                if is_flipped and not include_flipped:
                    continue
                rel = os.path.relpath(os.path.join(root, f), img_dir)
                dep = os.path.join(depth_dir, os.path.splitext(rel)[0] + '.png')
                if os.path.exists(dep):
                    self.records.append((os.path.join(root, f), dep,
                                         int(parts[0]), float(parts[2]), is_flipped))
        self.transform_rgb, self.transform_depth = transform_rgb, transform_depth

    def __len__(self):
        return len(self.records)

    def __getitem__(self, i):
        img_path, dep_path, grade, visibility, is_flipped = self.records[i]
        image = Image.open(img_path).convert('RGB')
        depth = cv2.imread(dep_path, cv2.IMREAD_ANYDEPTH).astype(np.float32) / 256.0
        if self.transform_rgb:
            image = self.transform_rgb(image)
        if self.transform_depth:
            depth = self.transform_depth(Image.fromarray(depth))   # 224x224 时等价于恒等
        return image, depth, grade, torch.tensor(visibility, dtype=torch.float32)
```

## 8. 引用

若使用本数据集，请引用：

原始数据集来源：

> YANG W, ZHAO Y, LI Q, et al. Multi visual feature fusion based fog visibility estimation
> for expressway surveillance using deep learning network[J]. Expert Systems with Applications,
> 2023, 234: 121-151.

## 9. 许可与联系方式

- **联系方式**：孙焱哲，长安大学未来交通学院，E-mail：481554940@qq.com

## 10. 版本记录

| 版本   | 日期      | 说明                                                                                                           |
| ---- | ------- | ------------------------------------------------------------------------------------------------------------ |
| v1.0 | 2026-10 | 首次发布：RGB 原图 4 500 张（train 3 600 / test 900），训练集含 3 600 张水平翻转副本，共 8 100 对；深度图为 224 × 224 16 位度量深度（与训练流程逐比特等价） |
