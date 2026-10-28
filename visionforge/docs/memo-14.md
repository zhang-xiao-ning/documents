# Memo 14: Framework / Application 分层重构

> 从单层 `src/` 到双层架构
> 时间：2026-10-11
> 版本：v0.1.2

---

## 一、目标

把项目拆成两层：

- **framework**：训练引擎，不依赖任何具体实现
- **application**：具体选择（模型、数据集、任务、指标）

验收标准：加新模型 / 数据集 / 任务 / 指标 = application 里加文件 + 注册表一行。框架代码不动。

---

## 二、最终结构

```text
src/
├── __init__.py
├── framework/                      # 引擎（application-agnostic）
│   ├── __init__.py                 # 公开 API re-export
│   ├── runtime.py                  # 常量 + get_device() + use_cuda()
│   ├── registry.py                 # ExperimentRegistry + DatasetRegistry
│   ├── interfaces/                 # Task / Model / Metric / DataBundle
│   ├── training/                   # train loop / hooks / strategy / tracker
│   ├── metrics/                    # Accuracy / CrossEntropy / Perplexity / corpus_bleu
│   ├── experiment/                 # spec / loader / artifacts / runner
│   └── utils/                      # logger / path / seed / env
│
└── application/                    # 内容（具体选择）
    ├── __init__.py                 # __version__
    ├── cli.py                      # argparse + YAML → RunParams
    ├── main.py                     # 入口
    ├── registry.py                 # 注册实验 + 数据集
    ├── models/                     # mlp / vit / mamba / captioning
    ├── datasets/                   # cifar10 / flickr8k / transforms / tokenizers
    ├── tasks/                      # classification / captioning
    ├── metrics/                    # bleu4
    ├── export/                     # ONNX 导出 CLI
    └── serving/                    # FastAPI 推理服务
```

---

## 三、关键设计

### 1. 单向依赖

```text
application  →  framework
framework    ✗→  application
```

### 2. 两个注册表（对称）

```text
ExperimentRegistry    # name → Experiment
DatasetRegistry       # name → build_bundle(ctx)
```

框架提供机制（空表 + register/get/build），应用填内容。`runner.py` 通过 `DATASETS.build(name, ctx)` 拿数据——不 import 具体数据集。

### 3. 边界测试

- `tests/contracts/test_framework_boundary.py`：遍历 `src/framework/` 下每个 `.py`，AST 解析 import，禁止引用 `application` / `data` / `models` 等旧顶层名
- `tests/contracts/test_application_boundary.py`：遍历 `src/application/`，禁止引用重构前的旧顶层名（`from cli import ...` 之类会被拦）

### 4. 接口在框架，实现在应用

| 归属框架                                        | 归属应用              |
| ----------------------------------------------- | --------------------- |
| `Task` / `Model` / `Metric` / `DataBundle`      | 具体实现              |
| 训练循环 / 调度器 / tracker                     | 具体任务逻辑          |
| 通用指标（accuracy / loss / perplexity / BLEU） | 任务特定指标（BLEU4） |
| `TrainConfig` / `Experiment` / `RunParams`      | 具体实验注册          |
| `cli.py` / YAML 加载                            | 具体模型 / 数据集     |

---

## 四、执行路径

### 阶段 1：拆接口

`src/tasks/base.py`、`src/models/base.py`、`src/evaluation/base.py`、`src/data/bundle.py` → `src/framework/interfaces/`。

### 阶段 2：`DatasetRegistry` + runner 解耦

`framework/registry.py` 加 `DatasetRegistry`。`runner.py` 从 `build_data(...)` 改成 `DATASETS.build(...)`。`application/registry.py` 注册数据集。

### 阶段 3：application 层建立

- `models/` / `tasks/` / `metrics/` / `serving/` / `export/` / `registry.py` → `src/application/`
- `data/` → `src/application/datasets/`
- `cli.py` / `main.py` → `src/application/`

### 阶段 4：边界 + 文档

- 填回 `framework/__init__.py` 完整 re-export
- `pyproject.toml`：`known-first-party = ["framework", "application"]`
- 加两个边界测试
- 更新 README 目录结构 + Extending 章节

---

## 五、踩的坑

### 坑 1：`framework/__init__.py` 触发循环 import

`__init__.py` eager re-export 所有子包。任何 `framework.xxx` 被 import 都先跑顶层 `__init__`，其中一条链回到 `runner.py → data.datasets` → 又回到 `framework.interfaces.bundle` → 循环。

**解法**：把 `runner.py` 对 `data.datasets` 的依赖换成 `DATASETS.build`。顶层 re-export 才能恢复。

### 坑 2：sed 替换丢失子包名

```bash
# 期望
s/from data\./from application.datasets./g
# "from data.tokenizers" → "from application.datasets.tokenizers"  ✓

# 但实际执行可能把中间的 .tokenizers 吃掉了：
# "from data.tokenizers" → "from application.datasets"  ✗
```

**教训**：sed 前先 `grep` 出所有路径，逐条写规则。或者干脆手动改。

### 坑 3：`path.py` 层数算错

`path.py` 从 `src/utils/` 搬到 `src/framework/utils/`，多了一层。`PROJECT_ROOT = Path(__file__).resolve().parent.parent.parent` 从"项目根"变成"src"。

**解法**：`parents[3]` 替代 `.parent.parent.parent`。加 `test_path.py` 断言 `PROJECT_ROOT / "pyproject.toml"` 存在。

### 坑 4：测试和 `monkeypatch` 名字硬耦合

```python
# 旧
monkeypatch.setattr("framework.experiment.runner.build_data", fake)

# 新
monkeypatch.setitem(DATASETS, "cifar10", fake)
```

名字耦合改成接口耦合。**未来 runner 内部怎么调 `DATASETS`，测试都不受影响。**

### 坑 5：`_eval_transform` 位置

`_eval_transform` 在 `flickr8k.py`（captioning 的 224×224 变换），不在 `transforms.py`（CIFAR-10 的 32×32 变换）。`sample_caption.py` 引错了。

**教训**：CIFAR-10 和 captioning 的 transform 完全独立。名字相似不代表同源。

### 坑 6：测试 fake 签名漂移

`_TwoGroupModel.__init__` 不接受 `num_classes`，但 `make_classification_builder` 默认往 `model_init` 塞 `{"num_classes": 10}` → `from_data` 崩。

**解法**：`make_classification_builder(model_init=None)`——默认 `{"num_classes": n}`，可显式传 `{}`。

---

## 六、测试现状

```text
259 passed, 6 deselected (integration + regression)
```

测试分层：

| 层                   | 内容                                                         |
| -------------------- | ------------------------------------------------------------ |
| `tests/unit/`        | 纯 fake，无 IO                                               |
| `tests/contracts/`   | Protocol 契约 + 边界契约                                     |
| `tests/integration/` | 真实数据（需 `RUN_INTEGRATION=1`）                           |
| `tests/fakes.py`     | 共享 fake（TinyModel / TinyTask / make_classification_builder） |

新增的契约测试：

- `test_framework_boundary.py`：28 个 parametrize，扫描 `src/framework/` 每个文件
- `test_application_boundary.py`：29 个 parametrize，扫描 `src/application/` 每个文件

---

## 七、下一步（记录但不急）

### 1. framework 独立发布（暂缓）

目标：让 framework 变成可 `pip install` 的独立包。

方案：uv workspace + 双包。

```text
packages/
├── framework/           # visionforge-framework
└── application/         # visionforge（依赖 framework）
```

改动面：物理移动 2 个包、加 3 个 pyproject.toml、更新 Makefile / scripts / Dockerfile、`uv.lock` 重新生成、pytest / ruff / mypy 路径更新。

**触发时机**：第二个项目要复用 framework，或要独立发布到 PyPI。

### 2. 测试重构深化（见 memo-11）

- ModelRegistry / TaskRegistry / MetricRegistry——让 runner 完全解耦具体实现
- 更多 fake 抽到 conftest
- 减少对 mock 调用次数/参数的断言

---

## 八、一句话

> **本阶段把 src/ 从"扁平单层"改成"framework / application 双层"。**
>
> **框架提供机制，应用提供内容。**
>
> **边界靠两个契约测试守护——改代码时 CI 拦住越界。**

---

*Last updated: 2026-10-11*