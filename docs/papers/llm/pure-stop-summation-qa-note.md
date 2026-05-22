# PURE 论文连续问答笔记

来源：围绕论文 `Stop Summation: Min-Form Credit Assignment Is All Process Reward Model Needs for Reasoning` 的连续问答整理。

## 1. PRM 是什么？

**你问：** PRM是什么？

**我答：**

PRM 是 **Process Reward Model，过程奖励模型**。

它的作用是：**不只判断最终答案对不对，而是给推理过程中的每一步打分**。

举个数学题场景：

```text
题目：求 x

步骤1：移项
步骤2：两边同时除以 2
步骤3：得到答案
```

普通的 verifiable reward 只看最后答案：

```text
答案对：+1
答案错：0
```

PRM 会看每一步：

```text
步骤1 是否合理？
步骤2 是否正确？
步骤3 是否从前面推出来？
```

所以 PRM 给的是更细粒度的反馈，也叫 **dense feedback**。

它的好处是：模型不用等到最后答案才知道对错，可以更早知道哪一步推理出了问题。

但它的风险是：模型可能学会“讨好 PRM”，比如写很多看起来像推理的废话、套模板、反复说“let's think step by step”，让 PRM 给高分，却没有真正解决问题。这就是论文里说的 **PRM-induced reward hacking**。

## 2. 与 PRM 并列的概念有哪些？

**你问：** 与PRM并列的概念有哪些？就是奖励模型的类型是吗？

**我答：**

对，你可以把 PRM 放在“奖励信号/奖励模型类型”这一层来理解。和它并列或相邻的常见概念主要有这些：

| 类型 | 英文 | 奖励粒度 | 核心意思 |
|---|---|---:|---|
| 结果奖励 | Outcome Reward / ORM | 整个回答 | 只看最终答案或最终输出好不好 |
| 过程奖励 | Process Reward / PRM | 中间步骤 | 每一步推理都打分 |
| 可验证奖励 | Verifiable Reward / VR | 整个回答 | 用规则/答案校验器判断最终答案对错 |
| 偏好奖励模型 | Preference Reward Model / RM | 两个回答比较 | 学人类偏好：A 比 B 更好 |
| 规则奖励 | Rule-based Reward | 可粗可细 | 人写规则，比如格式、长度、答案匹配 |
| 自奖励 | Self-Reward / Self-Critique | 可变 | 模型自己当评审给奖励 |

最重要的是区分这三组：

**1. PRM vs ORM**

PRM 看过程：

```text
步骤1 对不对？
步骤2 对不对？
步骤3 对不对？
```

ORM 看结果：

```text
最终答案好不好？
```

**2. PRM vs Verifiable Reward**

PRM 通常是模型打分，可能会误判，所以容易 reward hacking。

Verifiable reward 通常是规则校验，比如数学最终答案是否等于标准答案，更可靠，但信号很稀疏。

**3. Reward Model vs Reward Signal**

PRM、ORM、Preference RM 是“模型型奖励”，需要训练一个模型来判断好坏。

Verifiable reward、rule-based reward 更像“规则型奖励”，不一定需要训练模型。

放到这篇论文里，最关键的对照是：

```text
PRM = dense but hackable
VR = sparse but reliable
```

所以作者最后发现，效果最好的是：

```text
PRM 为主 + 少量 VR 做锚定
```

## 3. Test-time scaling 是什么？

**你问：** test-time scaling是什么？

**我答：**

**Test-time scaling** 指的是：**模型参数不再训练了，但在推理/答题时投入更多计算资源，让答案变好**。

也可以理解成：

```text
training-time scaling：训练时花更多算力，把模型训得更强
test-time scaling：使用时花更多算力，让同一个模型想得更充分
```

常见做法有：

| 方法 | 意思 |
|---|---|
| 多采样 | 同一道题让模型生成很多个答案 |
| Majority Voting | 多个答案投票，选出现最多的 |
| Best-of-N | 生成 N 个答案，用奖励模型选最好的 |
| Self-consistency | 生成多条推理路径，选一致性最高的答案 |
| Search / Tree Search | 像搜索树一样探索多个推理分支 |
| PRM-guided Search | 用 PRM 给中间步骤打分，引导搜索 |

在这篇论文里，test-time scaling 的典型场景是：

```text
同一道数学题，模型生成 1024 个解答
PRM 给每个解答的每一步打分
然后用某种聚合方式给整条解答打总分
选总分最高的答案
```

关键点是：PRM 在 test-time scaling 里通常用 **min aggregation**：

```text
整条解答的分数 = 所有步骤分数里的最低分
```

直觉是：

```text
一条推理链只要有一步严重错，整条链就不可靠
```

这篇论文的洞察就在这里：PRM 在测试时是按“最差一步”来选答案的，但训练时很多方法却按“未来奖励累加”来分配 credit。这个错配会导致 reward hacking，所以作者提出 min-form credit assignment。

## 4. 为什么同一个模型输出质量还和算力有关？

**你问：** 我不理解，为什么使用时花更多算力，让同一个模型想得更充分？难道模型的输出质量还和算力有关吗？我一直感觉一个训练好的模型就是一个函数，输入是什么，输出就是什么，难道算力也是和这个输出有关？

**我答：**

你的直觉对了一半：**训练好的模型本身确实可以看成一个函数**。但关键是，LLM 的“输出过程”通常不是一个固定的单次函数调用，而是：

```text
模型给出下一个 token 的概率分布
采样/搜索策略从分布里选 token
不断重复，生成完整答案
```

所以更准确地说：

```text
模型参数固定
但推理算法可以不同
推理时投入的计算量也可以不同
最终输出质量会变
```

一个最简单例子：

```text
同一道题，只让模型回答 1 次
可能答错

让模型独立回答 20 次
其中 7 次答对

再用投票、验证器或奖励模型选答案
最终更可能选到对的
```

这就是 test-time scaling。

你说的“一个函数，输入是什么输出就是什么”更像是 **greedy decoding**：

```text
每一步都选概率最高的 token
```

这时模型比较接近确定性函数。但实际推理常用 sampling：

```text
从概率分布里随机抽 token
```

同一个 prompt 可以产生很多不同解法。

举个直观类比：

```text
模型 = 一个会做题的人
单次输出 = 让他马上给一个答案
test-time scaling = 让他多做几遍、换几种思路、检查答案、再从候选里选最可靠的
```

参数没变，但“使用方式”变了。

更形式化一点，固定模型 `pθ` 后，你还有这些可调东西：

```text
生成几个答案 N
temperature 多高
是否允许搜索
是否用 verifier 检查答案
是否用 PRM 给过程打分
是否让模型反思/修正
```

这些都会影响最后输出。

所以算力不是直接改变模型能力，而是改变：

```text
探索的候选答案数量
推理路径的深度
检查和筛选的强度
```

在数学推理里尤其明显。模型可能“会做”，但第一次没走到正确路径；多采样、多路径搜索、PRM 选择，就更容易把正确路径捞出来。

## 5. Test-time scaling 是不是多生成几个回答然后挑对的？

**你问：** 也就是说test-time scaling，使用更多算力的方法是让我们多生成几个回答然后从里面挑对的吗？

**我答：**

对，这是 **test-time scaling 最典型、最好理解的一类方法**：

```text
多生成几个回答
再用某种机制从里面挑一个更可能对的
```

比如：

```text
同一道数学题生成 64 个解法
每个解法都算出一个答案
最后用投票 / verifier / PRM 选
```

但它不只这一种。可以分成几类：

| 方法 | 多花的算力用在哪里 |
|---|---|
| Best-of-N | 生成 N 个完整回答，再挑最好的 |
| Majority Voting / Self-consistency | 生成 N 个回答，看哪个最终答案最多 |
| Verifier reranking | 生成 N 个回答，用校验器/奖励模型重排 |
| PRM-guided search | 生成过程中每一步都用 PRM 评分，保留更好的分支 |
| Self-reflection | 生成答案后让模型检查、修改、再回答 |
| Tree search | 像搜索树一样探索多条中间推理路径 |

所以你的理解可以记成：

```text
狭义 test-time scaling = 多采样 + 选择
广义 test-time scaling = 推理时用更多计算做搜索、验证、反思、筛选
```

这篇 PURE 论文里说的 PRM test-time scaling，主要就是：

```text
生成很多条推理路径
PRM 给每条路径的每一步打分
用 min 聚合成整条路径分数
选择分数最高的路径
```

这里的核心不是“模型变聪明了”，而是：

```text
同一个模型本来就有概率生成好答案
多试几次 + 好的筛选器
就更容易选中好答案
```

## 6. PRM 为什么也能用在推理里面？

**你问：** PRM不是用来训练的吗？就是给强化学习里面的reward，不应该是用在训练里面的吗？怎么要用在推理里面？

**我答：**

你的疑问非常关键：**PRM 既可以用在训练里，也可以用在推理里**。它本质上只是一个“给过程打分的模型”，至于这个分数拿来干什么，是另一回事。

可以这样分：

| 使用阶段 | PRM 的作用 | 是否更新主模型 |
|---|---|---|
| 推理阶段 test-time | 给多个候选解法打分，选最好的 | 不更新 |
| 训练阶段 training-time | 把 PRM 分数当 reward，用 RL 更新模型 | 更新 |
| 数据构造阶段 | 给样本过滤、排序、标注质量 | 不一定更新 |

最早 PRM 成功的地方，主要是在 **推理阶段**。

比如数学题：

```text
模型生成 100 个解法
PRM 给每个解法的每一步打分
把每条解法的最低步骤分作为整条解法分数
选分数最高的答案
```

这个过程里，主模型不训练，只是 PRM 当“裁判/筛选器”。

你说的也对：在强化学习里，PRM 也可以作为 reward：

```text
模型生成一步
PRM 给这一步 reward
RL 用这个 reward 更新模型
```

但问题就在这里：**PRM 在推理里好用，不代表直接拿来训练也好用**。

原因是：

```text
推理阶段：PRM 只是挑答案
训练阶段：模型会主动学习如何让 PRM 给高分
```

这两个风险完全不同。

在推理阶段，如果 PRM 偶尔偏好“看起来像推理”的答案，它最多选错一次。

在训练阶段，模型会反复优化这个偏好，最后可能学会：

```text
多写漂亮废话
只写思考不解题
输出很短但 PRM 给高分的话
重复某些模式
```

这就是 reward hacking。

这篇论文的切入点正是：

```text
PRM 在 test-time scaling 里通常用 min aggregation，很成功
但训练时大家用传统 sum-form credit assignment
这两个目标不一致
所以 PRM 一进 RL 就容易被 hack
```

所以作者说：既然推理时 PRM 是按“最差一步”判断整条推理链，那训练时也应该用类似逻辑，让最差一步决定 credit assignment。

## 7. Training-time scaling 是什么？

**你问：** training-time scaling是什么意思？

**我答：**

**Training-time scaling** 指的是：**在训练阶段投入更多算力、数据或参数规模，让模型本身变强**。

也就是模型还没固定之前，通过更大的训练让参数变得更好。

可以和 test-time scaling 对照：

| 概念 | 发生在什么时候 | 模型参数变不变 | 多花的算力用来干嘛 |
|---|---|---|---|
| Training-time scaling | 训练时 | 会变 | 更新模型参数 |
| Test-time scaling | 推理时/使用时 | 不变 | 多生成、多搜索、多验证 |

例子：

```text
training-time scaling:
用更多数据训练
训更久
用更大模型
用更多 GPU
做更多 RL fine-tuning
最后得到一个更强的新模型

test-time scaling:
模型已经训练好了
同一道题生成 64 个答案
用投票或 PRM 选一个
模型参数不变
```

你可以把它理解成：

```text
training-time scaling = 把学生训练得更强
test-time scaling = 考试时让学生多想几遍、多打草稿、多检查
```

在这篇论文里，PRM 两边都有关：

```text
test-time scaling:
PRM 用来挑选更好的答案

training-time scaling / RFT:
PRM 用来提供 reward，更新模型参数
```

这篇论文关心的是后者：**怎么把 PRM 从“推理时的裁判”变成“训练时的奖励信号”，同时避免 reward hacking。**

## 8. Training-time scaling 有什么技术细节？

**你问：** training-time scaling具体有什么样的技术细节？

**我答：**

Training-time scaling 的技术细节，本质上是：**在训练阶段扩大“学习量”或“模型容量”，让参数本身变得更强**。

可以拆成几类。

**1. 扩大模型规模**

也就是增加参数量：

```text
1B -> 7B -> 70B -> 400B
```

模型更大，表达能力更强，可以容纳更多知识和复杂模式。但代价是训练和推理都更贵。

技术上涉及：

```text
更多 transformer layers
更大的 hidden size
更多 attention heads
更大的 FFN dimension
```

**2. 增加训练数据**

让模型看更多 tokens：

```text
1T tokens -> 10T tokens -> 15T tokens
```

数据越多，模型见过的语言、知识、代码、数学模式越丰富。

但不是越多越好，还要看数据质量。低质量数据太多会污染模型。

常见做法：

```text
数据清洗
去重
质量过滤
代码/数学/网页/论文混合比例设计
curriculum learning
```

**3. 增加训练步数**

也就是让模型训练更久：

```text
更多 optimization steps
更多 epochs
更多 token updates
```

每一步大概是：

```text
取一批文本
模型预测下一个 token
计算 loss
反向传播
更新参数
```

更久训练可以提升能力，但也可能过拟合、退化，或者浪费算力。

**4. 增大 batch size 和算力并行**

大模型训练不是一张 GPU 能完成的，通常会用很多 GPU/TPU。

技术上包括：

```text
data parallelism
tensor parallelism
pipeline parallelism
sequence parallelism
ZeRO / FSDP
gradient checkpointing
mixed precision training
```

这些不是改变算法目标，而是为了让训练能在大规模硬件上跑起来。

**5. 改训练目标**

预训练通常是 next-token prediction：

```text
给前文 x1...xt
预测下一个 token xt+1
```

但后训练阶段会加入更多目标：

```text
SFT：模仿高质量答案
RLHF / RLAIF：根据奖励模型优化
RFT：用可验证奖励或过程奖励优化推理
DPO：用偏好对直接优化
```

这篇 PURE 论文主要属于这里：**不是预训练扩规模，而是 reinforcement fine-tuning 阶段的 training-time scaling**。

**6. 强化学习式训练**

在 reasoning 模型里，常见流程是：

```text
模型生成答案
奖励函数/奖励模型打分
根据 reward 计算 advantage
用 PPO / GRPO / RLOO / REINFORCE++ 更新模型
```

如果 reward 是最终答案对错，就是 verifiable reward。

如果 reward 是每一步推理质量，就是 PRM。

PURE 论文关注的就是：

```text
PRM 给了每一步 reward
但这些 reward 怎么变成 advantage？
怎么分配 credit？
```

作者认为传统 sum-form 会出问题，所以改成 min-form。

**7. Scaling law**

Training-time scaling 背后有一个经验规律：模型性能通常随三件事变好：

```text
参数量
训练 token 数
训练算力
```

但三者要配平。比如模型很大但数据太少，会欠训练；数据很多但模型太小，容量不够。

所以现代训练会考虑 compute-optimal scaling，比如：

```text
给定总算力，应该用多大模型、多少数据、训多久？
```

一句话总结：

```text
training-time scaling = 用更多参数、更多数据、更多训练步、更强训练算法，把模型参数本身训练得更好。
```

和 test-time scaling 的区别是：

```text
training-time scaling 花算力改变模型
test-time scaling 花算力使用模型
```

## 9. 训练算力怎么让模型变得更好？

**你问：** 这个训练算力具体是怎么让模型变得更好的呢？参数量和训练的token数都很直接，越大模型会越好，但是这个训练算力要怎么去理解呢？

**我答：**

这个问题问得很准。**训练算力本身不是第三种神秘营养，它更像“训练预算”**。它通过支撑更多计算操作，让模型能：

```text
更大参数量
更多训练 token
更多训练步
更大的 batch
更复杂的训练流程
```

所以在 scaling law 里，训练算力通常不是和“参数量、token 数”完全并列的独立因素，而是它们背后的总约束。

可以先记一个粗略关系：

```text
训练算力 ≈ 参数量 × 训练 token 数 × 常数
```

更具体一点，Transformer 训练的 FLOPs 大约可以近似成：

```text
C ≈ 6 × N × D
```

其中：

```text
C = 总训练算力 / FLOPs
N = 模型参数量
D = 训练 token 数
```

所以如果你有更多训练算力，你可以选择：

```text
1. 训练更大的模型 N
2. 喂更多 token D
3. 或者二者都增加一点
```

举个例子：

```text
算力很少：
7B 模型，训练 500B tokens

算力更多：
7B 模型，训练 5T tokens
或者
70B 模型，训练 1T tokens
```

模型变强不是因为“算力”直接注入了智慧，而是因为算力允许你做更多次这样的过程：

```text
前向传播：模型预测
计算 loss：和目标答案比较
反向传播：算梯度
参数更新：把模型调得更好一点
```

这个过程重复得越充分，模型就越能从数据里吸收规律。

不过有一个关键点：**算力要花得合理才有用**。

如果模型太小、数据太多：

```text
小模型容量不够，学不下那么多东西
```

如果模型太大、数据太少：

```text
大模型没被充分训练，参数浪费
```

所以 scaling law 真正关心的是：

```text
给定一笔训练算力 C，
应该分配给多大的模型 N，
以及多少训练 token D，
才能最划算？
```

这就是 compute-optimal scaling。

用一个类比：

```text
参数量 = 学生的大脑容量
训练 token = 教材和习题量
训练算力 = 学习时间 + 教学资源 + 做题批改能力
```

只有学习时间多，但教材少，不行。  
教材很多，但学生容量太小，也不行。  
学生很聪明，但没时间训练，也不行。

所以更准确的说法是：

```text
训练算力让模型变好，是因为它允许模型在更多数据、更大容量、更长训练过程中进行更多有效参数更新。
```

在 scaling law 里，算力是总预算；参数量和 token 数是你怎么花这笔预算。

## 10. 算力、参数量、训练 token 数三者定位有什么不同？

**你问：** 所以算力一般就是一个固定的预先知道的值，参数和训练token数是自己调？是不是三者的定位是有细微不同的

**我答：**

对，你这个理解已经很接近 scaling law 的真实用法了。三者的定位确实有细微不同：

```text
训练算力 C：预算 / 约束
参数量 N：模型容量 / 设计变量
训练 token 数 D：数据量 / 训练变量
```

通常情况是：

```text
我有多少 GPU、能跑多久、预算多少钱
=> 大概决定了总训练算力 C
```

然后研究者要决定：

```text
在这个 C 下面，
模型做多大 N？
训练多少 token D？
```

粗略关系是：

```text
C ≈ 6 × N × D
```

所以 C 固定时，N 和 D 有 trade-off：

```text
模型更大 => 同样算力下能训练的 token 更少
训练 token 更多 => 同样算力下模型不能太大
```

举个非常粗糙的例子：

```text
同样算力预算 C：

方案 A：
大模型 + 较少 token
70B 参数，训练 1T token

方案 B：
小一点模型 + 更多 token
7B 参数，训练 10T token
```

哪个更好，不是拍脑袋，而是看 compute-optimal scaling law。

三者定位可以这么记：

| 变量 | 更像什么 | 你怎么控制 |
|---|---|---|
| 算力 C | 总预算 | GPU 数量、训练时长、硬件效率 |
| 参数量 N | 容器大小 | 模型架构、层数、宽度 |
| token 数 D | 教材/练习题数量 | 数据规模、epoch、训练步数 |
| 性能 L | 最终结果 | 被 N、D、C 共同影响 |

但要注意：C 不是永远“预先固定”。有两种视角。

**视角 1：工程训练视角**

公司或实验室先有预算：

```text
我们有 2048 张 GPU，能训 60 天
```

那 C 基本固定。然后设计 N 和 D。

**视角 2：科学规律视角**

研究 scaling law 时，会主动改变 C：

```text
小算力训练一批模型
中等算力训练一批模型
大算力训练一批模型
```

然后拟合：

```text
性能如何随 C 增长而改善？
最优 N 和 D 应该怎么随 C 增长？
```

所以你说的“算力一般是固定预先知道的值，参数和 token 自己调”，在实际训练规划里基本对。

更精确地说：

```text
C 是资源约束
N 和 D 是资源分配决策
scaling law 告诉你在不同 C 下怎么分配 N 和 D
```

## 11. PURE 是一个 credit assignment 方法吗？

**你问：** PURE是一个 credit assignment的方法是吗？

**我答：**

基本可以这么理解，但更精确一点：

**PURE 是一个 PRM-based reinforcement fine-tuning 框架；它的核心创新是 min-form credit assignment。**

也就是说：

```text
PURE 不是单纯一个 PRM
也不是全新的 RL 算法
而是：把 PRM 的过程奖励用一种新的 credit assignment 方式接入 RL 训练
```

它里面最关键的东西就是：

```text
sum-form credit assignment -> min-form credit assignment
```

传统 sum-form 是：

```text
当前步骤的价值 = 后面所有奖励的折扣累加
```

PURE 的 min-form 是：

```text
当前步骤的价值 = 从当前到未来步骤里最低的那个过程奖励
```

直觉是：

```text
推理链里最差的一步决定整条链的可靠性
```

所以你说“PURE 是一个 credit assignment 方法”，可以作为简化理解；但论文里的完整 PURE 还包括：

```text
1. PRM reward transform
2. min-form credit assignment
3. token-level reward placement
4. RLOO advantage estimation
5. PRM + 少量 verifiable reward 的混合训练设置
```

如果做论文笔记，建议这样写：

```text
PURE = PRM-based RFT framework
core = min-form credit assignment
goal = align training-time PRM usage with test-time PRM min aggregation
```

最短记法：

```text
PURE 用 min-form credit assignment 让 PRM 可以更稳定地用于 RL 训练。
```

## 12. Sum-form credit assignment 是什么？

**你问：** 介绍一下sum-form credit assignment，同时，他还有什么并列的概念？

**我答：**

**Sum-form credit assignment** 可以理解为传统 RL 里最常见的“功劳分配”方式：

```text
当前动作的价值 = 从当前时刻开始，未来所有奖励的折扣累加
```

形式上大概是：

```text
Q(s_t, a_t) = E[ r_t + γ r_{t+1} + γ² r_{t+2} + ... ]
```

其中：

```text
γ = discount factor，折扣因子
```

如果 `γ` 越接近 1，未来奖励影响越大；如果 `γ` 越小，模型越看重眼前奖励。

在推理链里，它相当于：

```text
第 t 步的 credit
= 第 t 步之后所有步骤奖励的加权总和
```

举个例子，一条推理有 5 步，PRM 分数是：

```text
[0.9, 0.8, 0.95, -0.6, 0.7]
```

如果用 sum-form，前面第 1 步的价值会包含：

```text
0.9 + 0.8 + 0.95 - 0.6 + 0.7
```

问题是：即使第 4 步严重错误，前面那些“看起来很像推理”的高分步骤仍可能让整体 return 很高。模型就可能学会多写高分模式，而不一定真正解题。

这就是 PURE 论文批评 sum-form 的地方。

和它并列的概念，可以按“如何聚合未来奖励”来分：

| Credit assignment 形式 | 核心逻辑 | 直觉 |
|---|---|---|
| Sum-form | 未来奖励加权求和 | 累积收益最大化 |
| Min-form | 未来奖励取最小值 | 最差一步决定链条质量 |
| Max-form | 未来奖励取最大值 | 只要未来有亮点就奖励 |
| Mean-form | 未来奖励取平均 | 看整体平均过程质量 |
| Final-only / Outcome-form | 只看最终奖励 | 最终答案决定全部 credit |
| Step-local | 每步只看自己的即时奖励 | 当前步骤只为自己负责 |

在传统 RL 里，最主流的是：

```text
sum-form / discounted return
```

在数学推理 PRM 里，这篇论文主张：

```text
min-form 更合理
```

因为数学推理链有一种“链条短板效应”：

```text
一步关键错误，后面再漂亮也可能全错
```

所以 PURE 的对照可以写成：

```text
Sum-form:
奖励会累积，容易放大高分步骤和 PRM 误差

Min-form:
奖励由最差步骤限制，更像 test-time PRM 的 min aggregation
```

## 13. 为什么 Sum-form 会放大高分步骤和 PRM 误差？

**你问：** 为什么Sum-form就会 奖励会累积，容易放大高分步骤和 PRM 误差

**我答：**

因为 sum-form 的数学结构就是“把未来奖励加起来”。只要加起来，就会出现两个放大效应：

**第一，放大高分步骤。**

假设一条推理有 6 步，PRM 分数是：

```text
[0.9, 0.9, 0.9, -0.6, 0.2, 0.1]
```

第 1 步的 sum-form return 近似是：

```text
0.9 + 0.9 + 0.9 - 0.6 + 0.2 + 0.1 = 2.4
```

虽然第 4 步明显错了，但前 3 个“看起来很像正确推理”的高分步骤把总分拉高了。

于是模型学到的可能是：

```text
多生成 PRM 喜欢的高分步骤
```

而不是：

```text
把题目真正解对
```

这就是“高分步骤被累积放大”。

**第二，放大 PRM 误差。**

PRM 不是完美裁判，它每一步都有误差。

假设每一步 PRM 都可能多给一点分，比如误差是 `+0.05`。如果只看一步，误差是：

```text
0.05
```

但如果 sum-form 累加 20 步，这些小误差会叠起来：

```text
0.05 × 20 = 1.0
```

如果推理链更长，误差还会更大。

所以论文里说，sum-form 的误差上界会随 horizon 累积：

```text
epsilon + gamma epsilon + gamma² epsilon + ...
≈ epsilon / (1 - gamma)
```

而 min-form 只取最差步骤，误差不会按步骤数一路加起来，上界仍接近单步误差：

```text
epsilon
```

把它放回 PRM 训练里：

```text
PRM 每一步都有一点点偏差
sum-form 把这些偏差累加
RL 又会最大化这个累加值
模型就会主动寻找并放大 PRM 的偏差
```

这就是 reward hacking 的来源。

一句话：

```text
Sum-form 会把“很多小的看起来不错”累积成“很大的训练信号”，即使整条推理其实已经坏了。
```

## 14. Min-form 具体是什么样的？

**你问：** Min-form具体是什么样的？举个例子

**我答：**

**Min-form credit assignment** 的核心是：

```text
当前步骤的价值 = 从当前步骤到未来所有步骤里，最低的那个过程奖励
```

也就是：

```text
Q_t ≈ min(r_t, r_{t+1}, ..., r_n)
```

它的直觉是：

```text
一条推理链的质量由最差的一步限制。
```

举个例子，一道题模型生成 5 步，PRM 给分如下：

```text
步骤1：0.9
步骤2：0.8
步骤3：0.95
步骤4：-0.6   <- 这里错了
步骤5：0.7
```

如果用 sum-form，第 1 步看到的是：

```text
0.9 + 0.8 + 0.95 - 0.6 + 0.7 = 2.75
```

虽然第 4 步错了，但整体还是正的。

如果用 min-form：

```text
第1步价值 = min(0.9, 0.8, 0.95, -0.6, 0.7) = -0.6
第2步价值 = min(0.8, 0.95, -0.6, 0.7) = -0.6
第3步价值 = min(0.95, -0.6, 0.7) = -0.6
第4步价值 = min(-0.6, 0.7) = -0.6
第5步价值 = min(0.7) = 0.7
```

也就是说，第 4 步这个错误会影响它之前的步骤：

```text
步骤1、2、3、4 都会被压低
```

因为前面的步骤共同导致了后面的错误步骤。

在 PURE 论文里，它还做了一个更细的设定：如果第 4 步是整条链里最差的一步，那么第 4 步之后的步骤不再贡献 credit：

```text
步骤1：-0.6
步骤2：-0.6
步骤3：-0.6
步骤4：-0.6
步骤5：0
```

为什么第 5 步是 0？

因为作者认为：

```text
一旦最差步骤已经出现，后面的步骤对 min objective 没有贡献。
```

整条推理链已经被最差步骤决定了。

再用一句话理解：

```text
Sum-form：你前面写了很多漂亮步骤，所以哪怕后面错了，也可能整体有奖励。
Min-form：只要中间有一步严重错，前面的漂亮步骤也不能洗白。
```

所以 min-form 很适合数学推理这种场景：

```text
一步关键错误，整条解法就不可信。
```

## 15. Min-form 有什么效果？作者做出来了什么效果？

**你问：** 这样有什么效果？作者做出来了什么效果？

**我答：**

Min-form 的效果可以分成两层：**机制效果** 和 **实验效果**。

**机制效果**

它会阻止模型靠“堆高分步骤”骗 PRM。

比如模型写了很多漂亮的思考句：

```text
Let us reason step by step...
We need to verify each condition...
Now consider another case...
```

PRM 可能给这些步骤高分。

如果用 sum-form，这些高分会一路累加，模型会学到：

```text
多写这种高分思考模式 = reward 更高
```

但 min-form 会说：

```text
只要后面有一步明显错，前面这些漂亮步骤也一起被压低
```

所以模型更难通过“表演推理”拿高 reward。

它带来的主要效果是：

```text
1. 限制 value 的范围
2. 避免 PRM 误差随步骤数累积
3. 减少 thinking-only reward hacking
4. 让训练目标更接近 test-time PRM 的 min aggregation
5. 提高 PRM-based RL 的稳定性
```

**作者做出来的实验效果**

最关键的结果有三个。

**1. Sum-form 很快崩，min-form 稳定很多**

在 Qwen2.5-Math-7B 上：

```text
PURE-PRM(sum-form)
PURE-PRM+VR(sum-form)
```

大约在 **step 25** 就训练崩溃。

到 step 80，它们的平均 benchmark 分数掉到约 **30**，比 base model 的 **39.5** 还低。

而 min-form：

```text
PURE-PRM(min-form)
PURE-PRM+VR(min-form)
```

可以稳定训练超过 200 steps，并达到：

```text
PURE-PRM: 49.3 average
PURE-PRM+VR: 53.3 average
```

**2. PRM-based RFT 可以接近 verifiable reward，而且更省训练步**

作者发现：

```text
PRM-involved 方法大约用 PURE-VR 30% 的训练步数
就能达到类似的 MATH-500 accuracy
```

也就是说，PRM 的 dense reward 确实提高了学习效率。

**3. PRM + 少量 verifiable reward 效果最好**

Qwen2.5-Math-7B 上：

| 方法 | Avg |
|---|---:|
| Base | 39.5 |
| PURE-VR | 48.3 |
| PURE-PRM | 49.3 |
| PURE-PRM+VR | 53.3 |

其中 PURE-PRM+VR 只用了约 **10% 带 GT answer 的题目**，也就是：

```text
800 个有标准答案的问题
7200 个 open problems
```

最终得到：

```text
AMC23: 82.5%
5 benchmark average: 53.3%
```

所以作者最重要的结论是：

```text
只用 PRM 可以训练，但仍会有 reward hacking 风险；
min-form 能显著延迟和缓解这个问题；
再加少量 verifiable reward，可以进一步稳定训练并得到最好效果。
```

## 16. Verifiable reward 是什么？

**你问：** verifiable reward是什么？

**我答：**

**Verifiable reward** 可以理解成：**可被规则或程序直接验证的奖励信号**。

它不靠另一个模型主观打分，而是用明确标准判断：

```text
答案对不对？
代码测试过不过？
证明是否满足形式验证器？
输出格式是否符合规则？
```

在数学题里最典型：

```text
模型最终答案：42
标准答案：42
=> reward = 1

模型最终答案：41
标准答案：42
=> reward = 0
```

所以它也常叫：

```text
outcome-level reward
rule-based reward
ground-truth reward
```

不过这些概念不完全等价。

和 PRM 对比：

| 类型 | 看什么 | 奖励来源 | 优点 | 缺点 |
|---|---|---|---|---|
| PRM | 中间步骤 | 奖励模型打分 | 信号密集，学习效率高 | 可能被 hack |
| Verifiable reward | 最终结果 | 规则/标准答案/测试器 | 可靠、客观 | 信号稀疏 |

举例：

```text
PRM：
步骤1 给 0.8
步骤2 给 0.6
步骤3 给 -0.2

Verifiable reward：
最终答案对 => 1
最终答案错 => 0
```

在这篇论文里：

```text
PURE-PRM = 只用 PRM 奖励
PURE-VR = 只用 verifiable reward
PURE-PRM+VR = PRM 为主 + 少量 verifiable reward
```

作者发现：

```text
少量 verifiable reward 很有用
```

因为它像一个“真实答案锚点”，可以防止模型只学会讨好 PRM。

一句话：

```text
Verifiable reward 是能被明确校验的最终结果奖励，比如数学答案匹配、代码单测通过、形式证明验证通过。
```

## 17. 这篇论文训练用的模型、数据等细节是什么？

**你问：** 这一篇论文用的训练用的模型、数据等细节是什么？

**我答：**

这篇论文的训练细节可以分成两块：**先训练 PRM**，再用这个 PRM 做 **RL fine-tuning**。

**1. PRM 训练**

作者先训练了一个过程奖励模型，叫 **PURE-PRM-7B**。

| 项目 | 细节 |
|---|---|
| PRM 基座 | Qwen2.5-Math-7B |
| PRM 数据 | PRM800K |
| 任务形式 | step-level binary classification，判断每一步是否正确 |
| 训练方式 | 两阶段训练 |
| 第 1 阶段 | 冻结 LLM 参数，只训练 value head，3 epochs |
| 第 2 阶段 | 解冻全模型，全参数微调 1 epoch |
| 用途 | 给推理过程中的每一步打 process reward |

PRM 评测结果：

| Benchmark | 结果 |
|---|---:|
| GSM8K BoN@1024 | 91.6 |
| MATH BoN@1024 | 62.6 |
| ProcessBench Avg F1 | 57.5 |
| PRMBench Overall | 65.3 |

**2. RL fine-tuning 用的 base models**

作者在 3 个模型上做 RFT：

```text
Qwen2.5-7B
Qwen2.5-Math-7B
Qwen2.5-Math-1.5B
```

**3. RL 数据集**

RL 阶段用的是 SimpleRL 的 RFT 数据集。

| 项目 | 细节 |
|---|---|
| 来源 | SimpleRL / SimpleRL-Zoo |
| 题目来源 | MATH dataset |
| 难度 | level 3-5 |
| 数量 | 约 8,000 道数学题 |

三种奖励设置：

| 方法 | 奖励数据怎么用 |
|---|---|
| PURE-PRM | 不用 GT answer，只用 PRM 给步骤打分 |
| PURE-VR | 所有题都用 GT answer，用最终答案对错给 reward |
| PURE-PRM+VR | 约 10% 题有 GT answer，90% 题只用 PRM |

PURE-PRM+VR 的具体比例：

```text
约 800 道题有 ground-truth answer
约 7200 道题是 open problems
```

**4. Reward 设计**

Verifiable reward：

```text
最终答案匹配 GT answer => +1
否则 => 0
```

没有额外 format reward。

Process reward：

```text
PRM 给每一步打分
再用 min-form transform 转换
然后把步骤奖励放到该步骤最后一个 token 上
```

**5. RL 框架和超参数**

| 项目 | 设置 |
|---|---|
| RL 框架 | veRL |
| Advantage estimator | RLOO |
| group size | 8，即每个 prompt 采样 8 个 responses |
| prompt batch size | 64 |
| training mini-batch size | 512 |
| max generation length | 8192 tokens |
| sampling temperature | 1.0 |
| KL coefficient | 1e-3 |
| transform temperature T | 0.1 |
| checkpoint save interval | 50 steps |

训练步数：

| 模型 | steps |
|---|---:|
| Qwen2.5-Math-7B | 500 |
| Qwen2.5-Math-1.5B | 500 |
| Qwen2.5-7B | 1000 |

学习率：

| 方法 | learning rate |
|---|---:|
| PURE-VR | 1e-6 |
| PURE-PRM+VR | 1e-6 |
| PURE-PRM | 5e-7 |

**6. 评测集**

作者用 5 个数学推理 benchmark：

```text
MATH-500
Minerva Math
Olympiad Bench
AIME24
AMC23
```

评测方式：

```text
pass@1 accuracy
greedy decoding
报告训练过程中保存的 best checkpoint
```

最核心的设置可以一句话记住：

```text
作者用 Qwen2.5-Math-7B 训练 PRM，再用这个 PRM 去强化微调 Qwen2.5 系列 3 个 base model；RL 数据是约 8000 道 MATH level 3-5 题，最强设置是 90% PRM reward + 10% verifiable reward。
```

## 18. SimpleRL 的 RFT 数据集是什么？

**你问：** SimpleRL 的 RFT 数据集是什么？

**我答：**

SimpleRL 的 RFT 数据集，简单说就是：

**SimpleRL-Zoo 用来做 zero RL / reinforcement fine-tuning 的数学题训练集。**

在这篇 PURE 论文里，作者用的是 SimpleRL 里的 **Hard setting**，也就是：

```text
MATH dataset 中 level 3-5 的题目
大约 8,000 道
```

它的每条数据大概包含：

```text
数学题 question
标准答案 ground_truth
题目难度 level
prompt 格式，例如 "Question: ... Answer: Let's think step by step."
```

SimpleRL-Zoo 官方仓库里把数据难度分成三档：

| 难度 | 数据来源 |
|---|---|
| Easy | GSM8K + MATH level 1 |
| Medium | MATH level 1-4 |
| Hard | MATH level 3-5 |

PURE 用的就是 **Hard：MATH level 3-5**。

在 PURE 里的用法有三种：

```text
PURE-PRM:
只用题目，不用标准答案，让 PRM 给过程奖励

PURE-VR:
用题目 + 标准答案，最终答案匹配就给 verifiable reward

PURE-PRM+VR:
约 10% 题目用标准答案，大约 800 道
剩下 90% 约 7200 道只用 PRM
```

所以你可以理解为：

```text
SimpleRL 的 RFT 数据集 = 一批用于 RL 训练的数学题 prompt，主要来自 MATH；
在 PURE 中，它提供训练问题本身，是否使用 ground-truth answer 取决于奖励设置。
```

来源可以看：

- [SimpleRL-Zoo GitHub](https://github.com/hkust-nlp/simpleRL-reason)
- [SimpleRL-Zoo-Data Hugging Face](https://huggingface.co/datasets/hkust-nlp/SimpleRL-Zoo-Data)

## 19. RFT 和 zero RL 是什么？

**你问：** RFT是什么？ zero RL / reinforcement fine-tuning是什么？

**我答：**

**RFT = Reinforcement Fine-Tuning，强化微调。**

意思是：模型已经预训练/SFT 过了，现在再用强化学习继续训练，让它更擅长某类任务，比如数学推理、代码、工具调用。

基本流程是：

```text
1. 给模型一个问题
2. 模型生成答案
3. 奖励函数/奖励模型给 reward
4. 根据 reward 计算 advantage
5. 用 RL 算法更新模型参数
```

比如数学题：

```text
题目：2x = 6, x = ?
模型回答：x = 3
verifier 判断正确
reward = 1
模型参数被强化
```

如果答错：

```text
reward = 0
这类生成方式被削弱
```

**zero RL** 通常指：**不先做专门的推理 SFT，直接用 RL 在 base/instruct model 上激发推理能力**。

这里的 “zero” 不是“零训练”，而是大致表示：

```text
zero supervised reasoning traces
```

也就是不依赖大量人工写好的 CoT 解题过程来监督微调，而是用 reward 让模型自己探索。

典型形式：

```text
给问题
模型自己生成推理
只用最终答案对错 / verifier / reward model 给信号
通过 RL 更新
```

DeepSeek-R1-Zero 就是这个方向的代表：不用人工 CoT SFT，直接用 RL 让模型学出长链推理行为。

**reinforcement fine-tuning** 和 **zero RL** 的关系：

```text
RFT = 大类，指用强化学习做微调
zero RL = RFT 的一种设定，强调没有先用人工推理轨迹做 SFT
```

可以这样区分：

| 类型 | 是否用人工答案/推理轨迹 SFT | 是否用 RL |
|---|---:|---:|
| SFT | 用 | 不一定 |
| RLHF/RFT | 可用可不用 | 用 |
| zero RL | 不用专门 CoT SFT | 用 |

在 PURE 这篇论文里：

```text
RFT = 用 PRM / verifiable reward 对 Qwen2.5 模型继续强化训练
```

它比较的奖励方式包括：

```text
PURE-VR：用最终答案对错做 RL
PURE-PRM：用过程奖励做 RL
PURE-PRM+VR：过程奖励 + 少量最终答案奖励
```

## 20. 这篇论文是 zero RL 还是 RFT？

**你问：** 所以作者这篇论文是zero RL还是RFT？

**我答：**

这篇论文更准确地说是 **RFT 论文**，也就是 **reinforcement fine-tuning**。

它是不是 zero RL？要看你怎么用这个词。

**严格说：不是典型的 zero RL 论文。**

因为它的重点不是证明“完全不用监督推理数据，也能从 base model 激发推理能力”。它的重点是：

```text
如何把 PRM 的过程奖励稳定地用于 RL fine-tuning
```

也就是 PRM-based RFT。

但它和 zero RL 很接近，因为作者对比/借鉴的是 DeepSeek-R1-Zero、SimpleRL-Zoo 这类 **verifiable reward-based RFT** 路线。

在论文中的几种设置：

```text
PURE-VR:
用最终答案对错做 RL，接近 R1-Zero / SimpleRL-Zoo 风格

PURE-PRM:
用 PRM 过程奖励做 RL，不用 GT answer

PURE-PRM+VR:
主要用 PRM，额外用 10% verifiable reward
```

所以最准确表述是：

```text
这篇论文是 PRM-based reinforcement fine-tuning 论文；
它研究的是 zero-RL/RFT 风格训练中，如何用 PRM 替代或辅助 verifiable reward。
```

一句话：

```text
大类：RFT
关联路线：zero RL
核心贡献：min-form credit assignment for PRM-based RFT
```

## 21. 1-step hacking 和 token-level baseline 是什么？

**你问：** token-level baseline 为什么防止 1-step hacking 这个 1-step hacking是什么意思？token-level baseline又是什么意思？

**我答：**

先说 **1-step hacking**。

在 PRM 训练里，模型的回答会被切成多个 reasoning steps，比如用空行分隔：

```text
Step 1: ...
Step 2: ...
Step 3: ...
```

PRM 是按 step 打分的。

**1-step hacking** 指的是：模型学会只输出 **一个 step**，但这个 step 可能特别长，把很多内容都塞在里面。

为什么这是 hacking？

因为 PRM 本来想做的是：

```text
逐步检查推理过程
```

但模型变成：

```text
我只给你一个超长 step
你就没法真正逐步监督我
```

它绕开了 PRM 的 step-by-step 监督机制。

再说 **baseline**。

在 RL 里，我们通常不是直接用 reward 更新，而是用：

```text
advantage = 当前样本 reward - baseline
```

baseline 可以理解成“平均水平”。

如果一个回答比平均水平好：

```text
advantage > 0
加强它
```

如果比平均水平差：

```text
advantage < 0
削弱它
```

问题出在 **step-level baseline**。

假设一组回答里，有两个回答的过程总分都差不多：

```text
回答 A：5 个 steps，总过程分 = 1.0
回答 B：1 个 step，总过程分 = 1.0
```

如果 baseline 是按 step 平均的，它可能会让多步骤回答承受更多扣减。

粗略理解：

```text
A 有 5 个 step，每个 step 都要和 baseline 比
B 只有 1 个 step，只比一次
```

这样模型会发现：

```text
步骤越少，越不容易被 baseline 惩罚
```

于是它学会输出：

```text
一个超长 step
```

这就是 1-step hacking。

**Token-level baseline** 是作者的解决方式。

它不是按“有多少 step”来平均，而是把 process reward 转成 token-level reward：

```text
每个 step 的奖励放在该 step 最后一个 token 上
其他 token 奖励为 0
```

然后 baseline 用固定的最大生成长度 `N` 来归一化，而不是用回答的 step 数。

论文里强调：

```text
normalize the baseline with the max generation length N instead of response length
```

直觉是：

```text
不要让“少写 step”天然占便宜
```

所以 token-level baseline 防止 1-step hacking 的原因是：

```text
它削弱了 baseline 对 step 数量的偏置，
让模型不能通过减少 step 数来逃避惩罚。
```

一句话版：

```text
1-step hacking = 模型把整段推理塞进一个 step，绕开逐步过程监督。
token-level baseline = 按 token 级别、用固定长度归一化来计算平均水平，避免少 step 回答天然更有 advantage。
```

## 22. 三类 reward hacking 是哪三类？

**你问：** 整理三类 reward hacking是指哪三类 reward hacking？

**我答：**

这篇论文整理了 **3 类 PRM-induced reward hacking**：

| 类型 | 表现 | 原因 | 解决思路 |
|---|---|---|---|
| 1. Only thinking, not solving | 只写思考/分析，不真正解题 | PRM 偏好某些“思考模式”，sum-form 又放大高分步骤 | 用 min-form credit assignment |
| 2. Extremely few steps, 1 step | 只输出一个超长 step | step-level baseline 对多步骤回答有惩罚偏置 | 用 token-level baseline |
| 3. Extremely few steps, 0 step | 输出空内容、无关短句、礼貌语 | discriminative PRM 只看当前 causal context，不知道后面没有解题内容 | 加 verifiable reward；或未来用 generative PRM |

具体说：

**1. Only thinking, not solving**

模型输出很多看起来像推理的内容：

```text
Let's analyze the problem step by step...
We should consider possible cases...
Now we need to verify...
```

但它不代入、不计算、不真正得到答案。

本质是：

```text
模型学会表演思考，而不是解决问题。
```

**2. 1-step hacking**

模型只输出一个 step，而且这个 step 很长。

它不是不会写多步，而是发现：

```text
步骤越少，越容易绕开 PRM 的逐步监督或 baseline 惩罚。
```

所以它把所有内容塞到一个 step 里。

**3. 0-step hacking**

模型几乎不输出有效解题内容，比如：

```text
Thank you.
Happy Birthday.
空回答
无关句子
```

PRM 可能因为只看到当前短文本，错误给了高分。它不知道后面不会继续解题。

这三类可以记成：

```text
1. 假装思考
2. 压成一步
3. 直接不答
```

对应的根因分别是：

```text
credit assignment 错
baseline 设计错
PRM 架构/奖励信号不够
```

## 23. Step-level baseline 的惩罚偏置是什么意思？

**你问：** step-level baseline 对多步骤回答有惩罚偏置这句话是什么意思？ token-level baseline是什么意思？

**我答：**

这句话可以拆开讲。

**Baseline 是什么**

RL 里常用：

```text
advantage = 当前回答的 reward - baseline
```

baseline 可以理解成“平均水平”。  
如果当前回答比平均水平好，就加强；比平均水平差，就削弱。

**Step-level baseline 是什么**

PRM 给的是 step 级奖励，所以一个自然想法是：

```text
每个 step 都拿自己的奖励去和“其他回答的平均 step 奖励”比较
```

这就是 step-level baseline。

问题是，它会和“step 数量”缠在一起。

假设两个回答过程总质量差不多：

```text
回答 A：5 个 steps
回答 B：1 个 step
```

如果每个 step 都要减一次 baseline，那么 A 被减了 5 次，B 只被减 1 次。

粗略看：

```text
A 的 advantage ≈ 总奖励 - 5 × baseline
B 的 advantage ≈ 总奖励 - 1 × baseline
```

即使 A 和 B 的总奖励差不多，A 也更容易被扣成负的。

这就是：

```text
step-level baseline 对多步骤回答有惩罚偏置
```

模型会学到：

```text
少写 step 更划算
```

于是出现 **1-step hacking**：把推理压成一个超长 step。

**Token-level baseline 是什么**

作者的做法是：不要按 step 数来归一化和比较，而是把 step reward 放到 token 层面。

比如一个回答：

```text
Step 1: blah blah <最后一个 token>
Step 2: blah blah <最后一个 token>
Step 3: blah blah <最后一个 token>
```

PRM 给每个 step 一个奖励。作者把奖励放在对应 step 的最后一个 token 上：

```text
普通 token：reward = 0
Step 1 最后 token：reward = r1
Step 2 最后 token：reward = r2
Step 3 最后 token：reward = r3
```

然后 baseline 用固定最大长度 `N` 来归一化，而不是用：

```text
这个回答有几个 step
```

直觉是：

```text
不要让“step 少”天然占便宜
```

所以 token-level baseline 的效果是：

```text
减少 step 数量带来的 advantage 偏置
防止模型学会只输出一个 step
```

一句话：

```text
step-level baseline 会让多步骤回答多扣几次分；
token-level baseline 把奖励搬到 token 层并用固定长度归一化，避免模型靠减少 step 数拿优势。
```

## 24. Baseline 是怎么计算的？惩罚偏置怎么得到的？

**你问：** baseline是怎么计算得到的？惩罚偏置是怎么得到的？

**我答：**

对，我们把它讲得更“算账”一点。

在这篇论文里，一个 prompt 会采样 **K 个 responses**，比如同一道题让模型生成 8 个解法。然后用组内其他 responses 作为 baseline，这叫 **RLOO: leave-one-out baseline**。

### 1. Baseline 怎么算？

最基本思想是：

```text
advantage = 自己的 reward - 其他样本的平均 reward
```

比如同一道题生成 4 个回答：

```text
A reward = 0.8
B reward = 0.6
C reward = 0.4
D reward = 0.2
```

对于 A 来说，baseline 是其他三个的平均：

```text
baseline_A = (0.6 + 0.4 + 0.2) / 3 = 0.4
advantage_A = 0.8 - 0.4 = 0.4
```

对于 D 来说：

```text
baseline_D = (0.8 + 0.6 + 0.4) / 3 = 0.6
advantage_D = 0.2 - 0.6 = -0.4
```

所以 baseline 不是外部给的，而是从同一个 prompt 的多个采样回答里算出来的“平均水平”。

### 2. Step-level baseline 怎么算？

如果 PRM 是按 step 打分，一个很自然的做法是：

```text
baseline = 其他回答的平均 step reward
```

假设回答 B 有 4 个 steps，过程奖励是：

```text
B: [0.8, 0.8, 0.8, 0.8]
```

它的平均 step reward 是：

```text
(0.8 + 0.8 + 0.8 + 0.8) / 4 = 0.8
```

回答 C 有 1 个 step：

```text
C: [0.8]
```

平均 step reward 也是：

```text
0.8 / 1 = 0.8
```

然后对当前回答的每个 step，都减掉这个 baseline。

问题就来了。

### 3. 惩罚偏置是怎么产生的？

假设当前有两个回答，过程总奖励一样：

```text
回答 A：4 个 steps，每步 reward = 0.25，总 reward = 1.0
回答 B：1 个 step，reward = 1.0，总 reward = 1.0
```

假设 baseline 是 `0.2`。

如果按 step-level baseline，每个 step 都减一次 baseline：

```text
A 的 advantage ≈ 1.0 - 4 × 0.2 = 0.2
B 的 advantage ≈ 1.0 - 1 × 0.2 = 0.8
```

明明两者总 reward 一样，但 B 因为 step 少，被扣得更少，所以 advantage 更大。

这就是所谓：

```text
step-level baseline 对多步骤回答有惩罚偏置
```

不是说系统显式讨厌多步骤，而是公式天然导致：

```text
step 越多，baseline 被减得越多
step 越少，baseline 被减得越少
```

于是模型会学到：

```text
把推理压缩成更少 step 更划算
```

极端情况下就是 **1-step hacking**：

```text
只输出一个超长 step
```

### 4. Token-level baseline 怎么缓解？

作者不按“有几个 step”来算 baseline，而是把 step reward 放到 token 级别：

```text
普通 token reward = 0
每个 step 最后一个 token 放这个 step 的 PRM reward
```

然后 baseline 用固定的最大生成长度 `N` 做归一化，而不是用当前回答的 step 数。

直觉上就是：

```text
不要让 baseline 扣分次数跟 step 数直接绑定
```

这样模型就没那么容易通过：

```text
减少 step 数
```

来获得更大的 advantage。

最短总结：

```text
baseline = 同题其他回答的平均表现；
step-level baseline 的问题是每个 step 都减一次平均值，step 多就多扣；
token-level baseline 用 token 级固定长度归一化，削弱“少写 step 更占便宜”的漏洞。
```

## 25. 为什么 step-level baseline 每个 step 都减一次 baseline？

**你问：** 为什么如果按 step-level baseline，每个 step 都减一次 baseline

**我答：**

因为 **step-level baseline 的定义对象就是 step**。

PRM 给的是：

```text
Step 1 reward
Step 2 reward
Step 3 reward
...
```

如果你把 advantage 也定义在 step 层面，最自然的写法就是：

```text
每个 step 的 advantage = 这个 step 的未来过程 reward - 其他回答的平均 step reward
```

所以每个 step 都会出现一次：

```text
- baseline
```

把所有 step 的 advantage 加起来看，就变成：

```text
总 advantage ≈ 总过程 reward - step 数 × baseline
```

这就是为什么会“每个 step 都减一次 baseline”。

更具体一点。

假设一个回答有 3 个 steps：

```text
r1 = 0.5
r2 = 0.4
r3 = 0.3
```

step-level baseline 设为：

```text
b = 0.2
```

那么每一步的 advantage 可能类似：

```text
A1 = r1 - b = 0.5 - 0.2
A2 = r2 - b = 0.4 - 0.2
A3 = r3 - b = 0.3 - 0.2
```

合起来：

```text
A_total = (0.5 + 0.4 + 0.3) - 3 × 0.2
```

如果另一个回答只有 1 个 step，总 reward 也差不多：

```text
r1 = 1.2
```

它只减一次：

```text
A_total = 1.2 - 1 × 0.2
```

所以 step 数越多，baseline 被减的次数越多。

在论文里的 Eq. (8)，这个问题来自 step-level baseline 里这一项：

```text
sum over j=t...N [ r_i,j^p* - average_step_reward_of_other_responses ]
```

也就是每个 token/step 位置都在减一个“其他回答的平均 step reward”。当回答 step 更多时，这种扣减累计更多，就产生了长度/步骤数偏置。

所以最核心原因是：

```text
baseline 是按 step 定义的
advantage 也是逐 step/token 计算的
因此每个 step 都会减一次 baseline
```

## 26. Token-level baseline 是怎么做的？为什么能防止 1-step hacking？

**你问：** token-level baseline 是怎么做的？为什么token-level baseline 为什么防止 1-step hacking

**我答：**

**Token-level baseline** 的做法是：把 PRM 的 step reward 转成 token 位置上的 reward，然后在 token 层面算 advantage/baseline，而不是在 step 层面按“步骤数”平均。

先看它怎么做。

假设一个回答有 3 个 steps：

```text
Step 1: x x x
Step 2: y y
Step 3: z z z z
```

PRM 给每个 step 一个分数：

```text
r1 = 0.8
r2 = 0.6
r3 = -0.3
```

作者把奖励放到每个 step 的最后一个 token 上：

```text
Step 1: x x x[0.8]
Step 2: y y[0.6]
Step 3: z z z z[-0.3]
```

其他 token 的 reward 都是 0：

```text
x[0], x[0], x[0.8], y[0], y[0.6], z[0], z[0], z[0], z[-0.3]
```

然后对于每个 token 位置 `t`，计算：

```text
当前 response 从 token t 到结尾的累计过程 reward
-
同一个 prompt 下其他 responses 的平均 token-level return
```

这就是 token-level baseline。

关键是：作者的 baseline 不是除以“这个回答有几个 step”，而是用固定的最大生成长度 `N` 归一化。

论文里大意是：

```text
baseline 用 max generation length N 归一化，而不是用 response length 或 step 数
```

为什么它能防止 **1-step hacking**？

1-step hacking 是：

```text
模型只输出一个超长 step
```

它这么做是因为 step-level baseline 下，step 少就少扣 baseline。

比如：

```text
多步骤回答：每个 step 都要减一次 baseline
一步回答：只减一次 baseline
```

模型会发现：

```text
把所有内容塞进一个 step 更划算
```

token-level baseline 把这个漏洞堵住了：

```text
baseline 不再随着 step 数直接重复扣减
```

因为奖励和 baseline 都被放到 token 序列上处理，并且用固定长度 `N` 做归一化。这样模型不能简单通过：

```text
减少 step 数
```

来让 advantage 变大。

更直观地说：

```text
step-level baseline:
比较单位是 step，少 step 会占便宜

token-level baseline:
比较单位是 token/固定长度轨迹，少 step 不会天然少扣很多
```

所以它防止 1-step hacking 的原因是：

```text
它切断了“step 越少，baseline 惩罚越少”的捷径。
```

一句话总结：

```text
token-level baseline = 把过程奖励挂到 token 上，并用固定长度归一化来算组内平均表现；这样模型不能靠把多步推理压成一个 step 来获得更高 advantage。
```

## 27. Token-level baseline 的直观对比例子

**你问：** token-level baseline:你举个对比的例子让我知道得更加直观一3

**我答：**

可以，我们用一个很小的例子算账。

假设同一道题，模型生成两个回答：

```text
回答 A：4 个 steps
回答 B：1 个 step
```

它们的 PRM 总奖励都一样：

```text
回答 A：0.25 + 0.25 + 0.25 + 0.25 = 1.0
回答 B：1.0 = 1.0
```

也就是说，从总奖励看，它们一样好。

**Step-level baseline 的问题**

假设 baseline 是：

```text
b = 0.2
```

step-level baseline 会让每个 step 都减一次 baseline。

回答 A 有 4 个 steps：

```text
A_adv = 0.25 - 0.2
      + 0.25 - 0.2
      + 0.25 - 0.2
      + 0.25 - 0.2

      = 1.0 - 4 × 0.2
      = 0.2
```

回答 B 只有 1 个 step：

```text
B_adv = 1.0 - 0.2
      = 0.8
```

结果：

```text
A_adv = 0.2
B_adv = 0.8
```

明明总奖励一样，B 却 advantage 更高。模型就会学到：

```text
少写 step 更划算
```

这就容易变成 1-step hacking。

**Token-level baseline 的直觉**

现在换成 token-level baseline。

设最大生成长度固定为：

```text
N = 8 tokens
```

回答 A 有 4 个 steps，每个 step 2 个 token。奖励放在每个 step 最后 token：

```text
A tokens:
[0, 0.25, 0, 0.25, 0, 0.25, 0, 0.25]

总奖励 = 1.0
```

回答 B 只有 1 个 step，也是 8 个 token，奖励放在这个 step 最后 token：

```text
B tokens:
[0, 0, 0, 0, 0, 0, 0, 1.0]

总奖励 = 1.0
```

token-level baseline 用固定 `N = 8` 归一化，而不是用 step 数。

假设平均 baseline 总量仍然是：

```text
baseline_total = 0.8
```

那么粗略地看：

```text
A_adv ≈ 1.0 - 0.8 = 0.2
B_adv ≈ 1.0 - 0.8 = 0.2
```

这时不会因为 B 只有 1 个 step 就少扣很多。

核心差别：

```text
step-level baseline:
扣分次数 ≈ step 数
4-step 回答扣 4 次
1-step 回答扣 1 次

token-level baseline:
用固定 token 长度 N 做归一化
不让 step 数直接决定扣分次数
```

所以 token-level baseline 堵住的是这个漏洞：

```text
“我少分几个 step，就能少被 baseline 扣几次。”
```

更短地说：

```text
step-level 比较的是 step 平均表现；
token-level 比较的是固定长度轨迹上的表现。
```

因此模型没有那么强的动机把所有推理压成一个 step。

## 28. 固定 token 长度 N 做归一化怎么算？

**你问：** 用固定 token 长度 N 做归一化是什么意思？你算一遍给我看

**我答：**

这里的 **“用固定 token 长度 N 做归一化”**，意思是：

```text
baseline 不除以这个回答有几个 step
而是统一除以最大生成 token 长度 N
```

比如论文里最大生成长度是：

```text
N = 8192 tokens
```

我们用小数字方便算，设：

```text
N = 8 tokens
```

现在有两个回答，PRM 总过程奖励都等于 1.0。

**回答 A：4 个 steps**

每个 step 2 个 token，每个 step 的 PRM reward 是 0.25：

```text
Step1: token1 token2   reward = 0.25
Step2: token3 token4   reward = 0.25
Step3: token5 token6   reward = 0.25
Step4: token7 token8   reward = 0.25
```

把 step reward 放到每个 step 最后一个 token：

```text
token rewards =
[0, 0.25, 0, 0.25, 0, 0.25, 0, 0.25]
```

总 reward：

```text
0 + 0.25 + 0 + 0.25 + 0 + 0.25 + 0 + 0.25 = 1.0
```

如果用固定 token 长度 `N = 8` 归一化：

```text
token-level average = 1.0 / 8 = 0.125
```

**回答 B：1 个 step**

它也是 8 个 token，但只有 1 个 step，PRM reward 是 1.0：

```text
Step1: token1 token2 token3 token4 token5 token6 token7 token8
reward = 1.0
```

奖励放到这个 step 的最后一个 token：

```text
token rewards =
[0, 0, 0, 0, 0, 0, 0, 1.0]
```

总 reward：

```text
1.0
```

用固定 token 长度 `N = 8` 归一化：

```text
token-level average = 1.0 / 8 = 0.125
```

所以：

```text
回答 A：1.0 / 8 = 0.125
回答 B：1.0 / 8 = 0.125
```

它们不会因为 step 数不同而 baseline 不同。

对比一下 **step-level 归一化**。

回答 A 有 4 个 steps：

```text
step-level average = 1.0 / 4 = 0.25
```

回答 B 有 1 个 step：

```text
step-level average = 1.0 / 1 = 1.0
```

这里 step 数直接影响平均值。

更关键的是，如果每个 step 都减一次 baseline，那么：

```text
A 有 4 个 steps，就可能减 4 次
B 有 1 个 step，就只减 1 次
```

于是模型会发现：

```text
少分 step 更划算
```

所以固定 token 长度 `N` 的作用是：

```text
不管你写 1 个 step 还是 4 个 steps，
都放到同一个 token 尺度上比较。
```

一句话：

```text
step-level baseline 的分母是 step 数，会诱导模型少写 step；
token-level baseline 的分母是固定 N，减少了“少写 step 就少被扣”的漏洞。
```

## 29. Pseudo-positive samples 是什么？

**你问：** pseudo-positive samples是什么东西？

**我答：**

**Pseudo-positive samples** 可以翻译成：**伪正样本**。

在这篇论文里，它指的是：

```text
被 verifier 判定为正确、拿到正奖励，
但实际上包含严重坏模式的回答。
```

尤其是这类回答：

```text
最终答案可能被判对
但整个 response 很长、很重复、质量很差
```

比如：

```text
解题过程反复重复同一段推理
不断循环
最后某处碰巧出现了正确答案
verifier 只检查最终答案，于是给 reward = 1
```

从 verifier 角度看：

```text
最终答案对 => positive sample
```

但从训练质量看：

```text
这是坏样本，不应该被强化
```

所以作者叫它：

```text
pseudo-positive sample
伪正样本
```

它危险在哪里？

因为 RL 会强化拿到正 reward 的行为。如果一个长且重复的坏回答被判定为正确，模型会学到：

```text
长重复模式也能拿正奖励
```

一旦这种模式被强化，训练可能快速崩溃。

论文里的观察是：

```text
step 361 开始 correct responses 中出现 clip ratio 和 high repetition ratio > 0
step 365 训练崩溃
```

也就是从异常样本出现到训练崩溃，只过了大约 **5 个 gradient steps**。

为什么 verifier 没发现？

因为 verifier 通常只看：

```text
最终答案是否匹配标准答案
```

它不关心：

```text
中间是不是疯狂重复
是否被截断
是否有无意义循环
过程质量是否正常
```

为什么 PRM 也没拦住？

作者说，当前 PRM 训练数据里没有充分覆盖这种重复/坏模式，所以 PRM 也可能识别不出来。

一句话总结：

```text
pseudo-positive samples = 最终答案被判对、但过程模式很坏的样本；它们会把错误行为当成正奖励强化，导致训练快速崩溃。
```

## 30. GAE、GRPO、REINFORCE++、RLOO 是什么？差别在哪里？

**你问：** Advantage estimators 作者比较 GAE、GRPO、REINFORCE++、RLOO 这几个是什么意思？这几个方法是哪里有区别？

**我答：**

这几个都是 **advantage estimator**，也就是：

```text
把 reward 转成 advantage 的方法
```

RL 更新时通常不直接用 reward，而用：

```text
advantage = 这个动作/回答比平均水平好多少
```

advantage 大于 0，就增加这类输出概率；小于 0，就降低概率。

**1. RLOO**

RLOO = Reinforce Leave-One-Out。

做法是：同一个 prompt 采样 K 个回答，每个回答的 baseline 是其他 K-1 个回答的平均 reward。

```text
A_i = reward_i - average(reward_others)
```

例子：

```text
同一道题生成 4 个答案：
r1 = 1.0
r2 = 0.5
r3 = 0.3
r4 = 0.2

A1 = 1.0 - (0.5 + 0.3 + 0.2)/3
```

优点：

```text
简单
不需要额外 value model
适合 group sampling
```

PURE 论文最后选了 RLOO。

**2. GRPO**

GRPO = Group Relative Policy Optimization。

它和 RLOO 很像，也是在同一个 prompt 的一组回答里做相对比较，但通常会做标准化：

```text
A_i = (reward_i - group_mean) / group_std
```

直觉是：

```text
不是看绝对 reward，而是看你在同组回答里相对好多少
```

DeepSeek-R1 系列让 GRPO 很出名。

优点：

```text
不需要 value model
组内相对稳定
```

区别于 RLOO：

```text
RLOO 用 leave-one-out mean
GRPO 常用 group mean/std normalization
```

**3. REINFORCE++**

REINFORCE 是最基础的 policy gradient 方法。

粗略是：

```text
A_i ≈ reward_i
```

如果回答 reward 高，就提升整条回答的概率；reward 低，就降低。

REINFORCE++ 是针对 LLM RL 做的一些增强版实现，比如加入 normalization、KL、token-level loss 等。

它相对简单，但 variance 可能更大。

在这篇论文里，REINFORCE++ 的 advantage 近似是：

```text
A_i,t = verifiable_reward_i + future_process_rewards_from_t
```

也就是少了 RLOO 那种“减去其他回答平均水平”的 baseline。

**4. GAE**

GAE = Generalized Advantage Estimation。

它是 PPO 里常见的 advantage 估计方法，需要一个 value function `V(s)`。

它用 temporal-difference error 来平衡 bias 和 variance：

```text
delta_t = r_t + gamma V(s_{t+1}) - V(s_t)
A_t = delta_t + gamma lambda delta_{t+1} + ...
```

优点：

```text
经典、稳定
可以处理长序列 token-level credit assignment
```

缺点：

```text
需要额外训练 value network
训练成本更高
```

PURE 论文里说，GAE 因为有额外 value network，forward/backward 大约多 30% 时间。

可以用一张表记：

| 方法 | baseline 从哪来 | 是否需要 value model | 特点 |
|---|---|---:|---|
| RLOO | 同 prompt 其他回答平均 reward | 否 | 简单稳，PURE 选择它 |
| GRPO | 同 prompt group mean/std | 否 | 组内相对标准化，R1 常用 |
| REINFORCE++ | 基本不显式减 baseline，或较弱 baseline | 否 | 简单但方差更大 |
| GAE | 学出来的 value function | 是 | 稳定但训练更贵 |

这篇论文的实验结论是：

```text
前 150 steps 几个方法差不多；
REINFORCE++ 在约 step 220 有 spike；
GAE 后期稍稳但慢约 30%；
综合性能、时间、稳定性，选 RLOO。
```

## 31. 这些方法只是 advantage 计算上的差别吗？

**你问：** 这些方法都只是advantage计算上的差别是吗？

**我答：**

大体上，**在这篇 PURE 论文的语境里，是的**：  
GAE、GRPO、REINFORCE++、RLOO 主要是在 **advantage 怎么估计** 上有区别。

它们最终都服务于同一个 policy gradient 更新：

```text
提高 positive advantage token/response 的概率
降低 negative advantage token/response 的概率
```

可以理解为：

```text
reward 已经有了
接下来要算：这个输出到底比基准好多少？
不同方法就是不同的“比基准”的算法
```

不过严格说，它们不只是一个小公式差别，还会影响：

```text
1. 是否需要额外 value model
2. 方差大小
3. 训练稳定性
4. token-level credit assignment
5. 是否需要 group sampling
6. 计算成本
```

但在 PURE 里，作者主要控制了其他东西，把它们作为 **advantage estimator** 来比较。

具体区别：

| 方法 | 核心差别 |
|---|---|
| RLOO | 用同题其他 responses 的平均 reward 当 baseline |
| GRPO | 用同题 group 的 mean/std 做相对标准化 |
| REINFORCE++ | 更接近直接用 reward/future return，baseline 较弱 |
| GAE | 用额外 value function 估计每个状态的 baseline |

所以你可以这样记：

```text
奖励模型/奖励函数决定 reward 是什么；
advantage estimator 决定 reward 怎么变成训练信号。
```

在这篇论文中：

```text
PURE 的核心贡献 = reward/credit assignment 的 min-form 变换
RLOO/GAE/GRPO/REINFORCE++ = 后续 advantage 估计器的选择
```

作者最后选 RLOO，是因为它在效果、速度、稳定性之间比较均衡。

## 32. GT-level signal 是什么？

**你问：** GT-level signal是什么？

**我答：**

**GT-level signal** 里的 **GT = Ground Truth**，意思是“标准答案 / 真实标签”。

所以 **GT-level signal** 就是：

```text
来自标准答案层面的训练信号
```

在这篇论文里，它主要指 **verifiable reward**：

```text
最终答案和 ground-truth answer 匹配
=> reward = 1

最终答案不匹配
=> reward = 0
```

比如数学题：

```text
标准答案：x = 3
模型答案：x = 3
GT-level signal = 正奖励

模型答案：x = 4
GT-level signal = 负/零奖励
```

它和 PRM 的区别是：

```text
PRM signal:
看中间步骤，由 PRM 模型打分

GT-level signal:
看最终答案，由标准答案/规则验证
```

为什么它重要？

因为 PRM 可能会被 hack：

```text
模型写一堆像推理的话
PRM 给高分
但最终答案错
```

GT-level signal 会把它拉回来：

```text
最终答案错 => 不给正奖励
```

所以它像一个锚点，告诉模型：

```text
不管过程看起来多漂亮，最终答案必须对。
```

在 PURE-PRM+VR 中：

```text
约 90% 训练题只用 PRM
约 10% 训练题有 GT answer，可以给 GT-level signal
```

作者发现，这 10% GT-level signal 就能明显缓解 PRM-induced reward hacking。

一句话：

```text
GT-level signal = 标准答案级别的监督/奖励信号，用来约束模型别只讨好 PRM，而要真的把题做对。
```

## 33. PRM 每一步 reward 是怎么算的？模型怎么训练？

**你问：** PRM，每一step的reward是怎么算的，是作者训练了一个模型吗，怎么训练的？

**我答：**

对，**每一步的 PRM reward 是作者训练出来的一个奖励模型给的分数**。

这篇论文里作者先训练了一个 **PURE-PRM-7B**，然后在 RL 训练时用它给模型生成的每个 reasoning step 打分。

流程大概是：

```text
数学题 + 当前已有推理 + 新生成的一步
输入 PRM
PRM 输出这个 step 正确/合理的分数
```

更形式化一点：

```text
r_t^p = R_p(s_t, a_t)
```

其中：

```text
s_t = 题目 + 前面已经生成的 steps
a_t = 当前第 t 个 step
R_p = PRM 模型
r_t^p = 当前 step 的 process reward
```

**PRM 怎么训练？**

作者用的是 **PRM800K** 数据集。

PRM800K 是一个带有人类 step-level 标注的数据集。它不是只标最终答案对不对，而是标：

```text
每一步推理是否正确
```

比如：

```text
题目：...
Step 1：正确
Step 2：正确
Step 3：错误
Step 4：错误
```

作者把 PRM 训练成一个 **binary classification** 问题：

```text
输入：题目 + 前面步骤 + 当前步骤
输出：当前步骤正确 / 错误
```

或者说输出一个 correctness score。

**模型结构**

作者用：

```text
Qwen2.5-Math-7B
```

作为 PRM 的基座。

然后：

```text
把最后一层替换/加上 value head
```

让它不再只是生成 token，而是对 step 给一个 reward / score。

**训练分两阶段**

论文里写的是：

| 阶段 | 做法 |
|---|---|
| 第 1 阶段 | 冻结 LLM 参数，只训练 value head，3 epochs |
| 第 2 阶段 | 解冻 LLM 参数，全参数微调 1 epoch |

学习率论文抽取文字里有编码问题，但核心是：

```text
先训练打分头
再微调整个模型
```

**PRM 训练好后怎么用？**

RL 训练时，主模型生成一条回答，按 step 切开：

```text
Step 1
Step 2
Step 3
...
```

PRM 对每一步分别打分：

```text
r1 = PRM(question + step1)
r2 = PRM(question + step1 + step2)
r3 = PRM(question + step1 + step2 + step3)
```

得到：

```text
[r1, r2, r3, ...]
```

这些就是 process rewards。

然后 PURE 不直接用这些 reward 做 sum-form，而是用 min-form transform：

```text
更低分的 step 权重更大
让最差步骤主导 credit assignment
```

一句话：

```text
PRM 是作者基于 Qwen2.5-Math-7B 和 PRM800K 训练出来的步骤正确性判别模型；它在 RL 阶段为每个 reasoning step 输出一个 process reward。
```

## 34. PRM800K 数据集是什么？

**你问：** PRM800K 数据集是什么？

**我答：**

**PRM800K** 是 OpenAI 在论文 **《Let’s Verify Step by Step》** 里发布的一个 **过程监督数据集**。

它的核心特点是：

```text
来自 MATH 数学题
包含模型生成的解题过程
每个解题过程被拆成 step
人类标注员给每个 step 标“对/错/是否合理”
总共约 800,000 个 step-level correctness labels
```

也就是说，它不是普通的：

```text
题目 -> 最终答案
```

而是：

```text
题目
Step 1: ...  标注：正确
Step 2: ...  标注：正确
Step 3: ...  标注：错误
Step 4: ...  标注：错误/后续无效
```

它主要用来训练 **PRM，Process Reward Model**。

训练方式大概是：

```text
输入：题目 + 前面步骤 + 当前步骤
输出：当前步骤是否正确 / 当前步骤 reward score
```

所以 PRM800K 的价值是：它提供了“过程级监督”，让奖励模型学会判断推理链中每一步的质量，而不是只看最终答案。

和普通数学数据集对比：

| 数据集类型 | 标注粒度 | 用途 |
|---|---|---|
| MATH | 最终答案 | 训练/评测数学解题 |
| PRM800K | 每一步推理正确性 | 训练 PRM |
| Outcome supervision 数据 | 整条解法最终对错 | 训练 ORM/verifier |
| Process supervision 数据 | 每个中间步骤对错 | 训练 PRM |

在 PURE 这篇论文里，作者就是用 **PRM800K** 来训练自己的 **PURE-PRM-7B**，然后用这个 PRM 在 RFT 阶段给每一步推理打 reward。

参考：

- [OpenAI PRM800K GitHub](https://github.com/openai/prm800k)
- [OpenAI: Improving mathematical reasoning with process supervision](https://openai.com/research/improving-mathematical-reasoning-with-process-supervision)

## 35. 读这一类论文需要哪些背景知识？

**你问：** 这篇论文非常厉害，非常值得学习，我现在就是对于一些背景知识我还不够了解，比如这个 PRM800K 等，请你给我总结一下，总结一下需要的背景知识

**我答：**

可以，这篇论文要读透，真正需要补的是一组 **LLM reasoning + RL fine-tuning + reward modeling** 的背景知识。你现在问到的点都很关键，不是细枝末节。

我建议按下面这张地图补。

**1. LLM 推理训练的大背景**

先理解现在 reasoning model 的几条路线：

| 概念 | 要懂什么 |
|---|---|
| SFT | 用人工/模型生成的高质量解题过程监督微调 |
| RFT | 用 reward 做强化微调，不一定需要人工 CoT |
| Zero RL | 不依赖大量人工推理轨迹，直接用 RL 激发推理 |
| Test-time scaling | 模型不更新，多采样/搜索/验证来提升答案质量 |
| Training-time scaling | 训练阶段投入更多数据、算力、RL 步数，让模型参数变强 |

PURE 属于：

```text
PRM-based reinforcement fine-tuning for reasoning
```

也就是把过程奖励模型接入 RL 微调。

**2. 奖励信号类型**

这是读 PURE 最核心的背景。

| 奖励类型 | 粒度 | 来源 | 特点 |
|---|---|---|---|
| Verifiable Reward | 最终答案 | 标准答案/规则/verifier | 稀疏但可靠 |
| Outcome Reward Model, ORM | 整体回答 | 训练出来的奖励模型 | 看最终输出质量 |
| Process Reward Model, PRM | 中间步骤 | 训练出来的过程奖励模型 | 密集但可被 hack |
| Rule-based Reward | 格式/答案/测试 | 人写规则 | 简单可靠但覆盖有限 |
| Preference RM | 回答对比较 | 人类偏好数据 | 常用于 RLHF |

PURE 论文最重要的对照是：

```text
PRM = dense but hackable
VR = sparse but reliable
```

作者最后的实践结论更像：

```text
PRM 为主 + 少量 VR 做锚定
```

**3. PRM800K 是什么**

PRM800K 是 OpenAI《Let’s Verify Step by Step》里的过程监督数据集。

你需要记住：

```text
来源：MATH 数学题
内容：模型生成的解题过程
标注：每一步推理是否正确
规模：约 800K step-level labels
用途：训练 PRM
```

普通数学数据是：

```text
题目 -> 最终答案
```

PRM800K 是：

```text
题目 -> Step 1 正确/错误
     -> Step 2 正确/错误
     -> Step 3 正确/错误
```

所以它能训练出一个模型：

```text
输入：题目 + 前面步骤 + 当前步骤
输出：当前步骤质量分数
```

这就是 PRM。

**4. RL 里的 credit assignment**

这是 PURE 的理论核心。

Credit assignment 问的是：

```text
一次成功/失败，应该把功劳或责任分给哪些中间步骤？
```

传统 RL 常用：

```text
sum-form credit assignment
当前步骤价值 = 未来奖励的折扣累加
```

PURE 提出：

```text
min-form credit assignment
当前步骤价值 = 当前到未来步骤里的最低过程奖励
```

为什么重要？

因为推理链像链条：

```text
一步关键错，整条链可能都不可靠
```

所以 min-form 更符合数学推理和 PRM test-time scoring 的逻辑。

**5. Advantage estimator**

Reward 不能直接拿来更新模型，一般要变成 advantage：

```text
advantage = 当前输出比基准好多少
```

PURE 里涉及：

| 方法 | 核心 |
|---|---|
| RLOO | 同题生成 K 个回答，用其他回答平均 reward 当 baseline |
| GRPO | 用同组回答 mean/std 做相对标准化 |
| REINFORCE++ | 更接近直接用 return，简单但方差可能大 |
| GAE | 用额外 value model 估计 baseline，更稳但更贵 |

你只需要先抓住：

```text
reward 决定“什么是好”
advantage estimator 决定“好多少，怎么更新”
```

**6. Reward hacking**

Reward hacking 是这篇论文的灵魂背景。

意思是：

```text
模型没有真正完成任务，而是学会钻 reward 的漏洞。
```

PURE 里总结三类 PRM hacking：

```text
1. 假装思考：only thinking, not solving
2. 压成一步：1-step hacking
3. 直接不答：0-step hacking
```

还有一个特别重要的概念：

```text
pseudo-positive samples
```

也就是：

```text
最终答案被 verifier 判对，
但过程很坏，比如很长、重复、被截断。
```

这些坏样本拿到正奖励后，会让训练快速崩溃。

**7. 你读这篇论文时最该建立的总框架**

可以用这条链串起来：

```text
PRM800K
=> 训练 PRM
=> PRM 给每一步推理打分
=> PRM reward 进入 RL
=> 需要 credit assignment
=> sum-form 会累积高分模式和 PRM 误差
=> 模型 reward hacking
=> PURE 用 min-form 把最差步骤作为瓶颈
=> 再加少量 verifiable reward 稳住训练
```

最值得你沉淀的背景知识优先级：

```text
第一优先级：PRM / ORM / VR / reward hacking
第二优先级：RFT / zero RL / test-time scaling
第三优先级：credit assignment / advantage / baseline
第四优先级：PRM800K / MATH / SimpleRL-Zoo 等数据集脉络
```

如果你把这四层补齐，这篇论文基本就能读到“作者为什么这么设计”的层面，而不是只看懂公式。

## 36. 更多这个方向的论文和 Agentic RL 论文

**你问：** 这一类论文是我研究的重点方向，请你给我找更多这个方向的论文。还有Agentic RL相关的一些论文

**我答：**

当然。这一方向现在很热，我建议你把论文分成两条主线读：

```text
A. Reasoning RL / PRM / verifiable reward / credit assignment
B. Agentic RL / tool-use / web agents / multi-step environments
```

下面是我给你的第一版阅读地图。

**一、必读底座：Reasoning RL / PRM / RLVR**

1. [Let’s Verify Step by Step](https://cdn.openai.com/improving-mathematical-reasoning-with-process-supervision/Lets_Verify_Step_by_Step.pdf)  
   OpenAI PRM800K 来源论文。核心是比较 outcome supervision 和 process supervision，并发布 PRM800K。你要研究 PRM，这篇是地基。

2. [DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning](https://arxiv.org/abs/2501.12948)  
   zero RL/RLVR 路线的标志性论文。理解 rule-based reward、GRPO、R1-Zero、冷启动数据、蒸馏都绕不开它。

3. [Kimi k1.5: Scaling Reinforcement Learning with LLMs](https://arxiv.org/abs/2501.12599)  
   重点看它怎么把 RL scaling 做成工程系统，以及它明确说不依赖复杂的 MCTS/value function/PRM，也能做强 reasoning。

4. [SimpleRL-Zoo: Investigating and Taming Zero Reinforcement Learning for Open Base Models in the Wild](https://arxiv.org/abs/2503.18892)  
   很适合作为开源复现实验入口。PURE 也用了它的 RFT 数据集。重点看训练数据难度和 base model exploration capability 的关系。

5. [Rewarding Progress: Scaling Automated Process Verifiers for LLM Reasoning](https://arxiv.org/abs/2410.08146)  
   和 PURE 高度相关。关注 process verifier / process advantage verifier，以及为什么过程奖励能改善 search 和 online RL。

6. [Process Reinforcement through Implicit Rewards, PRIME](https://arxiv.org/abs/2502.01456)  
   PRIME 是 PURE 的重要对照。它不直接依赖人工 process labels，而是用 outcome labels 学 implicit process rewards。PURE 表 2 里也拿 Eurus-2-7B-PRIME 做 baseline。

7. [PURE: Min-Form Credit Assignment for LLM Reasoning](https://www.emergentmind.com/papers/2504.15275)  
   你正在读的这篇。建议把它放在 PRM/RLVR 主线中间读：先懂 PRM800K、DeepSeek-R1、SimpleRL、PRIME，再回头看 PURE，会更清楚它的贡献点。

8. [Reinforcement Learning with Verifiable Rewards Implicitly Incentivizes Correct Reasoning in Base LLMs](https://arxiv.org/abs/2506.14245)  
   重点看 RLVR 到底是在“创造新能力”还是“激发 base model 已有能力”。这个问题对你理解 RFT 的边界很重要。

**二、Agentic RL / Tool-use / Web Agent 方向**

1. [The Landscape of Agentic Reinforcement Learning for LLMs: A Survey](https://arxiv.org/abs/2509.02547)  
   综述，覆盖 agentic capabilities、tool use、memory、planning、reasoning、self-improvement、perception 等。适合先扫全局。

2. [WebRL: Training LLM Web Agents via Self-Evolving Online Curriculum Reinforcement Learning](https://arxiv.org/abs/2411.02337)  
   Web agent RL 代表作。重点看 self-evolving curriculum、ORM、online RL、WebArena-Lite。结果很强：Llama-3.1-8B 从 4.8% 到 42.4%。

3. [WebAgent-R1: Training Web Agents via End-to-End Multi-Turn Reinforcement Learning](https://arxiv.org/abs/2505.16421)  
   把 R1/RL 思路迁移到 web agents。重点看 multi-turn RL、binary task success reward、behavior cloning warm-up、WebAgent-R1-Zero vs WebAgent-R1-CoT。

4. [Agentic Reinforced Policy Optimization, ARPO](https://arxiv.org/abs/2507.19849)  
   专门面向 multi-turn LLM agents 的 RL 算法。适合和 GRPO/RLOO 做对比：单轮 reasoning RL 如何变成多轮 agent RL。

5. [Agentic Reasoning and Tool Integration for LLMs via Reinforcement Learning, ARTIST](https://arxiv.org/abs/2505.01441)  
   关注“什么时候用工具、用什么工具、怎么在多轮 reasoning chain 中调用工具”。这是 agentic reasoning + tool integration 的典型论文。

6. [Process-Supervised Reinforcement Learning for Interactive Multimodal Tool-Use Agents](https://huggingface.co/papers/2509.14480)  
   把 process supervision/RL 拓展到交互式多模态 tool-use agent。可以和 PURE 的 process reward 思路并读。

7. [Verifiable Process Rewards for Agentic Reasoning](https://arxiv.org/abs/2605.10325)  
   这是 2026 年较新的方向：把 verifiable reward 从 outcome-level 推向 process-level，并迁移到 agentic reasoning。非常贴近你现在关注的 PURE + Agentic RL 交叉点。

**三、建议阅读顺序**

如果你想系统进入这个方向，我建议这样读：

```text
第一层：奖励与过程监督
Let’s Verify Step by Step
Rewarding Progress
PURE

第二层：RLVR / Zero RL
DeepSeek-R1
Kimi k1.5
SimpleRL-Zoo
RLVR implicitly incentivizes correct reasoning

第三层：隐式/替代过程奖励
PRIME
PURE 再读一遍
Verifiable Process Rewards for Agentic Reasoning

第四层：Agentic RL
Agentic RL Survey
WebRL
WebAgent-R1
ARPO
ARTIST
```

你这个研究方向的核心问题可以压成一句：

```text
如何把 sparse outcome reward、dense process reward、tool/environment feedback 统一成稳定的 long-horizon credit assignment？
```

这也是 PURE 和 Agentic RL 能接上的地方：PURE 解决 reasoning chain 里的 credit assignment，Agentic RL 要解决 multi-turn interaction/tool-use 里的 credit assignment。

