# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 项目概述

Qlib 是微软开发的面向AI的量化投资平台。它为量化投资提供了完整的机器学习流水线，包括数据处理、模型训练、回测和投资组合优化。该平台支持多种机器学习范式，包括监督学习、市场动态建模和强化学习。

## 开发命令

### 环境设置

```bash
# 以可编辑/开发模式安装（包含 pytest, statsmodels）
make dev

# 或分步设置
pip install numpy cython
make prerequisite  # 构建 Cython 扩展 (rolling.pyx, expanding.pyx)
make dependencies  # 以可编辑模式安装 qlib
```

### 测试

```bash
# 使用 pytest 运行测试（在 make dev 之后）
pytest tests/

# 运行特定测试文件
pytest tests/test_workflow.py

# 运行特定测试
pytest tests/test_workflow.py::test_specific_function -v
```

### 代码质量检查

```bash
# 使用 black 格式化代码（行长度 120）
make black

# 使用 pylint 检查
make pylint

# 使用 flake8 检查
make flake8

# 使用 mypy 检查
make mypy

# 使用 nbqa 检查 Jupyter notebooks
make nbqa

# 运行所有 lint 检查
make lint
```

### 构建和打包

```bash
# 构建 wheel 包
make build

# 上传到 PyPI
make upload
```

### 文档

```bash
# 使用 Sphinx 生成 HTML 文档
make docs-gen
```

## 架构概览

Qlib 采用分层架构，每个组件都设计为松耦合的模块，可以独立使用。

### 核心层

1. **数据层** (`qlib/data/`)
   - **提供者（Providers）**: `CalendarProvider`（日历）, `InstrumentProvider`（股票代码）, `FeatureProvider`（特征）, `ExpressionProvider`（表达式）, `DatasetProvider`（数据集）
   - **本地提供者**: 基于文件的数据访问（`LocalFeatureProvider`, `LocalPITProvider`）
   - **客户端提供者**: 远程数据服务器访问
   - **缓存**: `ExpressionCache`, `DatasetCache` 用于性能优化
   - **存储**: 为量化数据优化的二进制文件存储格式
   - **数据集** (`qlib/data/dataset/`): `DatasetH`（带处理器的数据集）, `DataHandler`, `DataHandlerLP` 用于数据预处理

2. **模型层** (`qlib/model/`)
   - `Model` 所有预测模型的基类
   - `trainer.py`: 带 MLflow 实验跟踪的训练流水线
   - `ens/`: 集成方法
   - `meta/`: 市场动态的元学习
   - `riskmodel/`: 风险建模

3. **工作流层** (`qlib/workflow/`)
   - 与 MLflow 集成的实验管理
   - `R` (Recorder): 用于保存模型、指标、工件的实验记录器
   - `ExpManager`: 实验管理器
   - `online/`: 在线服务和模型滚动

4. **策略层** (`qlib/strategy/`)
   - 交易策略基类
   - 从模型预测生成信号

5. **回测层** (`qlib/backtest/`)
   - `Exchange`: 市场模拟
   - `Executor`: 订单执行模拟
   - `Account`: 投资组合会计
   - `backtest_loop`: 主回测引擎
   - 支持多级策略的嵌套执行

6. **强化学习** (`qlib/rl/`)
   - 订单执行的 RL 环境
   - 与 Tianshou 库集成
   - `trainer/`: RL 训练流水线

7. **贡献层** (`qlib/contrib/`)
   - **模型库** (`contrib/model/`): 预实现的模型（LightGBM, LSTM, Transformer, GATs 等）
   - **数据处理器** (`contrib/data/handler.py`): Alpha158, Alpha360 数据集处理器
   - **策略** (`contrib/strategy/`): TopkStrategy, SoftTopkStrategy
   - **报告** (`contrib/report/`): 分析和可视化

### 数据流

1. 原始数据 → **数据提供者** → 特征表达式
2. 特征 → **DataHandler** → 预处理特征（如 Alpha158）
3. 预处理数据 → **Dataset** → 训练/推理数据
4. 数据集 → **模型** → 预测
5. 预测 → **策略** → 交易决策
6. 决策 → **回测** → 性能指标

### 关键设计模式

- **表达式系统**: 金融因子表达为表达式（如 `Ref($close, 1)`, `Mean($close, 3)`）并即时计算
- **提供者模式**: 可插拔的数据源（本地文件、远程服务器）
- **处理器模式**: 可配置的数据预处理流水线
- **记录器模式**: 使用 MLflow 自动实验跟踪

## 配置系统

Qlib 使用 YAML 配置文件进行工作流编排。`qrun` 命令执行工作流：

```bash
# 从配置文件运行工作流
qrun examples/benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml

# 或通过编程方式
python qlib/cli/run.py examples/benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml
```

### 典型工作流配置结构

```yaml
# 系统配置
sys:
  path: []  # 额外的 Python 路径

# Qlib 初始化
exp_manager:
  class: "MLflowExpManager"
  module_path: "qlib.workflow.expm"

# 数据集配置
dataset:
  class: "DatasetH"
  module_path: "qlib.data.dataset"
  kwargs:
    handler:
      class: "Alpha158"
      module_path: "qlib.contrib.data.handler"

# 模型配置
model:
  class: "LGBModel"
  module_path: "qlib.contrib.model.gbdt"

# 策略配置
strategy:
  class: "TopkStrategy"
  module_path: "qlib.contrib.strategy.strategy_topk"
```

## 代码规范

- **文档字符串**: 使用 Numpydoc 风格
- **行长度**: 最大 120 字符（由 black 强制执行）
- **类型提示**: 鼓励但不严格强制
- **测试**: 在 `tests/` 目录中使用 pytest 编写测试

## Qlib 详细使用方法

### 1. 数据准备

#### 下载示例数据

```bash
# 下载中国市场日频数据
python scripts/get_data.py qlib_data --name qlib_data_simple --target_dir ~/.qlib/qlib_data/cn_data --interval 1d --region cn

# 下载美国市场数据
python scripts/get_data.py qlib_data --name qlib_data_us_1d_latest --target_dir ~/.qlib/qlib_data/us_data --interval 1d --region us
```

#### 初始化 Qlib

```python
import qlib
from qlib.constant import REG_CN

# 初始化 Qlib（使用本地数据）
qlib.init(provider_uri='~/.qlib/qlib_data/cn_data', region=REG_CN)
```

### 2. 快速开始：完整的量化投资流程

#### 方法一：使用配置文件

```bash
# 运行 LightGBM 模型的完整工作流
cd examples
qrun benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml
```

#### 方法二：使用 Python 代码

```python
import qlib
from qlib.constant import REG_CN
from qlib.utils import init_instance_by_config
from qlib.workflow import R
from qlib.workflow.record_temp import SignalRecord, PortAnaRecord
from qlib.tests.data import GetData

# 初始化
GetData().qlib_data(target_dir='~/.qlib/qlib_data/cn_data', region=REG_CN, exists_skip=True)
qlib.init(provider_uri='~/.qlib/qlib_data/cn_data', region=REG_CN)

# 定义任务配置
market = "csi300"
benchmark = "SH000300"

# 数据处理器配置
data_handler_config = {
    "start_time": "2008-01-01",
    "end_time": "2020-08-01",
    "fit_start_time": "2008-01-01",
    "fit_end_time": "2014-12-31",
    "instruments": market,
}

# 任务配置
task = {
    "model": {
        "class": "LGBModel",
        "module_path": "qlib.contrib.model.gbdt",
        "kwargs": {
            "loss": "mse",
            "colsample_bytree": 0.8879,
            "learning_rate": 0.0421,
            "subsample": 0.8789,
            "lambda_l1": 205.6999,
            "lambda_l2": 580.9768,
            "max_depth": 8,
            "num_leaves": 210,
            "num_threads": 20,
        },
    },
    "dataset": {
        "class": "DatasetH",
        "module_path": "qlib.data.dataset",
        "kwargs": {
            "handler": {
                "class": "Alpha158",
                "module_path": "qlib.contrib.data.handler",
                "kwargs": data_handler_config,
            },
            "segments": {
                "train": ("2008-01-01", "2014-12-31"),
                "valid": ("2015-01-01", "2016-12-31"),
                "test": ("2017-01-01", "2020-08-01"),
            },
        },
    },
}

# 训练模型
with R.start(experiment_name="workflow"):
    R.log_params(**flatten_dict(task))
    model = init_instance_by_config(task["model"])
    dataset = init_instance_by_config(task["dataset"])

    # 训练
    model.fit(dataset)

    # 预测
    recorder = R.get_recorder()
    sr = SignalRecord(model, dataset, recorder)
    sr.generate()

    # 回测
    par = PortAnaRecord(recorder, port_analysis_config, "day")
    par.generate()
```

### 3. 数据处理

#### 使用内置数据处理器

```python
from qlib.data.dataset import DatasetH
from qlib.data.dataset.handler import DataHandlerLP

# Alpha158: 158个技术指标特征
handler_config = {
    "start_time": "2008-01-01",
    "end_time": "2020-08-01",
    "fit_start_time": "2008-01-01",
    "fit_end_time": "2014-12-31",
    "instruments": "csi300",
}

# 创建数据集
dataset = DatasetH(
    handler={
        "class": "Alpha158",
        "module_path": "qlib.contrib.data.handler",
        "kwargs": handler_config,
    },
    segments={
        "train": ("2008-01-01", "2014-12-31"),
        "valid": ("2015-01-01", "2016-12-31"),
        "test": ("2017-01-01", "2020-08-01"),
    },
)

# 获取数据
df_train, df_valid = dataset.prepare(["train", "valid"], col_set=["feature", "label"])
```

#### 自定义特征表达式

```python
from qlib.data import D

# 使用表达式计算特征
instruments = D.instruments(market='csi300')
fields = [
    "Ref($close, 1)/$close - 1",  # 昨日收益率
    "Mean($close, 5)/$close - 1",  # 5日均线偏离度
    "$volume/$volume.rolling(5).mean() - 1",  # 成交量相对强度
]

df = D.features(instruments, fields, start_time='2020-01-01', end_time='2020-12-31')
```

### 4. 模型训练

#### 使用内置模型

```python
from qlib.contrib.model.gbdt import LGBModel

# LightGBM 模型
model = LGBModel(
    loss="mse",
    learning_rate=0.05,
    max_depth=8,
    num_leaves=210,
)

# 训练
model.fit(dataset)

# 预测
pred = model.predict(dataset)
```

#### 使用深度学习模型

```python
from qlib.contrib.model.pytorch_lstm import LSTMModel

# LSTM 模型
model = LSTMModel(
    d_feat=158,  # 特征维度
    hidden_size=64,
    num_layers=2,
    dropout=0.0,
    n_epochs=100,
    lr=0.001,
    batch_size=2000,
)

model.fit(dataset)
pred = model.predict(dataset)
```

### 5. 回测和评估

#### 基本回测

```python
from qlib.contrib.strategy import TopkDropoutStrategy
from qlib.contrib.evaluate import backtest
from qlib.contrib.evaluate import risk_analysis

# 策略配置
strategy_config = {
    "topk": 50,
    "n_drop": 5,
}

# 回测配置
backtest_config = {
    "start_time": "2017-01-01",
    "end_time": "2020-08-01",
    "account": 100000000,
    "benchmark": "SH000300",
    "exchange_kwargs": {
        "freq": "day",
        "limit_threshold": 0.095,
        "deal_price": "close",
        "open_cost": 0.0005,
        "close_cost": 0.0015,
        "min_cost": 5,
    },
}

# 运行回测
strategy = TopkDropoutStrategy(**strategy_config)
report, positions = backtest(pred, strategy=strategy, **backtest_config)

# 分析结果
analysis = risk_analysis(report)
print(analysis)
```

#### 高级回测（使用 Executor）

```python
from qlib.backtest import backtest, executor
from qlib.contrib.strategy import TopkDropoutStrategy

# 创建策略
strategy = TopkDropoutStrategy(
    signal=pred,
    topk=50,
    n_drop=5,
)

# 创建执行器
executor_config = {
    "class": "SimulatorExecutor",
    "module_path": "qlib.backtest.executor",
    "kwargs": {
        "time_per_step": "day",
        "generate_portfolio_metrics": True,
    },
}

# 运行回测
portfolio_metric_dict, indicator_dict = backtest(
    strategy=strategy,
    executor=executor_config,
    start_time="2017-01-01",
    end_time="2020-08-01",
    account=100000000,
    benchmark="SH000300",
)
```

### 6. 在线服务和模型更新

#### 设置在线服务

```python
from qlib.workflow.online.manager import OnlineManager

# 在线管理器配置
online_config = {
    "tasks": [
        {
            "class": "OnlineStrategyTask",
            "module_path": "qlib.workflow.online.strategy",
        }
    ],
    "trainer": {
        "class": "OnlineTrainer",
        "module_path": "qlib.workflow.online.trainer",
    },
}

# 启动在线服务
online_manager = OnlineManager(online_config)
online_manager.run()
```

### 7. 实验管理（MLflow）

```python
from qlib.workflow import R

# 开始实验
with R.start(experiment_name="my_experiment"):
    # 记录参数
    R.log_params(learning_rate=0.01, max_depth=8)

    # 训练模型
    model.fit(dataset)

    # 记录指标
    R.log_metrics(train_loss=0.5, valid_loss=0.6)

    # 保存模型
    R.save_objects(model=model)

    # 保存预测结果
    R.save_objects(pred=pred)

# 查看实验结果
recorder = R.get_recorder()
print(recorder.list_metrics())
```

### 8. 常见使用场景

#### 场景1：因子挖掘

```python
# 定义多个候选因子
factors = [
    "($close-Ref($close,1))/Ref($close,1)",  # 收益率
    "($high-$low)/$open",  # 波动率
    "Corr($close, $volume, 10)",  # 价量相关性
    "Rank(Mean($close, 5))",  # 均线排名
]

# 批量测试因子
for factor in factors:
    df = D.features(instruments, [factor], start_time, end_time)
    # 计算因子IC
    ic = calculate_ic(df, returns)
    print(f"Factor: {factor}, IC: {ic}")
```

#### 场景2：模型集成

```python
from qlib.model.ens.ensemble import AverageEnsemble

# 训练多个模型
models = []
for model_config in model_configs:
    model = init_instance_by_config(model_config)
    model.fit(dataset)
    models.append(model)

# 集成预测
ensemble = AverageEnsemble(models)
pred = ensemble.predict(dataset)
```

#### 场景3：滚动训练

```python
from qlib.contrib.rolling import Rolling

# 滚动训练配置
rolling_config = {
    "train_start_date": "2008-01-01",
    "train_end_date": "2014-12-31",
    "validate_start_date": "2015-01-01",
    "validate_end_date": "2016-12-31",
    "test_start_date": "2017-01-01",
    "test_end_date": "2020-08-01",
    "rolling_period": 60,  # 每60天重新训练
}

# 执行滚动训练
rolling = Rolling(**rolling_config)
rolling.run()
```

## 运行示例

```bash
# 进入示例目录
cd examples

# 运行 LightGBM 工作流
qrun benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml

# 通过代码运行（更灵活）
python workflow_by_code.py

# 运行 Jupyter notebook 教程
jupyter notebook tutorial/
```

## 重要实现注意事项

### Cython 扩展
- `qlib/data/_libs/rolling.pyx` 和 `expanding.pyx` 必须在使用前编译
- 运行 `make prerequisite` 来构建这些扩展
- 这些扩展为滚动窗口操作提供了高性能实现

### 数据初始化
- 首次使用前必须下载数据：`python scripts/get_data.py qlib_data ...`
- 数据存储在 `~/.qlib/qlib_data/` 中
- 支持多个区域：cn（中国）、us（美国）、in（印度）等

### 性能优化
- 使用 `expression_cache` 和 `dataset_cache` 来加速重复计算
- 调整 `kernels` 参数以控制并行工作进程数
- 对于高频数据，设置 `maxtasksperchild=1`

### Windows 平台注意事项
- **重要**: 如果 Windows 用户名包含中文字符，系统会自动处理 Unicode 编码问题
- 修复已在 `qlib/utils/paral.py` 中实现，会自动创建 ASCII 临时目录
- 默认使用 `loky` 后端而不是 `multiprocessing` 以获得更好的兼容性

## 常见开发任务

### 添加新模型
1. 创建继承自 `qlib.model.base.Model` 的模型类
2. 实现 `fit()` 和 `predict()` 方法
3. 添加到 `qlib/contrib/model/` 或 `examples/benchmarks/`
4. 创建工作流配置 YAML 文件

### 添加新数据处理器
1. 创建继承自 `DataHandler` 或 `DataHandlerLP` 的处理器类
2. 在处理器中定义特征集
3. 添加到 `qlib/contrib/data/handler.py`

### 添加新策略
1. 创建继承自 `BaseStrategy` 的策略类
2. 实现 `generate_trade_decision()` 方法
3. 添加到 `qlib/contrib/strategy/`

## 需要理解的关键文件

- `qlib/__init__.py`: 初始化和配置
- `qlib/config.py`: 全局配置管理
- `qlib/data/data.py`: 数据提供者和 D facade API
- `qlib/data/dataset/handler.py`: DataHandler 类
- `qlib/model/trainer.py`: 训练流水线
- `qlib/workflow/__init__.py`: 记录器和实验管理
- `qlib/backtest/backtest.py`: 主回测循环
- `qlib/utils/paral.py`: 并行处理和 Unicode 修复

## 依赖项

- **核心**: numpy, pandas, lightgbm, mlflow, redis, filelock
- **可选**: torch（用于 RL）, tianshou（用于 RL）, plotly（用于分析）
- **开发**: pytest, black, pylint, mypy, flake8, nbqa

查看 `pyproject.toml` 获取完整的依赖列表和可选依赖组（`[dev]`, `[rl]`, `[lint]`, `[docs]` 等）。

## 故障排除

### 常见问题

1. **Unicode 编码错误（Windows 中文用户名）**
   - 已修复：系统会自动创建 ASCII 临时目录
   - 如果仍有问题，手动设置环境变量：`set TEMP=C:\temp`

2. **数据未找到错误**
   - 确保已下载数据：`python scripts/get_data.py qlib_data ...`
   - 检查 `provider_uri` 路径是否正确

3. **内存不足**
   - 减少 `kernels` 参数值
   - 使用数据缓存：`expression_cache=True`
   - 减小数据集时间范围

4. **模型训练慢**
   - 启用缓存：`dataset_cache=True`
   - 增加 `kernels` 以使用更多 CPU 核心
   - 考虑使用 GPU 版本的模型（如 PyTorch 模型）

## 参考资源

- 官方文档: https://qlib.readthedocs.io/
- GitHub 仓库: https://github.com/microsoft/qlib
- 论文: https://arxiv.org/abs/2009.11189
- 示例和教程: `examples/` 目录
