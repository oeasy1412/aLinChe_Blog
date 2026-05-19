---
title: PPO
published: 2026-05-12
description: PPO
tags: [RL, LLM, NLP]
category: LLM
draft: false
---

在大模型的标准训练范式中，我们通常遵循三步走：**预训练(Pre-training) -> 指令微调(SFT) -> 强化学习(RL)**。很多同学会问：既然SFT(监督微调)已经能让模型像人一样说话了，为什么还要费时费力去做强化学习？

让我们首先从近期大放异彩的 **DeepSeek**(尤其是 DeepSeekMath 和 DeepSeek-R1 的论文)中寻找答案。

## 一、 为什么LLM需要强化学习？
在很长一段时间里，工业界有一种错觉：只要SFT的数据足够多、足够好，模型就能无限变强。然而，DeepSeek的系列论文(尤其是震撼业界的DeepSeek-R1)彻底打破了这个迷思。
### 1. SFT的极限：模仿(Mimicry) vs. 精通(Mastery)
SFT 的本质是**行为克隆(Behavioral Cloning)**。你给模型输入一段高质量的人类解答，模型通过交叉熵损失(Cross-Entropy)来学习“预测下一个Token”。
*   **SFT的缺点**：它是通过“拟合人类输出”来学习的。人类的解题路径可能包含跳跃性思维，或者本身并不是最优解。SFT会让模型学会“人类的说话语气”，但并没有真正教给模型“如何通过探索找到正确的真理”。
### 2. RL的本质：目标优化与涌现能力(Emergence)
在 DeepSeek-R1 的技术报告中，研究人员做了一个极为大胆的尝试：**跨过大规模SFT，直接在基础模型上应用强化学习**。结果他们发现了令人震惊的 **“Aha Moment(顿悟时刻)”**。
*   **自我验证与反思**：通过RL(给定正确答案的Reward)，模型在没有任何人类思维链(CoT)数据示范的情况下，**自动学会了**在输出中写下“等等，让我重新想想”、“这个思路不对，我换个方法”。
*   **广阔的探索空间**：RL 允许模型在巨大的Token空间中进行**试错(Exploration)**。模型不再被局限于人类的解题模版中，而是通过奖励信号(Reward)的牵引，自行探索出最长的、逻辑最严密的推理路径。

**总结一句话**：SFT 是教模型“看起来像个专家”，而 RL 是通过制定规则和奖惩，逼迫模型在实战中真正“成为专家”。

---

## 二、 PPO的底层逻辑：为什么要“近端(Proximal)”？
既然决定了用强化学习，我们就要选择算法。标准的强化学习算法(如 REINFORCE 策略梯度)在 LLM 场景下会遇到极其严重的问题。

### 1. 传统策略梯度(Policy Gradient)的痛点
假设模型生成了一句话，得到了高分奖励。传统的做法是：增大这句话中所有Token生成的概率。
*   **痛点在于**：神经网络的更新是非线性的。如果步子迈得太大，权重更新过度，模型就会发生**灾难性遗忘(Catastrophic Forgetting)**。它可能会变成一个只会疯狂输出能够“骗取”奖励的乱码(`Reward Hacking`)，而失去了原本预训练赋予它的流畅语言能力。

### 2. PPO 的核心直觉：稳字当头(Trust Region)
PPO 的全称是“近端策略优化”。它的底层逻辑极其简单：**在更新模型时，新模型(New Policy)和旧模型(Old Policy)的表现不能差得太远。**
这就好比我作为一个教授指导你写论文：如果你的某个新思路(New Policy)得到了好评(高Reward)，我鼓励你顺着这个思路多写点，但**不能让你完全抛弃你原本扎实的学术写作规范(Old Policy)**。

### 3. LLM中的 PPO 的架构
在LLM的PPO训练(如RLHF)中，我们要同时在显存里塞下4个模型：
1.  **`Actor` Model(演员模型 $\pi_\theta$)**：**学生**，这是我们要训练的主模型，负责生成文本。其梯度更新目标为减小Loss。
2.  **`Reward` Model(奖励模型 $R_\phi$)**：**打分机器**，给Actor学生生成的完整回复打分。（冻结不训练）
3.  **`Critic` Model(价值模型 $V_\omega$)**：**私人教练**，Critic负责在模型生成每一个Token时，**预测**“当前这条路走下去，学生最终能拿多少分”，即**价值估计(Value Estimation)**。其梯度更新目标为更好地**预测**当前学生的reward表现。
4.  **`Reference` Model(参考模型 $\pi_{ref}$)**：**约束**，冻结了权重的原始模型。它就像一把尺子，Actor只要偏离原始模型太远，就会受到惩罚(KL散度惩罚)。（冻结不训练）

---

## 三、 PPO 的数学逻辑推演
接下来，我们进入数学部分。PPO是如何用数学语言实现上述“稳字当头”的底层逻辑的？

### 1. 优势函数(Advantage Function)
在 RL 中，我们不光要看绝对分数(Reward)，还要看**相对分数(Advantage)**。
$$ A_t = Q(s_t, a_t) - V(s_t) $$
*   $Q(s_t, a_t)$ 是在状态 $s_t$ 下采取行动 $a_t$(生成某个Token)后的实际总回报。
*   $V(s_t)$ 是 Critic 模型预测的期望回报(Baseline)。
*   **数学直觉**：如果 $A_t > 0$，说明当前Token比预期好，应该鼓励；如果 $A_t < 0$，说明不如预期，应该抑制。使用优势函数极大地**降低了方差**，使训练更稳定。

### 2. 重要性采样(Importance Sampling)
PPO 是一种**同轨(On-policy)** 变 **离轨(Off-policy)**的算法。我们用旧策略 $\pi_{old}$ 收集了一大批数据，要想用这些数据去更新新策略 $\pi_\theta$，就需要计算两者**概率**的比例：
$$ r_t(\theta) = \frac{\pi_\theta(a_t | s_t)}{\pi_{old}(a_t | s_t)} $$
*   PPO 的 Loss 计算与 $r_t(\theta) · \hat{A}_t$ 相关，即 $重要性\times优势$ 。
*   **数学直觉**：如果 $r_t(\theta) > 1$，说明新模型比旧模型更倾向于生成这个Token，如果是优势应该放大，如果是不好的更新，应该抑制，此时模型更新力度较大。如果 $r_t(\theta)$ 非常非常的小，说明新模型其实很难生成这次的Token，模型更新力度较小。

### 3. PPO的损失函数：截断代理目标函数(Clipped Surrogate Objective)
PPO 最伟大的数学贡献，就是这个带截断(Clip)的损失函数(注：这里写的是最大化目标，等价于损失函数的相反数)：
$$ L^{CLIP}(\theta) = \hat{\mathbb{E}}_t \left[ \min \left( r_t(\theta)\hat{A}_t, \, \text{clip}(r_t(\theta), 1-\epsilon, 1+\epsilon)\hat{A}_t \right) \right] $$

假设 $\epsilon = 0.2$：$r_t(\theta)$ 的会被 clip函数 限制在 $[0.8, 1.2]$。
通过这个极简的 `min` 和 `clip` 函数，PPO 巧妙地限制了策略更新的步长，减小了其“**训崩**”风险，实现了 Trust Region(信任域)。

### 4. LLM 场景下的 KL 惩罚奖励(KL Penalty)
除了 PPO 算法本身的限制，在从 Prompt x 生成完整文本 y 时，我们还在 Reward 层面加上了数学限制。最终用于强化学习的奖励 $R$ 并不是 Reward Model 直接给的分数，而是：

$$ \hat{R}_{total} = \underbrace{Reward_{model}(x, y)}_{\text{外部评分}} - \beta \underbrace{\mathbb{D}_{KL}(\pi_\theta || \pi_{ref})}_{\text{内部约束}} = Reward_{model}(x, y) -  \beta \sum_{t=1}^{T} \left[ \log \frac{\pi_\theta(y_t | x, y_{<t})}{\pi_{ref}(y_t | x, y_{<t})} \right]$$

后半部分就是 KL 散度惩罚。
*   **数学直觉**：如果 $\pi_\theta$(当前模型)生成的词汇分布偏离 $\pi_{ref}$(预训练模型)太远，哪怕 $R_{model}$ 给出了极高的分数，总奖励 $r$ 也会被扣成负数。这防止了模型退化成不说人话的刷分机器。所以 PPO 的 Reward 可以视为 `Reward - Critic - KL(Reference)` 的综合结果。

---

## 四、 扩展思考：从 PPO 到 DeepSeek-R1的GRPO 和 DPO 

讲完了 PPO，我们最后再呼应一下开头提到的 DeepSeek。
PPO 虽然稳定，但正如前文所述，它需要同时加载 4 个模型(Actor, Reward, Critic, Reference)，特别是Actor和Critic均需要训练，这对 GPU 显存是巨大的灾难。于是，业界开始思考：能不能**裁掉**一些模型，但保留强化学习的效果？

### 1. GRPO：裁掉“私人教练(Critic)” 并 缩小“打分机器(Reward)”
这是 DeepSeek 贡献给开源社区的神来之笔。在 DeepSeek-R1 的训练中，`GRPO`(Group Relative Policy Optimization) 大放异彩。
*   **它是怎么“裁员”Critic的？**
    PPO 需要 Critic 模型是为了预估一个“基准分(Baseline)”。GRPO 说：**我不需要专门的模型来预估基准，我让学生们“内卷”就行了。**
    *   对于同一个 Prompt，Actor 模型生成一组（比如 64 个）不同的回答。GRPO 计算这组回答奖励得分的**平均值**。
*   **为什么可行？**
$$ \hat{A}_{i} = \frac{r_i - \text{mean}(r)}{\text{std}(r)} $$
    通过这组回答的平均分，我们就能知道哪些回答是“超常发挥”（高于平均点），哪些是“失误”（低于平均点）。
    *   **优势**：省去了巨大的 Critic 模型空间（显存直接省掉 1/4 到 1/3）。
    *   **效果**：这种“群体相对评价”在**数学和逻辑推理**（如 DeepSeek-R1）中效果极好，因为这些领域存在有标准答案，奖励信号非常明确。
*   基于规则的奖励函数 (`Rule-based Reward Function`)：
    *   **准确性奖励(Accuracy Reward)**：比如数学题，直接用正则表达式提取模型输出的答案，看它等不等于正确答案（如 1+1=2）。对了给 1 分，错了给 0 分。代码编译器和数学公式不会被模型的“花言巧语”欺骗。
    *   **格式奖励(Format Reward)**：强制要求模型把思考过程写在 `<thought>` 标签里，结论写在 `<answer>` 标签里。不符合格式就扣分。
    *   **`顿悟`时刻(`Aha Moment`)的来源**：正是因为奖励信号极其**客观且纯粹**，模型在无数次试错中发现：只有不断反思、不断自我纠错(CoT)，才能拿到那个实打实的“1分”。
        *   这其实是一件很有意思的事情：PPO 时期，人们追求在一定程度上用 Reference 模型来约束 Actor 的输出，担心模型会退化成不说人话的长输出“**兜圈子**”的**Reward Hacking**刷分机器；而 GRPO 缺利用好了模型的长输出，在RL下结果反而让模型学会了“反思”和“自我纠错”的思考方式。
        > 例如：LLM: “20+50/2等于多少？20+50=70,70/2=35。等等，让我思考一下。哦，我做错了，除法是优先级更高的计算，我应该先计算50/2=25,20+25=45。”
        > GRPO 让模型在同一次采样中看到多个路径。如果路径 A 靠“猜”拿了 0 分，路径 B 靠“反思”拿了 1 分，GRPO 的**组内相对评分机制**会产生极强的信号，推着模型往路径 B 进化。

### 2. DPO：裁掉“打分机器(Reward)”
斯坦福提出的 **`DPO`(Direct Preference Optimization)** 不仅裁掉了Critic，还直接裁掉了Reward模型。
*   **核心数学直觉：**
    DPO 的损失函数计算的是：`新旧模型在好回答上的概率比值` vs `新旧模型在坏回答上的概率比值`。
    $$ L_{DPO} = -\mathbb{E}_{(x, y_w, y_l)} [\log \sigma (\beta \log \frac{\pi_\theta(y_w|x)}{\pi_{ref}(y_w|x)} - \beta \log \frac{\pi_\theta(y_l|x)}{\pi_{ref}(y_l|x)})] $$
    *   **思路**：即使新模型在好回答的概率没有提升甚至下降了，但只要它在坏回答上的概率下降得更多，整体的损失也Loss会降低，模型会被优化为更安全、更符合人类偏好的。（**适用场景**：**对话风格微调**、**安全**、**对齐**）
    *   **优势**：**快且稳**。它不需要采样（Sampling），不需要加载 Reward 和 Critic 模型，显存压力极小，训练起来就像 SFT 一样简单。
    *   **局限性**：DPO 属于“**离轨(Offline)**”学习。它只能在人类已经标注好的数据里打转，**无法像 GRPO/PPO 那样通过自我探索（Exploration）发现超出人类水平的新解法**。
