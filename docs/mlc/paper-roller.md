# ROLLER: Fast and Efficient Tensor Compilation for Deep Learning

[https://github.com/microsoft/nnfusion/tree/osdi22_artifact/artifacts](https://github.com/microsoft/nnfusion/tree/osdi22_artifact/artifacts)

## 0. Abstract
张量编译器在各种加速器（例如 GPU）上，为算子生成高效内核通常需要数小时，这是 DNN 中的计算密集型子任务。 这显著减慢了 DNN 开发周期，并给通用内核库和自定义内核的开发带来沉重负担，尤其是对于新硬件供应商而言。 编译过程缓慢原因是现有的 DLC 制定的搜索空间很大，必须使用 ML 算法才能找到好的解决方案。

在本文中，我们介绍了 ROLLER，它采用不同的基于构造的方法来生成内核。 ROLLER 的核心是 `rTile` ，这是一种新的 tile 抽象，**它封装了与底层加速器的关键特征对齐的张量形状，从而通过限制形状选择来实现高效执行**。 ROLLER 然后采用递归的基于 `rTile` 的构建算法来生成基于 `rTile` 的程序（rProgram），其性能可以通过 micro performance model 进行有效评估，而无需在真实设备中进行评估。 因此，ROLLER 可以在几秒钟内生成高效的内核，其性能可与 GPU 等流行加速器上的最先进解决方案相媲美，同时在 IPU 等较新的加速器上提供更好的内核。

## 8. Conclusion
ROLLER 对 DLC 采用了一种非常规的方法。 ROLLER 不是依靠昂贵的 ML 算法在大型搜索空间中找到好的解决方案，而是使用基于递归构造的算法生成高效 kernel， 该算法利用新的 `rTile` 抽象，具有更少的形状，可与多种硬件功能对齐。 构建的程序可以通过 micro performance model 进行评估，无需每次都在真实设备上运行。 因此，ROLLER 可以在几秒钟内编译出高性能内核，即使在不太成熟的加速器中也是如此。 更重要的是，ROLLER 为新的 AI 硬件供应商提供了一种独特且显着更有效的方法来构建有能力的供应商特定 DNN 库，弥合生态系统与市场领导者之间的差距，从而促进 AI 加速器的创新。

## 1. Intro

## 7. Related work
大多数张量编译器将 DNN 算子视为嵌套的多级循环计算，本质上定义了一个具有组合复杂性的大空间。 TVM [15] 继承了 Halide [27] 的 insight，并将 DNN 算子描述为循环优化调度原语。 后来，AutoTVM [16] 扩展了 TVM 以应用 ML 方法从手动编写的代码模板中搜索最佳配置。 FlexTensor [35] 提出无需手动模板即可自动探索空间。 Ansor [33] 进一步推进了这种自动化。 考虑到分层代码结构，它会生成更大的搜索空间，并采用 进化算法 来寻找高性能内核。 Tiramisu [14]、AKG [32] 和 Tensor Comprehensions [29] 等编译器**将基于多面体的技术应用于循环优化，将循环转换为整数规划问题并使用求解器找到良好的配置**。 所有这些方法都依赖于巨大的搜索空间来提供良好的内核，这**会导致较长的编译/求解时间**。 ROLLER 探索了一种不同的方法来构建与硬件特性一致的 rTiles。

张量处理原语 (TPP) [18] 定义了一组二维张量算子，用于在高维张量上组成复杂的算子，提供有限的表达能力。 相比之下，ROLLER 不限制 tile 形状的维度，可以应用于一般的张量表达式。 OpenAI Triton [28] 是一个编程框架和编译器，用于开发 block-based 的 GPU 内核。

Triton 依靠程序员来决定块大小和块调度，而这是 ROLLER 通过考虑硬件特性和张量形状来解决的关键问题。 MLIR [5] 和 Tensor IR [10] 计划在他们的 IR 中支持块级（即 tile）计算表示。 ROLLER 的 rTile 抽象和 rProgram 构造与这些计划兼容。

图级 DNN 编译器，如 XLA [11]、TVM [15] 和 Rammer [26]，专注于跨算子优化，例如算子融合/协同调度。 ROLLER 的内核生成与这些编译器兼容。 ROLLER 的 rTile 抽象补充了 Rammer [26] 中的 rTask 概念，因为它提供了一种构建 rTask 的有效方法。

最后，一些工作专注于特定于算子的优化。 CUTLASS [7] 是实现矩阵乘法的模板。 提出了一种分析模型 [24] 来仅针对多核 CPU 上的卷积算子找到最佳循环级优化配置。 DREW [30] 提出了一种使用数据压缩 [31] 优化 Winograd 卷积的新方法。 ROLLER 的优化方法适用于各种设备上的 DNN 算子。