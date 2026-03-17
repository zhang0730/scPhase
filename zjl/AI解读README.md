# scPhase: 基于注意力增强的表示学习探索表型相关的单细胞



## 项目概述

**scPhase** 是一个深度学习框架，旨在从单细胞RNA测序（scRNA-seq）数据中预测临床表型。它将每个患者样本视为单细胞的“包”，并使用多重实例学习（MIL）方法来学习其基因表达谱的全面表示。



## 主要特性

- **实例编码器**：使用多层感知机学习丰富的基因表示。
- **LinFormer 注意力模块**：高效捕捉细胞间关系。
- **基于混合专家（MoE）的MIL聚合层**：稳健地加权细胞重要性。
- **域适应模块**：通过梯度反转层减轻批次效应。
- **内置可解释性框架**：使用细胞注意力和基因归因分数（通过综合梯度）识别驱动表型预测的关键细胞和基因。



## 工作流程概述

1. **数据准备**：将scRNA-seq数据格式化为所需的`.h5ad`文件。
2. **配置**：编辑`config.json`文件以指定路径和模型参数。
3. **执行**：从命令行运行交叉验证和解释分析。
4. **结果解释**：包括基因归因、通路富集、细胞注意力分析和联合分析等一系列下游生物分析，以链接表型的细胞和分子驱动因素。



## 安装

1. 克隆仓库到本地机器：

   ```bash
   git clone https://github.com/wuqinhua/scPhase.git
   cd scPhase
   ```

2. 从`requirements.txt`文件安装所需的Python包：

   ```bash
   pip install -r requirements.txt
   ```



## 数据集要求

scPhase要求您的单细胞数据以预处理的**`.h5ad`文件**提供。该文件中的`AnnData`对象必须在其`.obs`属性中包含特定的元数据。

### 必需的`AnnData`结构

您的`AnnData`对象的`.obs`DataFrame必须包括以下列：

- **样本ID**：标识每个细胞所属的样本或患者的列。默认期望名称为`sample_id`。
- **表型**：包含每个样本的临床结果的列。可以是分类标签（用于分类）或连续值（用于回归）。默认期望名称为`phenotype`。
- **批次**：指示每个样本的批次、研究或队列的列。用于域适应模块以校正技术变异。默认期望名称为`batch`。

您可以在`config.json`文件的`data_params`部分轻松自定义这些默认列名。

### 推荐的预处理工作流程

1. **特征选择**：模型期望固定数量的输入特征。首先，从您的数据集中识别一组高度可变基因（HVGs）。模型的默认输入维度为**5000个HVGs**。您的最终`adata.X`矩阵应仅包含这些基因。
2. **细胞类型注释**：为您的细胞注释其相应的类型（例如，T细胞、B细胞）对于模型结果的下游解释至关重要。这些注释应存储在`.obs`列中。
3. **UMAP计算**：虽然不用于训练，但计算UMAP嵌入对于可视化解释阶段生成的细胞注意力分数是必要的。结果应存储在`adata.obsm['X_umap']`中。



## 数据可用性

所有处理过的数据，包括训练好的模型和可解释性指标，已存放在Zenodo，可通过以下链接访问：DOI: 10.5281/zenodo.18059900                



## 使用方法

整个工作流程由`run.py`脚本和一个中心的`config.json`文件控制。

### 步骤1：配置您的实验

在运行之前，编辑`config.json`以匹配您的数据和实验设置。需要更改的最重要参数包括：

| 参数             | 部分           | 描述                                                |
| ---------------- | -------------- | --------------------------------------------------- |
| `data_h5ad_file` | `path_params`  | **必需**：输入`.h5ad`数据文件的绝对路径。           |
| `RESULTS_DIR`    | `path_params`  | **必需**：保存所有输出文件的目录路径。              |
| `MODEL_NAME`     | `path_params`  | 您的实验名称（例如，“COVID19”，“Aging”）。          |
| `sample_col`     | `data_params`  | `.obs`中样本ID的列名。                              |
| `label_col`      | `data_params`  | `.obs`中表型标签的列名。                            |
| `batch_col`      | `data_params`  | `.obs`中批次/队列信息的列名。                       |
| `device_model`   | `run_params`   | 模型的主要GPU（例如，“cuda:0”）。                   |
| `device_encoder` | `run_params`   | 注意力编码器的次要GPU（可以与`device_model`相同）。 |
| `task_type`      | `run_params`   | 预测任务类型：`classification`或 `regression`。     |
| `n_classes`      | `model_params` | 预测输出类别的数量。例如，3类分类任务为3，回归为1。 |

### 步骤2：运行工作流程

使用命令行执行交叉验证（训练）或解释分析。

#### 运行交叉验证训练：

此命令使用`run_cv.py`中定义的交叉验证策略训练模型。如果检测到多个批次，则使用留一分组交叉验证；否则，使用k折交叉验证。

```bash
python run.py --config config.json --cv
```

#### 运行解释分析：

此命令必须在训练完成后运行。它加载每个折叠保存的模型并计算集成可解释性结果。

```bash
python run.py --config config.json --interpret
```





## 解读输出

所有结果都保存在`RESULTS_DIR`指定的目录中。关键输出包括：

- **训练好的模型**：每个CV折叠的最佳模型保存为`BestModel_{MODEL_NAME}_Fold{N}.pt`。
- **性能指标**：
  - `AllFolds_{MODEL_NAME}.csv`：每个验证折叠的详细性能指标。
  - `Summary_{MODEL_NAME}.csv`：所有折叠的聚合性能（均值 ± 标准差）。
- **可解释性结果**：
  - `ensemble_gene_attributions_{phenotype}.csv`/ `.pdf`：与每个表型相关的前排名基因的表格和条形图。
  - `sample_gene_attribution_mean.csv`：跨折叠的平均归因分数的样本-by-基因矩阵。
  - `ensemble_adata_with_attention.h5ad`：一个`AnnData`对象，其中包含所有折叠的平均细胞注意力分数，保存在`.obs['attention_weight_mean']`中。
  - `UMAP Plots (*.pdf)`：按细胞类型和细胞注意力着色的UMAP可视化，揭示模型用于其预测的细胞。