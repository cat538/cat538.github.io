# MLIR-IREE

## 0. Abstract

过去十年中，用于训练和执行的机器学习模型部署一直是工业界和学术界研究的重要课题。 大部分工作集中在开发特定工具链以支持加速硬件上。 本文介绍 IREE，这是一个统一的编译器和运行时 stack，其明确目标是将机器学习程序缩小到移动和边缘设备的最小占用空间，同时保持扩展到更大部署目标的能力。 IREE 采用基于编译器的方法，并通过使用 MLIR 编译器基础架构针对异构硬件加速器进行优化，MLIR 编译器基础架构提供了快速设计和实现多级编译器中间表示 (IR) 的方法。 更具体地说，本文关注的是 TinyIREE，它是 IREE 中的一组部署选项，可适应嵌入式系统和裸机平台中有限的内存和计算资源，同时还展示了 IREE 为不同 ISA 扩展生成工作负载的直观工作流和 ABIs through LLVM.

## 1. Intro
随着新模型架构和更大数据集的出现，对硬件的需求也在增加。 这加速了 ML 模型训练和推理的新架构的开发。 然而，要充分利用此类加速硬件，开发工具链需要新的部署和编译流程。 此外，由于系统资源和功率限制，将模型部署到移动和边缘设备时必须特别考虑。

该行业的大部分投资都针对可扩展到最强大的部署目标的用例，而在低功耗和嵌入式设备的表现却很差。 IREE（Intermediate Representation Execution Environment）通过设计一个统一的编译器和运行时堆栈来解决这种不平衡，其目标是最小化空间占用，同时保留其扩展到更大部署的能力（并与更高的集成） 级分布式运行时）。 这种对模型可移植性的关注促使 IREE 有效地将裸机/嵌入式 CPU 和微控制器作为目标。 本文介绍了如何实现这种可移植性。

传统上，ML 推理框架(以微控制器 （Microcontrollers） 为目标) 主要是由基于内核运行时完成的，这些运行时针对一小组常用算子进行手动优化。 一种简化将模型部署到微控制器的方法是 TensorFlow Lite for Microcontrollers (TFLM)，这是一种专门设计用于支持微控制器上的模型执行的框架 [1]。 它使用支持 TensorFlow 算子子集的小型运行时库（对于 Arm Cortex-M3 约为 16 kB），具有特定于硬件的高级算子内核实现。 图优化仍然利用 TensorFlow Lite 编译流程，并在算子级别进行优化。

尽管 TFLM 是一个具有有限硬件特定内核的运行时库，但基于编译器的方法最近也已应用于加速器和嵌入式系统。 其中一个编译器是 Glow [2]，它使用两阶段中间表示 (IR) 来降低神经网络图。 使用高级中间表示来应用特定领域的优化。 较低级别的 IR 允许编译器应用与内存相关的优化。 这些优化包括指令调度、静态内存分配和复制消除。

另一个编译器是 Apache TVM [3]，它在各种中间表示中执行高级图重新配置和低级操作优化。 虽然 TVM 面向CPU、GPU、DSP等各种设备后端，但对于裸机微控制器，需要 MicroTVM 扩展来支持模型执行调度和内存管理，以弥补操作系统的缺失。

这让我们了解了我们如何通过 IREE 应对这一挑战。 IREE 是一个端到端的编译器和运行时框架，用于基于 MLIR 编译器基础设施 [4] 的模型执行。 它遵循基于编译器的方法，将 ML 模型转换为中间表示，允许分析和优化 ML 模型，同时针对异构硬件加速器[5] 生成代码 。 如 图 1 所示，IREE 可以将各种模型表示作为输入，并为不同的硬件目标生成可执行格式。

<div class="autocb" style="text-align:center;"><img src="./mlir-iree.assets\autocb_0.png" style="zoom: 45%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

IREE 将 ML 模型视为另一个程序，用各种 MLIR 方言表示它，并最终生成可以在目标架构上运行的二进制文件。 一种 MLIR dialect 可以认为是逻辑上的一层 IR。 这种通过一系列可组合 IR [5] 进行多阶段编译是一个关键的区别因素，因为它使我们能够更有效地重新定位和概括用于针对更大规模部件的方法和组件，以实现专门的、深度嵌入的用途。 这种泛化在经典的、基于发射器的两阶段编译器中更难实现，并且在 TFLM 等执行器中根本不可能实现。

IREE 旨在针对各种边缘设备和加速器，包括 CPU、GPU 和各种 ML 加速器。 因此，移动部署的各个方面，如小二进制文件大小、与应用程序的互操作性和可移植性，都是头等大事。 在本文中，我们重点介绍 TinyIREE，它是 IREE 选项的子集，可生成紧凑的工作负载和运行时库，该库针对没有操作系统的嵌入式系统进行了优化。 这些选项可以无缝支持不同的 CPU 架构和 ISA 扩展，同时还提供一个接口来支持嵌入式 ML 加速器。 在图 1 中，TinyIREE 选项是彩色的，而一般的 IREE 编译和部署选项是灰色的。

在以下部分中，我们将介绍 IREE 的编译流程并解释一些 TinyIREE 部署选项。 此外，我们在给出结论之前展示了已编译的 MobileNet 单次检测器 (SSD) 模型的结果。

## 5. CONCLUSION
我们介绍了 IREE，这是一种基于 compiler 的 ML 执行框架，它充分利用了 MLIR，且可针对不同目标设备轻松配置。 此外，TinyIREE 选项可以 复用 相同的编译流程，通过调整 runtime 以降低开销从而适应嵌入式系统。 我们还展示了 通过调整 LLVM 编译标志支持不同 CPU 架构和 ISA 扩展的能力。 因此，提出的框架可以同时针对多个目标后端进行优化，同时保持足够的定制灵活性。


## 2. IREE COMPILATION FLOW 💡

IREE 的编译流程使用 MLIR 的方法，将 ML 程序逐步降低为目标硬件的可执行文件。 MLIR 实现这一目标的主要工具是 方言 dialect。 MLIR 中的方言是 operations 和 associated types 的集合。 可以使用不同的方言在不同的抽象层次上对程序进行建模。

例如，图 1 中的 IREE importer 显示了 IREE 中用于将 ML 程序转换为机器可执行代码的一些不同方言。

- TOSA（Tensor Operator Set Architecture）[6] / MHLO（Meta High Level Optimizer）dialects **在张量运算级别对 ML 程序进行建模**。
- Linalg 方言以简洁、易于操作的方式表示完美嵌套循环，使得实现融合、平铺、循环交换等转换变得容易。
- Vector 方言 将程序的一部分建模成虚拟向量运算。
- 最后，LLVM 方言是一种 leaf dialect，它允许将 MLIR 程序翻译成 LLVM IR。 

这些方言中的每一个都是使用通用基础结构定义的，所以在每个级别上，都可以应用通用的编译器优化，如公共子表达式消除 (CSE)、死代码消除 (DCE) 等。每个方言也可以应用自定义优化 使用该方言中的操作捕获的语义信息。 例如，在 TOSA 方言中，人们可以很容易地识别 1×1 二维卷积运算并将它们转换为矩阵乘法运算，然后可以更有效地执行。 从更高级别表示（如 TOSA/MHLO）的方言通过中间方言转换为可执行二进制文件的过程——被称为 progressive lowering 。 本节的其余部分概述了 IREE importer 支持的每种方言的程序表示。

TOSA 和 MHLO 等前端方言使用加法、卷积、点积等操作表示程序。 例如，清单 1 显示了一个简单的序列，其中包含一个点积，然后是 TOSA 方言中的偏置加法。 在 MLIR 内置类型 Tensor 中使用 `?` 表示动态维度（即在编译时未知的维度）。 因此，MLIR 对表示动态形状的计算提供了一流的支持。

<div class="autocb" style="text-align:center;"><img src="./mlir-iree.assets\autocb_2.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

然后将程序降低到 **Linalg 方言**。 在这个级别，操作被表示为完美嵌套循环计算，最内层循环的主体执行标量操作。 这种表示旨在更轻松地应用传统的循环转换，如循环融合、循环平铺和循环交换。 这种建模的灵感来自多面体编译，其使用正交空间表示计算，计算的每个循环都有一个轴。 这个正交空间被称为计算的迭代空间。 循环中归纳变量的值表示为沿相应轴的点。 这个正交空间中的每个点都代表在最内层循环中执行的标量操作序列。

> 清单 2 显示了形式为 $D_{ij} = C_{ij} + A_{ik} \times B_{kj}$ 的 GEMM 运算的 Linalg 方言表示。 循环数由列表迭代器类型中的条目数捕获。 循环迭代之间的数据依赖被指定为并行（无依赖）或归约（归约型依赖）。
> 
> <div class="autocb" style="text-align:center;"><img src="./mlir-iree.assets\autocb_1.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

与迭代空间类似，每个操作数也由与操作数相同维数的标准正交空间表示。 该空间称为操作数的数据空间。 例如，在 GEMM 操作中，所有操作数都由 2D 数据空间表示。 索引映射中的映射捕获每个操作数的数据访问模式。 在每个映射中，域表示迭代空间中的一个点，范围表示映射访问的操作数数据空间中的点。

*region* of the operation （`{` and `}` 之间的操作序列）表示在迭代空间中的每个点执行的计算。 每个 Linalg 操作都有一个带有单个基本块的区域，其参数表示标量值，这些值是使用索引映射从操作数中获得的。 产生的值是迭代空间中该点的计算结果。 例如，在清单 2 中，迭代空间中的每个点都执行乘加运算。

TOSA/MHLO 的所有张量运算都可以降级为 Linalg 方言的 `linalg.generic` 运算。 通过仅使用迭代器类型和索引映射中的信息，可以将两个具有**生产者-消费者关系**的此类操作融合到另一个 `linalg.generic` 操作中。 这种融合包含所有逐元素融合优化（在 XLA 中也称为循环融合），而无需推断每个操作执行的实际计算。 例如，在 TOSA/MHLO 级别进行相同的融合会导致融合可能性数量的组合爆炸。 请注意，可以以这种方式融合的操作类别大于纯元素操作，包括排列和广播。

Linalg 方言中的操作也可以仅使用迭代器类型和索引映射提供的信息来平铺。 这种转换允许将每一层的计算拆分为类似计算的较小块。 例如，一个 GEMM 操作可以被分割成几个更小的 GEMM 操作。 这些较小的图块可以并行计算，因此可以分配给不同的线程。 然后，每个 tile 表示的计算被封装在一个调度区域内。 每个调度区域都包含必须以原子方式在设备上执行的代码。 此时，每个计算被分成两部分执行：分派区域（分块计算）和执行主机端计算的代码（VM 命令），它指定了分派区域的执行顺序。 然后使用融合、循环交换等进一步转换调度区域内的代码，以实现更好的缓存局部性和数据访问模式。

最后，代码被降低到 Vector 方言中。 这种方言表示目标硬件上可用的高级、可重定向的矢量指令。 这种方言中的转换允许生成的代码使用目标体系结构上可用的高效矢量读/写和特殊固定功能单元。

设备端编译的最后一步是将程序降低为 LLVM 方言，这允许程序机械地转换为 LLVM IR。 然后 LLVM 编译堆栈可以为目标架构生成二进制代码。 LLVM target 标志可用于选择要使用的 CPU 架构、ABI 和 ISA 扩展。 例如，要为 x86-64 CPU 构建 module，应用以下标志：

<div class="autocb" style="text-align:center;"><img src="./mlir-iree.assets\autocb_3.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

前面提到的优化过程可以很容易地应用于其他非 ML 特定的线性代数运算，只要这些运算采用适当的格式即可。 换句话说，IREE 还可以将执行输入预处理和输出后处理的操作作为程序的一部分进行优化。 例如，视觉 ML 程序通常会执行颜色空间转换或图像数据流的图像大小调整，并且这些操作可以通过遵循相同的编译流程在程序中融合。

## 3. TINYIREE DEPLOYMENT OPTIONS
上述编译流程的输出是一个 IREE module，其中包含一段用于控制缓冲区设置的 VM commands 和一组作为工作负载的调度区域。 这些工作负载包含可以 分发 到各个目标设备的 ML 程序。 如图 1 的底部所示，当以嵌入式系统为目标时，用户需要构建一个 IREE 运行时应用程序，其中包括用于配置 VM 设置的 IREE 运行时库、工作负载加载器、工作负载调度器。 后两者被定义为设备硬件抽象层 (HAL) 驱动程序。

在编译最终的可执行文件之前，我们可以进一步分解具有不同部署策略的 IREE 模块结构，如以下小节所述。 如图 2 所示，IREE 为 CPU 上的各种部署场景提供了一组灵活的工具，包括静态和动态嵌入式库，而 VM 控制支持可以使用 bytecode renderer 或 C source code 来实现。 在这里，VM 字节码以 FlatBuffer format (vmfb) 存储在一个文件中，可以选择以共享动态库的格式嵌入设备工作负载。

除了嵌入式 CPU 的这些部署路径之外，开发人员还可以利用现有的 HAL 接口与定制的 ML 加速器进行交互。 此外，虽然 IREE 运行时可以支持动态部署，但嵌入式系统甚至可以完全绕过 IREE 运行时并部署特定的工作负载以实现最高效的执行。

### 3.1. Virtual machine control with EmitC

VM commands 控制资源所有权和执行流程，它们被编码为另一种 MLIR 方言：**VM dialect。 这些操作可以序列化为在运行时解释的字节码 module**。 另一种选择是将这些操作序列化为 C 源代码。 这是通过**将 VM 操作转换为 EmitC dialect** 来实现的，EmitC dialect 是一种允许从 MLIR 生成 C/C++ 的 MLIR dialect。 将 VM 操作转换为 EmitC 操作后，这些操作将通过 Cpp emitter 转换为 C。 EmitC 包含一个表示 C 函数调用的 call operation 。 另外，这种类型的操作用于直接调用 VM API，而不是将调用序列化为等效的字节码表示。 因此，字节码解释器没有链接到可执行文件中，从而减少了它的体积。

IREE repo 中的 `static_library_demo_c` 示例中提供了使用 EmitC 的示例。

### 3.2. ML workload as a static or dynamic library
ML 工作负载可以包含在静态或动态库中。 在这两种情况下，IREE 使用 LLVM 生成对特定目标优化的指令流。 静态 ML 工作负载库可以与序列化的 VM 字节码模块结合使用，形成 IREE module. 但如果要缩减二进制文件大小，静态库应该与转换为 C 调用的 VM commands 结合使用。

另一方面，动态 ML 工作负载库提供了更大的灵活性。 例如，可以创建多个具有相同入口点和出口点的针对不同体系结构优化的动态库。 然后在运行时决定使用哪个动态库。 由于嵌入式系统往往不提供动态库支持，IREE使用固定的动态库ABI，**并在运行时库中集成了自己的低开销动态库加载器**。

### 3.3. Scheduler of the HAL driver
一旦host VM 正确设置了命令缓冲区，工作负载调度程序就会负责调度命令缓冲区中设置的设备工作负载。 受 GPU 调度程序和计算 API 的启发，IREE 针对包含数据缓冲区和调度信息的特定 worker 结构采用 3D 网格拓扑进行工作负载调度，如清单 3 所示。

<div class="autocb" style="text-align:center;"><img src="./mlir-iree.assets\autocb_4.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

对于支持多线程的系统，IREE 提供了一个异步任务调度程序，以更高效地执行有向无环图 (DAG) 来拆分工作负载。 这允许乱序和流水线执行，可以实现更好的并行性和目标设备的更好利用。 但是，IREE 还支持通过顺序发布工作负载来同步调度程序执行。 对于没有线程处理程序的嵌入式系统或没有操作系统支持的 CPU，同步调度程序提供了一种直观的方式来分派工作负载。

### 3.4. Stream execution
由 IREE 运行时调度的 work 以及 work 之间的依赖信息 被提供给底层系统调度程序 —— 可能是  hardware command processor 或 CPU 线程池等。 当系统共享有限的设备资源时，这提供了最终级别的即时调度、工作平衡和安全抢占点。 异步工作期间所需的内存 is reserved from a pool that is aware of the streaming behavior，并且只会分配并发可执行工作所需要的内存。 这使得深入的工作管道能够提交给设备，而不会过度使用有限的内存资源。 因为在典型推理中使用的大部分内存要么是常量，要么是临时值，这种方法将加载程序的静态内存消耗减少到仅几千字节 ，常量所需的内存通常可以映射到 discardable 内存中。 这允许加载多个程序——即使在跨调用保留状态时——然后可以安排交错执行，同时仅消耗任何单个调用使用的峰值内存。

### 3.5. Buffer allocation and permission control
IREE 运行时使用设备提供的内存分配器来处理 device buffer configuration and preparation，并使用系统分配器来处理 host-side VM buffers。 分配器可以显式设置 host (VM) 和 device 之间缓冲区的可见性，同时使用标准内存分配库进行正确对齐。 内存块可以在一侧（主机或设备）分配， 同时可以从另一端有选择地访问。 例如，ML 程序的输入缓冲区可以在主机上分配，这样它就可以用正确的输入值启动，同时设置为对具有读取访问权限的设备可见，这样它的内容就可以被工作负载消耗。 **这种可见性和权限控制与支持 enclave 计算的系统相结合，提供了一个安全的 ML 执行环境。**


## 4. Result
通过应用 IREE 的编译流程并选择裸机友好的配置来部署 ML 程序，我们可以减少 IREE module 和 IREE runtime 的大小。

表 1 显示了使用 [IREE snapshot 20211203.686](https://github.com/google/iree/releases/tag/snapshot-20211203.686) 针对不同的 target 和 使用不同编译 mode 编译 MobileNet SSD V2 [7] 的结果。 为了针对 Armv7E-M（i.e. Arm Cortex-M4）目标进行编译，参数被量化。 它显示了打包到 FlatBuffer 文件中的 IREE module 的大小，以及跨不同 LLVM 编译目标的可执行工作负载（嵌入 module FlatBuffer 中）的大小，每个编译目标都有四种 IREE 编译模式：Debug-dylib（动态库 包含调试符号；默认）、Dylib（无调试符号）、Embedded（嵌入式系统友好的动态库）和 Static（静态库）。 后两种模式是 TinyIREE 模式。 结果显示工作负载大小随 LLVM 目标而变化。 特别是，由于针对此类目标进行了 LLVM 优化，嵌入式系统目标（例如 Armv7E-M）的工作负载大小要小得多。

<div class="autocb" style="text-align:center;"><img src="./mlir-iree.assets\autocb_5.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

图 3 描绘了不同 target 和 mode 之间的工作负载大小比较，以及 TinyIREE 模式对嵌入式系统的优化。 注意对于静态库模式，ML 工作负载成为一个可以在链接时优化的库，因此其最终大小取决于用户应用程序如何集成工作负载库。 例如，当从静态库切换到使用 EmitC 的虚拟机控制时，可以删除字节码解释器，这在为 Armv7E-M 编译时节省了大约 15 kB。

在执行方面，IREE 运行时还提供了更高效的内存使用。 表 2 显示了 x86-64 TensorFlow Lite (TFLite) benchmark tool [8] 和使用 dynamic embedded library mode 以及 synchronous HAL driver 构建的 IREE 应用程序之间的峰值内存使用比较。 与 TFLite 解释器相比， IREE 可以使用更小的运行时库来部署相同的工作负载。 请注意，TFLM 可以提供类似大小的宿主库，但它只支持 算字库 的一个子集。 因此，并非所有 TFLite artifacts 都与 TFLM 兼容。 例如，the pre-2.5.0-rc0 TFLM可以部署 表 2 中所示的相同 MobileNet V2 SSD 模型，具有 4.91 MB 的峰值内存使用和 80.02 kB x86-64 host lib，但 TFLM 此后放弃了对 unsigned operators 的支持，因此无法再支持这个模型。

<div class="autocb" style="text-align:center;"><img src="./mlir-iree.assets\autocb_6.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>