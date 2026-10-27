# TODO

> 只记"还没做"的事。
> 完成 → 移到 `COMPLETED.md`。
> 决定不做 → 移到本文件底部"已排除"。

---

## 短期（接下来做）

- [ ] **LLaVA-A 推理脚本**（1 天）
  - `scripts/llava_infer.py`：加载 HF 权重（4bit 量化）
  - 不改 framework，先跑通推理
  - `transformers` 只出现在 `scripts/`，不进 `src/`

- [ ] **评估层扩展**：CIDEr / METEOR / ROUGE
  - 依赖 `pycocoevalcap` 或 `nltk`
  - 与 BLEU4 同一模式（Metric + registry）

---

## 中期（撞到需求再做）

- [ ] **加载预训练权重**（CLIP / HF）
- [ ] **KV cache**（生成式推理）
- [ ] **量化**（int8 / 4bit / bitsandbytes）
- [ ] **参数冻结 + LoRA 骨架**
- [ ] **多阶段训练**（LLaVA 对齐 + 指令微调）
- [ ] **AMP 实测**（大模型）

---

## 长期（做完整 LLaVA / LM 训练时）

- [ ] **数据缓存**（tokenized 结果存盘）
- [ ] **packed sequence**（多句拼接）
- [ ] **vLLM / TGI 集成**
- [ ] **FSDP / DeepSpeed**
- [ ] **多机多卡**（跨机 DDP）

---

## 工程细节

- [ ] **`DataBundle` / `EvalBundle` 强类型化**：等 LLaVA 接入后字段稳定
- [ ] **`Trainer` 类重构**：等第 3 类任务撞出新 hook / config 需求
- [ ] **`from_data` 优化**：`DatasetInfo` 合并到 `DataBundle`
- [ ] **`is_main_process` 散落治理**：rank 差异应由 strategy 吸收
- [ ] **`CHANGELOG.md`**（Keep a Changelog 格式）
- [ ] **覆盖率报告**（pytest-cov）
- [ ] **CI 矩阵**（Python 3.12 + 3.13）

---

## 可选

- [ ] TensorBoard 远端服务器
- [ ] WandB 集成
- [ ] MLflow 实验管理
- [ ] mkdocs / pdoc API 文档

---

## 暂缓

### Framework 独立发布

**目标**：把 framework 变成可 pip install 的独立包。

**方案**：uv workspace + 双包。

```text
packages/
├── framework/           # visionforge-framework
│   ├── pyproject.toml
│   └── src/framework/
└── application/         # visionforge（依赖 framework）
    ├── pyproject.toml
    └── src/application/