# Learning to Optimize Tensor Programs

文章发在 2018 年 NIPS， 后续工作 有 Ansor 等

## 0. Abstract
我们引入了一个基于学习的框架来优化张量程序。 张量算子的高效实现，是深度学习系统的关键， 当前的系统依赖于手动优化的库，例如 cuDNN，但它仅支持小范围的服务器级 GPU。这种依赖限制了高级图优化的适用性，并且在部署到新的硬件目标时会产生大量的工程成本。 我们使用 learning 来消除这种工程负担。 我们学习特定领域的统计成本模型(domain-specific statistical cost models)，以指导在数十亿种可能的程序变体中搜索张量算子实现。 我们使用跨工作负载的有效模型迁移进一步加速搜索。 实验结果表明，我们的框架提供的性能可与用于低功耗 CPU、移动 GPU 和服务器级 GPU 的最先进的手动优化库相媲美。

## 1. Intro
为了优化张量算子，程序员需要考虑由于线程、内存重用、流水线和其他硬件因素做实现。 因此，支持不同的硬件后端需要付出巨大的工程努力。 即使在当前支持的硬件上，开发 DL 框架和模型也会受到库中优化算子集的限制，从而阻止算子融合等优化。

这项研究探讨了以下问题：我们能否利用学习来减轻这种工程负担并自动优化给定硬件平台的张量算子程序？ 我们肯定的答案基于我们的统计成本模型，该模型使用给定的低级程序预测程序运行时间。 这些成本模型指导我们探索可能的程序空间，使用可迁移的表示来概括不同的工作负载以加速搜索。 我们做出以下贡献：

- 形式化描述了 **使用 learning 优化张量程序** 这个problem ， 并且总结了其显著特点
- 提出了一个ML 框架来解决这个问题
- 使用迁移学习加速优化2-10倍

我们对该框架中的组件设计选择进行了详细的实证分析。 实验结果表明，我们的框架比现有框架产生了 1.2 倍到 3.8 倍的端到端性能改进。

## 7. Conclusion
文章介绍了 AutoTVM：一种基于机器学习的框架，可自动优化 DL 系统中张量算子的实现。 我们的统计成本模型(statistical cost model)允许在工作负载之间进行有效的模型共享，并通过模型迁移加速优化过程。 除了我们的解决方案框架之外，这个新问题的具体特征使其成为相关领域创新的理想试验台，例如神经程序建模、贝叶斯优化、迁移学习和强化学习。 在系统方面， learning to optimize tensor programs 可以在不同的硬件后端实现更多融合的算子、数据布局和数据类型——这对于改进 DL 系统至关重要。 我们的框架可以在 https 找到： [https://tvm.apache.org/](https://tvm.apache.org/)

## 2. Problem Formalization
以 图1 为例。我们使用索引表达式(index expressions)表示张量算子（如，$C_{ij} = \sum_k A_{ki}B_{kj}$）。 用 $\varepsilon$ 表示其空间。 索引表达式不指定实现细节，例如循环顺序、内存范围和线程。 因此，我们可以生成逻辑上等同于给定 $e \in \varepsilon$ 的表达式的低级代码的多个变体。我们使用 $S_e$ 来表示从 e 到低级代码的可能的调度空间。 对于 $s \in S_e$，设 $x = g(e, s)$ 为生成的低级代码。 这里，g 代表一个编译器框架，它从 e、s 生成低级代码。 我**们感兴趣的是最小化 $f(x)$，这是硬件上的实际运行时间成本**。 重要的是，我们不知道 f(x) 的解析表达式，但可以通过在硬件上运行实验来查询它。 对于给定的 (g, e, Se, f) 元组，我们的问题可以形式化为以下目标：

$$
\texttt{arg min} f(g(e,s))
$$

这很类似传统的超参数搜索问题，但是有以下几个显著不同：

- **相对较低的实验成本**。 传统上超参数优化问题查询 f 成本很高，即运行实验可能需要数天。 <u>然而，编译和运行张量程序的成本是几秒钟</u>。 此属性要求模型训练和推理速度快， 否则与在真实硬件上分析执行相比没有任何好处。 这也意味着我们可以在优化过程中收集更多的训练数据。

- **特定领域的问题结构**。 大多数现有的超参数优化算法将问题视为黑盒。 当我们优化程序时，我们可以利用它们丰富的结构来构建有效的模型。

- **大量类似的算子**。 端到端 DL 系统必须针对不同的输入大小、形状和数据布局配置优化张量算子程序。 这些任务相似，可以为迁移学习提供机会。

我们描述了与手动优化代码竞争的自动代码生成的两个关键先决条件。:

1. 我们需要定义一个详尽的搜索空间 $S_e$，涵盖手动调整库中的所有硬件感知优化。
2. 我们需要有效地找到 $S_e$ 中的最优调度。

有许多用于代码生成的领域特定语言 (DSL) [32、36、15、37、20、30]，每种语言都有不同的 $E$、$S_e$ 和 $g$。 **多面体模型 [5, 42, 41] 是 $S_e$ 的流行选择**； 他们将循环域建模为整数线性约束。 一种源自 Halide [32] 的替代方法使用一组转换原语定义了一个调度空间。 改善$S_e$是一个重要的研究方向，超出了本文的范围； 我们选择了丰富的 $S_e$，并在本文的其余部分专注于调度优化。

我们使用来自现有代码生成框架 [9] 的原语来形成 $S_e$。 我们的搜索空间包括每个循环轴上的多层平铺、循环排序、GPU 的共享内存缓存以及展开和矢量化等注释。 搜索空间大小

## 3. Learning to Optimize Tensor Programs
我们提出了一个基于 ML 的框架来解决这个问题。 图 2 展示了框架及其模块。 我们建立了一个统计成本模型 statisticalcost models $\hat{f}(x)$ 来估计每个低级程序 x 的成本。 Exploration Module 提出新的调度以在硬件上运行。 运行时统计信息收集在数据库 D = {(ei , si , ci)} 中，该数据库又可用于更新 $\hat{f}$。 我们在以下小节中讨论特定于 module 的设计选择。

### 3.1. Statistical Cost Model
我们支持的第一个统计模型是基于梯度提升树 [11] (GBTs)。 我们从给定的低级 AST $x$ 中提取领域特定特征。 包括循环结构信息（如内存访问计数和数据重用率）和通用注释（例如，矢量化、展开、线程绑定）。 我们使用 XGBoost [7]，它在过去的问题中被证明是一个强大的基于特征的模型。 我们的第二个模型是 TreeGRU[39]，它递归地将低级 AST 编码为嵌入向量。 我们使用线性层将嵌入向量映射到最终预测成本。

GBT 和 TreeGRU 代表了两种不同的 ML 问题解决方法。 GBT 依赖于精确的特征提取，使用 CPU 进行快速预测。 TreeGRU 是一种基于 DL 的方法，具有可扩展性，不需要特征工程，但在训练和预测速度方面滞后。 我们将批处理应用于 TreeGRU 模型并使用 GPU 加速使训练和预测速度足够快。

### 3.2. Training Objective Function
我们可以从多个目标函数中进行选择，为给定的数据集合 $\mathcal{D} = {(e_i , s_i , c_i)}$ 训练统计成本模型。 一个常见的选择是回归损失函数 $\sum_i (\hat{f}(x_i)-c_i)^2$ ，它鼓励模型准确预测成本。 另一方面，由于我们在选择过程中只关心程序运行时间的相对顺序而不是它们的绝对值，因此我们可以改用以下秩损失函数[6]：

$$
\sum_{i,j}\log(1+e^{-sign(c_i-c_j)(\hat{f}(x_i)-\hat{f}(x_j))})
$$

我们可以使用预测函数 $\hat{f}(x)$ 来选择最好的实现。

### 3.3. Exploration Module
!!! warning "疑问"
    没看懂？不懂的概念比较多


探索模块控制搜索循环，算法 1 对其进行了总结。在每次迭代中，它必须根据 $\hat{f}(x)$ 和在真实硬件上查询 $f(x)$ 选择一批候选程序。 由于搜索空间的大小，我们不能简单地枚举 $S_e$ 的整个空间并选择 top-b 候选者。 相反，我们使用模拟退火 [19] 和 f(x) 作为 energy function。

具体来说，我们使用一批并行马尔可夫链来提高统计成本模型的预测吞吐量。 我们选择表现最好的候选批次在真实硬件上运行。 收集的性能数据用于更新 $\hat{f}$。 我们使马尔可夫链的状态在 $\hat{f}$ 更新中保持不变。 我们还应用 $\epsilon$-greedy 随机选择 $\epsilon$b（例如 0.05）个候选者以确保探索。

**Diversity-Aware Exploration**。 在为硬件评估选择 b 个候选者时，我们会同时考虑质量和多样性。 假设调度配置 $s$ 可以分解为 m 个分量 $s=[s_1,s_2,\cdots s_m]$。 我们最大化以下目标以从顶部 $\lambda b$ 候选者中选择候选集 $S$：

$$
L(S) = -\sum_{s\in S}\hat{f}(g(e,s)) + \alpha \sum_{j=1}^m |\cup_{s\in S}\{s_j\}|
$$

第一个任期鼓励我们选择运行时间成本低的候选人。 第二项计算 S 所覆盖的不同配置组件的数量。L(S) 是一个子模函数，我们可以应用贪心算法 [29, 22] 来获得近似解。 

**不确定性估计器**。 当 $\hat{f}$ 的不确定性估计可用时，贝叶斯优化方法 [34、33、35、17] 使用获取函数而不是均值。 典型的选择包括预期改进 (EI) 和置信上限 (UCB)。 我们可以使用引导程序来获得模型的不确定性估计并验证这些方法的有效性。 正如我们将在第 6 节中看到的，考虑不确定性并不会改进我们问题的搜索。 然而，采集功能的选择仍然是一个值得进一步探索的候选者。

## 4. Accelerating Optimization via Transfer Learning
到目前为止，我们只关注学习优化单个张量算子的工作负载。 在实践中，我们需要针对具有不同输入形状和数据类型的许多张量算子进行优化。 在现实世界的设置中，系统从以前看到的工作负载中收集历史数据 D0。 我们可以应用迁移学习来有效地使用 D0 来加速优化。

迁移学习的关键是创建一个对源域和目标域不变的可迁移表示。 然后，我们可以使用跨域的通用表示来共享成本模型。 不同的表示选择可能具有不同级别的不变性。

贝叶斯优化方法中的一种常见做法是直接使用配置作为模型的输入。 但是，搜索空间规范可能会因不同的工作负载而改变，或者当用户为相同的工作负载指定新的搜索空间时。 配置表示 s 对于搜索空间的变化不是不变的。

另一方面，低级循环 AST $x$（图 3a）是对搜索空间不变的程序的共享表示。 为了利用这种不变性，我们的成本模型 $f(x)$ 将低级循环 AST $x$ 作为输入。 我们还需要将 $x$ 编码到向量空间中以进行预测。 $x$ 的特定编码也可以导致不同级别的不变性

**GBT 的上下文关系特征**。 我们在每个循环级别定义上下文特征来表示循环特征。 上下文特征的简单表示是一个向量（例如，在图 3b 中，每个循环都有一行特征）。 上下文特征可以提供信息，但至关重要的是，不能概括不同的循环嵌套模式； 我们定义上下文关系特征来克服这个问题。

!!! warning "疑问"
    这部分也看不太懂，暂时跳过


## 5. Prior Work
黑盒优化（auto-tuning）用于高性能计算库，如 ATLAS [43] 和 FFTW [12]。 或者，可以构建一个依赖于硬件的成本模型来指导搜索 [28、5]。 Polyhedral methods [5, 42] 使用整数线性规划来优化成本。 Tensor Comprehensions [41] 结合了这两种方法，使用黑盒优化来选择线程块的参数和多面体优化来生成内部循环。 黑盒方法可能需要许多实验试验才能探索巨大的$S_e$。 另一方面，预定义的成本模型可能不够准确，无法捕捉现代硬件的复杂性，必须为每个新硬件目标手动重新定义

以前，  statistical cost models 已用于优化 SAT 求解器 [17、18]。 我们将这一想法应用于我们的问题，并构建了一个特定领域的成本模型，以实现工作负载之间的有效转移。 最近的趋势是使用深度神经网络来执行程序分析 [3、10]。 我们 new problem setting and experiment environment 可以作为相关方向未开发研究机会的试验台。

## 6. Experiment
TODO: