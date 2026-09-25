# MAFA 会话交接文档（v0.7.36）

**版本：** v0.7.36 · **自检：** 169 passed / 0 failed · **交接时间：** 2026-09-24
**上一会话完整记录：** `RUN.md` §20–§25（§25 是三大目标的现状盘点）

> 新会话开场请先读本文，再读 `RUN.md` 的 §20–§25。项目规则：**每轮对话之后更新 RUN.md**。

---

## 零、三个大目标的现状（最重要）

| 目标 | 状态 |
|---|---|
| 1. 基于 **MedAgentAudit** 体系做人机协同标注 | **这就是现在在跑的东西。** 标注器用的是 `taxonomy.SEED_MODES` 十个码（`provenance: "T0:MedAgentAudit"`） |
| 2. 基于 **MAST** 体系也能做 | **可表达，未建成。** 缺 MAST 种子文件，详见下面第五节 |
| 3. Stage 4 由用户指定、**默认关闭** | **已做成真开关**（0.7.36）。之前只是「碰巧关着」 |

**三个概念不要混：** AEGIS 语料是**底料**；MAST 是模拟 split 的**注入真值**（`mafa/mast.py`，**绝不能进标注器提示词**，否则循环论证）；被标注的**标签空间**是 MedAgentAudit 的。

---

## 一、恢复环境（新会话容器是空的）

### 需要重新上传的文件

| 文件 | 解压到 | 内容 |
|---|---|---|
| `MAFA-v0_7_36.zip` | `/home/claude/work/mafa/` | 代码 + 所有结果 + 运行快照 |
| `AEGIS-custom-clean-dataset.zip` | `/home/claude/work/data/clean/` | 440 条 |
| `AEGIS-custom-real-faults-dataset.zip` | `/home/claude/work/data/faulty/` | 1,112 条 |
| `AEGIS-custom-simulated-faults-dataset-副本.zip` | `/home/claude/work/data/simulated/` | 3,519 条，带注入日志 |
| `medxpertqa_images.zip` + `.z01`–`.z04` | `/home/claude/work/img/` | 2,858 张，分卷 |
| `MedAgentAudit-datasets_and_logs.zip` | `/home/claude/work/maa2/` | 14,370 条 audit_results，6 框架，**含 human_eval** |

**`MedAgentAudit.zip`（9.2 版）不需要再传。** 已核实 `maa2/github_release_files/human_eval` 就是同一份：`human_kappa.py` 重生成的 `human_gold.json` 与快照**逐字节一致**，κ 范围同为 0.273–0.744。软链接即可。
`MedAgentAudit-4.23-observation.rar` 是另一样东西（3,600 个原始 MAS 输出，1.86 GB，无 human_eval），当前工作用不到。

### 恢复命令

```bash
mkdir -p /home/claude/work/mafa /home/claude/work/data/clean /home/claude/work/data/faulty \
         /home/claude/work/data/simulated /home/claude/work/img /home/claude/work/maa2 /home/claude/work/runs
cd /home/claude/work
unzip -q MAFA-v0_7_36.zip -d mafa
R=/home/claude/work/mafa/MAFA-multi-agent-failure-attribution

# 三个 AEGIS 包解压后都多一层目录，要摊平成 data/<split>/*.jsonl
# 期望行数：clean 440 / faulty 1112 / simulated 3519
ln -sfn /home/claude/work/data $R/data

# 图像是分卷 zip，先合卷：zip -s 0 medxpertqa_images.zip --out full.zip
unzip -q MedAgentAudit-datasets_and_logs.zip -d maa2
mkdir -p maa/MedAgentAudit-9.2 && ln -sfn /home/claude/work/maa2 maa/MedAgentAudit-9.2/datasets_and_logs

cp -r $R/results/runs_snapshot/. /home/claude/work/runs/
cp $R/results/runs_snapshot/ab4_180_ids.txt /home/claude/work/
cp $R/scripts/keepalive_ds_armb.sh   /home/claude/work/ka11.sh
cp $R/scripts/keepalive_consistency.sh /home/claude/work/ka12.sh
chmod +x /home/claude/work/ka1*.sh

pip install -q -r $R/requirements.txt --break-system-packages   # 新容器没有 openai！
cd $R && python scripts/selfcheck_offline.py | tail -2          # 期望 169 passed / 0 failed
```

**上次踩的三个坑（配方里没写的）：**
1. **新容器没有 `openai` 包。** keepalive 会照常报「已启动」，进程却立刻 `ModuleNotFoundError` 退出——它只检查进程在不在，不检查有没有真开始工作。
2. `maa2` 不解压时，自检会显示 **146 passed / 0 failed**——ColaCare 的四项检查藏在没有 `else` 的 `if cola:` 里，静默消失。**已修**：语料缺失现在记为失败。
3. shell 是 `sh`，**不支持花括号展开**；`pkill -f <脚本名>` 会匹配到自己的命令行，把当前 shell 一起杀掉。

---

## 二、进行中的任务

| 任务 | 进度 | 续跑 |
|---|---|---|
| **deepseek Arm B**（六码消融） | **154/180**，卡在 MDAgents 段（19/45） | `/home/claude/work/ka11.sh` |
| **step 层一致性校准** | 206 槽位还差 44 个 | **暂停中**，见下 |

### deepseek 为什么卡住（首要运维问题）

日志里有 **41 次 "resuming"**，进度在 153–154 之间反复。原因：**Stage 1 只在整条轨迹完成时才写盘**，而剩下的全是 MDAgents，一条要十几分钟；容器在工具调用之间回收后台进程，整批在途工作丢失，重启又从头做。

**光靠勤轮询解决不了**，前几轮已经证明。真正的修法二选一：
- 让 Stage 1 按轨迹增量落盘（改 `run_stage1.py` 的写入时机）；
- 或把剩下 26 条 MDAgents 拆成小批（`--trace-ids` 每次 3–5 条）分批跑完。

**我推荐后者**，改动小、当轮见效。

### step 一致性校准为什么停

glm **账户级限流**：单个请求 0.4 秒就返回 `429 code 1302`，与并发无关（已降到 3 仍然如此）。三个 worker 全卡在退避里空转，文件 40 分钟零写入。已把客户端 `timeout=300, max_retries=3` 改成 `timeout=120, max_retries=1`（原配置下单票最坏 60 分钟不写盘，从外面看和死掉一模一样）。

**等配额恢复再启。不能换模型**——k=5 的投票已经积了一半，换标注器这批就废了。

---

## 三、本会话做了什么（写论文要用）

### Stage 0：提议-验证（L3），已建成

新模块 `mafa/stage0_auto.py`。模型只从**路径清单**里挑字段（不从正文猜说话人），六项**不用模型**的检查裁决，最多 3 轮反馈：

1. 文本守恒（逐路径，带副本检测）2. 有效读取（用**真实解析器**解析，不只看清单）3. 重复摄入 4. 阶段与数据流一致 5. 能力键必须合法 6. **不可观测 ≠ False**

**有真值的验证：** ColaCare 当作未见框架，模型提议的 spec 在**全部 2,395 条**上与手写 spec 语义等价——19,800 个步骤的内容、阶段、轮次全一致，三项能力 2,395/2,395 一致，只有聚合者命名不同。4 个未见框架（reconcile/mac/medagent/healthcareagent）在**留出记录**上重验 0 问题。

**修掉的静默失败（每项都曾骗过全部检查）：**
- `detect()` 曾按记录形状回退匹配，把 4 个未见框架（9,580 条）**全部摄入成 ColaCare**，其中 2 个验证零问题。现在只按文件名识别，形状相似只写进报错。
- `get()` 对相对路径里的 `[*]` **静默返回 None** → 所有步骤输入为空、同伴可见性全 False，四项检查全过。现已支持 `[*]`。
- 重复摄入无人查（`final_decision_log` 被计两次）；模型自造的能力键被静默丢弃；数据流检查把 Planner 的追问算成同伴。

### Stage 3：三项改动

- **`ThresholdBackend`**（`--router-backend threshold`）：用户分段阈值，可按组；不继承保证，`fit` 改为**测量**规则（自动占比、错误率、Clopper-Pearson 上界）。区间留空**报错**不默认放行。
- **`calibrate_bands()`**：给**错误预算**反推阈值（Learn-then-Test 单参数情形）；**没有阈值能满足时返回 `None` 而不是最严阈值**。
- **`export_level_buckets()`**：**按层路由**。

### 关键实测数字

610 个有标签槽位（原始 `p_yes`，**未经 Stage 2 校准**）：

| 规则 | 自动入库 | 错误率 | 上界95% |
|---|---|---|---|
| 0.05 / 0.95 | 94% | 0.128 | 0.153 |
| 0.1 / 0.9 | 95% | 0.132 | 0.158 |
| 0.2 / 0.8 | 98% | 0.138 | 0.163 |

手设阈值几乎失灵（分数堆在两端）。反方向：预算 10% 需要 `t_auto=1.0`，自动占比降到 39%；**预算 5% 和 2% 无解**。

85 条归档轨迹，alpha=0.10，有监督头 ECE 0.046：

| | 折叠到轨迹层 | 按层拆分 |
|---|---|---|
| 入库 | 6 条 | **模式层 63**、智能体层 33、步骤层 12 |
| 全量复审 | 28 条 | 只剩步骤层 28 |

**28 条 REVIEW 全部由步骤层触发**（27 条纯因 `step:set_size`），其中模式层已完全定论的有 15 条。所以 **REVIEW 不能丢**（Manski 界宽 33%），但也不必全读——按层拆分即可。

alpha 扫描：**AUTO 在任何 alpha 下都不超过 11%**；alpha 过 0.2 后预测集变空，NOVEL 飙到 44.7%、77.6%。**放宽 alpha 买到的是假新颖，不是自动化。** AUTO 还非单调（7.1 → 10.6 → 4.7）。

### Stage 4 开关（目标 3）

`EVOLUTION_DEFAULT = False`、`PipelineConfig.enable_evolution = False`、`evolution_enabled(cfg)` 唯一真相来源、`require_evolution(cfg)` 在 `run_evolution.py` 与 `ablate_seed.py` 入口抛 `EvolutionDisabled`，两者都加了 `--enable-evolution`。无该字段的配置对象读作关闭。

---

## 四、已批准的下一步（按顺序）

1. **跑完 deepseek Arm B**（按第二节的分批方案），然后 Stage 4 分诊 + `run_gate.py --min-cluster-size 5` 查候选码 **c3**（glm 那边 support 9 vs 要求 10，差一个；deepseek novel 率 0.65/条，残差只可能出现在最后 26 条 MDAgents 里，过门约五五开）。**注意 Stage 4 现在要显式 `--enable-evolution`。**
2. **单槽位补标的对照实验**（我上一轮改过的推荐）：路线 A 只改 2.2.2 一个槽位，逐槽位投票设施已存在，成本约整臂的 1/20，所以**四个臂都能改**，不必把 gpt/deepseek 臂标成「修正前」。**但先测**：单独问 2.2.2 与联合产出是不同条件化，先在**本来不该变**的轨迹上（glm 臂多轮 DyLAN）比一致率。高 → 四臂全补；低 → 退回只全量重标 glm 两臂。
3. **合入路线 A (b)**：mdagents / clinical_agent(5) 的 spec 里把聚合者映射为 `synthesis`（已核实：凡允许 `decision` 的码都允许 `synthesis`，不会有码失去资格），同伴可见性改为按记录判定，`STAGE0_SEMANTICS` 升 `stage0-v2`。**必须等第 1 步完成**，否则 ka11 重启会让一个臂一半新一半旧。
4. **下游接住 `unobservable`**（优先级已提高，是目标 2 的前置）：现在 reconcile/mac 这类无输入记录的框架，同伴可见性仍落成 False，2.2.x 被当作「不适用」跳过，正确说法是「不可测」。两者在论文里混淆会把「发生率为零」读成「没发生」。
5. glm 配额恢复后收尾 step 验证（还差 44 槽位）。

---

## 五、目标 2：MAST 种子文件（待办，已讨论清楚）

`Taxonomy.load()` 读 JSON，`provenance` 字段早就预留了 `"MAST = general seed"`。缺的是种子文件本身：MAST 在 `mast.py` 里只有散文定义，**没有 `allowed_phases`、没有 `requires`、没有逐码问题**，Stage 1 建不出槽位计划。

两个难点：
1. **适用条件要逐码判定。** FM-1.x 多为单智能体、无能力前提；**FM-2.x 全依赖同伴可见或多轮**——正好撞上第四节第 4 项那个缺口，所以**先做 `unobservable`**。
2. **模拟语料不能用。** 拿 MAST 注入的轨迹做 MAST 标注是循环的。MAST 臂只能跑在真实 AEGIS split（clean 440 / real-faults 1,112）或 MedAgentAudit 语料上，那里没有 MAST 真值，只能靠人工标签或一致性验证。

**我上一轮提出、用户尚未答复的问题：** 要不要先做一份 MAST 十六码的结构化适用条件草案（逐条给 `requires` 与 `allowed_phases`）供审？不需要调模型，不影响在跑任务。

---

## 六、五个外部框架的接入判断（已调研，无一有样例日志）

| 框架 | 写什么 | 结论 |
|---|---|---|
| **LungNoduleAgent** | `conversation[*].responses[*]` 带 `agent`/`role`/`content`，外层 `round_num`、`summary` | **L1 可直接接**，已按其序列化代码写 spec 并在构造记录上验证 0 问题 |
| **ConSensus**（nokia） | 纯文本 `log.txt`：`[名] [SYSTEM/HUMAN/AI]` | 需确定性文本→JSON 前置解析（正则，不用模型）。`HUMAN` 块即输入，**同伴可见性可恢复** |
| **CodeSeeker** | 四阶段各一份 HF dataset，只有 `output` 列 | 先按案例跨文件连接。顺序流水线，无轮次，2.2.x **结构上不适用** |
| **OEMA** | 每查询一行 JSONL，按 `idx` | 三个智能体是三个脚本，不是同一记录的轮次；token 级 NER，体系用得上的部分少 |
| **TriageAgent** | **仓库无代码**，只有 prompts/datasets | 无可摄入物 |

顺带查出一个静默默认：轮次源自身就有 `round_num` 时，`{"field": "^round_num"}` 只搜祖先，找不到静默回退第 1 轮。阶段映射记录 `was_mapped`，**轮次不记**，值得修。

---

## 七、端点使用要点

| 端点 | 要点 |
|---|---|
| glm-5.3-flash（智谱） | **本会话遭遇账户级限流**（429 code 1302，0.4 秒即返回），与并发无关。强制思考，思维链计入 `max_tokens`，用阶梯 10k/20k/40k |
| deepseek-flash（官方） | 必须 `--reasoner-effort low --reasoner-max-tokens 24000`；Responses 后端有 20 个 logprob 候选；Stage 1 约 0.35 条/分；**只在整条轨迹完成时写盘** |
| gpt-5.6-sol-disc（aihubmix） | logprobs 仅 1 候选；并发 45 会 `cannot be served`，12 稳定 |
| claude-opus-5（aiberm） | Stage 4 的 judge，跨家族；**L3 的 spec 提议器用的也是它**（`anthropic` SDK，`base_url='https://aiberm.com'`，不带 `/v1`） |
| doubao / baidu-deepseek-v4.1-flash | **不可用** |

---

## 八、这个项目反复出现的错误模式

**「用一个便宜的代理量替代真实量，却从没在它会失效的边界上检查过」。** 本会话又出现多次，而且**每一次都骗过了当时的全部检查**：

- keepalive 检查**进程在不在**，而不是**有没有在工作**；
- 验证器检查**路径清单**，而不是**解析器实际读出了什么**；
- 自检用**通过计数**报告健康，而缺失的检查**静默消失**（146 vs 150）；
- 用**记录形状**当框架身份，而形状是整个发布包共有的；
- 用**完成计数**判断 deepseek 是否前进，而计数在重启后原地打转。

**新会话里做任何状态判断之前，先确认这个指标量的到底是什么、它在什么情况下会说谎。** 判断是否卡住要看**文件修改时间**，不要看计数器。
