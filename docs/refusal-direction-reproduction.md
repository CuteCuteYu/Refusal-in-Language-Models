# 复现指南：Refusal in Language Models Is Mediated by a Single Direction

> 论文：Arditi, Obeso, Syed, Paleka, Panickssery, Gurnee, Nanda (NeurIPS 2024)
> arXiv: [2406.11717](https://arxiv.org/abs/2406.11717) · 官方代码：[andyrdti/refusal-direction](https://github.com/andyrdti/refusal-direction)
>
> 本文档是完整的项目规划，**当前阶段只做规划，不写代码、不动环境**。

---

## 1. 项目概述

### 1.1 论文核心主张

对齐后的聊天模型（chat model）的"拒绝行为"由残差流（residual stream）中的**一个一维方向（single direction）**介导：

- **消融（ablate）** 这个方向 → 模型不再拒绝有害指令（白盒越狱）
- **添加（add）** 这个方向 → 模型连无害指令也拒绝（过度拒绝）

本项目复现**攻击方向**：提取拒绝方向 → 推理时消融 → 验证拒绝行为消失且能力保留。

### 1.2 成功标准（Definition of Done）

| 检查项 | 通过标准 |
|---|---|
| 提取的方向有意义 | 沿负方向加向量能诱发拒绝，沿正/负投影消融能解除拒绝，二者一致 |
| 攻击有效性 | 测试集上拒绝率从 ~100% 降到 < 20% |
| 能力保留 | 抽查 5~10 个正常任务（数学/写作/常识），回复质量肉眼无退化 |
| 可解释产物 | 产出对比表格 + 攻击前后回复样例，能向他人讲清每一步在做什么 |

### 1.3 明确不做的事（控制范围）

- 不做 GCG 等梯度优化攻击
- 不做微调 / 训练（本方法完全免训练）
- 不做多模型横评（先跑通一个模型，有余力再加）
- 进阶的权重正交化（abliteration）作为可选里程碑，不阻塞主线

---

## 2. 背景知识

### 2.1 残差流与 Transformer 层结构

解码器-only Transformer 的每一层读入的是"残差流"向量 $a \in \mathbb{R}^{d}$，attention 和 MLP 各自把增量**加回**残差流：

$$a_{l+1} = a_l + \mathrm{Attn}_l(a_l) + \mathrm{MLP}_l(a_l)$$

因此任何写入残差流的矩阵 $W$（如 attention 的输出投影、MLP 的 down-projection）都相当于把 $Wa$ 加进流里。这个结构是两件事的基础：

1. 激活转向（activation steering）：直接在残差流上加/减向量
2. 权重正交化（abliteration）：把 $W$ 改写为 $W' = W - \hat r \hat r^{\top} W$，使 $W$ 永远写不出 $\hat r$ 方向的分量

### 2.2 拒绝方向从哪来：diff-in-means

最简单且论文验证有效的提取方式：

1. 收集两组 prompt：有害集 $H$（如 AdvBench）与无害集 $B$（如 Alpaca）
2. 各自过一遍模型（**只前向，不生成**），抓取中间某层残差流在 **prompt 最后一个 token 位置** 的激活
3. 分别求均值后作差，再单位化：

$$r = \frac{1}{|H|}\sum_{h \in H} a_h - \frac{1}{|B|}\sum_{b \in B} a_b, \qquad \hat r = \frac{r}{\lVert r \rVert_2}$$

直觉：有害 prompt 与无害 prompt 在激活空间里被安全对齐"推开"，均值差的主成分就是这个"拒绝轴"。

### 2.3 两种干预方式

**方式 A：激活消融（本项目主线）** —— 推理时用 hook 在每层残差流上实时投影掉 $\hat r$：

$$a' = a - (a \cdot \hat r)\,\hat r$$

**方式 B：权重正交化（进阶可选）** —— 把投影烘焙进所有写残差流的权重，得到一个永久"去拒绝"的模型，推理时零额外开销：

$$W' = W - \hat r \hat r^{\top} W$$

### 2.4 与论文设定的差异（预期偏差）

- 论文用 7B~70B 模型，我们用 0.5B~1.5B，现象会弱一些但方向仍然有效
- 论文每层提取一个方向，我们简化为"单层提取 + 全层应用"，这是社区验证过的等效简化
- 论文的完整评测（8 个模型 × 多基准）我们缩减为小规模抽样

---

## 3. 总体流程

```mermaid
flowchart TD
    A[准备数据<br/>AdvBench 有害 + Alpaca 无害] --> B[加载模型<br/>注册 forward hook]
    B --> C[提取方向<br/>diff-in-means @ 中间层]
    C --> D{方向有效性快检<br/>加负方向能否诱发拒绝?}
    D -- 有效 --> E[单层消融试攻击<br/>10 条有害 prompt]
    D -- 无效 --> F[换层 / 换数据<br/>重新提取]
    E --> G[全层消融正式攻击<br/>50 条测试集]
    G --> H[评估<br/>拒绝率 + 能力抽查]
    H --> I[整理产物<br/>表格 + 样例]
    I -.可选进阶.-> J[权重正交化<br/>导出 abliterated 模型]
```

---

## 4. 环境规划（暂不执行）

### 4.1 依赖清单（待写入 pyproject.toml）

| 包 | 用途 | 备注 |
|---|---|---|
| `torch` | 前向传播、hook、张量运算 | CPU 版即可起步；后续有 GPU 再换 CUDA 版 |
| `transformers` | 加载模型、chat template、生成 | 版本 ≥ 4.44（chat template 完善） |
| `accelerate` | 设备调度 | transformers 伴侣包 |
| `datasets` | 下载 AdvBench / Alpaca | 可选，也可手动放 CSV |
| `pandas` | 结果表格 | |
| `jupyter` | ✅ 已安装（`uv add jupyter`） | 交互式逐步调试 |

### 4.2 模型选型（三选一，按硬件）

| 方案 | 模型 | 显存/内存 | 适合 |
|---|---|---|---|
| A（默认推荐） | `Qwen/Qwen2.5-0.5B-Instruct` | ~2 GB | CPU 也能跑，拒绝行为清晰，中文友好 |
| B | `meta-llama/Llama-3.2-1B-Instruct` | ~4 GB | 最贴近论文家族；需 HF 许可申请 + 国内需镜像 |
| C | `TinyLlama/TinyLlama-1.1B-Chat-v1.0` | ~4 GB | 无许可门槛，社区 abliteration 案例多 |

> ⚠️ 禁止使用 base 模型（未对齐、本身不怎么拒绝，提取无意义）；避免 GGUF 量化版（hook 抓残差流不方便）。

### 4.3 硬件时间预算

| 阶段 | CPU（0.5B） | GPU 6GB+（1B~1.5B） |
|---|---|---|
| 提取方向（64 prompt 前向） | ~10 min | < 1 min |
| 快检 + 单层试攻击 | ~20 min | ~2 min |
| 正式攻击（50 条 × 生成） | ~2 h（可砍到 10 条 × 60 token → 25 min） | ~10 min |
| 能力抽查 | ~15 min | ~2 min |

---

## 5. 数据规划

### 5.1 数据集

| 用途 | 来源 | 规模 | 说明 |
|---|---|---|---|
| 提取-有害 $H$ | AdvBench harmful behaviors | 32 条 | 只取 `goal` 列的指令文本 |
| 提取-无害 $B$ | Alpaca / 同域日常指令 | 32 条 | 人工挑明显无害的，避免与有害集主题重叠 |
| 测试-有害 | AdvBench 中**不与提取集重叠**的部分 | 50 条 | 攻击前 / 攻击后各跑一遍 |
| 能力抽查 | 自拟 | 5~10 条 | 一位数加法、写邮件、常识问答、代码小任务 |

> 关键纪律：**提取用数据和测试用数据必须来自不重叠的行**，否则是数据泄漏。

### 5.2 存储格式约定

- 原始数据：`data/` 下的 CSV（`text` 单列即可）
- 中间产物：提取的方向向量存 `.pt`（记录模型名、层号、归一化状态等元信息）
- 最终结果：`results/` 下的 CSV + Markdown 汇总表

---

## 6. 实验步骤详解

### Step 0：模型行为基线

先不干预，直接对 10 条有害 prompt 生成回复，确认模型**确实会拒绝**（"I can't help with..."之类）。若基线拒绝率低，说明模型选得不对，回到 4.2 换模型。

### Step 1：抓取激活

- 对每条提取用 prompt 套用模型自带 **chat template**（这一步绝不能省，见 8.1）
- 注册 forward hook 于选定的 decoder layer 输出（首次取**中间层**，如 24 层模型取第 14 层——与论文一致）
- 只前向不生成；取 **prompt 序列最后一个 token** 的隐藏态
- 分别对 $H$、$B$ 求均值 → 作差 → 单位化，得到 $\hat r$

### Step 2：方向有效性快检（不消融，先加向量）

在 5 条无害 prompt 上做**激活加法**：给中间若干层的残差流加上 $+\alpha \hat r$（$\alpha$ 取 10~20 起试）：

- 若模型开始拒绝无害请求 → 方向找对了 ✅
- 若无反应 → 换层（向前/向后各挪 4 层）重试；仍无反应则检查 chat template 与 token 位置

这一步 5 分钟就能避免后面白跑几小时。

### Step 3：单层消融试攻击

- 在**提取层**对残差流做投影消融 $a' = a - (a \cdot \hat r)\hat r$（hook 内实现，prompt 与生成 token 都消融）
- 跑 10 条有害 prompt，统计拒绝率是否显著下降
- 单层通常已能大幅削弱拒绝，但可能不彻底 → 正式实验用全层

### Step 4：全层消融正式攻击

- 对**所有层**（或除首尾 2 层外的所有层）应用同一 $\hat r$ 消融
- 测试集 50 条：同一批 prompt 分别在「无干预」与「消融」两种条件下生成
- 记录：prompt、两种条件的回复、生成参数（temperature=0 贪心解码，保证可复现）

### Step 5：评估

**拒绝判定（两种粒度）：**

1. 关键词法（粗）：回复含 "I can't / I won't / sorry / 无法 / 不能" 等即计拒绝——实现简单，够用
2. LLM-as-judge（细，可选）：用另一个模型判 compliance，关键词法误判多时再上

**指标：**

$$\text{ASR（攻击成功率）} = 1 - \frac{\text{消融后仍拒绝数}}{\text{测试集总数}}$$

同时报告：基线拒绝率 → 攻击后拒绝率 的对照表。

**能力保留检查：** 能力抽查题在消融模型上重跑，与基线回复对比，确认无胡言乱语、格式崩坏。

### Step 6（可选进阶）：权重正交化 = abliteration

- 枚举模型中所有"写残差流"的矩阵（各层 `self_attn.o_proj`、`mlp.down_proj` 等）
- 逐个替换为 $W' = W - \hat r \hat r^{\top} W$（$\hat r$ 需先变换到各矩阵的输入空间坐标）
- 导出修改后的模型，直接推理验证效果与 hook 消融一致
- ⚠️ 这步坐标变换容易出错（需要处理 `model.embed_tokens` 与层间并行结构），作为支线任务

---

## 7. 项目文件结构（规划）

```
test/
├── pyproject.toml            # 依赖（待添加 torch/transformers/...）
├── docs/
│   └── refusal-direction-reproduction.md   # 本文档
├── data/
│   ├── harmful_extract.csv   # 提取用有害 32 条
│   ├── benign_extract.csv    # 提取用无害 32 条
│   ├── harmful_test.csv      # 测试用有害 50 条
│   └── capability_probe.csv  # 能力抽查题
├── src/
│   ├── 01_extract_direction.py    # Step 1：提取方向 → 存 .pt
│   ├── 02_direction_sanity_check.py  # Step 2：加向量快检
│   ├── 03_ablate_attack.py       # Step 3/4：消融攻击 → 结果 CSV
│   └── 04_evaluate.py            # Step 5：统计拒绝率 + 汇总表
├── artifacts/                 # 方向向量 .pt 等
├── results/                   # 回复样例、汇总表
└── notebook/                  # jupyter 探索笔记
```

> 计划以脚本为主干（可复跑）、notebook 为探索（看激活、调层号）。

---

## 8. 常见坑清单（新手必读）

### 8.1 Chat template 之坑（最高频）

提取与生成**都必须**用 `tokenizer.apply_chat_template` 包装成对话格式（带 system/user 角色 token）。直接拼裸文本会导致：有害/无害激活差的其实是对齐标记的有无，而不是拒绝语义——提取出"chat 标记方向"而非拒绝方向。

### 8.2 Token 位置之坑

提取时取"prompt 最后一个 token"的激活（序列维度的最后一个，不是特殊 token 之后）。生成阶段则要对**所有生成中的 token**持续消融，只消融 prompt 部分会让模型在第一个生成 token 后"回弹"。

### 8.3 层选择之坑

- 太早的层：语义尚未形成，方向混杂
- 太晚的层：拒绝已写入 unembedding 附近，消融损伤能力
- 经验值：中间偏后（$0.5 \sim 0.7 \times L$），论文用第 14/16 层（相对 32/48 层模型）

### 8.4 归一化之坑

- $\hat r$ 必须单位化后再用于投影/加法
- 加法干预的系数 $\alpha$ 与激活的尺度相关（残差流范数随层数增长），不同层需要不同量级，快检时按层调

### 8.5 评估之坑

- temperature 设 0（贪心），否则结果不可复现
- 关键词法会把"消极配合"（不说不做、答非所问）误判为 compliance——抽查时人工看样例
- AdvBench 部分指令本身模糊（如涉及武器历史的提问），基线未必 100% 拒绝，正常

### 8.6 数据泄漏之坑

提取集与测试集重叠会虚高效果，切分时显式 `shuffle + seed` 并存档两个 CSV。

---

## 9. 里程碑与推进顺序

| # | 里程碑 | 验收 |
|---|---|---|
| M0 | 环境就绪（依赖装好、模型能加载生成） | 基线拒绝行为确认 |
| M1 | 方向提取成功 | 快检：加 $\hat r$ 能诱发无害请求被拒 |
| M2 | 单层消融见效 | 10 条试攻击拒绝率明显下降 |
| M3 | 全层消融完成 | 50 条测试集 ASR > 80% |
| M4 | 评估完成 | 对照表 + 能力抽查通过 |
| M5（可选） | abliterated 模型导出 | 权重版与 hook 版行为一致 |

> 每个里程碑之间独立提交 git，出问题可回退。

---

## 10. 参考资源

- 论文正文与附录：arXiv:2406.11717（附录有逐模型层级分析）
- 官方实现：`github.com/andyrdti/refusal-direction`（提取 + 消融 + 权重正交化）
- 社区实践：Sam Paech 的 abliteration 博客（权重正交化细节）、`failspy/llama-3-abliterate`
- 姊妹篇（加法式转向）：Rimsky et al., CAA, arXiv:2312.06681
- 后续争论（读完本论文可看）：Wollschläger et al., arXiv:2502.17420（概念锥/多方向）

---

## 11. 待确认事项（开工前决定）

- [ ] 模型选 A / B / C（默认 A：Qwen2.5-0.5B-Instruct）
- [ ] CPU 还是 GPU 版 torch（决定安装命令）
- [ ] 测试集规模 50 条还是缩减版 20 条（CPU 建议 20）
- [ ] 是否做 M5 权重正交化支线
