
# Compiling ONNX Neural Network Models Using MLIR

IBM 于 20年8月 挂在 Arxiv。 onnx-mlir 现在作为 ONNX 的子项目之一： [ONNX-MLIR](https://github.com/onnx/onnx-mlir)

## 0. 摘要

本文对 onnx-mlir 编译器的做初步 high-level 报告。 Onnx-mlir 是一个开源编译器，使用 MLIR 基础架构实现。 Onnx-mlir 依赖 MLIR 的方言概念来实现其功能。 我们在这里提出两种新的方言：（1）一种 ONNX 特定方言，它对 ONNX 标准语义进行编码（2）一种基于循环的方言，为所有 ONNX 方言操作提供一个共同的降低点。 每个中间表示分别促进其自身的图级和基于循环优化的特征集。 我们通过提出的表示遵循几个模型来说明我们的方法，并且我们包括一些早期的优化工作和性能结果。

## 1. Intro

首先 diss 了依赖于第三方加速库的方法： 

> 1. 可以重写的模型数量受到库中提供的函数的限制
> 2. 通常情况下，用户需要安装附加包才能使库正常运行
> 3. 它缺乏针对不同问题定制代码的能力

接着提出了 ONNX-MLIR compiler： 使用编译器长期发展过程中开发的许多成熟优化技术，例如针对特定问题定制代码的能力、内存优化和并行化。 选择了开放神经网络交换 (ONNX) [1] 作为表示编译器输入模型的格式。 使用 MLIR [5] 编写——这是一种用于多级中间表示的现代开源编译器基础架构。 尽管它仍处于开发阶段，但它已经可以将一些流行的模型（如 MNIST 和 ResNet50）编译为 x86 机器、IBM Power Systems 和 IBM System Z 上的本地代码。

文章从 4 个方面介绍这个编译器：

1. 介绍编译器的总体设计和架构，
2. 引入的两种新方言： `onnx` 方言编码 ONNX 标准语义， `krnl` 方言为所有 ONNX 方言操作提供一个共同的降低点。
3. 优化 passes ，例如图重写、常量传播和内存管理
4. 讨论为不同架构生成代码时遇到的一些问题。

章节2讨论了 ONNX 和 MLIR。 章节3介绍编译器的设计原理和架构， 还在本节中讨论了两种新方言，即 `onnx` 和 `krnl` ，以及一些优化pass。 章节4，展示 IBM Power Systems 上 MNIST 和 ResNet50 模型的一些初步实验结果。 章节5总结论文并讨论了未来的工作。

## 5. Conclusion
onnx-mlir 开源编译器用于将 ONNX 模型编译成 native code。 MLIR 被用作构建编译器的基础设施，并引入了 onnx 和 krnl 两种方言。值得注意的是，由于使用 MLIR 基础设施，新的优化可以很容易地集成到 onnx-mlir 中。 未来将添加更多优化，例如多面体优化、循环融合、SIMD 优化，并为加速器代码生成添加支持。

## 2. Background
### 2.1. ONNX
ONNX 定义了一个可扩展的计算图模型、算子和标准数据类型，为不同的框架提供了一个通用的 IR。 有两种 ONNX 变体：
1. the neural-network-only ONNX 仅将张量识别为输入和输出类型
2. the classic machine learning ONNX-ML 也识别序列和映射。 ONNX-ML 使用不基于神经网络的机器学习算法扩展了 ONNX 算子集。 

在本文中，我们关注仅神经网络的 ONNX 变体，并将其简称为 ONNX。

### 2.2. MLIR
MLIR [5] 是一种可重用和可扩展的现代编译器基础架构。 它通过 facilitating the design and implementation of 不同抽象级别的代码生成器、翻译器和优化器，降低了构建特定领域编译器的成本。在本节中，我们简要回顾了用于构建我们的编译器的 MLIR 中的一些特性。

与 LLVM 类似，MLIR 中的 IR 是基于静态单一分配 (SSA) 的 三地址码，其中值在使用前定义，并且具有由它们的支配关系定义的范围。 操作可能会产生零个或多个结果，并且每个操作都是一个不同的 SSA 值，具有由类型系统定义的自己的类型。 MLIR 中的类型系统是开放的，可以定义特定于应用程序的类型。 有许多基本类型，例如 `integers`，以及用于表示张量和内存缓冲区的聚合类型，例如 `Tensor` 和 `MemRef`。 `Tensor` 类型是较为高阶的抽象，没有指向数据的指针，`MemRef` 类型是较低级的表示，指向特定的内存区域。 在 MLIR 中，`Tensor` 和 `MemRef` 类型在语法上表示为 `tensor<D1×D2× ... ×DN×dtype>` 和 `memref<D1×D2× ... ×DN×dtype>`, 其中 D1, D2, ... , DN 是表示 tensor 或 memref 维度的整数，dtype 是张量或 memref 中元素的类型，例如，f32 表示 float32。`<D1×D2× ... ×DN>`称为 `tensor` 或 `memref` 的形状。 `Tensor` 和 `MemRef` 类型在形状未知时可以被标记为 unranked 。 在 MLIR 中，unranked Tensor 和 MemRef 类型在语法上分别表示为 `tensor<∗×dtype>` 和 `memref<∗×dtype>`。

MLIR 的operation 提供了对于 嵌套区域(nested regions) 的一流支持， 这对于表示模型中的控制流比较友好。

使用 MLIR 二次开发，需要自定义 dialects and optimization passes 。 MLIR 中有现成可用的方言，例如 llvm、std、scf 和 affine。 llvm方言是一种低级方言。 它将 LLVM IR 类型和指令包装到 MLIR 类型和操作中。 std 方言包括标准操作，如 load、store、addi、addf、absf 和 call。 scf 方言定义了控制流操作，例如 for 和 if。 affine 方言为 affine 操作和分析提供了抽象。

## 3. Compiling ONNX Models
本节介绍 onnx-mlir 。 首先讨论它的整体架构。 然后介绍两种新方言 onnx 和 krnl。 最后，介绍用于执行优化的 MLIR passes。

### 3.1. Overview
图 2 显示了 onnx-mlir 的整体架构。 输入是一个 ONNX 模型，输出是一个包含编译代码的库。 输出库包含一个名为 "_dyn_entry_point_main_graph" 的入口函数， 其输入和输出分别类似于 ONNX 模型的输入和输出。 为了使用输出库进行推理，用户编写程序通过将输入传递给函数来调用入口函数并获得结果。

<div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_10.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

onnx-mlir 中有五种主要方言： onnx、krnl、affine、std 和 llvm，分为四个抽象级别。
1. 第一个抽象级别是 ONNX 操作的高级表示。 由 onnx 和 std 方言中的 operation 组成，其中 onnx 方言是通过 importer 自动生成的。 
2. 第二个抽象级别包括 krnl、affine 和 std 方言。 krnl dialect 提供了一种适用于循环优化的表示，能够轻松地进行 tiling、 skew、 和 permutation 等变换。 它充当中间方言，有效地将 onnx 方言降级为更低级方言（例如，affine、std 和 llvm）。 
3. 第三个抽象级别包括 affine 和 std 方言，其中可以自由应用 MLIR 中现有的优化 pass。 
4. 第四个抽象级别仅包括真正生成 bitcode 的 llvm 方言。

MLIR passes 用于将一种方言转换为另一种方言，以及针对特定方言进行优化。 onnx 方言通过 pass `--convert-onnx-to-krnl` 转换为 krnl 方言。 然后通过 pass `--convert-krnl-to-affine` 将 krnl 方言（except some of its operations）转换为 affine 和 std 方言。 krnl dialect 中的剩余操作和 affine 和 std dialect 中的操作通过 pass `--convert-krnl-to-llvm` 直接转换为 llvm 中的指令。 图 2 的右侧显示了可以在每个抽象级别执行的优化过程(我们在这里只列举了重要的优化)。

在详细讨论方言和优化过程之前，我们给出一个简短的运行示例，并在 onnx-mlir 中介绍方言。 此示例是 ONNX 中的一个测试用例模型，它执行逐元素加法。 图 3 显示了测试用例的这个 ONNX 模型。 add 操作接受两个 `<3x4x5xf32>` 类型的 tensor 并返回相同类型的结果，即 `sum`。 清单 3、4 和 5 分别显示了在 onnx、krnl、affine 方言中的程序。 限于篇幅省略了llvm中的程序。

<div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_11.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

<div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_12.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>


在 onnx 方言中，操作的表示类似于它们在 ONNX 中的描述。 ONNX模型转化为函数 `main_graph` 。 为了生成接收用户输入的入口点函数，我们在 onnx 方言中创建了一个辅助操作，即 `onnx.EntryPoint`，它将元数据保存在操作的属性中，例如要调用的函数名称和输入的数量以及输出。

在 krnl 方言中，`onnx.Add` 操作被翻译成基于循环的计算，由 krnl 方言中的操作表示，其中标量计算由 affine 和 std 方言中的原始操作表示。 我们可以将循环优化（例如 tiling、 skew 或 tranpose）应用于基于循环的计算。 **在此级别，我们为输出张量分配内存，并可以执行内存管理**。

在 affine 方言中，krnl 方言中优化的基于循环的计算被转换为 `affine.for` 循环。 在这一层，我们在 krnl 中还有一个操作，即 `krnl.entry_point`。 这样的操作与主计算无关，会直接转为 llvm。 在降低为 llvm 方言中的指令之前， affine 方言中的操作将转换为 std 和 scf 方言中的操作。

### 3.2. onnx dialect
onnx 方言是 onnx-mlir 中的第一个抽象级别。 我们编写了一个 python 脚本来自动将 ONNX 操作导入到 MLIR 中基于 TableGen 的操作定义中。 这些导入的操作被组织到 onnx 方言中。 多亏了 tablegen，onnx 方言中的操作定义与 ONNX 中的操作描述非常相似，我们能够将所有必要的信息（例如输入、输出、属性和描述）以人类可读的文本形式定义表示到一个基于 tablegen 的表中。

我们还在 onnx 方言中创建了一个新操作，即 `onnx.EntryPoint` 以保存与 ONNX 模型中的 dynamic list of inputs 相关的信息。 这个操作会被降级生成生成库的入口函数 `dyn_entry_point_main_graph`。

清单 6 显示了通过 onnx-mlir 导入 relu 操作的定义。 所有输入和输出都将作为 MLIR 中的张量导入。 importer 自动推断输入、属性和输出的元素类型。 然而，张量的形状将通过 `--shape-inference-pass` 推断出来， 这是 LeakyRelu 操作（第 2 行）中的一个 trait 。 MLIR 从其 Tablegen-based definition  中为 operation 生成 C++ 类定义。 如果用户想在类中定义自定义声明，可以通过"ex-traClassDeclaration" 字段（第 7 行）来完成。

### 3.3. krnl dialect
神经网络工作负载中的计算内核具有局部结构简单性，其中循环嵌套通常很简单，例如，超矩形和语句带有非常简单的算术语义。 这种特性非常适合在多面体模型中表示以进行优化 [8]。 krnl 方言旨在于单一表示中同时托管 循环优化 和 标量语义优化 。 它有望提供可解释性，不仅多面体表示可读，而且还使程序语义（或执行什么）和程序调度（如何以及何时执行）独立。 换句话说，**我们的目标不仅是优化程序，还优化各个调度的组合，这是其他现有系统通常缺乏的功能。**

!!! warning "疑问"
    Halide 和 TVM 中的计算调度分离？


下面是一个在 krnl 中定义嵌套循环的例子：

```c++
%ii, %jj = krnl.define_loops 2
krnl.iterate ( %ii , %jj ) with ( %ii -> %i = 0 to 10 , %jj -> %j = 0 to 10) {
    %foo = std.addi %i , %j : index
}
```

其中 `krnl.define_loops` 定义了两个循环，称为 `ii` 和 `jj`。 这些循环变量将用于表达程序语义和调度。 操作 `krnl.iterate` 在语义上接受两种类型的循环变量：原始循环的变量和预定循环的变量。 在语法糖形式中，我们通过关键字 `with` 将两种类型的循环分开，即 (scheduled loops) `with` (original loops)。 归纳变量，例如上例中的 i 和 j，将使用原始循环定义。 如果没有调度（例如 block, skew 等），调度循环类似于原始循环。

接下来是一个 block(或 称为tiling) 调度的例子：

```c++
%ii = krnl.define_loops 1
%ib, %il = krnl.block %ii 2 : (!krnl.loop) ->(!krnl.loop, !krnl.loop)
krnl.iterate (%ib, %il) with (%ii -> %i = 0 to 10){
  %foo = std.addi %i , %i : index
}
```

Operation `krnl.block`（第 2 行）将循环和整数作为输入，其中整数是tile size（即一个tile块的大小）。 结果是内外层两个循环变量。 这两个循环将用作调度的结果并传递给 `krnl.iterate`（第 3 行）。 值得注意的是，在插入调度时， `krnl.iterate` 中的原始循环和计算保持不变，这正是我们在 krnl 方言中分离程序语义和调度所需要的。

`--convert-krnl-to-affine` 过程自动生成优化的 `affine.for` 循环，如下所示。

```c++
#map0 = affine_map <(d0) -> (d0)>
#map1 = affine_map <(d0) -> (d0 + 2)>
affine.for %arg0 = 0 to 10 step 2 {
  affine.for %arg1 = #map0 (%arg0) to #map1 (%arg0) {
    %0 = addi %arg1 , %arg1 : index
  }
}
```

外层 `affine.for` 迭代步长为 2，即 tile size， 内部 `affine.for` 迭代一个 `tile` 中的元素。

以类似的方式使用其他调度，例如偏斜和排列。 所有计划都是可组合的并且可以嵌套。

### 3.4. Optimization Passes
本节讨论 onnx-mlir 中的一些优化 pass。 由于 MLIR 的表达能力，许多优化可以通过声明重写规则 (DRR) 使用 tablegen 轻松表达

1. **<u>Operation Decomposition</u>**

    ONNX 中许多操作可用其他基本操作表示。 如，向量 x 上的 ReduceL1(即L1范数) 即为对 x 中元素的绝对值求和。 换句话说，我们有

    $$\texttt{ReduceL1} (x) = \texttt{ReduceSum} (\texttt{Abs} (x))$$

    我们只需将 onnx 方言中的操作子集降低到 krnl 方言中，而onnx方言中的其余操作将分解为子集中的操作。

    使用 MLIR 中的 DRR，操作分解简明地写成以下模式：

    ```py
    def ReduceL1Pattern: Pat<
        (ReduceL1Op $x, $axes, $keepdims),
        (ReduceSumOp (AbsOp $x), $axes, $keepdims)
    >;
    ```

    其中 `ReduceL1Op`、`ReduceSumOp` 和 `AbsOp` 分别是 onnx.ReduceL1、onnx.ReduceSum 和 onnx.Abs 的可编程操作形式。 变量 x、axes 和 keepdims 用于保存操作 ReduceL1Op 的输入值。 重写模式 `ReduceL1Pattern` 将会匹配 `ReduceL1Op` 替换为 `ReduceSumOp` 和 `AbsOp`。

2. **<u>Shape Inference</u>**

    `--shape-inference` pass 尝试在 onnx 的程序中推断所有张量的形状。 pass 遍历程序中的所有操作，推断具有 unranked shape 的张量的形状（即 $\texttt{tensor}\langle * \times f32\rangle$），将 ranked shape 传播到 consuming operations ， 并在所有张量具有 ranked shapes 后终止。 对于一个操作，如果其输入具有静态形状，则 `--shape-inference` pass 很可能能够为其输出推断静态形状。 如果输入具有动态形状 (如 $\texttt{tensor} \langle ?\times ?\times ?\times f32\rangle$)， 则输出也将具有动态形状(除了一些在操作属性中指定输出张量形状的操作)。

3. **<u>Graph Rewriting</u>**

    在 MLIR 中，图重写规则可以方便地使用 DRR 表示。例如，下面的规则是在 `MatMulOp` 的结果只被 `AddOp` 消费的条件下，将 onnx.Add 和 onnx.MatMul 融合为一个onnx.Gemm 的操作：

    ```py
    def MulAddToGemmPattern: Pat<
      (AddOp(MatMulOp: $res $m1, $m2), $m3),
      (GemmOp $m1, $m2, $m3),
      [(HasOneUse $res)]
    >;
    ```

    另一个例子是通过将其输入直接传递给它的消费操作来删除 IdentityOp 操作。

    ```py
    def IdentityEliminationPattern: Pat<
      (ONNXIdentityOp $arg),
      (replaceWithValue $arg)
    >;
    ```

4. **<u>Constant propagation</u>**

    常量传播有两个关键思想：（1）如果一个操作的所有输入都是常量，则在编译期间计算其输出并删除该操作，（2）如果存在常量和非常量的混合， normalize the operation。 归一化是为了增加不断传播的可能性，并且在很大程度上取决于操作的数学属性。 以下是 onnx-mlir 中 onnx.Add 操作的一些规范化规则，其属性是 associative and communicative。 Normalization rules 同样使用 DRR 表示。

    (1) $c + x \Rightarrow x + c$

    (2) $(x + c_1) + c_2 \Rightarrow x + (c_1 + c_2)$

    (3) $(x + c) + y \Rightarrow (x + y) + c$

    (4) $x + (y + c) \Rightarrow (x + y) + c$

    (5) $(x + c_1) + (y + c_2) \Rightarrow (x + y) + (c_1 + c_2)$

## 4. Preliminary Experiments
### 4.1. ONNX operation support and testcases
ONNX 为每个操作提供了一组测试用例。 当我们支持 onnx-mlir 中的任何操作时，我们启用它的 ONNX 测试用例来检查操作是否正确运行并产生正确的结果。 在撰写本文时（2020年），onnx-mlir 支持 ONNX 中 139 个操作中的 51 个操作，包括卷积、池化、Gemm 和 LSTM 等。 这些足以编译和执行 MNIST 和 ResNet50 等主要网络。 

> 2023 年 3月 11日， 172 models from ONNX Model Zoo: 120 passed, 19 failed, 33 skipped

- MNIST 是一个小型模型，具有两个卷积运算、一个最大池化运算和一个矩阵乘法，然后是逐元素加法。 编译 MNIST 模型并进行推理的速度相当快，不到一秒即可完成。 在 MNIST 模型中，图重写规则 `MulAddToGemm-Pattern` 。 

- ResNet50 是一个复杂的深度模型，由 50 层卷积和池化等操作组成。 该模型大约有 100MB， 包括学习的权重。 **对于 ResNet50，当前版本的 onnx-mlir 在编译期间没有对模型应用任何优化**。 但是，我们认为编译时间看起来合理，推理时间也没有那么慢。 我们希望一旦我们在不久的将来集成重要的优化，如多面体优化、SIMD 优化和循环融合，推理时间将显着减少。

虽然 onnx-mlir 完全建立在广泛使用的开源软件（如 ONNX 和 MLIR）之上，但我们发现了一个与支持不同系统相关的问题。 我们无法在 IBM System Z (s390-linux) 上的 Linux 上运行 ONNX 模型，因为大端格式在 ONNX 和 MLIR 中没有得到很好的支持。 出现这样的问题有两个原因。 首先，ONNX 中大量的 public 输入数据和模型都是以小端格式存储的。 因此，在大端系统中使用它们之前，必须将它们转换为大端格式。 其次，我们发现 ONNX 模型中的常量值未正确加载到 MLIR 中。 LLVM 在 big-endian 中得到了很好的支持，但 MLIR 却没有。 我们创建了两个补丁来解决这个问题：一个在 ONNX 中，一个在 MLIR 中，它们现在可以在 ONNX 和 MLIR 的主分支上使用。 因此，onnx-mlir 现在支持 Linux on x86 (x86-Linux)、Linux on Power Systems (ppc64le-Linux)、Linux on IBM Z (s390-Linux) 和 Windows。