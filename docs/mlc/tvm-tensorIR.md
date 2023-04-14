# TVM-TensorIR

## (一)论文 

**TensorIR: An Abstraction for Automatic Tensorized Program Optimization**

### 0. 摘要
在各种设备上部署深度学习模型已经成为一个重要的话题。 各种张量计算加速器为张量计算带来了一组多样化的加速原语。 这些新的加速原语以及新兴的机器学习模型带来了巨大的工程挑战。 本文介绍 TensorIR，一种新的编译器抽象，用于使用这些张量计算原语优化程序。 TensorIR 归纳了现有DLC中使用的循环嵌套表示，将张量计算作为一等公民。 工作在抽象之上构建了一个端到端框架，以自动优化给定张量计算原语的深度学习模型。 实验结果表明，TensorIR 对于给定的硬件后端，可以自动应用张量计算原语，并提供与跨平台的最先进的手动优化系统相媲美的性能。

### 1. Intro
在机器学习加速目标的驱动下，现代硬件后端（例如，Nvidia Tensor Core [31]，Google TPU [22]）引入了专门的原语来加速张量计算。 领域专家也开始开发微内核原语(micro-kernel primitives)，它精心组织一系列高度优化的指令来执行子计算，以加速特定领域的张量算子库。 这些硬件指令和微内核原语通常在多维张量区域上运行，并有效地执行 多维load、点积 和 矩阵乘等张量运算（图 1）。 

**文章将这些张量计算加速器生成的指令称为 *tensorized intrinsics* ； 将利用tensorized intrinsics 对程序做变换的过程称为  *tensorization***。 为了充分利用这些硬件后端，现代机器学习系统需要优化包含 <u>分层循环嵌套</u>、 <u>multi-dimensional loads</u> 和 <u>tensor intrinsics</u> 的程序——文章称这个problem为 *tensorized program optimization problem*

<div class="autocb" style="text-align:center;"><img src="./tvm-tensorIR.assets\autocb_0.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

!!! warning "疑问"
    文章中提到的 "micro-kernel primitives"是什么？

当前的大多数张量程序都由领域专家优化，他们将 **张量化原语** 与 **多维循环** 、 **线程模式** 和 **数据缓存** 组合在一起，以制作专用内核库，例如 Intel MKL-DNN [20]、ARM Compute Library [3] 和 NVIDIA cuDNN [11]。 这些库随后被 TensorFlow [1]、PyTorch 等机器学习框架使用。 然而，需要巨大的工程努力来支持不断增长的模型和后端集，并且这些库需要迭代周期才能适应快速变化和增长的机器学习应用程序。

本文使用自动编译方法解决 tensorized program optimization problem 。 大多数过去的 DLC 工作 [9, 43] 针对程序中多层循环做自动化搜索调优，但没有做程序张量化这件事情。 将 自动程序优化 引入tensorized programs将充分利用张量计算加速器。 我们确定了实现这一目标的以下主要挑战：

**Abstraction for Tensorized Programs**。 要为张量化程序构建自动编译器，我们需要一种抽象，它可以实用地捕获给定机器学习算子的可能等效张量化计算。 

1. 这层抽象需要能够 表示不同硬件后端的 **多维访存**、 **线程层次结构** 和 **张量化计算原语** 。 
2. 抽象还需要具有足够的表现力来表示 ML 的大多数算子。

**Large Design Space of Possible Tensorized Program Optimizations**。 另一个挑战是为给定的算子自动生成高度优化的 张量化程序 。 DLC 需要使用领域专家可能使用的 包括 loop tiling, threading, layout transformations 等技术。 重要的是，这些变换现在需要结合张量计算进行，从而给分析和自动化带来额外的复杂性。 这些转换的组合形成了一个大的搜索空间，需要一种有效的方法来为给定的搜索空间找到优化的张量程序。

文章提出 TensorIR，an abstraction for automatic tensor program optimization 来解决上述挑战。 首先引入了一个名为 `block` 的新结构，可以 **从外部循环嵌套中划分和隔离张量化计算区域(tensorized computation region)**。 新的抽象使我们能够有效地表示张量化计算，并将它们与 循环嵌套、 线程 和 内存层级 相结合。工作还引入了程序转换原语来表达丰富的潜在优化空间。 我们在抽象和转换原语之上构建了一种新颖的自动调度算法。 此外，TensorIR 还可以表示和优化包含不规则计算和张量计算混合的程序，将可能的支持扩展到普通张量表达式 [9] 之外。 本文做出以下贡献：

- 针对张量化程序提出了一种全新的抽象；在抽象层级上可以将 **计算张量化** 从传统的 **循环变换** 中分离出来； 该抽象还允许我们对于 各种形式的 张量化计算指令 、 以及各种各样的张量积算加速器 进行统一的表示
- 构建了 变换原语 能够生成 丰富的搜索空间
- 实现了一个 tensorization-aware automatic scheduler

### 7. 总结
文章提出了 TensorIR，一种自动张量化程序优化的抽象。 

1. 设计了一个称为 `block` 的关键抽象，它可以隔离张量化计算并为程序优化提供有效的转换原语。
2. 构建了一个自动调度算法，与其他优化联合执行张量化并生成高性能程序。

### 3. TENSORIR ABSTRACTION

#### 3.1. Block
TensorIR 中的 一个 block 表示对 多维缓冲区 的一个子区域进行 的一处张量化计算 。 图 5 显示了矩阵乘法 (matmul) 计算的示例。 block 的主体由一组 block 迭代器变量 𝑣𝑦、𝑣𝑥、𝑣𝑘 参数化，代表抽象的张量化计算。 使用这些 block 迭代器变量的不同值组合实例化， 可以得到不同的具体 block 运行实例。 这些迭代器变量可以绑定到包含外部循环迭代器的表达式——这样做隐式地限定了 block 实例的执行顺序。

<div class="autocb" style="text-align:center;"><img src="./tvm-tensorIR.assets\autocb_1.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

<u>*Rationale*</u>:  block 的主要设计思想是隔离张量计算—— **我们希望能够在不查看 block body 的情况下转换 block 外的循环嵌套**。 然而，与标量计算不同，我们可能无法从不透明的张量计算体中提取转换所需的依赖信息。 **因此，我们引入了一个 block signature**， 其中包含足够的转换依赖信息。 我们在 [§3.2](#32-scheduling-transformations) 中讨论了这些转换。 此外，签名可用于在转换过程中独立验证迭代器绑定的正确性（更多细节见[§3.3](#33-validation)）。

<u>*Block Iterator Domain*</u>:  虽然可以通过将 block 迭代器绑定到任何循环嵌套来实例化 block 的主体计算，但大多数实例化并不对应于相同的计算。 为了保证转换之间计算的一致性，我们将 **iterator domain information 和 迭代器的约束存** 储在 block 签名中。 对于图 5 中的特定示例，我们知道 𝑣𝑥、𝑣𝑦 和 𝑣𝑘 必须绑定到 iterators in domain domain [0, 16)。 **此外，因为 𝑣𝑘 是归约轴，我们知道我们不能将它绑定到并行循环，除非归约是原子的**。 域约束仍然为外循环转换留下了巨大的空间，因为有多种方法可以构建满足约束的循环。 我们的域签名可以被视为表示整数域集和迭代器关系的特定方式。 我们选择这种特定表示是因为它的实现效率和推理简单性，但也需要指出相同的设计理念适用于整数集和 relation 的其他形式化 domain 表示 [42]。

!!! warning "疑问"
    We choose the particular representation due to its implementation efficiency and simplicity in reasoning, but would also point out that **the same design philosophy applies to other formal domain representations of integer sets and relations [42]**.

    整数集？ 多面体模型？

<u>*Access Region and Dependency*</u>: 为了提供足够的依赖信息，block signature 包含block相对于多维缓冲区的访问区域和读/写依赖关系。 在图5中，该 block:

- 读取𝐴[𝑣𝑦∗ 4：𝑣𝑦∗ 4 + 4，𝑣𝑘∗ 4：𝑣𝑘 ∗ 4 + 4], 𝐵[𝑣𝑘 ∗ 4 : 𝑣𝑘 ∗ 4 + 4, 𝑣𝑥 ∗ 4 : 𝑣𝑥 ∗ 4 + 4]
- 写入𝐶[𝑣𝑦 ∗ 4 : 𝑣𝑦 ∗ 4 + 4, 𝑣𝑥 ∗ 4 : 𝑣𝑥 ∗ 4 + 4]

这种依赖信息将会在转换过程中被使用。 我们只标记每个block相对于多维缓冲区而不是其他语句（block）的依赖性。 这种间接性使得我们能够支持更广泛的变换， 例如数据布局转换和 re-computation ， 这在张量程序优化中是必不可少的。

!!! warning "疑问"
    We only mark each block’s dependency with respect to the multidimensional buffers instead of other statements (blocks). **This indirection enables a broader range of transformations, such as data layout transformation and re-computation which are essential in tensorized program optimization.**

    是什么意思？


<u>*Reduction Block and Initialization*</u>:   reduction 计算通常包含初始化步骤和更新步骤。 我们可以自然地将 reduction 计算映射到两个 block 中。 但是，另一方面，共同制定这两个步骤的调度决策（例如 tiling 和 conputation location）通常是有帮助的。 我们为执行 reduction 的 block 引入了一个可选的初始化语句。 在 reduction 的第一次迭代期间执行初始化语句。 这种 reduction  block 表示在转换过程中很有用。 我们提供转换原语以在基于两个 block 的表示和基于初始 block 的表示之间进行转换，因此我们可以为低级代码生成选择最佳表示。

#### 3.2. Scheduling Transformations
对于给定的输入程序，我们需要生成具有等效语义的程序的搜索空间。 我们引入原语将 TensorIR 程序转换为等效的优化程序。 遵循张量程序优化的现有约定 [5、9、35]，我们将此过程称为 Scheduling。

如果一个 block 只包含 loop nests with sub-block as its leaves.，则该 block 是可调度的。 我们可以通过分析子 block 签名及其依赖信息来转换可调度 block 内的循环嵌套和子 block 计算位置。 值得注意的是，可调度 block 可以包含不可调度的子 block （例如，不透明的 Tensor Core 计算）。 不透明 block 还可以包含可调度的子 block 。 基于 block 隔离，我们仍然可以有效地独立探索可调度部分的搜索空间，同时保持相同的不透明 block 。 我们将在本小节的其余部分描述调度原语。

<u>*Loop Transformations*</u>: 循环tiling（split、 reorder ）和计算位置 mutation 等循环转换是优化程序以获得更好的内存局部性的重要方法。 我们还提供了这些循环转换原语（参见图 6 中的示例）。 与现有张量编译器直接提取每个叶标量计算语句的依赖关系的做法不同， 我们仅通过检查 block 签名来计算依赖关系。 除了循环转换，我们还支持将循环绑定到 GPU 线程的原语，并提供向量化和展开等注释提示。 请注意， block 隔离不会阻止许多重要的跨 block 协作优化（例如内联、协作获取）。 我们的循环变换涵盖了之前工作提供的循环变换，它允许 TensorIR 重现第 2 节中提到的搜索空间。

<div class="autocb" style="text-align:center;"><img src="./tvm-tensorIR.assets\autocb_2.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

<u>*Blockization*</u>:  循环转换原语保留 block 的整体层次结构。 正如我们在第 2 节中提到的，有时通过将子区域计算隔离到一个新的子 block 中来划分问题是有帮助的。 我们将这种变换称为 **blockization** 。 blockization 程序不再基于标量，因为新的子 block 对应于张量化计算。 我们可以使用 blockization 来隔离张量化的可能候选对象。 除了 blockization ，我们还引入了可以改变 block 层次结构的原语。 例如，我们提供了引入子 block 以将输入数据缓存到共享内存中的缓存原语。 我们还提供将单个 reduction-block 转换到 init-block 加 update-block 的变换原语。

<div class="autocb" style="text-align:center;"><img src="./tvm-tensorIR.assets\autocb_3.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

<u>*Separation of Scheduling and TensorIR*</u>: 许多以前的张量编译器(TVM-TE, Halide) **依赖于声明式调度语言来构建调度树**。 向这些编译器添加新的调度原语需要更改这些编译器中的调度树数据结构和相应的 lowering 规则。 我们采用不同的方法，**将每个调度原语实现为从一个 TensorIR 程序到另一个程序的独立转换**(类似于一个 pass )。 这种设计更易于扩展，因为不同的开发人员可以基于稳定的 TensorIR 抽象同时开发新的原语。 此外，开发人员可以在任何转换阶段打印出程序进行调试，并将自动重写与调度转换相结合。

#### 3.3. Validation
block 和其中记录的 读写关系 可用于验证 循环嵌套的正确性以及线程分配的正确性。

<u>*Loop Nest Validation*</u>: 循环嵌套验证检查循环嵌套提供的迭代器绑定是否匹配 iterator domain 的约束，包括domain 大小 和迭代器独立性信息。 例如，如果两个数据并行块迭代器绑定为 𝑣1 = 𝑖； 𝑣2 = 𝑖 ∗ 2，则对应的程序无效，因为𝑣1和𝑣2不独立。 但是𝑣1 = 𝑖/4； 𝑣2 = 𝑖%4 可以是合法绑定。我们构建 pattern-matchers 以找到从循环迭代器到块迭代器变量的准仿射映射，并使用该模式来验证绑定的 independence 和 domain 。 除了迭代器域验证之外，检查生产者-消费者关系以确保写入缓冲区的生产者块始终覆盖下游消费者的读取区域也很重要。

<u>*Threading Validation*</u>: 在为 GPU 和其他支持线程的加速器构建程序时，我们还需要对线程和内存层次结构进行额外的验证。 我们进行三种验证：

1. Threading Binding: 确保绑定到同一个线程的不同迭代器是一致的，并且满足后端的launching constraints
2. Cooperative memory access: 对于跨线程协作生成存储在共享内存中的缓冲区的 block，我们需要确保该 block 涵盖同一组中所有线程的下游需求。 同时，为该块提供输入的上游块需要覆盖该组中所有线程对该块的读取需求
3. Execution scope: 验证张量内在函数是否在正确的执行范围内运行（例如，TensorCore 需要在 warp 级运行）

!!! warning "疑问"
    回头再看这一节

<u>*Correctness of Schedule Primitives*</u>: 我们向每个调度原语添加检查以确保转换的正确性。 当调度原语仅更改循环嵌套时，我们还可以使用验证过程来确保正确性。 因为在这些情况下，块迭代域和依赖关系保持不变。我们发现更改块的调度原语（例如，块化）的特定于原语的必要条件。

请注意，loop nest validation and threading validation 用作检查以过滤掉无效的 TensorIR 程序，并且调度原语检查用于确保转换前后 TensorIR 程序的等效性。 当用户错误地手动制作、导入和调度 TensorIR 程序时，他们将收到警告或错误信息。 当用户使用编译器自动生成程序（将在第 4 节中讨论）时，验证可以帮助在搜索空间的探索过程中过滤掉误报案例。 因此，用户程序和编译程序都将受益于验证。

### 4. AUTO-SCHEDULING TENSORIZED PROGRAMS
本节中，我们描述了一个 tensorization-aware 自动调度程序来搜索调优。

图 8 显示了我们方法的概览。 我们的系统将来自用户的工作负载描述和关于硬件平台的张量内在描述作为输入。 自动调度程序首先通过**检查计算模式来生成用于张量化的候选对象**。 然后它生成使用 **张量化计算** 的程序草图候选，然后**根据计算模式决定数据移动**。 对于由张化程序草图引起的给定搜索空间，我们执行由 ML-based cost model 指导的进化搜索。 整个过程以张量化为中心，并利用新的块抽象来隔离张量化计算。 我们将在后续小节中讨论每个步骤的细节。

#### 4.1. Abstraction for Tensor Intrinsics
为了在优化中使用张量指令，需要一种方法来向系统提供它的语义和后端实现。 我们用相同的 TensorIR 抽象来描述给定硬件后端的张量指令。 对于每个张量化指令，我们引入了一个由两个块组成的 `TensorIntrin` 结构。 **一个块描述了计算语义，另一个提供了张量化计算的低级实现**。

在图 8 的示例中，我们使用带有标量主体𝐶 [𝑖, 𝑗] += 𝐴[𝑖, 𝑘] ∗ 𝐵[𝑘, 𝑗] 的普通循环嵌套来表示计算语义并使用内部点积指令实现指令 `𝑎𝑐𝑐𝑒𝑙.𝑑𝑜𝑡`。 我们还通过 TensorIntrin 中的多维缓冲区规范包括数据类型、存储范围、内存布局和连续性约束。 这些约束在验证步骤中使用。

值得注意的是，张量指令通常与通用平台中的特殊内存范围、数据布局和相应的加载/存储指令一起应用。 例如，在 NVIDIA GPU 上，如果我们决定使用 `nvcuda::wmma::mma_sync` API 来执行密集计算，那么我们需要应用 `nvcuda::wmma::load_matrix_sync` 和 `nvcuda::wmma::store_matrix_sync` 来准备输入操作数和 分别检索输出结果。 在 ARM CPU 上，像 `a64_gemm_u8_8×12` 这样的微内核要求操作数以交错布局存储。 开发人员可以通过为张量计算的每个输入和输出操作数指定特殊的存储范围来告知系统这些约束。

#### 4.2. Tensorization Candidate Generation
给定一对后端目标和一个输入程序，我们首先将程序主体与可能的 TensorIntrin 进行匹配，以生成张量化候选对象。 比赛以渐进的方式进行。 我们首先匹配表达式模式 𝐶 [.] += 𝐴[.] × 𝐵[.] 而不考虑索引。 然后，我们通过提出索引之间可能的映射来优化匹配。 图 9 给出了一个遍历匹配过程的示例。 在此示例中，我们将矩阵乘法内在函数作为后端描述。 这个张量本征的计算可以用下面的公式来描述

#### 4.3. Tensorized Program Sketch Generation
对于一组候选者，我们需要构建包含张量化的大型程序搜索空间。 我们将现有的分层搜索空间生成 [48] 推广到张量计算。 我们通过生成包含张量化计算的程序草图来构建搜索空间，然后枚举由生成的草图引起的选择。 如图 8 右侧所示，程序草图修复了部分程序结构，同时为剩余的参数选择留出空间，例如循环平铺大小和计算缓存决策。 我们通过迭代应用预定义的草图生成规则来生成草图。 重要的是，我们需要通过查看块签名来构建适用于张量计算的草图生成规则，并在我们的分析过程中利用访问区域信息。

**作为一等公民的数据移动**。 现有的张量程序自动调度器将其设计重点放在计算调度上，并以次要优先级处理不同内存范围之间的数据移动。 然而，由于张量指令极大地提高了计算的吞吐量，数据移动成为张量程序的瓶颈。 此外，数据移动决策通常取决于计算调度决策，如平铺、线程绑定、执行范围和生产者-消费者数据流粒度。 我们采用这些见解并将数据移动作为我们的自动调度程序中的一等公民，并将它们与计算调度分离。 具体来说，我们将 AutoCopy 块插入草图生成规则决定执行数据移动的位置（图 8）。 复制块隐藏了内存调度细节，只在块签名级别公开必要的缓冲区访问信息。 隔离的副本块允许草图生成独立做出计算调度决策，而无需考虑如何进行数据移动。 AutoCopy 块的主体描述了数据移动任务的详细信息，包括缓冲区位置映射、线程和存储范围要求。 数据移动调度程序将此信息作为输入并执行与内存相关的调度转换，例如插入中间缓存阶段、利用数据移动张量指令、矢量化、协作获取或跨步填充以避免存储体冲突。


#### 4.4. Evolutionary Search
在张量化程序草图生成阶段之后，我们可以获得数十亿个可能的诱导程序。 我们使用进化搜索来探索空间并找到优化的张量程序。 我们的搜索从给定程序草图的选择的随机初始化开始。 然后我们对当前的程序集执行突变。 然后，我们从变异的候选程序中选择有前途的程序，并在我们感兴趣的硬件后端上对它们进行基准测试。 我们从评估阶段收集数据以更新学习成本模型。

**张量计算的成本模型**。 我们构建了一个基于成本模型的提升树集成 [7]，该模型使用从程序中提取的特征。 特征向量包含与内存访问模式、重用和循环注释相关的信息。 重要的是，**我们以隔离的方式从区块签名和区块主体中提取特征**（例如，标记 Tensor Core 的使用）。 我们的成本模型可以看作是对先前张量程序方法的泛化。 我们认为，张量化程序的有效成本模型是未来研究的一个有前途的领域。

**Validation** 由于张量内在约束或无效循环嵌套候选者未满足约束，在搜索过程中随机变异程序可能会生成无效程序。 误报的可能性需要在搜索过程中进行验证步骤。 我们应用 3.3 小节中的技术来验证进化搜索中的候选程序，以识别和拒绝无效程序。 验证步骤减轻了进化搜索算法的负担，并允许我们在搜索过程中产生少量误报。



### 6. RELATEDWORKS

- 很多 DL 框架使用vendor optimized libraries (e.g., cuDNN [11], MKLDNN [20], TensorRT [32], ArmComputeLibrary [3]) 对计算进行加速； 计算密集型线性代数算子（矩阵乘， 点乘）的加速 在 HPC领域已经相当成熟； divide-and-conquer 是其中的重要思想

- DLC 对于 张量程序 引入了不同的抽象： 

    1. Halide 和 TVM 使用计算-调度分离的思想，用scheduling language描述计算； Tensor Comprehensions [43], Tiramisu [5] and MLIR/Affine [26] 使用 polyhedral model [42] 来分析循环依赖；这些工作 optimize loop nests with scalar computation in a bottom-up way 。 
    2. Fireiron [17] and Stripe [46] 使用嵌套多面体结构 以 top-down fashion 对张量程序建模；IREE [40] 是一个基于MLIR的端到端compiler which utilizes platform-specific optimization pipelines； TACO [12, 24, 38] is a compiler for sparse tensor algebra； Cortex [15] generalized tensor compilation to recursive computations； tensorIR 的工作与这些工作正交

- 自动优化是DLC的一个重要话题：

    AutoTVM [10] 使用机器学习的方法， 基于模板和训练好的 cost model 进行自动搜索； Triton [41] 引入了 tile-based template representation； FlexTensor [50] 可以自动生成模板； Halide 使用 Monte-Carlo tree search [2] 来构建一个自动调度器； Ansor [48] 使用 hierarchical search space 改进了自动调度；

- 自动向量化是compiler 领域的一个重要话题：

    Tensorization can be viewed as a generalization of the vectorization problem to enable tensor intrinsic。 张量化(Tensorization)可以被视为一种 泛化的 向量化(vectorization) 问题， 只不过使用的是 tensor intrinsic 而不是传统的 向量化指令。有 AKG [47]， UNIT [45]， AMOS [49] 等工作关注这个话题。


---

## (二)代码

### 0. TVM-Script

### 1. Tensor Expression

### 2. TOPI

虽然可以通过 TensorIR 或张量表达式 (TE) 为每个用例直接构造算子，但这带来很多重复工作。 TOPI (Tensor operator inventory) 提供一组预定义的算子（在 TE 或 TIR 中），由 numpy 定义并在常见的深度学习工作负载中找到。我们还提供了一组通用调度模板，以获得跨不同目标平台的高性能实现。

