# TVM-Relay

> 本文 **基于 tlc-pack/relax dc7072efe290d7e8c69d8e216311510981fc82e1**

## (一) 论文： Relay: A New IR for Machine Learning Frameworks

文章发在PLDI 2018， 这时候的 Relay 已经支持控制流，但还不支持动态shape， 后续nimble 在relay基础上做了一些改进。

### ABSTRACT
文章提出了一种高级IR Relay。 Relay 被设计为一种纯函数式、静态类型的语言，目标是平衡高效编译、表现力和可移植性。 我们讨论 Relay 的目标并强调其重要的设计约束。 Relay 实现为 NNVM 编译器框架的一部分。

### 1. Intro

- 1.1 **现有的高级 IR**

    这项工作的重点是重新设计 TVM 编译栈的顶层。这一层称为 NNVM，基于计算图，计算图是最流行的可微分计算表示。

    机器学习通常依赖于可微分的计算，即可以计算数学导数的计算。 为了保证用户程序的这种特性，现有框架限制了程序的计算表达能力。 TensorFlow 等框架使用静态图表示可微分计算，静态图是具有固定拓扑结构的数据流图。
    
    静态图很容易优化，但需要用户在深度嵌入的领域特定语言 (eDSL) 中构建程序，而且没有高阶函数这样的抽象。

    Pytorch 等构建具有动态拓扑的图，这些拓扑 **可以依赖于运行时数据** 并支持大多数命令式计算的微分。 这种表达能力对用户来说很方便，但限制了现有框架优化图的能力。 此外，PyTorch 的模型需要 Python 解释器，这使得部署到新的加速器和 FPGA 极具挑战性。

    **总之，静态图易于优化，但缺乏高级语言的表现力； 动态图提供了这种缺失的表现力，但引入了新的编译和执行挑战，尤其是在异构硬件和 FPGA 上。**

- 1.2 **Relay**

    Relay 旨在通过在函数式编程语言的支持下结合 静态图 和 动态图 方法的见解来平衡高效编译、表达能力 和 可移植性。

    也就是说，我们设计 Relay 不是从计算图的角度，而是从可微计算的编程语言的角度。 这种 PL 视角将使 Relay 能够利用在 函数式编程、 类型系统、 synthesis、 rewrite systems 和经典编译器技术方面数十年的研究。 我们想在未来工作中证明这些特点将降低以新加速器为目标的成本， 并实现更多优化以改善训练和推理时间、能源消耗和空间利用率。

贡献：

- Relay 一种用于表达机器学习模型的新型可微分语言。
- Relay 的高阶自动微分。
- Relay 的 shape-dependent tensor type system
- A baseline evaluator 和基于TVM 构建的类型专用算子编译器。



### 2. Background and Related Work
当前的 DL IR，在很大程度上受到了数据流编程和相关计算图抽象的启发，这些计算抽象在以前的框架中很流行。

例如，TensorFlow [4] 是 Google 之前 DistBelief [11] 等工作的迭代。 这些框架从数据流编程范例中发展而来，在这些范例中，抽象是具有输入和输出连接的算子。 这些语言提供的语义已在之前的工作 [5] 中进行了概述。

TensorFlow 使用 原语算子和受限的控制边（control edges）构造数据流图来表示可微程序。 TensorFlow 可以被视为深度嵌入式 DSL (eDSL)，其中执行用户 Python 脚本的结果是一个计算图，然后可以在执行前对其进行优化和转换。 此外，由于该图仅公开高级节点，因此程序可以移植到异构设备，并且在给定设备上执行子图只需要为该设备实现那些操作符。 不幸的是，这种编程模型有局限性。 因为拓扑结构在执行之前是固定的，所以 TensorFlow（1.0） 不适合，有动态形状的模型。 虽然确实存在一个库来缓解这个特定问题（参见 [24]），但这种模式表明，如果将来对新的依赖项感兴趣，也必须编写类似的库来解决每个问题，这需要大量的工程工作。

PyTorch [28]、Gluon 和 TensorFlow eager-mode [33] 等动态框架 moving from the define-then-run model to the define-by-run model。 PyTorch 在 Python 中嵌入了构建动态数据流图的原语。 控制流在 Python 解释器中执行，数据流由框架代码执行。 但是，当使用动态框架时，有关控制流的信息会丢失，从而降低了优化它们的能力。 此外，动态框架需要在图拓扑发生变化时重新优化，这会消耗 CPU 周期以及在主机和加速器之间移动数据的开销。 这可以通过转换 Python 代码来解决，但实际上与 Python 是输入 IR 的静态框架相同。

先前在高阶微分方面的工作为 Relay 设计提供了指导。我们从自动微分的各种实现中汲取了灵感 [1、2、7、13、20、29、36]。 我们特别感兴趣的是可以计算高阶程序的高阶梯度的技术。

最近对 TVM 的研究一直集中在产生高效的算子，例如广义矩阵乘法 (GEMM) 或卷积。 这一系列研究侧重于低级性能，但证明需要协同调整高级计算图、算子和加速器才能实现最佳性能。 输入程序的高级转换对于张量化问题尤为重要。 **张量化是现有编译器中向量化的类似过程** ， 涉及优化器将程序分解和匹配到暴露的底层硬件张量操作。 与类似的 SIMD 原语不同，由于是多维的、混合大小的和非有限的，这个问题更具挑战性。

TVM 旨在实现一系列基本优化：

- 高级优化，例如运算符融合和布局更改
- 在图和运算符级别的内存重用
- 张量化计算
- 延迟隐藏（传统硬件提供这种抽象，但新的加速器将这种负担推给了编译器编写者）

有多项相关的工程工作，主要来自谷歌和 Facebook。 Facebook 一直在构建一个由许多项目组成的高效 ML 堆栈，包括 Tensor Comprehensions [35] 和 Glow [31]。 Tensor Comprehensions 位于与 TVM 相似的空间中，但采用不同的技术，例如**使用多面体编译而不是算法调度**。 Glow 编译器 [31] 类似于 NNVM，旨在成为高级计算图的编译器。 Glow 的设计更接近现有的计算图，似乎不是一种完整的语言，也不太注重全栈调优。

TensorFlow 的 XLA 与完整的 TVM 编译栈非常相似，专注于为 TensorFlow 的计算图提供较低级别的中间表示。 Relay 旨在用更高层次的抽象来取代用户可见的图，让用户可以用纯Python编写像TensorFlow和PyTorch这样的框架。

### 3. Language
**Relay 是一种静态类型的、纯函数式的、可微分的 IR**。 Relay 不是用于编写和优化高性能内核的低级 IR； 相反，它旨在取代 NNVM 的计算图作为 NNVM 的输入层。 我们允许使用 C 或 C++ 等外部语言或 TVM 或 Tensor Comprehensions 等较低级别的 IR 实现原始运算符。 因为 Relay 旨在作为 TVM 堆栈的顶层 [9]，我们与 TVM 紧密集成并使用它来实现和优化内核。

我们的目的是让我们的新 IR 成为研究人员以 Edward 和 Pyro 的风格实施新的可微分编程语言和深度概率编程语言的便捷手段。

正如我们在第 2 节中讨论的那样，大多数流行的机器学习框架构建了代表用户程序的计算图。 由于这些图本质上是抽象语法树 (AST) 的修改形式，我们将对计算图执行的转换和分析视为程序转换和程序分析。 虽然其他 DL 框架也采用了这种观点，但它们基于图形的方法使得难以使用传统编译器和编程语言技术的全部武器库。

静态类型支持将模型直接编译到嵌入式硬件和加速器中，这已在 TVM 中得到证明。 拥有像 Relay 这样的 IR 可以为自然语言处理等应用程序部署更丰富的动态模型。 通过这种观点，我们可以利用数十年的编程语言研究来帮助我们表达和理解这些深度学习模型，不是将其作为一种受限的数据流语言，而是作为一种完整的编程语言。

#### 3.1. Grammar and Design
完整语言的语法可以在图 3 中找到。

Relay 是一种具有闭包、递归、条件、运算符和张量的函数式语言。 Relay 的 IR 在计算图上有两个主要的设计贡献：添加函数和可以捕获张量操作关系的丰富类型系统。

为了支持高阶（在高阶函数的意义上）可微程序，我们需要能够支持计算任意函数的梯度。 我们通过引入 higher-order、 higher-order (in both senses) reverse mode operator [29] 来实现这一点。 这个运算符允许我们计算高阶程序的 n 阶导数，开启了区分用函数编码的任意控制结构的能力。

部分受到 DLVM [37] 的启发，DLVM [37] 是一种神经网络 DSL，它支持用于深度学习程序的 CFG 式 IR，它引入了一种基于恒定张量形状和类型的张量类型系统， Relay 支持丰富的类型系统，包括 dependent typing for tensor shapes， 从而允许函数类型签名指定参数（例如属性或其他张量）与生成的张量形状之间的关系。

### 4. System Design
NNVM 目前将 DL 程序表示为包含算子和输入/输出数据流的静态计算图。 该图的拓扑结构是固定的，允许直接编译到 TVM 的图运行时。

我们首先在 Python 中构建了一个原型来验证我们的想法，并尝试进行转换，例如部分求值和自动微分。

Relay由一系列互操作的基本模块组成：

- Python 前端，将 Python 代码转换为 Relay 的 C++ 数据结构
- 用于relay程序自动微分的模块
- 形状相关的张量类型系统
- 用于原型设计和调试的简单evaluator 
- 基于TVM 构建的type-specialized operator compiler
- 高效的运行时系统（仍在开发中）

下面，我们将描述已经原型化的模块的设计和实现，并讨论 5 中正在进行的和尚未实现的组件。

#### 4.1. Frontend
Relay 目前有两个接口：一个可以用 Python 或 C++ 编写的文本 AST 和一个 Python 前端。 我们打算添加一个 JSON 序列化接口，以便与其他编译器轻松集成。

Python 前端是 Relay 预期的面向用户的交互模式，而其他接口允许以编程方式使用 Relay 的 AST。

Python 接口由两部分组成：一个库和一对装饰器。 该库包含标准的 DL 算子和一些特定于 relay 的函数。 这对装饰器将 vanilla Python 代码的一个子集转换为 Relay 文本 AST 表示，并生成一个包装函数，该函数将使用 Relay 的一种求值机制来执行该代码。

尽管 Relay 的核心是用 C++ 编写的，但我们能够通过重用 TVM 的 Node 系统将内部结构暴露给 Python，这允许两种语言之间的低工作量互操作性。 

Python 前端受到许多其他项目的启发，这些项目使用类似的机制来重写 Python AST，例如 Tangent [38] [8]。使用 Python 作为源语言还允许用户使用他们用于进行数据处理和部署的相同语言来编写和扩展 Relay。

图 4 演示了如何使用装饰器，我们将在下面简要概述它们的语义。
让我们先说明装饰器的描述，注意当前并未实现此示例中的所有功能，而是代表我们前端的设计和理想语法。

<div class="autocb" style="text-align:center;"><img src="./tvm-relay.assets\autocb_0.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

为了说明装饰器，我们简要跟踪一下前端如何将 4 中的程序转换为 Relay。 在这个程序中，修饰了三个Python函数：

- `lenet` ：LeNet 模型的声明
- `loss` ：模型的损失函数
- `train_lenet` ：训练循环


函数的每个参数都需要一个显式的类型注释，但是局部变量赋值的类型可以省略，稍后由后端推断。

relay 命名空间中的任何函数调用都将转换为内部标识符，它必须在 Relay 外部实现并在运行时注册。 TVM 是实现它们的首选机制。

为了防止将模型参数传递给每个需要它们的函数，我们有两个独立的装饰器： `relay` 和 `relay_model` 。 relay 装饰器声明了一个可以在没有任何隐藏状态的情况下运行的函数（因此，无法调用需要隐藏状态的函数）。 `relay_model` 装饰器声明了一个不能默认运行的函数，而是必须首先通过调用 `relay.create_model` 来实例化。 当为 `relay_model` 修饰的函数创建模型时，会在函数体中搜索任何需要隐藏参数的调用； 然后初始化这些调用的任何参数。 请注意，多次调用同一函数仍会生成多组隐藏参数。 比如 lenet 函数中， conv1 和 conv2 都有自己的隐藏参数。 当前假设所有模型参数的初始化为高斯分布，其中 µ = 0 和一些小的 σ 。

为了训练模型，我们根据我们的模型（即 lenet）定义损失函数，并且在我们的训练循环中，我们使用 `relay.grad` 来计算参数相对于输出的梯度。 然后我们将生成的梯度通过管道传输到 `relay.update_model_params` 以更新我们的参数（此示例使用普通随机梯度下降）。 虽然 Relay IR 通常是功能性的，但为了方便起见，我们将 relay.update_model_params 公开为一种有限的突变形式。

训练完成后， `relay.export_model` 返回训练模型的可调用版本，然后可以在原始 Python 中使用。

#### 4.2. Automatic Differentiation
在 [29] 中，作者证明了可以通过使用引入引用的本地程序转换在函数式语言中执行反向模式自动微分。 我们的方法受到他们的启发，并且与使用双数执行前向模式自动微分密切相关。 在对偶数方法中，实数值被转换为原始值和函数在该值处的导数对（称为“对偶数”）。 然后提升函数中的所有操作以对双数进行操作。

我们不是像正向模式那样将每个值与其偏导数配对，而是将每个实数值与 real 类型的引用配对，表示反向模式偏导数。 实数的逆模偏导数是实数相对于表示函数最终结果的变量的导数 [26]。 此外，对于我们执行的每个反向模式 AD 转换，我们从一个单元到另一个单元返回对函数的引用，称为“反向传播器”。 对于之前生成的每个实数，反向传播器都会根据链式法则进行更新，以获取其偏导数并通过引用将其向上游传递。 然后反向传播器调用它自己的旧版本，从而形成一个闭包链来更新每个偏导数。 有关更详细的解释，请参见 [29]。

我们用返回原始值和零初始化引用的转换操作替换实数上的每个操作，然后更新反向传播器以清除梯度参考，向前传播梯度参考，并调用旧的反向传播器。 用更具体的 AD 术语来表达它，Wengert 列表是在创建新实数时动态构建的。 Wengert 列表表示为创建反向传播器的闭包列表，更新列表的操作与列表捆绑在一起。

对于每个泛型操作，包括控制流和高阶函数，我们只需要转换内部表达式并提升类型以适应新表达式。 这与传统双数方法中所做的相同。

此外，我们使用梯度节点 (Grad expr) 扩展语法。 在梯度节点中，expr 应该是实数与实数的乘积的函数。 转换后的表达式是一个新函数，它计算与所有偏导数捆绑在一起的原始函数的结果。 该节点是通过我们实现反向模式自动微分来转换内部 AST 来实现的。 对于每个实型参数，我们将原始参数与新的零初始化引用捆绑在一起传递给转换后的函数。 我们调用反向传播器，提取传递的引用中的值，清除引用，并将提取的值返回到具有原始结果的产品中。

此转换要求我们转换传递的函数内的每个值，因此该函数不得包含自由变量。 （这个限制总是可以通过 lambda 提升来规避。）给定一个没有自由变量的中继程序，转换总是产生一个有效的中继程序，这意味着它具有闭包属性。 因此，我们有一个高阶反向模式，即使在包含闭包的程序中也是如此。 我们采用这种方法而不是 [29] 中的方法，原因有以下三个：

1. **简单性**：Pearlmutter 和 Siskind 的方法需要对 AST 和闭包转换进行反射，这意味着如果我们要遵循 [29]，我们将需要用我们自己的语言实现反射、代数数据类型和闭包转换。
2. **类型化**：此外，[29] 中生成的反向传播器具有依赖于闭包内自由变量的类型。 这意味着反向传播器的类型是动态的，这会使我们的类型系统复杂化。
3.**效率**：反射和遍历AST并不快。 虽然 Pearlmutter 和 Siskind 提议使用部分评估来消除这种开销，但它引入了另一层复杂性。

目前，我们仅通过公开 Grad 操作来保持 Relay 的 purity 。 用户代码永远不能与上述过程中产生的引用进行交互； 该过程完全抽象掉引用并仅返回结果值。 我们还可以通过将转换生成的代码输入为 lazy functional state threads （monads）来使转换产生的代码变得pure， 如 [22] 中所示。

Relay 中自动微分的实现包括 449 行 C++，而 C++ 后端总共大约 10,000 行。

#### 4.3. Type System
我们的类型系统是基于作者之前使用和实现 dependent type theory 的经验。 受到使用 small core languages 的类型系统设计的启发，我们一直保持小型语言 [12、18]。

我们的类型系统允许形状依赖。 也就是说，它允许类型在可以出现在表达式和类型中的形状上是多态的。 这种设计允许我们在编译时捕获重要的属性，尽管它摆脱了传统依赖类型系统的复杂性。 重要的是，我们有种类规则强制形状和基本类型都与值类型不同——即张量、乘积和箭头类型。

在这个范例中，知道所有值都是张量允许编译器编写者以统一的方式设计和实现对 AST 的优化。 例如，如果 Relay 的用户想要编写一个将计算提升一个维度的优化，他们可以统一添加一个维度，而无需处理标量情况。 这对于更改维度的优化（例如，自动批处理、空间打包或布局更改）非常有用。 我们在 5 中讨论了对类型系统的可能扩展。

将张量形状纳入类型系统而不是将其作为单独的 "分析" 的决定允许在优化管道的任何阶段轻松存储和推理形状信息，并使用户更容易明确 张量形状及其预期效果。


### 5. Future Work
#### 5.1. Runtime System
我们当前的 evaluator 是用于微分测试和实验的参考实现。 该 evaluator 不足以进行实验评估，我们当前工作的主要方向是其高效的对应部分。 这个求值器的一个有趣方面是它使用 TVM 作为 JIT 来生成类型专用的张量运算符。 优化的运行时系统，旨在作为部署和执行 Relay 程序的主要方式，仍在大力开发中。

传统语言已经针对非常具体的执行配置文件优化了它们的执行引擎的虚拟机，具有长寿命的堆分配和相对较小的堆栈值。 DL 工作负载具有截然不同的执行配置文件，通常不在传统 CPU 上执行，而是在 GPU 和加速器等专用设备上执行。

关于值的生命周期以及如何处理就地更新、分配、回收等，存在许多问题。 运行时系统需要函数调用堆栈的新表示、围绕作用域的新分配模式以及身份和分配的不同概念。

#### 5.2. Optimizations
Relay 旨在提供深度学习程序的全程序表示，使我们能够解决主机切片 [3]、动态网络、布局更改、延迟隐藏以及并行和分布式调度等问题。 我们在设计 Relay 时考虑到了这些目标，并帮助解决 [9] 中确定的关键优化。

我们设想能够在优化通过 Relay 程序时添加其他系统的功能，例如 DyNet [25] 中的 auto-batching 、在当前 NNVM 框架中完成的运算符融合， 或更改张量的布局。 自动批处理依赖于了解未批处理操作和批处理操作之间的一组转换的能力，在正确的位置插入适当的聚合指令，例如求和。 给定类型信息，可以使用额外的批次维度扩展某些程序，插入适当的运算符以保留类型和语义。

#### 5.3. Software Engineering
Relay 既支持单步调试器，也支持将 Relay 程序编译为 Python 以针对其他机器学习框架进行调试和差异测试的能力。 我们使用它来测试自动微分，方法是将 Relay 程序编译为 Python，使用“autograd”Python 库计算梯度，然后使用property-based testing [10] 检查梯度的结果。

#### 5.4. Numeric Accuracy
ML 工作负载已被证明对精度不敏感。 鉴于这种对低精度算术的容忍度，我们渴望采用最新的技术来自动重写 numerical code 来加速计算。 未来我们希望进一步扩展这些工具并针对专门的数值表示，包括混合宽度和定点计算； 阻塞浮点； 非标准的、特定于加速器的数字； 以及新兴的替代标准（例如，关于 unums 和 posits [16] 的工作）。

#### 5.5. Type System Extensions
**一种计划中的类型系统扩展是处理具有部分指定形状的张量，即某些维度未知的形状。** 这对于许多 NLP 应用程序很有用，在这些应用程序中，数据可能在一个或多个维度上呈锯齿状，并且无法用固定形状表示。

另一个扩展是扩展类型系统以跟踪  individual tensors’ data layouts。 Motivation是我们在写change-of-layout optimizations 时遇到的困难，这两者都必须推断现有布局并确保所有用途都已转换。 这些类型的错误导致难以调试的代码悄无声息地产生不正确的结果或崩溃。 通过使这些布局更改操作明确化，可以围绕值的自动装箱和拆箱以这种方式执行优化。

一个更重要的扩展是集成一个 effect system，以允许我们分离操作不同资源的代码，例如随机数生成器、状态、I/O 等。 这种变化更为激进，目前作为必须由编译器执行的分析而保留。


### 6. Conclusion
我们描述了一个 in-progress implementation of Relay： 一种用于高效编译和执行机器学习模型的新 IR。 我们的初始原型实现了我们关于研究人员如何在 Relay 中编写模型的愿景，具有原始 Python 的人体工程学以及 PyTorch 和 TensorFlow 等系统所享有的优势。 Relay 的实现仍在不断变化，我们专注于探索第 5 节中的主题。我们相信 Relay 是 TVM 的重要组成部分，它将促进当前和未来的研究工作。

## type system

Relay的类型系统包括以下三个子类:

- `PrimType`: type of primitive type values used in the low-level IR.
- `FuncType`: type of a function.
- `TensorType`: type of certain Tensor values in the expression.


```cpp
class TypeNode : public Object {
 public:
  // 记录源码位置信息 for debug
  mutable Span span;

  static constexpr const char* _type_key = "Type";
  static constexpr const uint32_t _type_child_slots = 14;
  TVM_DECLARE_BASE_OBJECT_INFO(TypeNode, Object);
};
```

PrimType 直接对应到Low-level IR的基础数据类型
```cpp
class PrimTypeNode : public TypeNode {
 public:
  runtime::DataType dtype;
  // ...
};
```

## 表达式 Expr

### Let-Bindings
见 [TVM: type system](./tvm-type.md)

### ADT

在 TVM 中， 使用 ADT 定义作为描述结构化数据的统一类，其中ADT相关的定义位于`include/tvm/ir/adt.h` 和 `include/tvm/relay/adt.h`

Algebraic data types (ADTs) 的概念来自函数式编程，简单来说，一个ADT定义含有多个构造函数( `Constructor` )，每个构造函数有不同的参数类型，每个ADT实例构造出来后只是简单地包含其构造时的所有参数值。

下面是在Relay中定义一个ADT的具体$x$例子:

```python
g_type_var = relay.GlobalTypeVar("Either")
type_var_a = relay.TypeVar("A")
type_var_b = relay.TypeVar("B")
prog = relay.TypeData(
    g_type_var,
    [type_var_a, type_var_b],
    [
        relay.Constructor("Left", [type_var_a], g_type_var),
        relay.Constructor("Right", [type_var_b], g_type_var),
    ],
)
mod = tvm.IRModule()
mod[glob_typ_var] = prog
# we get ==>
"""
type Either[A, B] {
  Left(A),
  Right(B),
}
"""
```

其中用于表示一个 ADT 的数据类型是 `TypeData`， 一个 TypeData 包含:

1. 该 ADT 类型的名称 (rust 中的 `std::Result` )
2. 该 ADT 类型所接收的**类型参数**列表 (rust `Result` 对应的 `T`, `E`)
3. 该 ADT 类型的构造函数列表 (rust `Result` 对应的 `Ok()`, `Err()`)

我们可以向 ADT 的一个构造函数中传入若干类型作为参数，得到一个新的类型； `TypeData` 定义如下(位于`include/tvm/ir/adt.h`)：

```c++
class TypeDataNode : public TypeNode {
 public:
  GlobalTypeVar header; // differently-named ADT-defs with same cons have different types
  Array<TypeVar> type_vars; // 该 TypeData 的类型参数
  Array<Constructor> constructors;  // 该 TypeData 的构造函数
  static constexpr const char* _type_key = "relay.TypeData";
  TVM_DECLARE_FINAL_OBJECT_INFO(TypeDataNode, TypeNode);
};
```

可以看到 3 个 fileds 分别对应刚才提到的一个 ADT 的3部分。其中 `Constructor` 定义如下：

```c++
class ConstructorNode : public RelayExprNode {
 public:
  String name_hint; // 构造函数的名字 (only a hint)
  Array<Type> inputs; // 构造函数的参数列表(每个参数都是一个类型)
  GlobalTypeVar belong_to;  // 该构造函数将会构造出哪一种(由一个GlobalTypeVar标识的)数据类型
  mutable int32_t tag = -1; // 在构造函数表中的index， 当该type被注册的时候将被设置
  ConstructorNode() {}
  static constexpr const char* _type_key = "relay.Constructor";
  TVM_DECLARE_FINAL_OBJECT_INFO(ConstructorNode, RelayExprNode);
};
```

而在 Relay 中使用 ADT 的例子如下（以使用刚才定义的 `Either` 为例）

```c++

```