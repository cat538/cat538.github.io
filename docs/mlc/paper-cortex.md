# CORTEX: A COMPILER FOR RECURSIVE DEEP LEARNING MODELS

文章基于 TVM 发表在2021 MLSys， 针对深度学习中的**递归模型**做了专门的优化，后续工作有 dietcode等

## 0. 摘要
优 DL 模型一般分两步进行。(i)高层次的图优化，如内核融合；(ii)低层次的内核优化，如 vendor 库中的优化。这种方法往往会留下显著的性能，特别是对于递归深度学习模型的情况。在本文中，我们提出了CORTEX，一种基于编译器的方法，为递归模型生成高效的代码，用于低延迟推理。我们的编译器方法和对厂商库的低依赖性使我们能够进行端到端的优化， 使得推理延迟比过去的工作低14倍，适应多后端。

## 1. Intro
DL 应用对模型的推理延迟有严格要求。因此，各种各样的硬件，包括CPU（Zhang等人，2018）、GPU（Nvidia AI，2019）和专门的加速器（Jouppi等人，2017），都在生产中用于低延迟推理。

对于具有递归和其他动态控制流的模型来说，加速推理尤其困难。以解析树表示的文本数据可以被送入TreeLSTM（Tai等人，2015）和 MV-RNN（Socher等人，2012b）等模型。图像中的层次和空间关系可以通过将其建模为树（Socher等人，2012a）或图（Shuai等人，2015）来学习。这些递归模型通常是为顺序数据设计的模型的扩展，如 LSTM （Hochre-iter & Schmidhuber, 1997）和 GRU （Cho等人，2014）。一个简单的递归模型如 图1 所示。我们在整个文章中使用这个模型为例。

<div class="autocb" style="text-align:center;"><img src="./paper-cortex.assets\autocb_0.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

过去关于递归和动态模型的工作，如DyNet（Neubig等人，2017a;b）、Cavs（Xu等人，2018）和 PyTorch（Paszke等人，2019）都依赖于特定硬件、高度优化的 vendor 库，如用于 Nvidia GPU 的 cuDNN（Chetlur等人，2014）和用于 Intel CPU 的 MKL（In-tel，2020a）。厂商库使这些框架能够为用户提供一个通用接口，在运行时采用专门的高性能内核实现，并有效利用需要针对的广泛的后端。

然而，厂商库在 model coverage 和 development effort 方面不理想。由于这些库是高度优化的，实现它们是一个非常复杂的过程。因此，它们只包含最常用的模型和内核的实现。例如，cuDNN 包含 LSTM 和 GRU 模型的实现，但不包含不太常用的 TreeLSTM 和 MV-RNN 模型。

此外，**厂商库中的每个内核都是单独优化的。这通常排除了诸如内核融合（将多个内核调用合并为一个调用）的优化，而这些优化已被证明是相当有益的（微软，2020）**。 *Model persistence*， 即把每轮迭代中都会用到的参数存在访问更快的 on-chip memory 中进行重用是另一个重要的优化（Zhang等人，2018；Holmes等人，2019；Diamos等人，2016）。但在使用厂商库时很难利用这种重用，尤其是在具有手动管理缓存的 GPU 等加速器上（Liu et al., 2019; Vasilache et al., 2018; Chen et al., 2018a）。 Nimble（Shen 等人，2020 年）等框架也在这些困境，它依赖于单个内核的自动调整实现。

在这项工作中，我们没有依赖厂商库或自动调整的内核，而是提出了一种基于编译器的方法，这使得我们能够进行优化，如 kernel fusion 和 model persistence 。 虽然过去有一些工作可以编译普通的前馈模型，但将这种方法应用于递归模型有以下挑战：

1. <u>递归控制流的有效表示</u>：图 1 展示了一个包含动态控制流 及常规张量的递归模型。 此类模型需要一种 IR， 通过递归控制流对张量计算进行编译器优化和代码生成。

2. <u>优化递归控制流</u>：加速递归模型推理需要有效的方法来执行控制流，且不妨碍内核融合等优化。

3. <u>静态优化</u>：动态模型通常在运行时通过构建展开所有递归的数据流图进行优化，并使 dynamic batching 等优化更容易。 此类优化必须以基于编译器的方法静态执行。

考虑到这些挑战，我们提出了 CORTEX(COmpiler for Recursive Tensor EXecution) ， 一个编译器框架，使用户能够表达迭代和递归模型，并跨不同后端（CPU 和 GPU）生成高效代码。 为了克服挑战 C.1，我们观察到递归模型中的控制流通常仅取决于输入数据结构。 这种见解以及 §2 中讨论的其他一些见解，使我们能够将递归计算降低为基于循环的高效计算（如图 2 所示）。 为了克服 C.2 和 C.3，我们使用调度原语来执行优化，例如专业化和动态批处理（Neubig 等人，2017b；Looks 等人，2017），以及编译时优化，例如 computation hoisting（计算提升）。

CORTEX 基于编译器的方法能够以端到端的方式优化模型计算，而不必像使用厂商库时那样将 算子 作为黑箱函数调用。这就实现了广泛的内核融合（见 7.3），同时避免了与动态批处理优化相关的一些开销（见 7.2）。作为CORTEX设计的一部分，我们重用了过去关于张量编译器的工作： extend a tensor compiler （Ragan-Kelley等人，2013；Chen等人，2018a；Baghdadi等人，2019；Kjolstad等人，2017）。这也为通过 auto-tuning （Mullapudi等人，2016；Adams等人，2019；Chen等人，2018b；Zheng等人，2020）来优化这些模型打开了方便之门。 table 1提供了CORTEX与递归模型相关工作的定性比较。

<div class="autocb" style="text-align:center;"><img src="./paper-cortex.assets\autocb_1.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

贡献：

1. 我们设计了CORTEX，一个基于编译器的框架，能够对递归深度学习模型的低延迟推理进行端到端的优化和高效的代码生成。

2. 作为设计的一部分， 我们扩展了张量编译器提供的抽象， 并提出了新的调度原语 和 递归模型的优化。

3. 我们对所提出的框架进行了原型设计，并针对最先进的递归深度学习框架进行了评估（Xu等人，2018；Neubig等人，2017a；Paszke等人，2019），在 Nvidia GPU 和 Intel 及 ARM CPU 上有显著性能提升（高达14倍）。

## 9. CONCLUSION
在本文中，我们介绍了 CORTEX，一种**用于优化递归深度学习模型**以实现快速推理的编译器。 CORTEX 实现了从**递归控制流到张量代数计算的积极内核融合和端到端优化**。 这使 CORTEX 能够显著降低推理延迟。 过去关于 MLC 以及 DL的工作表明，支持 ML 模型中各种动态的高效执行是非常可取的。 CORTEX 证明，这样做的一种富有成效的方法是利用过去在通用编译方面的工作，例如检查器-执行器技术或稀疏多面体框架。 我们认为，扩展当今使用的高度专业化 ML 框架和技术的范围也很重要（在不损害其优化静态前馈模型的能力的情况下），例如我们在 ILIR 中所做的。 将来，我们希望 (i) 应用这些见解来开发类似的技术来训练和服务具有潜在非递归动态控制流的模型，以及 (ii) 将 CORTEX 集成到更高级别的编程抽象中。

## 2. OVERVIEW

递归深度学习模型通常在执行张量计算时遍历 recursive data structures。 高效执行此类模型具有挑战性，因为它们的动态控制流通常会排除常见的优化， 例如内核融合。 在 CORTEX 中，我们观察到**递归模型中的控制流通常满足某些属性，使我们能够有效地将其降低为基于循环的迭代控制流**。 特别是，我们注意到许多递归模型具有以下属性：

1. 所有**控制流都取决于数据结构的连通性**，而不是动态计算的数据
2. 可以在执行张量计算之前进行所有的递归调用
3. 对数据结构节点的子节点的递归调用是相互独立的：一个调用的参数不依赖于先前调用的结果

属性 P.1 意味着**模型中的所有控制流都封装在输入数据结构中**。 属性 P.2 意味着计算可以从数据结构的叶子开始，向上移动到根。 属性 P.3 允许我们并行处理兄弟节点。 总而言之，这些属性使得为这些递归模型计算生成高效的基于循环的代码成为可能。

!!! warning "疑问"
    完全看不懂在说什么？

<div class="autocb" style="text-align:center;"><img src="./paper-cortex.assets\autocb_2.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

我们现在看看利用这些见解的 CORTEX 的编译和运行时工作流程（如图 2 所示）。 编译从Recursive API (RA) 中表示的递归模型计算 ① 开始。 用户还可以在此阶段指定一些调度原语 ② 来控制如何 lower 递归计算。 然后，编译器根据用户提供的调度原语生成与输入计算对应的 Irregular Loop IR (ILIR) ③。 **ILIR 是张量编译器使用的 IR 的扩展，旨在支持 间接内存访问 和 可变循环边界 等功能**。 它完全基于循环并且与数据结构无关。 因此， RA lowering 阶段将所有递归控制流降低到循环 ， 并将所有数据结构访问降低到该阶段潜在的 间接内存访问 。 可以在此处执行在张量编译器中执行的展开、tiling 等循环优化，之后生成目标特定代码 ④ 作为 ILIR 降低的一部分。

!!! warning "疑问"
    这里的 indirect memory accesses 指的是？

运行时工作流程反映了编译期间的 lowering 。 我们从指针链接的递归数据结构 ⑤ 开始，例如链表、树 或 DAG，将其降低为数组 ⑥ ，即使用 linearizer 执行线性化。 这种线性化使得生成迭代代码遍历数据结构成为可能。 linearizer 必须确保在执行此 lowering 时满足数据结构节点之间的数据依赖性。 请注意，线性化阶段不涉及任何张量计算。 **这是因为属性 P.1 允许我们将递归控制流与张量计算分开。 因此，我们在主机 CPU 上执行线性化**。

解析来讨论上面提到的每个编译和执行 stage。

## 3. RECURSIVE API (RA)
CORTEX 需要端到端的模型计算视图，以便执行内核融合等优化。 因此，输入程序需要包含有关模型中执行的张量操作的足够信息，以便在降低到 ILIR 时启用调度。 因此，RA 将输入计算建模为算子的 DAG， 其中每个算子都被指定为循环嵌套。 这在 Listing 1 中可以看出，它用 RA 表示图 1 中 的模型。 除了 RA 计算，用户还需要提供有关输入数据结构的基本信息，例如每个节点的最大子节点数以及数据结构的类型（链表、树或 DAG）。 此信息在编译期间使用，并且可以在运行时轻松验证。

<div class="autocb" style="text-align:center;"><img src="./paper-cortex.assets\autocb_3.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

### 3.1. Recursion Scheduling Primitives
将递归计算降为循环时，需要**保证数据结构节点之间的数据依赖关系得到满足**。 由于这些依赖性通常仅指定节点的部分排序，因此我们在安排计算时具有很大的自由度。 不同的调度可以提供不同程度的并行性，或者允许数据重用。 Listing 1 第 25 行和第 26 行指定了这些调度原语。我们建议使用以下调度原语来利用这些优化机会：

1. **Dynamic Batching** ：动态批处理（Neubig et al., 2017b; Gao et al., 2018; Looks et al., 2017; Xu et al., 2018）涉及动态批处理算子以利用模型批处理中的并行性 动态控制流。 由于我们研究的模型中的控制流仅取决于输入数据结构（属性 P.1），因此我们在线性化期间执行动态批处理。 使用动态批处理，树中的节点从上到下处理，如图 2 中的 6 所示。

2. **Specialization**: 递归计算往往有频繁的条件检查来检查基本条件。 这些检查可能会阻碍计算提升和常量传播（§4.3）等优化，并且会产生自己的执行开销。 因此，我们允许用户 specialize the program for the two branches of a conditional check 。 listing 2 显示了为递归模型生成的 ILIR。 **请注意它如何使用单独的循环嵌套来计算叶子和内部节点，因为叶子检查被要求 Specialization**（清单 1 中的第 26 行）。

    <div class="autocb" style="text-align:center;"><img src="./paper-cortex.assets\autocb_6.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>


3. **Unrolling**： 展开递归改变了处理节点的顺序（如图 3 所示），使节点的计算在时间上更接近其子节点的计算。 这允许通过 fast on-chip cache 重用孩子的隐藏状态，而不是较慢的片外内存。 例如，在 图3 中，递归调用中的每条边（图中的黄色框）都可以进行重用。 展开也为内核融合创造了机会，因为我们可以在子计算中融合算子。

    <div class="autocb" style="text-align:center;"><img src="./paper-cortex.assets\autocb_4.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

4. **Recursive Refactoring**：内核融合更难跨递归调用边界执行。 在这种情况下，可以使用递归重构来更改递归后边。 考虑图 4 左侧的计算。A1、A2 和 B 表示张量算子，因此存在从 A1 到 A2 的相关性。 在这种情况下，递归后边从 B 或 A2 到 A1。 融合 A1(n) 和 A2(n.left) 或 A2(n.right) 中的内核会很困难，因为内核位于递归调用边界。 重构改变了这个边界（backedge 现在从 A1 到 A2）。 因此，A1(n)、A2(n.left) 和 A2(n.right) 现在位于同一个调用中，可以很容易地融合。

    <div class="autocb" style="text-align:center;"><img src="./paper-cortex.assets\autocb_7.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

    !!! warning "疑问"
        没看明白？
    
## 4. LOWERING RECURSION TO LOOPS
### 4.1. RA Lowering
从RA 到 ILIR 的 lowering，本质上是一个从递归到迭代的 lowering 。因此，我们在 lowering 过程中**使所有的临时张量显性化**。例如，在 listing 2 中的 ILIR 中，我们明确地创建了10个张量 lh 和 rh 。我们还将张量 `rnn` 实体化，它存储了计算的结果。这三个张量中的每一个都存储了每个递归调用的数据，在这种情况下，这相当于每个树节点。

<div class="autocb" style="text-align:center;"><img src="./paper-cortex.assets\autocb_6.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

递归重构 和 循环展开 的调度原语是通过在 lowering 之前适当地转换输入 RA 计算来处理的。特殊分支是通过生成两个版本的计算来处理的，每个版本都是针对分支的一个目标的。 data structure linearlizer 为这些特殊分支划分节点， ILIR 为各自的节点划分采用正确的计算版本。 lowering 阶段产生适当的循环嵌套，对 data structure linearlizer 的输出进行迭代。默认情况下， ILIR 对节点进行迭代，但如果用户指定了 dynamic batching ， ILIR iterates over batches of nodes（如 listing 2）

### 4.2. Data Structure Linearization
在运行时， data structure linearlizer 会遍历输入的结构，并将其排列成数组，供基于下层循环的计算迭代。我们的运行实例的线性化器的伪代码显示如下。

<div class="autocb" style="text-align:center;"><img src="./paper-cortex.assets\autocb_8.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

data structure linearlizer 是在 RA 降低过程中产生的。在没有 specialization 和 dynamic batching 的情况下，线性化器基本上要像输入程序那样遍历数据结构，同时跟踪遇到的节点的顺序。这种对节点的排序将满足数据的依赖性，并可在张量计算中使用。因此，在这种简单的情况下， data structure linearlizer 本质上是输入程序，剥离了所有张量计算。对于标记为 specialization 的条件检查，线性化器将分别收集检查的两个分支之后的节点。对于 dynamic batching ，我们生成代码来遍历数据结构，并识别可以被并行处理的批量节点。

### 4.3. Computation Hoisting and Constant Propagation
> 编译器分析和移动代码，一般有以下几个步骤：
> 
> 1. 编译器会对循环进行数据流分析，确定循环的入口和出口，以及循环中的变量的定义和使用。
> 2. 编译器会识别出循环不变的代码，即在循环中每次执行都不会改变的代码，比如常量或者不依赖于循环变量的表达式。
> 3. 编译器会检查循环不变的代码是否有副作用，比如修改全局变量或者调用函数。如果有副作用，编译器会保留循环不变的代码在循环内部，以保证程序的语义不变。
> 4. 编译器会把没有副作用的循环不变的代码移到循环外面，只计算一次，然后把结果保存在一个变量里。这个变量会替换循环内部的循环不变的代码。

递归和迭代模型通常使用一个初始值作为 base case。 如果这个初始值对所有的叶子都是一样的，那么对所有的叶子都要进行相同的计算，这是多余的。当降低到 ILIR 时，这样的计算被从递归中 hoisted out。 我们还特别优化了初始值为零张量时的情况。

## 5. IRREGULAR LOOPS IR (ILIR)
我们已经简单地提到， ILIR 是张量编译器所使用的程序表示法的扩展。因此，计算和优化在 ILIR 中被单独指定。计算被表达为一个算子的DAG，每个算子都通过吸收先前产生的或输入的张量来产生一个张量。在调度原语的帮助下，可以进行优化，如 loop tiling, loop unrolling, vectorization 等。

RA lowering 后产生 ILIR 。ILIR 是基于循环和数据结构的 agnostic ， 这种降低产生了间接内存访问和具有可变循环边界的循环。请注意，在listing 2中，第 1 行的循环中用于索引张量 rnn 的变量 node 是循环变量 `n_idx` 的一个非 affine 函数。此外，第 7 行的循环在一批节点上迭代，有一个变量边界，因为批次可以有不同的长度。为了支持这些特征，我们用(1)非affine索引表达式，(2)具有可变边界的循环，和(3)条件算子来扩展张量编译器。下面我们将进一步详细描述这些修改。

### 5.1. Indirect Memory Accesses
我们将作为间接内存访问的一部分产生的非affine索引表达式表示为循环变量的未解释函数（Strout等人，2018）。间接内存访问需要进一步的改变，这将在接下来描述。

**Bounds Inference:** 编译过程中，张量编译器为输入程序中的所有算子推断循环边界。对于每个产生张量t的算子，编译器首先计算t的哪些区域是其消费者需要的。然后，这个数量被转化为op的循环边界。在传统的张量编译器中，这是直接的，因为算子的循环和相应的张量尺寸之间存在着一对一的对应关系。然而，ILIR的情况并非如此，正如清单2所显示的那样。张量lh、rh和rnn各有两个维度，但生成的 ILIR 对其相应的算子都有三个循环。因此，我们要求ILIR明确说明张量尺寸与相应算子的循环巢中的循环之间的关系。这将在附录的A.2节中进一步讨论。

**Tensor Data Layouts:** 中间十张量的数据布局经常需要改变，以便有效地使用内存子系统。为了实现这样的优化，ILIR公开了数据布局基元，允许张量尺寸被分割、重新排序和融合，类似于相应的循环转换。

当中间张量存储在scratchpad内存中时，如图5中的A1，用非affine表达式为其建立索引会导致张量的稀疏填充。这样一个稀疏填充的张量占据了过多的内存，这是有问题的，因为刮板内存空间往往是很宝贵的。这可以从图5的左边尺寸看出，A1的一半是未使用的。在这种情况下，我们可以通过循环迭代空间对张量进行索引，而不是像图5的右边那样。注意我们现在需要在scratchpad内存中分配一个小得多的张量。这种转换也通过将间接内存访问变成仿射访问来降低索引成本。它也被暴露为一个调度基元。

### 5.2. Conditional Operator
为了降低条件检查，如我们模型中的isleaf检查，我们在ILIR中添加了一个条件算子。它需要两个子图和一个条件检查作为输入，并被降低为一个if语句。如果用户没有专门的叶子检查，条件算子就会在我们的运行实例的ILIR中生成。
关于ILIR降低的更多细节以及我们在其中做的一些小的优化可以在附录中找到。

## 6. IMPLEMENTATION
为了评估的目的，我们对普通情况下的 CORTEX pipeline 进行了原型化。在这一节中，我们谈一谈有关的一些实现细节。

**RA Lowering**。作为RA降低的一部分，我们已经实现了对动态批处理和专业化的支持，用于叶子检查的常见情况。

**ILIR Lowering**。我们扩展了TVM（Chen等人，2018a）v0.6，一个深度学习框架和一个张量编译器。我们目前的原型实现没有对生成的 ILIR 进行自动排程。因此，用于评估的模型im-plementations是基于手动定义的时间表。然后，我们通过网格搜索进行自动调谐，以搜索某些时间表参数的空间。之前关于自动调度的工作是对我们技术的补充，并且可以很容易地应用到原型上。

**Data Structure Linearizers**。我们实施了数据结构线性化器（树和DAG各一个），用于我们的评估。我们对数据结构节点使用了一个编号方案，在附录的§B中描述了这个方案，通常可以减少叶子检查和批量迭代的成本。

## 7. EVALUATION

## 8. RELATED WORK
**<u>MLC</u>**： Tensor 编译器，例如 TVM、Halide、Tiramisu、Tensor Com-prehensions 和 Taco 。 Taco 支持的稀疏张量计算与 ILIR 之间存在相似之处，这导致了相似的实现技术。 例如，the idea of dense layouts for 中间张量(§5.1) 类似于 (Kjolstad et al., 2019) 中引入的 Taco 工作空间的概念。 然而，更一般地说，CORTEX 扩展了张量编译器提供的抽象以支持递归计算，并为此开发了专门的优化。

XLA (Team, 2017) 和 Glow (Rotem et al., 2018) 等 DLC 优化了静态前馈模型，可以执行部分内核融合和代码生成。 此外，在（Radul 等人，2020 年）中，作者开发了一些技术，可以在为 XLA 工具链执行动态批处理时有效地将递归降低到迭代控制流中。 TensorRT（NVIDIA，2020）和 OpenVINO（英特尔，2020b）等推理引擎优化推理模型执行。 我们在本文中开发的技术可以用作这些 DLC 和优化器的低级后端。 **MLIR（Lattner 等人，2020 年）提供构建 DLC 的基础设施，并且 CORTEX 有可能使用 MLIR 构建**。

**<u>Optimizing Dynamic Neural Networks</u>**： 有大量工作旨在优化递归和更一般的动态神经网络。
动态批处理的变体已用于 DyNet、Cavs、BatchMaker、Ten-sorFlow Fold 和 Matchbox 等框架中。 与这些不同，CORTEX 在张量计算之前执行动态批处理。 模型持久性首先由 Persistent RNNs 提出，随后用于 GRNN 和 VPPS，并适用于 DeepCPU 中的 CPU (Zhang et al., 2018) 。 CORTEX 能够将这些优化扩展到递归模型，并将它们形式化为编译器中的转换原语。

Nimble 采用 DLC 技术以更好地支持动态模型。Janus 推测性地创建了可以优化以加速动态模型的数据流图。 与 DyNet 类似，这会导致运行时的开销。 在 (Jeong et al., 2018) 中，作者使用递归扩展了 TensorFlow 的静态数据流图。 此外，虽然 CORTEX 目前专注于非循环数据结构，但 ILIR 基础设施也可用于支持更多图的深度学习，正如 DGL 所支持的那样（Wang 等人，2019）。

正如我们在第 3 节中看到的，与上述框架相比，CORTEX 提供了较低级别的编程抽象。 我们相信 CORTEX 有可能用作这些框架的后端，这将减轻使用 §1 中讨论的厂商库的缺点。

**<u>Sparse Polyhedral Framework</u>**： 稀疏多面体框架 (SPF)（Strout 等人，2018 年；Mohammadi 等人，2019 年；Nandy 等人，2018 年）针对稀疏张量计算的情况扩展了多面体模型。 CORTEX 借鉴了这些作品的技术，例如使用未解释的函数来表示间接内存访问。 CORTEX 中的 data structure linearlizer 可以看作是检查器-执行器技术的一个实例 (Agrawal et al., 1995)。 在 (van der Spek et al., 2010) 中也提出了使用这种技术来降低数据结构。
