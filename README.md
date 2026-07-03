# Shrimp Identification

## 1. 背景分析和简单数据描述

### 1.1 背景分析

随着水产养殖业规模化发展、生鲜电商行业快速崛起以及水产品市场监管需求的提升，虾类物种的快速准确识别成为行业痛点。传统虾类识别依赖人工经验，存在效率低、主观性强、易受环境因素影响等问题，难以满足大规模分拣、品质分级、溯源监管等场景的标准化需求。

在人工智能计算机视觉技术飞速发展的背景下，图像分类与目标检测技术已在农产品识别、物种分类等领域展现出显著优势。从技术演进来看，目标检测算法经历了从 R-CNN 系列的两阶段检测到 YOLO、SSD 等单阶段检测的迭代，图像分类模型也从 ResNet 等基础网络发展到融合注意力机制的增强型架构，为细分类别、复杂背景下的虾类识别提供了成熟的技术支撑。

本项目聚焦 10 类常见经济虾类的智能识别需求，结合图像分类与目标检测技术，构建高效、准确的识别模型。项目成果可应用于水产养殖现场分拣、生鲜电商平台商品核验、市场监管品质检测等场景，有效降低人工成本、提升识别标准化水平，具有重要的实际应用价值和行业参考意义。

### 1.2 简单数据描述

#### 1.2.1 数据集来源与结构

本项目使用的数据集包含两个核心数据目录（Niko 与 Niko2），其中 Niko 目录存储虾类图像数据，Niko2 目录存储对应的 XML 格式标注文件。数据集覆盖 10 类常见经济虾类，具体类别包括：对虾、斑节虾、毛虾、河虾、法国蓝龙虾、波士顿龙虾、牡丹虾、琵琶虾、皮皮虾、鼓虾，涵盖海产、淡水等不同生长环境的代表性虾类物种。

#### 1.2.2 数据规模与格式

- **图像数据**：支持 JPG、JPEG、PNG 三种常见格式，原始图像尺寸不统一，通过后续预处理调整为统一输入尺寸（分类模型 224×224 像素，检测模型 300~640 像素）。
- **标注数据**：采用 XML 格式存储，包含物种类别标签及目标边界框信息（目标检测任务专用），通过解析 XML 文件提取类别名称，确保图像与标签的精准对应。

#### 1.2.3 数据划分方式

为保证模型训练的泛化能力，采用分层抽样方式将数据集划分为训练集、验证集和测试集，划分比例为 7:1.5:1.5，具体如下：

- **训练集**：占比 70%，用于模型参数学习与特征提取，通过数据增强提升模型鲁棒性；
- **验证集**：占比 15%，用于训练过程中的超参数调整与模型性能监控，避免过拟合；
- **测试集**：占比 15%，用于模型最终性能评估，确保评估结果的客观性（测试集数据未参与任何训练过程）。

#### 1.2.4 数据预处理说明

为适配模型输入要求并提升训练效果，对数据进行以下预处理操作：

- **分类任务**：训练集采用随机裁剪、水平/垂直翻转、旋转、颜色抖动、高斯模糊等数据增强策略；验证集与测试集仅进行尺寸调整、中心裁剪和标准化（均值 [0.485, 0.456, 0.406]，标准差 [0.229, 0.224, 0.225]）。
- **检测任务**：统一调整图像尺寸至模型输入规格，保留原始边界框坐标并进行相对坐标转换，确保标注信息与图像尺寸适配。
- **数据校验**：通过图像读取验证、XML 标签解析校验，过滤无效数据，确保数据集质量。

## 2. 人工智能算法模型设计、选择、解决思路和具体方案

### 一、模型选择依据

本项目针对虾类图像识别与目标检测任务，结合数据集特点、任务需求（分类/检测）及硬件资源约束，筛选并优化了多种主流深度学习模型。核心选择依据如下：

- **任务场景适配**：虾类识别需兼顾"类别分类准确性"与"目标定位精度"，因此同时覆盖图像分类模型（DenseNet、ResNet18）和目标检测模型（SSD、YOLOv5/YOLOv8），分别满足"批量分类"和"单图多目标检测"需求。
- **模型性能平衡**：优先选择参数量与精度兼顾的模型，如 DenseNet121、YOLOv5s。
- **可扩展性与改进空间**：选择支持模块化改进的模型，如在 DenseNet、ResNet 中插入 SE 注意力机制，在 YOLO 系列中支持超参数自动优化，便于后续性能迭代。
- **工程实践成熟度**：选用 PyTorch 官方支持、社区活跃的模型（如 torchvision 内置 DenseNet/ResNet、Ultralytics YOLO），确保训练稳定性和部署可行性。

最终选定的核心模型包括：**图像分类模型（DenseNet121、ResNet18）** 和 **目标检测模型（SSD300、YOLOv5s、YOLOv8n）**，以下围绕模型设计、解决思路及实施细节展开说明。

### 二、核心解决思路

本项目的核心解决思路遵循"数据驱动 - 模型适配 - 训练优化 - 迭代评估"的闭环流程，具体逻辑如下：

- **数据层**：先通过数据清洗与增强解决"数据集质量低、样本量不足、分布不均"问题，为模型训练提供可靠输入；
- **模型层**：针对分类/检测任务分别选择适配模型，通过模块化改进（如注意力机制、分类头重构）提升特征提取与任务适配能力；
- **训练层**：采用"预训练权重初始化 + 精细化调参 + 正则化约束"策略，平衡训练效率与泛化能力，避免过拟合；
- **评估层**：建立多维度评估体系（准确率、精确率、召回率、mAP、混淆矩阵），实时监控训练效果并反向优化参数。

### 三、具体实施方案

#### （一）数据预处理方案

数据预处理是模型性能的基础，本项目针对虾类图像的复杂性（光照差异、角度变化、背景干扰）设计了分层处理流程：

**数据清洗：**

- 解析 XML 标注文件获取类别标签，过滤缺失 object 标签或无法读取的 XML 文件；
- 校验图像有效性，剔除无法通过 OpenCV 读取的损坏图像，最终保留有效数据比例 ≥80%；
- 采用分层抽样划分数据集（训练集:验证集:测试集 = 7:1.5:1.5），确保各类别在不同数据集分布一致，避免类别偏倚。

**数据增强：**

- 训练集增强策略（强增强，缓解过拟合）：
  - 几何变换：随机裁剪（scale=0.7~1.3）、水平翻转（p=0.5）、垂直翻转（p=0.2）、随机旋转（±20°）、平移变换（±10%）；
  - 像素级变换：颜色抖动（亮度/对比度/饱和度 ±30%、色相 ±10%）、高斯模糊（kernel_size=3×3，sigma=0.1~1.0）；
- 验证/测试集增强策略（弱增强，保证评估真实性）：仅保留尺寸调整（Resize→CenterCrop）、标准化（均值=[0.485,0.456,0.406]，标准差=[0.229,0.224,0.225]）。

**数据格式适配：**

- 分类任务：将图像转为 RGB 格式，标签映射为整数编码（cls2id 字典）；
- 检测任务：解析 XML 中的边界框坐标，转换为模型所需的相对坐标格式，构建 boxes 和 labels 字典。

#### （二）模型构建方案

##### 1. 图像分类模型（DenseNet/ResNet18）

核心设计思路：基于预训练模型微调，通过注意力机制增强特征提取能力，适配虾类细分类需求。

**DenseNet 系列（核心分类模型）：**

- 结构改进：加载 ImageNet 预训练权重（DenseNet121_Weights.IMAGENET1K_V1），在每个 DenseBlock 后插入 SE 注意力模块（通道数严格匹配各 Block 输出：256/512/1024/1024 等），强化有效特征权重；
- 分类头重构：替换原全连接层为 "Dropout (0.3)+Linear" 结构，初始化采用 Kaiming 正态分布，避免梯度消失；
- 模型选型逻辑：DenseNet 的"特征复用"特性适合小样本场景，能有效利用虾类图像的局部特征（如虾须、虾壳纹理）。

**ResNet18：**

- 结构改进：在 layer4 后插入 SEBlock（通道数 512），增强深层特征的判别能力；替换全连接层适配数据集类别数；
- 选型逻辑：参数量小（约 1100 万），训练速度快，适合硬件资源有限的场景。

##### 2. 目标检测模型（SSD/YOLO 系列）

核心设计思路：针对"单图多虾类目标识别"需求，选择实时性与精度平衡的检测模型，优化锚点与分类头适配虾类目标尺寸。

**SSD300-VGG16：**

- 结构改进：加载预训练权重，重构分类头（输入通道 [512,1024,512,256,256,256]，锚点数量 [4,6,6,6,4,4]），适配 10 类虾类目标；
- 选型逻辑：anchor-based 检测框架，对中等尺寸虾类目标定位准确，训练收敛速度快。

**YOLOv5s/YOLOv8n：**

- 结构改进：选用轻量型主干网络（CSPDarknet），配置适配虾类尺寸的锚点，通过 YAML 文件指定数据集路径、类别数等参数；
- 选型逻辑：端到端检测框架，兼顾实时性与精度，支持自动混合精度训练、超参数进化，适合批量处理含多目标的虾类图像。

#### （三）训练实施方案

##### 1. 训练环境配置

| 配置项 | 具体参数 | 选择依据 |
|--------|----------|----------|
| 硬件设备 | GPU（优先）/CPU | GPU 加速混合精度训练，CPU 适配无显卡环境 |
| 图像尺寸 | 分类任务 224×224，检测任务 300×300（SSD）/320×320（YOLO） | 平衡分辨率与显存占用，适配模型输入要求 |
| 批次大小（Batch Size） | 分类任务 16，检测任务 4 | 基于 GPU 显存动态调整（如 16GB 显存支持 Batch=16，4GB 显存降至 Batch=4），避免 OOM 错误 |
| 迭代次数（Epochs） | 分类任务 30，检测任务 50，YOLO 系列 300 | 分类任务收敛快（30 轮足够），检测任务需更多迭代优化定位精度 |

##### 2. 核心训练组件

- **优化器**：AdamW 优化器（权重衰减 1e-5），相比 SGD 更快收敛，权重衰减抑制过拟合；
- **损失函数**：分类任务采用带标签平滑（label_smoothing=0.1）的交叉熵损失，缓解类别不平衡；检测任务采用 SSD 多任务损失（分类损失 + 边界框回归损失）、YOLO 置信度损失；
- **学习率调度**：ReduceLROnPlateau 策略（mode='max'，factor=0.5，patience=3），基于验证集准确率/损失动态调整，最小学习率 1e-6；
- **正则化策略**：Dropout（比例 0.3）、权重衰减（1e-4~1e-5）、标签平滑，多重抑制过拟合；
- **训练加速**：混合精度训练（torch.amp），在不损失精度的前提下提升训练速度，降低显存占用。

##### 3. 训练流程

1. **数据加载**：解析 XML 标签 → 过滤无效数据 → 分层划分训练/验证/测试集 → 应用数据增强；
2. **模型初始化**：加载预训练权重 → 重构分类头/检测头 → 插入 SE 注意力模块（可选）；
3. **迭代训练**：
   - 训练阶段：前向传播计算损失 → 反向传播更新参数 → 梯度裁剪防止梯度爆炸；
   - 验证阶段：每轮迭代后计算验证集损失与准确率，保存最佳模型（基于验证集性能）；
   - 早停机制（可选）：检测任务设置 patience=8~10，连续多轮无性能提升则停止训练；
4. **模型评估**：训练结束后在测试集上计算准确率、精确率、召回率、F1 分数（分类任务）或 mAP（检测任务），绘制混淆矩阵与训练曲线。

### 四、调参过程与核心技巧

#### （一）关键参数调参过程

##### 1. 学习率（Learning Rate）

- **初始参数**：1e-4（分类任务）、1e-3（检测任务）；
- **调参逻辑**：
  - 若训练损失不下降，说明学习率过低，提升至 2e-4；
  - 若训练损失快速下降但验证损失上升（过拟合），说明学习率过高，降至 5e-5；
- **最终稳定范围**：分类任务 1e-4~2e-4，检测任务 5e-4~1e-3。

##### 2. 正则化参数

- **权重衰减（Weight Decay）**：初始 1e-5，若过拟合（训练准确率远高于验证准确率），提升至 1e-4；若欠拟合，降至 5e-6；
- **Dropout 比例**：分类任务固定 0.3，检测任务 0.2~0.3，比例过高导致欠拟合，过低无法抑制过拟合；
- **标签平滑**：固定 0.1，平衡分类边界的模糊性与判别力。

##### 3. 数据增强强度

- **初始配置**：随机裁剪 scale=(0.7,1.3)、颜色抖动 brightness=0.3；
- **调参逻辑**：
  - 小样本场景（训练集 < 1000 张）：增强强度提升（brightness=0.4，添加随机剪切）；
  - 样本充足场景：降低强度（brightness=0.2），避免数据失真；
- **关键技巧**：验证集不使用强增强，确保评估真实性。

##### 4. 早停与模型保存

- **早停耐心值（Patience）**：分类任务无需早停（30 轮迭代有限），检测任务设置 8~10，避免过度训练；
- **模型保存**：仅保存验证集性能最佳的模型，避免保存过拟合模型。

#### （二）核心调参技巧

- 预训练权重优先：所有模型均加载 ImageNet 预训练权重，初始学习率设置为 1e-4（远低于随机初始化的 1e-2），避免破坏预训练特征；
- 批次大小适配显存：遵循"显存占用≈批次大小 × 图像尺寸 × 通道数 × 参数规模"，若出现 OOM 错误，优先减半批次大小，而非缩小图像尺寸；
- 注意力机制适配：SE 注意力模块的压缩比（reduction）固定 16，通道数严格匹配模型 Block 输出（如 DenseNet121 的 DenseBlock 输出为 256/512 等），避免维度错误；
- 学习率调度时机：ReduceLROnPlateau 的 patience 设置为 3，既不过早调整（patience=1），也不延迟优化（patience>5）；
- 类别不平衡处理：通过分层抽样划分数据集，确保各虾类类别在训练/验证集中比例一致，无需额外加权损失；
- 混合精度训练适配：仅在 GPU 环境启用 torch.amp，CPU 环境禁用，避免兼容性错误；
- 超参数进化（YOLO 系列）：利用 YOLOv5 的 evolve 参数，自动搜索最优超参数（如学习率、数据增强强度），适合复杂数据集。

#### （三）调参避坑指南

- 避免盲目增大批次大小：超过 GPU 显存阈值后，训练速度无提升且易导致梯度消失；
- 学习率调整不频繁：每 3 轮观察验证集性能，再决定是否调整，避免震荡；
- 数据增强不过度：过度模糊（sigma>1.0）或裁剪（scale<0.5）会导致模型学习无效特征；
- 正则化参数不叠加：同时启用 Dropout、权重衰减、标签平滑时，需降低各参数强度，避免欠拟合。

### 五、方案优势总结

- **模型适配性强**：分类与检测模型覆盖不同任务场景，SE 注意力机制针对性提升虾类特征提取能力；
- **训练稳定性高**：混合精度训练、学习率调度、多重正则化确保模型不震荡、不过拟合；
- **调参效率高**：基于硬件资源与数据集规模的动态调参策略，无需盲目尝试；
- **工程化程度高**：代码中包含路径自动创建、无效数据过滤、模型校验等模块，支持直接部署。

## 3. 模型算法实现环境（人工智能框架、是否 GPU 加速）

### 一、环境配置整体概述

本项目模型算法的实现环境围绕"兼容性强、性能高效、部署便捷"的原则搭建，充分适配图像分类（DenseNet/ResNet）与目标检测（SSD/YOLOv5/YOLOv8）两类任务的训练需求，同时兼顾学术研究场景下的硬件资源多样性（有显卡/无显卡环境均可运行）。整体环境分为软件环境（人工智能框架、依赖库等）与硬件环境（训练设备）两大模块，具体配置细节如下。

### 二、软件实现环境

#### （一）核心人工智能框架

本项目采用 **PyTorch** 作为核心深度学习框架，搭配其生态工具完成模型构建、训练与评估，框架选择及版本说明如下：

| 框架名称 | 版本选择 | 核心用途 | 选择依据 |
|----------|----------|----------|----------|
| PyTorch | 1.13.1 / 2.0.1 | 模型构建、正向/反向传播、参数优化、混合精度训练 | 1. 动态计算图特性便于模型模块化改进（如 SE 注意力模块插入），调试灵活，适合学术研究；2. torchvision 内置 DenseNet/ResNet等预训练模型，无需手动实现主干网络；3. 原生支持 CUDA 加速与 torch.amp 混合精度训练，大幅提升训练效率；4. YOLOv5/YOLOv8 官方基于 PyTorch 开发，生态兼容度高，便于检测模型的调优与部署。 |
| torchvision | 0.14.1 / 0.15.2 | 图像预处理、数据增强、预训练模型加载、数据集构建 | 与 PyTorch 版本严格适配，提供 transforms 工具集与 Dataset/DataLoader 接口，完美支撑分类任务的数据流水线搭建。 |
| Ultralytics | 8.0.200+ | YOLOv8 模型训练、评估、可视化 | 官方封装的 YOLO 生态工具，提供简洁 API，支持超参数进化、训练日志自动记录，降低检测模型的使用门槛。 |

#### （二）辅助工具库

除核心框架外，项目依赖多个辅助工具库完成数据处理、可视化、评估等任务，关键依赖清单如下：

| 工具库名称 | 版本 | 核心用途 |
|------------|------|----------|
| OpenCV-Python | 4.7.0.72 | 图像读取、损坏图像过滤、XML 标注文件解析、边界框可视化 |
| NumPy | 1.24.3 | 数值计算、图像数组处理、标签编码与转换 |
| Pandas | 2.0.3 | 数据集划分（分层抽样）、类别分布统计、结果保存 |
| Matplotlib | 3.7.1 | 训练/验证损失曲线绘制、混淆矩阵可视化、准确率趋势图生成 |
| Scikit-learn | 1.2.2 | 分层抽样（train_test_split）、分类指标计算（精确率/召回率/F1 分数） |
| tqdm | 4.65.0 | 训练进度条显示，直观监控迭代过程 |
| Pillow | 9.5.0 | 图像格式转换、精细化数据增强（如随机裁剪） |
| xml.etree.ElementTree | 内置 | XML 标注文件解析，提取目标类别与边界框坐标 |

#### （三）软件环境详细配置

**环境**：Windows 10（64 位）
**优势**：操作界面友好，适合新手调试代码，支持 PyTorch GPU 加速

**Python 版本**：推荐版本 Python 3.13

**依赖安装清单（requirements.txt）**：为便于快速搭建环境，提供完整的依赖配置文件，可通过 `pip install -r requirements.txt` 一键安装：

```txt
# 核心深度学习框架
torch==1.13.1+cu117
torchvision==0.14.1+cu117
ultralytics==8.0.200

# 数据处理与可视化
opencv-python==4.7.0.72
numpy==1.24.3
pandas==2.0.3
matplotlib==3.7.1
scikit-learn==1.2.2
tqdm==4.65.0
pillow==9.5.0

# 可选：GPU加速依赖（自动随PyTorch安装）
# torch-cuda==11.7
```

### 三、硬件训练设备

#### 训练设备（CPU 环境）

项目为 CPU 训练，具体配置与局限性如下：

| 硬件类型 | 最低配置 | 适配场景 | 局限性 |
|----------|----------|----------|--------|
| 中央处理器（CPU） | Intel Core i5-10400 / AMD Ryzen 5 5600G（6 核 12 线程及以上） | 小型数据集分类任务（样本量 < 1000 张）、模型调试、代码验证 | 训练速度慢（ResNet18 训练 30 轮需 2~4 小时，GPU 仅需 10~20 分钟）；无法支持大批次训练（Batch Size 需降至 2~4）；不支持混合精度训练，需禁用 torch.amp。 |
| 内存（RAM） | 16GB DDR4（最低） | 基础数据处理与模型训练 | 易出现内存溢出，需减小图像尺寸（如分类任务改为 112×112）。 |
| 存储设备 | SSD 256GB 及以上 | 小型数据集存储 | 对训练效率影响较小，主要满足数据读取需求。 |

**CPU 环境搭建：**

1. 安装 Python 3.13；
2. 安装 PyTorch CPU 版本：
   ```bash
   pip3 install torch==1.13.1 torchvision==0.14.1 --extra-index-url https://download.pytorch.org/whl/cpu
   ```
3. 安装其他依赖：执行 `pip install -r requirements.txt`；
4. 验证环境：运行上述验证代码，输出 "CUDA is available: False" 即为配置成功。

## 4. 实现步骤和代码

### 4.1 实现步骤总览

项目初始化 → 数据预处理 → 模型构建 → 训练配置 → 模型训练 → 模型评估 → 结果保存

### 4.2 详细实现步骤

#### 4.2.1 分类任务详细实现步骤

**步骤 1：项目初始化与环境准备**

- 创建项目目录结构，划分数据存储、模型保存、代码模块等文件夹；
- 搭建 Python 环境（Python 3.13），安装 PyTorch 及相关依赖（参考 3. 模型算法实现环境）；
- 准备虾类图像数据集与 XML 标注文件，放入指定数据目录。

**步骤 2：数据预处理**

- 数据清洗：遍历图像与 XML 文件，过滤损坏图像、缺失 object 标签的 XML 文件，生成有效数据清单；
- 数据集划分：采用分层抽样法，按 7:1.5:1.5 比例划分训练集、验证集、测试集，确保各类别分布一致；
- 数据增强配置：训练集配置强增强（几何变换 + 像素变换），验证/测试集配置弱增强（仅尺寸调整 + 标准化）；
- 数据加载器构建：自定义 Dataset 类解析数据与标签，通过 DataLoader 实现批量加载与多线程加速。

**步骤 3：模型构建**

- 导入预训练模型（DenseNet121/ResNet18），加载 ImageNet 预训练权重；
- 在模型关键特征层插入 SE 注意力模块，强化虾类特征提取能力；
- 重构分类头，替换全连接层适配虾类类别数，添加 Dropout 层缓解过拟合；
- 初始化分类头参数，将模型移至指定设备（GPU/CPU）。

**步骤 4：训练配置**

- 配置训练参数（批次大小、迭代次数、学习率等）；
- 定义损失函数（带标签平滑的交叉熵损失）、优化器（AdamW）、学习率调度器（ReduceLROnPlateau）；
- 开启混合精度训练（torch.amp），提升训练效率并降低显存占用。

**步骤 5：模型训练与验证**

- 遍历训练轮次，每轮执行训练循环与验证循环；
- 训练循环：加载批量数据 → 前向传播计算损失 → 反向传播更新参数 → 梯度裁剪防止梯度爆炸；
- 验证循环：加载验证集数据 → 前向传播预测 → 计算验证损失与准确率 → 记录训练日志；
- 保存验证集性能最佳的模型权重，触发早停机制（若需）。

**步骤 6：模型评估与保存**

- 加载最佳模型权重，在测试集上进行预测；
- 计算分类任务评估指标（准确率、精确率、召回率、F1 分数），绘制混淆矩阵与训练/验证损失曲线；
- 保存模型权重、评估指标结果与可视化图像，完成分类任务实现。

#### 4.2.2 检测任务详细实现步骤

**步骤 1：前期准备**

- 继承分类任务的项目目录与环境配置，补充 YOLOv5/YOLOv8 专属依赖；
- 将 XML 标注文件转换为 YOLO 格式（txt 文件，存储相对坐标与类别 ID）；
- 编写 YOLO 数据集配置文件（YAML），指定数据集路径、类别数与类别名称。

**步骤 2：模型配置**

- 选择 YOLOv5s/YOLOv8s 轻量型模型，下载官方预训练权重；
- 调整模型锚点尺寸，适配虾类目标大小；
- 配置训练超参数（批次大小、迭代次数、学习率等），开启超参数进化（可选）。

**步骤 3：模型训练**

- 调用 YOLO 训练 API，传入配置文件、数据集路径与训练参数；
- 开启训练过程监控，实时查看训练损失、mAP 等指标变化；
- 训练结束后，自动保存最佳模型权重与训练日志。

**步骤 4：模型评估与可视化**

- 加载训练完成的模型，在测试集上进行目标检测；
- 计算检测任务评估指标（mAP@0.5、mAP@0.5:0.95）；
- 可视化检测结果（绘制边界框与类别标签），生成检测报告；
- 保存检测模型权重、评估指标与可视化结果。

### 4.2.3 代码部分

#### 1. SSD 核心训练代码

```python
# 数据集类
class SSDDataset(Dataset):
    def __init__(self, data_list, transform=None):
        self.data_list = data_list  # 包含图像路径和标注信息
        self.transform = transform

    def __len__(self):
        return len(self.data_list)

    def __getitem__(self, idx):
        img_path, annotations = self.data_list[idx]
        img = cv2.imread(str(img_path))
        img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
        
        # 处理标注（边界框和类别）
        boxes = annotations['boxes']
        labels = annotations['labels']
        
        if self.transform:
            img, boxes, labels = self.transform(img, boxes, labels)
        
        # 转换为SSD需要的目标格式（张量）
        boxes = torch.tensor(boxes, dtype=torch.float32)
        labels = torch.tensor(labels, dtype=torch.long)
        return img, boxes, labels

# 训练核心循环
def train_ssd():
    # 数据加载
    train_dataset = SSDDataset(train_data_list, train_transform)
    train_loader = DataLoader(train_dataset, batch_size=BATCH_SIZE, shuffle=True)
    
    # 模型初始化
    model = SSD300(num_classes=NUM_CLASSES).to(DEVICE)
    criterion = SSDLoss()  # SSD专用损失函数（含分类和定位损失）
    optimizer = optim.SGD(model.parameters(), lr=0.001, momentum=0.9, weight_decay=5e-4)
    
    # 训练循环
    for epoch in range(EPOCHS):
        model.train()
        total_loss = 0.0
        for imgs, boxes, labels in train_loader:
            imgs, boxes, labels = imgs.to(DEVICE), boxes.to(DEVICE), labels.to(DEVICE)
            
            optimizer.zero_grad()
            # 前向传播：获取预测的边界框和类别分数
            preds = model(imgs)
            # 计算损失（分类损失+定位损失）
            loss = criterion(preds, boxes, labels)
            
            loss.backward()
            optimizer.step()
            total_loss += loss.item()
        
        # 验证与模型保存
        val_loss = evaluate_ssd(model, val_loader, criterion)
        print(f"Epoch {epoch}: Train Loss={total_loss/len(train_loader)}, Val Loss={val_loss}")
        if val_loss < best_val_loss:
            torch.save(model.state_dict(), "best_ssd.pt")
            best_val_loss = val_loss
```

#### 2. YOLOv5 核心训练代码

```python
class EnhancedYOLOv5Trainer:
    def train(self):
        # 环境与数据集设置
        self.setup_environment()
        self.analyze_dataset()
        self.auto_optimize()  # 自动优化超参数、批次大小等
        
        # 模型加载
        self.model = Model(self.config.get('cfg') or weights,
                          ch=3,
                          nc=self.dataset_analyzer.analysis.get('num_classes', 80))
        if weights and os.path.exists(weights):
            ckpt = torch.load(weights, map_location='cpu')
            csd = ckpt['model'].float().state_dict()
            self.model.load_state_dict(csd, strict=False)
        
        # 训练准备
        self.setup_training()
        self.monitor = YOLOv5Monitor(self.save_dir)  # 训练监控器
        self.start_time = time.time()
        
        # 核心训练循环
        for epoch in range(1, self.config.get('epochs', 100) + 1):
            # 训练阶段
            self.model.train()
            train_loss, box_loss, obj_loss, cls_loss = 0.0, 0.0, 0.0, 0.0
            for imgs, targets in train_loader:
                imgs, targets = imgs.to(self.device), targets.to(self.device)
                optimizer.zero_grad()
                
                # 前向传播：获取损失
                loss, loss_items = self.model(imgs, targets)  # YOLOv5损失直接在模型中计算
                loss.backward()
                optimizer.step()
                
                # 累计损失
                train_loss += loss.item()
                box_loss += loss_items[0].item()
                obj_loss += loss_items[1].item()
                cls_loss += loss_items[2].item()
            
            # 验证阶段
            val_results = self.validate()  # 计算验证集mAP等指标
            
            # 监控与更新
            results = {
                'train/loss': train_loss/len(train_loader),
                'train/box_loss': box_loss/len(train_loader),
                'val/loss': val_results['loss'],
                'metrics/mAP_0.5:0.95': val_results['mAP_0.5:0.95']
            }
            self.monitor.update_epoch(epoch, results, optimizer.param_groups[0]['lr'])
            
            # 保存最佳模型
            if self.monitor.update_best(val_results['mAP_0.5:0.95'], epoch):
                torch.save(self.model.state_dict(), self.save_dir / 'best.pt')
        
        # 训练后处理
        self.monitor.close()
        self.generate_report()
```

#### 3. YOLOv8 核心训练代码

```python
class YOLOv8Trainer:
    def __init__(self, config):
        self.config = config
        self.device = select_device(config['device'])
        self.model = YOLOv8Model(config['cfg'], num_classes=config['num_classes']).to(self.device)
        self.loss_fn = YOLOv8Loss()  # 改进的损失函数（含CIoU等）
        
    def train_loop(self):
        # 数据加载
        train_loader = self.get_data_loader('train')
        val_loader = self.get_data_loader('val')
        
        # 优化器与调度器
        optimizer = optim.AdamW(self.model.parameters(), lr=0.002, weight_decay=0.0005)
        scheduler = CosineAnnealingLR(optimizer, T_max=self.config['epochs'])
        
        best_map = 0.0
        for epoch in range(self.config['epochs']):
            self.model.train()
            total_loss = 0.0
            
            for batch in train_loader:
                imgs, targets = batch['img'].to(self.device), batch['targets'].to(self.device)
                optimizer.zero_grad()
                
                # 前向传播：输出预测框、分数
                outputs = self.model(imgs)
                loss = self.loss_fn(outputs, targets)
                
                loss.backward()
                optimizer.step()
                total_loss += loss.item()
            
            # 验证（计算mAP）
            val_map = self.evaluate(val_loader)
            scheduler.step()
            
            # 保存最佳模型
            if val_map > best_map:
                best_map = val_map
                torch.save(self.model.state_dict(), f"yolov8_best_{epoch}.pt")
            
            print(f"Epoch {epoch}: Loss={total_loss/len(train_loader)}, mAP={val_map:.4f}")
```

#### 4. ResNet18 核心训练代码

```python
# SE注意力模块（可集成到ResNet18）
class SEBlock(nn.Module):
    def __init__(self, channel, reduction=16):
        super().__init__()
        self.avg_pool = nn.AdaptiveAvgPool2d(1)
        self.fc = nn.Sequential(
            nn.Linear(channel, channel // reduction, bias=False),
            nn.ReLU(inplace=True),
            nn.Linear(channel // reduction, channel, bias=False),
            nn.Sigmoid()
        )

    def forward(self, x):
        b, c, _, _ = x.size()
        y = self.avg_pool(x).view(b, c)
        y = self.fc(y).view(b, c, 1, 1)
        return x * y.expand_as(x)

# 模型构建
def build_resnet18(num_classes, use_se=True):
    model = models.resnet18(pretrained=True)
    if use_se:
        model.layer4.add_module('se', SEBlock(512))  # 在layer4后添加SE模块
    model.fc = nn.Linear(model.fc.in_features, num_classes)  # 替换分类头
    return model.to(DEVICE)

# 训练核心
def train_resnet18():
    # 数据加载
    train_dataset = ShrimpDataset(train_data, train_transform)
    train_loader = DataLoader(train_dataset, batch_size=BATCH_SIZE, shuffle=True)
    val_loader = DataLoader(val_dataset, batch_size=BATCH_SIZE)
    
    # 模型与训练组件
    model = build_resnet18(num_classes=len(train_dataset.classes))
    criterion = nn.CrossEntropyLoss(label_smoothing=0.1)
    optimizer = optim.AdamW(model.parameters(), lr=0.001, weight_decay=1e-5)
    scheduler = ReduceLROnPlateau(optimizer, mode='max', factor=0.5, patience=3)
    
    best_val_acc = 0.0
    for epoch in range(EPOCHS):
        # 训练阶段
        model.train()
        train_correct, train_total = 0, 0
        for imgs, labels in train_loader:
            imgs, labels = imgs.to(DEVICE), labels.to(DEVICE)
            optimizer.zero_grad()
            
            outputs = model(imgs)
            loss = criterion(outputs, labels)
            loss.backward()
            optimizer.step()
            
            # 统计准确率
            _, preds = torch.max(outputs, 1)
            train_total += labels.size(0)
            train_correct += (preds == labels).sum().item()
        
        # 验证阶段
        val_acc = evaluate(model, val_loader)
        scheduler.step(val_acc)
        
        # 保存最佳模型
        if val_acc > best_val_acc:
            best_val_acc = val_acc
            torch.save(model.state_dict(), "resnet18_best.pt")
        
        print(f"Epoch {epoch}: Train Acc={train_correct/train_total:.4f}, Val Acc={val_acc:.4f}")
```

#### 5. DenseNet 核心训练代码

```python
# 数据集处理
class DenseNetDataset(Dataset):
    def __init__(self, data_list, transform=None):
        self.data_list = self._filter_invalid_data(data_list)  # 过滤无效数据
        self.transform = transform
        self.classes = sorted(list(set([cls for _, cls in self.data_list])))
        self.cls2id = {cls: i for i, cls in enumerate(self.classes)}

    def _filter_invalid_data(self, data_list):
        valid_data = []
        for img_path, cls_name in data_list:
            try:
                if cv2.imread(str(img_path)) is not None:
                    valid_data.append((img_path, cls_name))
            except:
                continue
        return valid_data

    def __len__(self):
        return len(self.data_list)

    def __getitem__(self, idx):
        img_path, cls_name = self.data_list[idx]
        img = cv2.imread(str(img_path))
        img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
        label = self.cls2id[cls_name]
        if self.transform:
            img = self.transform(img)
        return img, torch.tensor(label, dtype=torch.long)

# 训练核心
def train_densenet():
    # 数据增强
    train_transform, val_transform = get_transforms()  # 复用ResNet的数据增强策略
    train_dataset = DenseNetDataset(train_data_list, train_transform)
    val_dataset = DenseNetDataset(val_data_list, val_transform)
    
    train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True)
    val_loader = DataLoader(val_dataset, batch_size=32)
    
    # 加载DenseNet模型
    model = models.densenet121(pretrained=True)
    num_ftrs = model.classifier.in_features
    model.classifier = nn.Linear(num_ftrs, len(train_dataset.classes))  # 替换分类头
    model = model.to(DEVICE)
    
    # 训练组件
    criterion = nn.CrossEntropyLoss()
    optimizer = optim.SGD(model.parameters(), lr=0.001, momentum=0.9)
    scheduler = StepLR(optimizer, step_size=7, gamma=0.1)
    
    # 训练循环
    for epoch in range(25):
        model.train()
        running_loss = 0.0
        for imgs, labels in train_loader:
            imgs, labels = imgs.to(DEVICE), labels.to(DEVICE)
            optimizer.zero_grad()
            
            outputs = model(imgs)
            loss = criterion(outputs, labels)
            loss.backward()
            optimizer.step()
            
            running_loss += loss.item() * imgs.size(0)
        
        # 验证
        val_acc = evaluate(model, val_loader)
        scheduler.step()
        
        print(f"Epoch {epoch}: Loss={running_loss/len(train_dataset)}, Val Acc={val_acc:.4f}")
```
