# Stop Summation 论文精读笔记

原文：`Stop Summation: Min-Form Credit Assignment Is All Process Reward Model Needs for Reasoning`

PDF：`D:/WeChat Files/wxid_w4vjkrkpk86m21/FileStorage/File/2026-05/Stop Summation Min-Form Credit Assignment Is All Process Reward Model Needs for Reasoning(1).pdf`

会议标注：NeurIPS 2025

作者：Jie Cheng, Gang Xiong, Ruixi Qiao, Lijun Li, Chao Guo, Junle Wang, Yisheng Lv, Fei-Yue Wang

代码/模型：`https://github.com/CJReinforce/PURE`

## 1. 一句话结论

这篇论文的核心判断是：PRM 不是不能用于强化微调，问题主要出在传统 RL 的 sum-form credit assignment 会把未来所有高分步骤累加进当前动作价值，诱导模型 hack 高奖励模式；如果把过程奖励改成 min-form credit assignment，也就是让“未来最差一步”决定当前步骤价值，PRM-based RFT 会稳定很多，并且能用更少训练步达到接近 verifiable reward 的推理效果。

我的压缩理解：

> 用 PRM 训练推理模型时，不要奖励“累计看起来不错的过程”，而要惩罚“链条里最弱的一环”。

## 2. 研究问题

论文试图回答两个问题：

1. 为什么 PRM 在 test-time scaling 中好用，但在 reinforcement fine-tuning 中容易 reward hacking？
2. 如何让 PRM 的 dense feedback 在训练阶段也稳定有效，而不是被模型利用？

作者认为关键错配在于：

- Test-time scaling 中，PRM 常用 `min(process rewards)` 给整条推理路径打分。只要有一步明显错，整条答案就应被压低。
- 训练时如果用传统 discounted return，即 `sum(gamma^t * future rewards)`，高分的“思考样式”会被累积放大，即使后续解题错了，前面的漂亮废话也可能得到正向强化。

## 3. 背景概念

### PRM 与 verifiable reward

Verifiable reward 是整条回答级别的稀疏奖励。例如数学题最终答案对就给 +1，错就给 0。

PRM 是步骤级奖励。每一步生成后，PRM 判断这一步过程是否合理。优势是信号密集，缺点是模型可能学习“让 PRM 喜欢的样子”，而不是真的解题。

### Credit assignment

Credit assignment 问的是：一次成功或失败，应该把功劳/责任分配给哪些中间动作？

在传统 RL 中，某一步的 value 通常是未来奖励的折扣和：

```text
Q(st, at) = E[sum_{i>=t} gamma^(i-t) * r_i]
```

论文称这类做法为 sum-form credit assignment。

## 4. 方法：PURE

PURE 全称是 Process sUpervised Reinforcement lEarning。它不是新 PRM，也不是新 RL 框架，而是一个过程奖励变换和优势估计方案。

### 4.1 Min-form credit assignment

对一条 n 步推理路径，设第 w 步是 PRM 打分最低的 worst step：

```text
w = argmin(r_1^p, ..., r_n^p)
```

作者定义：

```text
G(st, at | tau) = min(r_t^p, ..., r_n^p), if t <= w
                = 0,                  if t > w
```

直觉：

- 最差步骤决定整条推理链的质量。
- 最差步骤之前的步骤也要承担责任，因为它们构成了导致错误步骤的上下文。
- 最差步骤之后的步骤不应再获得“补救式”的累计价值，因为 test-time 的 min aggregation 已经认为整条链被最差步骤限制住了。

### 4.2 实现方式

作者没有重写 PPO/RL 训练逻辑，而是先变换 process rewards：

```text
r_i^p* = softmax(-r_i^p / T) * r_i^p
```

更低的 PRM 分数会得到更大的权重。温度 T 越低，越接近只保留最差步骤奖励。论文主实验使用 `T = 0.1`。

然后把每个步骤级奖励放到该步骤最后一个 token 上，其他 token 奖励为 0。Verifiable reward 则放在整条回答的最后一个 token 上。

### 4.3 Advantage estimator

作者最终选择 RLOO，并为 process rewards 使用 token-level baseline。关键原因是避免 step-level baseline 偏向短回答。

如果 baseline 按步骤数平均，当两条回答的过程总分相近时，步骤更多的回答会被惩罚更重，模型可能学会只输出一步甚至不输出有效步骤。

## 5. 理论分析

论文给出两个假设：

- PRM 每步奖励估计误差有界：`|r_p - r*| <= epsilon`
- 奖励本身有界

结论：

- Sum-form 的 Q-value 误差会随 horizon 和 discount factor 累积，近似上界是 `epsilon / (1 - gamma)`。
- Min-form 的 Q-value 误差不随步骤数累积，上界仍是单步误差 `epsilon`。

这给了一个重要解释：推理链越长，sum-form 越容易把 PRM 的小错误累积成很大的价值偏差；min-form 则把错误限制在单步奖励尺度内。

## 6. 实验设计

### 6.1 PRM 训练

作者基于 Qwen2.5-Math-7B 训练 PURE-PRM-7B：

- 数据：PRM800K
- 任务：二分类步骤正确性
- 第一阶段：冻结 LLM，只训练 value head，3 epochs
- 第二阶段：解冻全模型，继续微调 1 epoch

PRM 评测：

- BoN@1024：GSM8K 91.6，MATH 62.6
- ProcessBench 平均 F1：57.5，略高于 Qwen2.5-Math-7B-PRM800K 的 56.5
- PRMBench overall：65.3，接近 Qwen2.5-Math-7B-PRM800K 的 65.5

### 6.2 RL 训练设置

三类奖励设置：

- PURE-PRM：只用过程奖励，不用 GT answer
- PURE-VR：只用 verifiable reward，类似 R1-Zero 风格
- PURE-PRM+VR：过程奖励为主，10% 题目带 GT answer，约 800 个 GT 问题 + 7200 个 open problems

模型：

- Qwen2.5-7B
- Qwen2.5-Math-7B
- Qwen2.5-Math-1.5B

评测集：

- MATH-500
- Minerva Math
- Olympiad Bench
- AIME24
- AMC23

## 7. 主要结果

### 7.1 性能结果

Qwen2.5-Math-7B 上：

| 方法 | MATH-500 | Minerva | Olympiad | AIME24 | AMC23 | Avg |
|---|---:|---:|---:|---:|---:|---:|
| Base | 71.8 | 29.8 | 35.1 | 13.3 | 47.5 | 39.5 |
| PURE-PRM | 81.8 | 38.2 | 44.7 | 16.7 | 60.0 | 49.3 |
| PURE-VR | 79.4 | 36.8 | 41.8 | 23.3 | 60.0 | 48.3 |
| PURE-PRM+VR | 82.6 | 37.1 | 44.1 | 20.0 | 82.5 | 53.3 |

关键观察：

- 只用 PRM，在 min-form 下可以达到与 verifiable reward 相近甚至略高的平均分。
- PRM+VR 最强，平均 53.3，在 AMC23 上尤其高，为 82.5。
- 10% GT signals 对缓解 reward hacking 很有帮助。

### 7.2 训练稳定性

论文最强的证据不是最终分数，而是训练曲线：

- Sum-form 下，PURE-PRM 和 PURE-PRM+VR 在约 step 25 就 collapse。
- 到 step 80，平均 benchmark 分数跌到约 30，低于 base model 的 39.5。
- Min-form 方法能稳定超过 200 steps，并达到 49.3/53.3 的平均分。

### 7.3 学习效率

作者声称 PRM-involved 方法约用 PURE-VR 30% 的训练步数，就能达到相同 MATH-500 accuracy。也就是说，过程奖励确实提高了训练效率，但前提是 credit assignment 不能让模型 exploit PRM。

## 8. Reward hacking 类型

论文总结了三类 PRM-induced reward hacking。

### Case 1：Only thinking, not solving

模型只输出“分析问题、列计划、描述思路”，但不真正求解。原因是 PRM 更喜欢某些 thinking pattern，sum-form 又把这些高分步骤强化得更厉害。

这点很有启发：奖励“思考痕迹”可能会把模型训练成会表演思考，而不是会解决问题。

### Case 2：Extremely few steps, 1 step

如果 advantage baseline 设计不当，模型会偏向更少步骤，因为多步骤回答在 baseline 中被惩罚更多。最终模型可能输出一个巨长步骤，绕开逐步评估。

作者用 token-level baseline 解决这个偏差。

### Case 3：Extremely few steps, 0 step

模型输出无关内容、空回答、礼貌语等，例如 “Thank you.”。由于 discriminative PRM 是 causal attention，它看到当前短文本时可能给高分，却不知道后面没有实际解题内容。

作者认为这类问题靠 min-form 仍不能根除，需要 GT-level signal 辅助，或者未来用 generative PRM。

## 9. 训练崩溃原因

作者进一步分析一个故意放大 process reward 的实验：

- 在 Qwen2.5-7B 上用 PURE-PRM+VR，并把 process rewards 加倍。
- step 365 出现训练崩溃，reward 和 MATH-500 accuracy 急剧下降。
- step 361 前后，correct responses 中开始出现长且高度重复的样本。
- 这些被 verifier 判定为正确、但模式质量很差的样本被作者称为 pseudo-positive samples。

核心结论：

> Long and highly repetitive “correct” responses can collapse training within 5 gradient steps.

这说明 verifier 只看最终答案还不够；PRM 如果训练数据没覆盖重复/坏模式，也无法可靠拦截。未来 PRM 需要同时评估 step correctness 和 pattern quality。

## 10. 消融结果

### Transform temperature

在 Qwen2.5-7B + PURE-PRM+VR 上：

| T | Avg |
|---:|---:|
| 0.01 | 42.8 |
| 0.1 | 45.6 |
| 1.0 | 42.8 |

论文选择 `T = 0.1`。解释上，T 太小可能过度只看最差步，T 太大又不够接近 min-form。

### Advantage estimators

作者比较 GAE、GRPO、REINFORCE++、RLOO：

- 前 150 steps 类似。
- REINFORCE++ 在约 step 220 有 spike。
- GAE 后期更稳一点，但因为额外 value network，训练前后向耗时多约 30%。
- 综合表现、耗时和稳定性，作者选 RLOO。

## 11. 贡献

1. 把 PRM 训练失败的问题定位到 training-time sum-form credit assignment 和 test-time min aggregation 的错配。
2. 提出 min-form credit assignment，把“最差步骤”作为推理链质量瓶颈。
3. 给出误差上界分析，说明 sum-form 会累积 PRM 误差，而 min-form 不会随步骤数累积。
4. 在 3 个 Qwen 基座和 5 个数学 benchmark 上验证 PRM-based RFT 的可行性。
5. 系统归纳了 PRM reward hacking 的三种案例和训练崩溃中的 pseudo-positive samples。

## 12. 局限与质疑

### 12.1 Min-form 可能过于悲观

如果 PRM 某一步误判为低分，min-form 会让整条链受到强惩罚。理论上它限制误差累积，但实践中对单个 false negative 可能非常敏感。

### 12.2 任务集中在数学推理

实验主要是数学 benchmark。对代码、长文本规划、开放式问答、多轮 agent 任务是否同样成立，还不能直接外推。

### 12.3 PRM 自身质量仍是瓶颈

作者的 PURE-PRM-7B 本身已经是强 PRM。如果 PRM 较弱，min-form 是否仍能稳定提升，需要额外验证。

### 12.4 10% GT signals 很关键

最优结果来自 PRM+VR，不是纯 PRM。论文标题强调 “PRM needs”，但正文结果表明少量 verifiable reward 对最终稳定性和抗 hacking 仍很重要。

### 12.5 Best checkpoint 评估可能偏乐观

论文报告 training 中保存的 best checkpoint。若实际训练中没有可靠选择准则，线上使用可能仍要依赖验证集监控 reward hacking。

## 13. 对我有用的启发

### 13.1 奖励不是越密越好，credit assignment 决定奖励如何被学习

PRM 给 dense feedback，但 dense feedback 进入 RL 后会被 advantage/return 机制重塑。奖励模型质量、奖励聚合方式、baseline 设计是一个整体。

### 13.2 推理链应按瓶颈质量建模

对数学推理这类“错一步可能全错”的任务，min aggregation 很自然。可迁移到：

- theorem proving
- code generation with intermediate tests
- multi-step tool use
- long-horizon agent planning

但对允许局部错误后自我修正的任务，min-form 是否过严，需要具体分析。

### 13.3 训练监控要看模式指标

只看 reward 和 accuracy 不够，还要看：

- response length
- clip ratio
- repetition score
- high repetition ratio
- empty/irrelevant response ratio
- thinking-only ratio

很多 collapse 不是慢慢坏掉，而是几步内突然坏掉。

### 13.4 PRM 数据集要覆盖坏模式

如果 PRM 训练数据只有“数学步骤对/错”，却没有“重复、空回答、模板化思考、无解题内容”等负样本，模型迟早可能钻空子。

## 14. 可以复用的研究假设

1. 在长链推理任务中，min-form credit assignment 比 sum-form 更抗 PRM 误差累积。
2. 对纯 PRM 训练，少量 verifiable reward 可以充当 anti-hacking anchor。
3. Step-level baseline 会诱导短步骤偏好，token-level baseline 更适合 process reward。
4. PRM 训练数据中的 pattern-negative examples 对防止 collapse 可能和 correctness labels 一样重要。
5. 对“任一步失败即整体失败”的任务，min-form 比 sum-form 更符合任务结构。

## 15. 复读路线

第一次复读：

- 读 §3.1，理解 min-form 的三条直觉。
- 看 Figure 1，理解 sum-form 为什么会奖励 thinking-only。
- 看 Table 2，记住 PRM、VR、PRM+VR 的相对关系。

第二次复读：

- 读 §3.4，重点看 token-level baseline 为什么防止 1-step hacking。
- 读 §5.1，整理三类 reward hacking。
- 读 §5.2，理解 pseudo-positive samples 如何让训练 5 步内崩溃。

第三次复读：

- 看 Appendix B 的 theorem proof。
- 看 Appendix F 的 temperature 和 advantage estimator 消融。
- 思考 min-form 在非数学任务中的边界。

## 16. 我的最终判断

这篇论文的价值不只是提出一个简单有效的 reward transform，而是提醒我们：PRM-based RL 的失败未必来自 PRM 本身，也可能来自“训练时如何使用 PRM”。它把 test-time scaling 中已经被验证有效的 min aggregation，迁移成 training-time credit assignment，逻辑上比较漂亮，工程上也足够轻量。

不过，论文并没有证明 PRM 可以完全替代 verifiable reward。更稳妥的实践结论是：

> PRM 可以作为主训练信号，但最好用少量 ground-truth/verifiable reward 做锚定，并持续监控长度、重复和无效输出等模式指标。

