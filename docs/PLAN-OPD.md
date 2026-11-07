# OPD 论文复现计划

> 版本：v0.1（2026-11-06）
> 范围：先完成 OPD 方向的两篇论文复现；本计划也是后续 Agentic RL、Harness 两个方向的工程模板。

## 1. 项目定位

ReproLab 是一个以**可信复现**为核心的论文复现开源项目，围绕“模型如何被监督与优化”组织三条方向：OPD → Agentic RL → Harness。

与 agentic-rl-lab 等教学仓库刻意错位：

- 别人做“算法入门覆盖广度”，本项目做“少数论文的**复现深度**”：官方代码锁定、消融、成本、失败复现。
- 每条方向采用**双轨制**：教学轨让人看懂并跑起来，复现轨让人相信并能验证。

## 2. 双轨制

| 轨道 | 目录 | 回答的问题 | 交付标准 |
|---|---|---|---|
| 复现轨（严谨轨） | `papers/opd/` | 论文结论在我这里是否成立 | 官方代码 commit 锁定、原始/降规模两档配置、RunManifest、复现卡、成本台账、消融 |
| 教学轨 | `demos/opd/` | 这个方法到底在算什么 | 几百行自包含最小实现、一条命令 smoke、概念讲解；**不承诺数值复现** |

共享层（双轨必须复用同一套）：

- 数据契约：RunManifest / Metrics / Trajectory 三个 Schema；
- 概念文档：`docs/concepts/`（importance sampling、reverse KL、OPD、GRPO 等）；
- 训练后端抽象：本地自托管为主（verl），远程 API（PyTRIO/Tinker）可选可换，同一份实验配置可切换；
- 成本口径：GPU·小时、token 量、教师调用次数（不只记钱数）。

边界纪律：**教学 demo 的简化结果不得作为论文结论的证据**；README 两个入口需明确标注区别。

## 3. 复现对象与顺序

### 第一篇：One-Shot OPD（链路与展示先行）

- 论文：arXiv:2609.04172《Rethinking OPD II: One Training Example》
- 官方代码：https://github.com/Thinking-Space/One-Shot-OPD（Apache-2.0）
- 核心结论：单条 query 训练数百步可恢复 full-data OPD 的大部分收益；16 条语义多样 query 匹配全量；OPD “data-overfed but algorithm-starved”。
- 为什么先做：仓库自带 smoke 模式（10 步、24 条评测样本）；展示性强（“一条数据持续涨点”）；与第二篇同团队、同 verl 技术栈，环境可复用。

### 第二篇：Rethinking OPD（深度消融）

- 论文：arXiv:2604.13016《Rethinking OPD: Phenomenology, Mechanism, and Recipe》（ICML 2026 FoGen Workshop）
- 官方代码：https://github.com/thunlp/OPD（**未声明许可证**）
- 复现内容：思维模式相容性、弱→强反向蒸馏、top-k overlap / 熵差诊断、离策略冷启动 SFT→OPD、教师对齐 prompt 选择。
- 可用资产：作者已释放 Qwen3-1.7B-SFT 等中间 checkpoint 与数据集。

### 参考基线（不作为主复现）

- MiniLLM（arXiv:2306.08543，MIT）：reverse-KL OPD 经典锚点，优先用 TRL 内置实现，不折腾其旧版定制 transformers。
- GKD（arXiv:2306.13649）：OPD 溯源，TRL `GKDTrainer` 实现。
- Thinking Machines On-Policy Distillation：仅作损失与伪代码参考，训练后端为付费 Tinker API，不作为复现目标。
- agentic-rl-lab 的 `general-opd` 章节：教学轨前置阅读与实现参考（复用代码需保留其许可声明）。

## 4. 与上游代码的结合方式

原则：**上游代码是“被复现的对象”，不是本项目代码的一部分**；本项目在其之上提供运行、记录、评测与展示层。禁止 fork 后改成自己的仓库。

- One-Shot-OPD（Apache-2.0）：以 **Git Submodule** 固定 commit，置于 `third_party/one-shot-opd`，只读引用。
- thunlp/OPD（无许可证）：**不拷贝源码**。在 `external/rethinking-opd.yaml` 记录 url / commit_sha / 放置路径，由安装脚本拉取到 gitignore 的 `third_party/`；自有适配代码仅以独立模块 import。去仓库 issue 请求作者补充许可证后再调整。
- 上游 bug（如 verl v0.7.0 验证分数低估 5–7 个点）：不改 submodule 内容，用 `patches/` 放最小补丁，安装脚本 apply，并在复现卡注明“原始行为 vs 已知修正”。
- 环境不做全局统一：每篇论文一个锁定环境（Dockerfile 或 conda lock），公共 Python 包只放纯数据处理与 Schema，避免绑定特定 verl 版本。

## 5. 目标仓库结构

```text
ReproLab/
├── README.md                      # 双轨入口、复现状态总表
├── registry/
│   └── papers.yaml                # 论文-仓库-commit-许可证-硬件-状态总登记
├── third_party/                   # submodule（One-Shot）或脚本拉取（无许可），gitignore 实际代码
├── external/
│   └── rethinking-opd.yaml
├── papers/
│   └── opd/
│       ├── one_shot_opd/
│       │   ├── repro-card.md      # 目标结论、模型、数据、预算、成功判据、与论文差异
│       │   ├── configs/           # 统一实验 YAML
│       │   ├── launch/            # 调用上游入口的薄封装
│       │   ├── patches/
│       │   └── results/           # 指标 JSON、图表、轨迹抽样、成本
│       └── rethinking_opd/        # 同上
├── demos/
│   └── opd/
│       └── general_opd/           # 教学轨最小实现（本地后端可跑）
├── docs/
│   ├── PLAN-OPD.md                # 本文件
│   └── concepts/                  # IS / reverse KL / OPD / GRPO
├── src/reprolab/
│   ├── contracts/                 # RunManifest / Metrics / Trajectory Schema
│   ├── runners/                   # 配置解析 → 调用上游 → 收集产物
│   ├── tracking/                  # SwanLab/W&B + 原始 JSONL 双写
│   ├── evaluation/                # 不信任上游打印分数，独立复算
│   └── cost/                      # GPU·小时、token、教师调用次数
└── environments/                  # 每篇论文一个锁定环境
```

## 6. 共享数据契约（最先做）

1. **RunManifest**：run_id、论文与实验名、上游仓库 url+commit_sha+补丁列表、镜像/依赖锁、模型与数据版本哈希、GPU 型号数量、种子、启动命令、开始结束时间、成本。
2. **Metrics**：按步记录的统一指标，至少含 `opd/reverse_kl_mean|std`、overlap ratio、熵差、评测分数、token 吞吐、步耗时；同时支持“按步”和“按实际成本”两条曲线。
3. **Trajectory**：抽样保存 prompt、student completion、student/teacher 逐 token logprob、advantage，支持事后做“哪些 token 被罚”的展示。

## 7. 阶段计划

### P0 地基（无 GPU 即可完成）

- [ ] 初始化仓库结构、README 双轨入口、许可证（本项目代码建议 Apache-2.0）
- [ ] 定义并落地三个 Schema（含示例 JSON 与校验脚本）
- [ ] 建立 `registry/papers.yaml` 登记表
- [ ] 编写两篇论文的复现卡初稿
- 验收：`pytest` 能校验示例 manifest/metrics/trajectory 通过。

### P1 接入 One-Shot OPD（只验证结合方式）

- [ ] submodule 接入并固定 commit；环境锁定文件
- [ ] runner 把统一 YAML 映射到上游参数；`patches/` 机制就绪
- [ ] 跑通 smoke（10 步、24 样本），产物经 Schema 落盘
- [ ] tracking 双写（SwanLab 可关闭，JSONL 必须有）
- 验收：一条命令完成 smoke，输出合法 RunManifest + Metrics。

### P2 教学轨 general-opd

- [ ] 参考 agentic-rl-lab 章节，写自包含最小实现（学生采样→教师 logprob→reverse KL→importance sampling 更新）
- [ ] 默认 smoke 参数、本地后端优先、远程 API 后端可选
- [ ] `docs/concepts/` 补 OPD/reverse KL/IS 三篇概念文档
- 验收：新人按 README 10 分钟内跑通并看懂核心 30 行逻辑。

### P3 One-Shot OPD 小规模方法复现

- [ ] 1.5B 师生档：1 query / 16 query / full-data 三组对照
- [ ] 独立评测脚本复算 MATH-500 等分数；记录 state coverage（可降规模近似）
- [ ] 多 seed、同成本曲线；成本台账完整
- 验收：趋势与论文一致（1 query 恢复大部分收益、16 query 接近全量），复现卡记录偏差与原因。

### P4 One-Shot OPD 原始规模指引（有算力时）

- [ ] 8×80GB 参考配置、checkpoint 续跑说明、外部贡献者提交通道
- [ ] 完整结果入 `results/`，复现卡标注“流程/方法/结论/数值”四级达成度

### P5 Rethinking OPD 消融

- [ ] 安装清单方式接入无许可上游
- [ ] 直接 OPD / 仅冷启动 SFT / SFT→OPD 三组，成本分列
- [ ] overlap ratio、熵差、overlap-token advantage 诊断独立复算
- [ ] 反向蒸馏（弱教师→强学生）失败条件复现；冷启动恢复实验
- 验收：成功组与失败组呈现论文所述的动态签名差异；失败也要形成分析结论。

## 8. 合规与风险

- thunlp/OPD 无许可证：不拷源码、不分发其产物之外的代码，安装脚本由用户自行拉取。
- 模型/数据集许可随 RunManifest 记录；教学轨只使用公开数据（如 DeepMath-103K）。
- 已知坑：非 thinking 模型训练需关闭 thinking 模板；verl 版本验证路径差异；teacher/student token 对齐必须强校验、失败即中断（fail-fast）。
- 远程 API（PyTRIO 等）仅作体验后端，严谨实验默认本地自托管，避免供应商绑定。

## 9. 近期下一步

执行 P0：初始化仓库、三个 Schema、登记表与两份复现卡初稿。
