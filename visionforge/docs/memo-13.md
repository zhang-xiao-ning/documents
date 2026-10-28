# Memo 11: 测试重构 TODO

> 状态：待办
> 记录时间：2026-10-11
> 触发：framework/application 分层重构中，测试改动量远超实现改动量

---

## 一、问题

每次重构 framework 的接口，实现改 1-2 行，测试却要改 10+ 处。

具体表现：

- `monkeypatch.setattr("module.func_name", fake)` 和实现的名字硬耦合
- fake 函数的签名照抄真实签名，签名变了就全崩
- 同一份 fake 在多个测试文件里重复，改一处要改 N 处

---

## 二、根本原因

测试测的是"实现的名字 / 签名 / 调用细节"，不是"外部可观察的行为"。

monkeypatch 的字符串路径本身编码了实现事实：

- 哪个模块
- 哪个名字
- 是不是可替换的函数

任何一个变了 → 测试崩，即使行为完全没变。

---

## 三、目标

测试只依赖"契约"和"行为"，不依赖"实现的名字"。

---

## 四、行动项

### A. 用注册表 / 容器注入替代 monkeypatch 名字

- ✅ 已完成：`DatasetRegistry`（`monkeypatch.setitem(DATASETS, ...)`）
- ⬜ 待做：`ModelRegistry` / `TaskRegistry` / `MetricRegistry`
  - 让 `runner.py` 从注册表拿 model / task / metric
  - 测试用 `setitem` 注入 fake
- ⬜ 待做：`StrategyRegistry`（如果将来有 FSDP / DeepSpeed）
- ⬜ 待做：评估是否需要 `ArtifactsFactory` 或类似抽象

### B. 共享 fake 放进 conftest.py

- ⬜ 把 `_fake_data_bundle` 抽到 `tests/conftest.py`（现在 3 份）
- ⬜ 把 `_TinyModel` / `_TinyTask` / `_CountingMetric` 等公共 fake 抽到 conftest
- ⬜ 提供一个 `tests/fakes/` 或 `tests/factories.py` 模块（函数工厂优先于 fixture）

### C. 尽量测"外部可观察的行为"

- ⬜ 审查现有测试，去掉对 mock 调用次数 / 参数的断言（除非那是契约）
- ⬜ 优先断言：返回值 / 副作用（文件、日志、checkpoint）/ 状态变化
- ⬜ 反对：`assert mock.called_with(...)`、`assert spy.call_count == N`

### D. 测试分层与契约测试

- ⬜ 契约测试（contracts/）只依赖 Protocol / 注册表接口，不依赖具体实现
- ⬜ framework 单元测试用纯 fake（不 import application）
- ⬜ application 单元测试才允许 import 具体实现

### E. 引入 Protocol / 依赖注入（可选）

- ⬜ 如果注册表方案还不够，考虑用 Protocol 定义接口契约
- ⬜ runner 接收注入的依赖（而非自己去注册表查）
- ⚠️ 权衡：会引入更多抽象，先做 A-D 看效果

---

## 五、判断标准

重构 framework 接口后，满足以下任一条即认为测试已解耦：

1. 改 1 行实现 → 改 0 行测试
2. 改 1 行实现 → 改 1 行测试（且那 1 行是"注册表 key"）
3. 加新模型 / 数据集 / 任务 → 只改 application，测试自动覆盖

---

## 六、优先级

1. **B**（共享 fake）—— 立刻能减轻痛苦，半小时
2. **A**（注册表）—— 已经在做，继续推进
3. **C**（测行为不测实现）—— 重构时顺手清理
4. **D**（分层）—— 与 A/B 并行
5. **E**（Protocol）—— 等 A-D 完成后再评估

---

## 七、触发时机

- **每次 framework 接口变化**：顺手做 C（去 mock 断言）
- **下一轮重构前**：做 B（共享 fake）
- **准备独立发布 framework 前**：做 A + D
- **测试再出现"改 1 行实现改 10 行测试"**：立刻停下来做 B

---

## 八、一句话

> 测试难改不是因为"测试多"，是因为"测试和实现的耦合方式"。
>
> 让测试依赖"接口 / 行为"，不依赖"名字 / 签名 / 调用细节"。
>
> 这次 `DatasetRegistry` 是对的方向，继续推。