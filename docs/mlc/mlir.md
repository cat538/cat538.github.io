# MLIR: Scaling Compiler Infrastructure for Domain Specific Computation

## 0. 摘要
本文提出了MLIR，这是一种**构建可重用、可扩展编译器基础结构**的新方法。MLIR旨在解决软件碎片化，改进异构硬件的编译过程，大大**降低了构建领域特定编译器的成本**，并有助于**将现有的编译器连接在一起**。MLIR还有助于在不同抽象级别、不同跨应用程序域、不同硬件目标和执行环境，改善代码生成器、翻译器和优化器的设计和实现。本文的贡献包括（1）讨论MLIR作为本文研究成果可能的扩展和进化，并指出这个新设计方法在设计、语义、优化规范、系统和工程等方面带来的挑战和机遇。（2）评估MLIR作为可减少构建编译器成本的通用架构，通过描述各种用例，显示本文研究成果在未来编程语言、编译器、执行环境和计算机体系结构方面的研究和教学机会。本文还介绍了MLIR设计基本原理、结构和语义。

## 1. Intro

编译器设计是一个成熟的领域，包括许多广为人知的算法，可用于代码生成、静态分析、程序转换等。编译器设计领域已发展出许多成熟技术平台，这些平台现在已经在整个编译器社区大规模应用，包括LLVM [25]、JVM [26] 等系统。这些流行系统的一个共同特征是它们的“one size fits all”方法，即系统接口只有一套抽象，例如LLVM IR 可被视为 C with vectors，而 JVM 提供了“有 gc 的面向对象类型系统（object-oriented type system with a garbage collector）”抽象。这种“one size fits all”的方法非常有价值，因为从源语言（C/C++和Java）到这些抽象领域的映射非常直接。

同时应该指出，许多问题在较高或较低级别的抽象层级建模会更好。例如在LLVM IR上，对 C++ 代码进行源代码级分析非常困难。我们注意到，许多语言（例如Swift，Rust，Julia，Fortran）都开发了自己的 IR，以解决这些语言领域特定的问题，例如语言/库相关的优化、流敏感（ flow-sensitive）类型检查（例如线性类型）和改善降级（lowering）流程的实现。类似地，机器学习系统通常将“ML图”(ML graph)用作领域特定的抽象。

尽管领域特定IR的开发是一项已经被充分研究的技术，但其工程和实现成本仍然很高。对于这些系统的实现者而言，有时候基础结构的质量不一定是优先考虑的因素。这可能导致编译器系统的实现质量降低，这包括一些用户可见的问题，例如编译时间慢、错误的实现、诊断质量欠佳、优化代码的调试体验差等等。

MLIR项目的目的就是要应对这些编程语言设计和实现方面的挑战，采用的方法是通过非常简便地定义和引入新的抽象级别，并提供“现成的（in the box）”基础架构来解决常见的编译器工程问题。 MLIR的做法是：（1）标准化基于静态单赋值（SSA）的IR数据结构，（2）提供用于定义IR dialect的声明系统，（3）提供广泛的通用基础结构（包括文档、解析和打印逻辑、位置跟踪、多线程编译支持、pass管理等）。

本文探讨了MLIR系统的各个设计要点，将我们的经验应用于不同的问题，并讨论了这项工作可能对编程语言设计和教学产生的影响。

### 1.1. 主要贡献
尽管 MLIR 系统的大部分基于已知的编译器算法，但是它提供许多有趣的研究的机会，这依然是足够创新的。这篇文章的贡献包括

- 对一个在工业界和研究中有广泛应用的编译器基础架构的描述
- 一种构建可扩展的模块化的编译器系统的新方法
- 探索了几个 MLIR 在多个领域的应用，并对这些系统的共性进行说明
- 分享基于MLIR 基础架构构建系统的经验

### 1.2. Where did MLIR come from?
MLIR 起源于我们意识到：机器学习框架由许多不同的 编译器、图技术 和 运行时系统 组成（参见 图1 ）， 但他们并没有共享公共的基础架构和 design point， 也并不是每一个都遵循编译器设计的最佳实践。这在很多用户可见的方面十分明显，包括糟糕的错误信息，边界条件下的错误，难以预测的性能以及将当前 stack 泛化到新硬件的难度。

<div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_7.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>


我们很快意识到，整个编译器行业都存在一个类似的问题，那就是，诸如 LLVM 之类的现有编译系统在跨多语言实现的统一和集成方面非常成功，但是现代高级语言通常最终会构建自己的高级IR， 并重复发明许多相同的更高层抽象技术（请参见图2）。 同时，在 LLVM 社区经常出现一些争论， 比如， 如何最好地表示并行结构， 如何共享常见的前端 lowering 基础架构实现（例如，用于C调用约定或诸如OpenMP之类的跨语言功能），但都没有得出令人满意的解决方案。

<div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_0.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

面对这些挑战，我们认为我们无法承担改进 N 个编译器的工作量，因此要构建一个更通用的解决方案。我们可以花精力开发一套高质量的基础架构，这会让多个领域受益，会让我们能够逐步升级现有系统，让我们能够更轻松地解决眼下紧迫的问题，例如专用加速器的异构编译。

现在，我们在构建和部署基 于MLIR 的系统方面积累了大量经验，可以回顾一下 MLIR 基础架构的原理和设计，并讨论为什么要朝这个方向发展。

## 8. 结论与展望
文章介绍了**MLIR——一种用作构造编译器的灵活且可扩展的基础结构**。本文描述了 MLIR 的具体设计，展示了其在一系列领域中的适用性，并描述了许多原创研究和工程意义。

展望未来，我们期待看到一有的编译器社区(如 clang )和领域专家能够从这种高级的语言相关的 IR 中受益。期待看到由这个基础设施催生或者加速的全新的研究领域。

**未来方向**： 

有几个MLIR 未来发展的方向。

1. 在机器学习和 HPC 领域，包括**从具有 symbolic shapes 的秩多态（rank-polymorphic）规范中推断出高效的Op实现**。还包括实现更广泛的数据结构（ sparse, graph ）及其相应转换，以及将**符号推理（例如自动微分和算法简化）与更常规的数据流 及 基于控制流优化的结合**。 

2. 除 ML 和 HPC 之外，开发者还可以考虑 MLIR 在其它相关领域的适用性，例如安全编译、安全敏感（ safety-critical ）系统、数据分析和图形处理，关系查询优化等。

回到通用语言领域，一个明显缺失的前端是从 Clang 派生的 C++ 中级表示形式。比如说，"CIL" 在本质上与 Swift 的 SIL 和 Rust 的 MIR 相似，这将有助于优化常见的 C++ idioms ， 这些 idioms 目前需要从 lowered code 中重建（例如，treating `std::vector` as an array rather than pointer manipulation）。支持 gc 的语言、**支持 higher-order and polymorphic type systems with type inference 也是未解决的挑战**。

在 LLVM 中探索并行性和并发构造非常困难，主要是因为所需的修改是侵入性的，且不容易分层（例如，注入元数据并检查所有pass以确保元数据被传播，同时由于抽象级别太低而失去优化机会）。With MLIR, parallel constructs can be first-class operations, using regions and parallel idiom-specific verification. 这将能够在降低到 例如 LLVM 之前支持更高级别的转换，在 LLVM 中可以对已经降低的代码执行常规转换。

除了调试和测试之外，IR的文本形式对于教学也很有用。**通过工具展示高性能编译中优化过程的交互信息有助于新生手了解编译器**。 IR设计是开发新的编译器或优化框架不可或缺的部分，但是许多本科生的编译器课程并未涵盖IR设计。 MLIR为此类课程提供了新的探索方法。

## 2. 设计原则

本节探讨了指导 MLIR 设计的一些需求

1. 尽可能少的内置, 尽可能多的定制(little builtin, everything customizable)

    MLIR 系统基于最少量的基本概念，大部分中间表示都完全可定制。在设计时，应当用少量抽象（types, operations and attributes，这是IR中最常见的）表示其它所有内容，从而可以使抽象更少、更一致，也让这些抽象易于理解、扩展和采用。从广义上讲，可定制性确保编译系统可以适应不断变化的需求，并且更有可能适用于未来的问题。从这个意义上讲，我们应该将 IR 构建为支持其中间语言的语法和语义、具有可重用组件和编程抽象的基础结构。

    **定制化成功的标准是可以表达多种抽象，包括机器学习图、AST、数学抽象（例如多面体）、控制流图（CFG）和指令级IR（例如LLVM IR），而且从抽象到编译系统无需任何硬编码**。

    当然，由于兼容性不佳，可定制性会带来内部碎片的风险。虽然不可能有一种纯粹的技术解决方案来解决生态系统碎片化问题，但系统应鼓励设计可重用抽象，并假定这些抽象会在设计的预料范围之外使用。

2. SSA 和区域(region)

    SSA [15]是编译器IR中广泛使用的表示形式。静态单赋值有许多优点，包括使数据流分析变得简单而稀疏，并且由于静态单赋值与延续传递风格（Continuation-Passing Style, CPS）的关系，编译器开发社区对静态单赋值已有广泛理解，并且已被应用到主要框架中。尽管许多现有的IR使用扁平的（flat）、线性CFG，**但表示更高级别的抽象却推动将 嵌套区域 （nested regions） 作为IR中最重要的概念**。这超越了传统的区域形式，提升了抽象级别（例如，loop trees），加快了编译过程、指令提取或SIMD并行性 [22，21，37]。为了支持异构编译，系统必须支持结构化控制流、并发构造、源语言中的闭包等等。一个具体的挑战就是在嵌套区域之上构造CFG分析和转换。

    !!! warning "疑问"
        Continuation-Passing Style, CPS 是什么？

        nested regions 和 线性IR 在现有 IR 中分别对应的例子？

        结构化控制流/控制流 ？ 参考 https://en.wikipedia.org/wiki/Control_flow

    在此过程中，会牺牲LLVM的归一化（ normalization ），有时甚至牺牲其规范化（ canonicalization ）属性。能够将各种数据和控制结构降级为更小的归一化（normalized）表示集合，这对于控制编译器的复杂性至为重要。具有pre-header、header、latch、body的规范循环（canonical loop）结构是前端语言中各种循环构造的线性化控制流表示的典型情况。MLIR的目的是为用户提供一种选择，即，根据编译流程中pass的编译算法，可以将嵌套循环捕获为嵌套区域或线性化控制流。通过提供这种选择，我们可以脱离LLVM的normalization-only方向，同时保留了在必要时处理更高级别抽象的能力。反过来，采用MLIR的这些方法也产生了如何控制抽象规范化（normalization）的问题，这是下一段的主题。

    !!! warning "疑问"
        normalization 和 canonicalization 分别是什么意思？

    > 【译注：“规范（canonical）”形式和“归一（normal）”形式之间的区别因不同领域而异。通常，规范形式为每个对象指定唯一的表示形式，而归一形式仅指定其形式，而无需唯一。因此，规范化比归一化要求更严格】

3. 渐进式降级（Progressive lowering）

    系统支持渐进式的的lownering，即，以较小的步幅，依次经过多个抽象级别，从较高级别的表示降低到最低级别。多级抽象的需求来自通用编译器基础架构支持的多重平台和编程模型。

    之前的编译器都在自己的 pipeline 中引入了固定级的抽象，如 Open64 WHIRL 表示有五个抽象等级，clang 编译器的翻译过程则包含了从 AST 到 LLVM IR 到 SelectionDAG 到 MachineInstr 到 MCInst 的抽象等级。这些方案的设计都太过于固化，需要更加灵活的设计以支持扩展性。

    这对于程序变换的 pass 的顺序有重要影响。当编译器专家开始实现越来越多的 pass 的时候，不同 pass 之间开始出现复杂交互。不同 pass 整合到一起的最显著的收益是混合(mix)常量传播，值编号( value numbering )和死代码消除。从一个更通用的角度来说，编译 pass 可以被大概分成四种角色： (1)优化变换 (2)使能(enabling)变换 (3)翻译(lowering) (4)清理。 系统应该在**单个操作(single operation)的粒度**允许混合并且匹配四种角色，rather than sequencing passes on the full compilation unit

4. 保留高级语义

    系统应该保留程序分析和性能优化所需的高级语义和计算结构。一旦降低语义，再试图提高语义会很难，并且将这种信息强行塞进一个低层次IR的环境中通常是侵入性的 invasive (例如，在使用调试信息来记录数据结构的情况下，所有 pass 都需要进行验证/重新访问)。相反，系统应该保留计算的结构并逐步 lower 到硬件抽象。只有在确定不再需要保留计算的结构时，再丢弃结构。例如，系统应在整个相关转换过程中保留结构化的控制流，例如循环结构。删除此结构，即转到基于 CFG 的控制流，实质上意味着将不再在此级别上执行任何转换。最新的生产级的并行编译建模表明了这个任务是多么困难 [23, 42]。

    !!! warning "疑问"
        "在使用调试信息来记录数据结构的情况下，所有 pass 都需要进行验证/重新访问"， 这里是什么意思？

    为了允许编译系统的一部分 IR 保留较高层级的抽象，而另一部分被降低IR层级，**在同一 IR 中混合不同级别的抽象和不同概念必然成为系统的关键属性**。比如，自定义加速器的编译器可以在 IR 中复用系统定义的一些高级结构和抽象， IR 同时也可表达加速器特有的基本标量/矢量指令。

5. IR验证

    生态系统的开放性要求有宽泛的验证机制。验证和测试不仅对于检测编译器错误很有用，而且在可扩展的系统中，对验证方法和工具健壮性的需求也在不断提高。验证机制应使得定义简洁和实用，并可以作为正确结果的唯一来源。

    验证机制的长期目标是复制翻译验证[35，29，50，51]和现代编译器测试方法[12]的成功经验。在可扩展的编译器生态系统中，验证和测试都还是有待解决的两个问题。

6. 声明式重写模式（Declarative rewrite patterns）

    定义 IR modifier 应该和定义 abstraction 一样简单；编译器基础设施的好坏取决于它支持的转换。通用转换应可实现为声明式表达的重写规则，并以机器可分析的格式推理出重写的属性，例如复杂性和完成度。重写系统的健全性和效率很高，因此被广泛研究，并已被应用于从类型系统（type systems） 到指令选择的众多编译问题。**MLIR的目标是实现前所未有的可扩展性和渐进降级功能**，可以通过许多途径将程序转换建模为重写系统。这也提出了如何表示重写规则和策略，以及如何构建机器描述的问题。这里的机器描述应能够通过多个抽象级别引导重写策略。系统需要在解决这些问题的同时，保持可扩展性并执行合理、单调和可复制的行为。

7. 源位置跟踪和可追溯性（Source location tracking and traceability）

    操作的来源（包括其原始位置和应用的 pass ）应易于在系统中追溯。这是为了解决在复杂编译系统中常见的缺乏透明性问题，在复杂编译系统中，很难了解最终表示是如何从原始表示中构造出来的完整过程。在编译安全性至关重要的敏感应用程序时，这是一个突出的问题，在这类程序中，跟踪降级和优化步骤是软件认证程序的重要组成部分[43]。**当使用安全代码（例如加密协议，或对隐私敏感的数据进行操作的算法）进行操作时，编译器常会碰到看似冗余或繁琐的计算，这些计算会嵌入未被源程序的功能语义完全捕获的安全性或私有属性，而安全代码可以防止旁路暴露或加强代码以防止网络攻击或故障攻击。优化可能会改变或使此类保护完全失效[56]**；这种缺乏透明性在安全编译中称为WYSINWYX[6]。准确地将高层次信息传播到较低层的一个间接目标就是帮助实现安全且可追溯的编译过程。

## 3. IR 设计细节
本节根据 第2节 中阐述的设计原则，介绍 MLIR 中 IR 的设计细节。

1. <u>**Operations**</u>

    MLIR 的语义单位是 "operation" ，称为 Op 。MLIR 系统中，从 "instruction" 到 "function" 再到 "module" ，一切都是 Op 。 MLIR 没有固定的 Op 集合，允许并鼓励用户自定义扩展Op。编译器 pass 会保守地对待未知Op， MLIR 支持通过特征（traits）、特权操作 hook 和优化接口等方式向 pass 描述 Op 语义，详见 [6.1](#61-reusable-compiler-passes)。

    Op 具有唯一的操作码(opcode)。opcode 字面上是一个字符串，用于标识其 dialect 和操作。 Op 可有零个或多个值作为操作数和结果，并以 SSA 的形式维护操作数和结果。所有值都有一个类型，类似于 LLVM IR 。除了操作码、操作数和结果外，Op还可能具有属性、区域、块参数和位置信息（Attributes, Regions, Block Arguments, and Location Information）， 图3 说明了这一点。
    
    <div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_1.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

    图4 说明了 values and Ops ，`%`标识符表示 (packs of) named values，如果 pack 中有多个值，`:` 后指定其中值的数量（注：如 `%results:2`，表示返回值有2个），而 `#` 表示特定值。在一般的文本表示形式中，操作名是用引号括起来的字符串，后跟括号括起来操作数。

    <div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_2.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

2. <u>**Attributes**</u>

    MLIR attribute 是结构化的**编译时静态信息**，例如整数常量值、字符串 或 常量浮点值列表。 属性是有类型的，每个 Op 实例都有一个开放的键值字典，**从字符串名称到属性值**。 在通用语法中，属性位于 Op 操作数及其类型之间，作为大括号括起来的 逗号分隔的 键值对列表。 例如，图 4 使用属性定义已知为常量仿射形式 (constant affine forms) 的循环边界：{lower_bound = () -> (0), step = 1 : index, upper_bound = #map3} 其中 `lower_bound`、`upper_bound` 和 `step` 是属性名。 `() -> (0)` notation is used for inline affine forms，在这种情况下生成仿射函数，返回常量 0 值。 `#map3` 表示法用于属性别名，它允许预先将属性值与标签相关联，并在需要属性值的任何地方引用该标签。

    与 opcodes 一样，没有固定的属性集。 属性从 Op 语义或与它们相关联的方言（第 3 节）中获取它们的含义。 属性也是可扩展的，允许直接引用外部数据结构，这对于与现有系统集成很有用。 例如，属性可能指的是 ML 系统中（在编译时已知）数据存储的内容。

3. <u>**位置信息**</u>

    MLIR 提供位置信息的紧凑表示，并鼓励在整个系统中处理和传播此信息。 它可用于保留产生 Op 的源程序堆栈跟踪，以生成调试信息。 它标准化了从编译器发出诊断的方式，并被广泛的测试工具使用。

    位置信息也是可扩展的，允许编译器引用现有的位置跟踪系统、高级 AST 节点、LLVM 样式的文件行列地址、DWARF 调试信息或高质量实现所需的任何其他内容。

4. <u>**Regions and blocks**</u>

    Op 的一个实例可能有 a list of attached regions。 区域提供了 MLIR 中嵌套结构的机制：区域包含块列表，块包含操作列表（可能包含区域）。 与 attribute 一样，区域的语义由它们所附加的操作定义，但是 regions 内的 block（如果有多个 block ）形成一个控制流图 (CFG)。 例如，图 4 中的 `affine.for` 操作是一个循环，单块主体作为一个区域附加在（`{ and }`）分隔符之间。 Op 指定跨区域的控制流。 在这个例子中，主体被重复执行直到达到上限。

    每个 region 的主体是 a list of blocks，每个块都以 terminator operation 结束——可能将控制流转移到的 successor blocks 。 每个terminator （例如 "switch", "conditional branch" 或 "unwind" ）定义了自己的语义。 它可能会选择将控制流转移到同一区域中的另一个块，或者将其返回到封闭该区域的 Op。 The graph of successor 定义了一个 CFG，允许在一个区域内使用标准的基于 SSA 的控制流。

    MLIR 没有使用 $\Phi$ 节点，而是使用 SSA [2] 的 functional form， 其中 terminator 将值传递到由 successor 定义的 block arguments 中。 每个 block 都有一个 typed block arguments 列表（可能为空）。 terminator Ops 的语义定义了在控制权转移后 block 参数将采用什么值。 对于该区域的第一个（入口）块，值由包含 Op 的语义定义。例如，`affine.for` 使用入口块参数 `％arg4` 作为循环归纳变量。

5. <u>**Value dominance and visibility**</u>

    Ops 只能使用范围内的值，即根据 SSA 支配、嵌套和封闭操作强加的语义限制可见的值。 如果值遵循标准的 SSA 支配关系，则值在 CFG 中是可见的，在这种关系中，控制保证在到达使用之前通过定义。

    基于区域的可见性是基于简单的区域嵌套定义的：如果 Op 的操作数在当前区域之外，那么它必须在词法上定义在使用区域之上和之外。 这就是允许 affine.for 操作中的 Ops 使用外部作用域中定义的值的原因。

    MLIR 还允许将操作定义为与上面隔离，表明该操作是一个范围障碍——例如 “std.func”Op 定义了一个函数，函数内的操作引用函数外定义的值是无效的。 除了提供有用的语义检查之外，包含独立于上层操作的模块可以由 MLIR 编译器并行处理，因为没有 use-def 链可以跨越隔离屏障。 这对于编译利用多核机器很重要。

6. <u>**Symbols and symbol tables**</u>

    Ops 可以附加一个符号表。 该表是将名称（表示为字符串）与 IR 对象（称为符号）相关联的标准化方法。 IR 没有规定使用什么符号，这取决于 Op 定义。 符号对于不需要遵守 SSA 的命名实体最有用：它们不能在同一个表中重新定义，但可以在定义之前使用。 例如，全局变量、函数或命名模块可以表示为符号。 如果没有这种机制，就不可能定义，例如，在定义中引用自身的递归函数。 如果附有符号表的 Op 具有包含相似 Ops 的关联区域，则可以嵌套符号表。 MLIR 提供了一种机制来引用 Op 中的符号，包括嵌套符号。

7. <u>**Dialects**</u>

    MLIR 使用 Dialects 管理可扩展性，**Dialects 在唯一的命名空间下提供 Ops、属性和类型的逻辑分组**。 方言本身不引入任何新语义，但作为一种逻辑分组机制，可用于提供方言通用操作支持（例如，方言中所有操作的常量折叠行为）。 方言命名空间在 opcode 中显示为以点分隔的前缀，例如，图 4 使用仿射和标准方言。

    将 Ops、类型和属性分离为方言是概念性的，类似于设计一组模块化库。 例如，一种方言可以包含用于对硬件向量进行操作的操作和类型（例如，shuffle、插入/提取元素、掩码），而另一种方言可以包含用于对代数向量进行操作的操作和类型（例如，绝对值、点积等）。 两种方言是否使用相同的矢量类型以及这种类型属于何处是留给 MLIR 用户的设计决定。

    虽然可以将所有操作、类型和属性放在一个方言中，但由于大量同时出现的概念和名称冲突等问题，它很快就会变得难以管理。 尽管每个 Op、类型和属性只属于一种方言，但 MLIR 明确支持方言的混合以实现渐进降低。 来自不同方言的操作可以随时在 IR 的任何级别共存，它们可以使用以不同方言定义的类型等。方言的混合允许更大的重用、可扩展性并提供灵活性，否则将需要开发人员诉诸各种 不可组合的解决方法。

8. <u>**Type system**</u>

    如同LLVM， MLIR 中的每个值都有一个类型，通常在 Op 中、或 block 参数中指定。 类型为 IR 提供编译时语义。 MLIR 中的类型系统是用户可扩展的，并且可以引用现有的外部类型系统（例如 llvm::Type 或 clang::Type）。 MLIR 强制执行严格的类型一致性检查并且不提供类型转换规则。 Ops 使用 trailing function-like syntax 列出它们的输入和结果类型。 在图 4 中，`std.load` 从内存引用和索引类型映射到它加载的值的类型。

    从类型论的角度来看，MLIR 只支持非依赖类型，包括 trivial、parametric、function、sum 和 product 类型。 虽然可以通过在 Curry-Howard 同构的文字解释中将 Ops 与符号和用户定义类型组合来实现依赖类型系统，但此类类型对 IR 来说是不透明的。

9.  <u>**Standard types**</u>

    MLIR 提供一组标准化的常用类型，包括 任意精度整数、 标准浮点 和 简单的通用容器——元组、 多维向量 和 张量。 这些类型只是对方言作者有用的便利，但非必需。

10. <u>**Functions and modules**</u>

    与传统 IR 类似，MLIR 通常结构化为函数和 module 。 然而，这些在 MLIR 中不是独立的概念：它们在内置方言中作为 Ops 实现。

    A module is an Op with a single region containing a single block，并由不转移控制流的 a dummy Op 终止。 一个 module 定义了一个符号，可以被引用。 与任何块一样，它的主体包含一个 Ops 列表，这些 Ops 可能是函数、全局变量、编译器元数据或其他顶级结构。

    A function is an Op with a single region，其参数对应于函数参数。 它定义了一个符号，可以通过名字来引用。 使用函数调用 Op 将控制流转移到函数中。 一旦进入，控制流将遵循该区域中块的 CFG。 "return" 终止符没有successors，终止区域执行，将控制流转移回函数的调用点。 "return" Op 操作数是函数的返回值。

## 4. IR 基础设施

除了 IR 本身，MLIR 还提供用于定义 IR 元素的基础设施，例如 dialects、Ops、pattern rewrites、verification 和 reusable passes。 MLIR 的基础设施方面对于在定义新抽象和使用 MLIR 作为优化工具包时提供可扩展性和易用性至关重要。

### 4.1. Operation 描述

MLIR 使用 TableGen[47] 规范定义操作描述（Operation Descriptions, ODS），以声明的方式定义 Op 的结构及其验证程序组件。 TableGen 是一种在 LLVM 中广泛使用的数据建模工具，目的是帮助定义和维护 domain-specific 信息的记录。选择 TableGen 因为它是一个业界标准选择。 ODS 可以看作是 一种嵌入到 TableGen 的、用于声明 MLIR Op 定义的 EDSL。因此 ODS 语法由 TableGen 规定，但 MLIR 特定的语义由 ODS 规定。 ODS 定义最终会转换为C++代码，这些代码可以与编译系统的其余部分互操作。

> [LLVM-TableGen](https://llvm.org/docs/TableGen/)

MLIR 的 ODS 使用 TableGen 的 `Op` 类对 Op 建模。图5 显示了 Op 的 ODS 定义示例。

- 每个 Op 定义都有一个名称，该名称是一个唯一标识符
- Op 的 trait 列表描述了 Op 属性
- Op 的argument（参数）列表指定 Op 的操作数和属性
- Op 定义中还有一个 result （结果）列表

Op 的参数和结果具有名称和类型约束（例如 float 或 int32 的固定形状张量）。 Op 定义还可以指定人类可读的 Op 描述。当 Op 定义需要比 ODS 更精细的控制时，可以通过 builder、printer、parser、verifier 语句注入其他C++代码。操作 trait 可以是通用的，例如 "has no side-effects"， 也可以是特定于 dialect 或 ODS 的，例如 "has custom exporter" 。 

ODS中的 trait 可以由定义 trait 行为的 C++ 类支持。 MLIR 没有固定的 trait 集合，但是有些 trait 对 ODS 是已知的（例如，"shape result and operand type" 表示当完全捕获给定输入类型时，输出类型的约束），或对优化器是已知的（例如，"has no side-effects"，请参阅 [6.1](#61-reusable-compiler-passes)）。

<div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_3.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

类型约束会检查 arguments/results 类型的属性， 并且可由用户/dialect扩展。 MLIR 基础结构还提供了许多预定义的类型约束，例如 "any type" 、"tensor with element satisfying the given constraint"、 "vector of given rank" 等。 **ODS有限地支持了使用 traits 的限制自动推断操作数的结果类型**。更多信息参见 [4.2](#42-declarative-rewrites)


### 4.2. Declarative rewrites

许多 MLIR 转换涉及 Op 操作，尽管某些转换需要对 IR 进行复杂的修改，但许多其它转换可以表示为对 DAG defined by SSA use-def relations 的简单重写。 MLIR提供了一个 graph 重写框架，并辅以声明性重写规则（Declarative Rewrite Rule, DRR）系统，使模式表达变得简单。

与 ODS 相似，DRR是嵌入到 TableGen 中的 EDSL 。 DRR 表示源和目标 DAG 模式以及约束（包括动态约束[49]）和模式优先级的收益。模式可以捕获和重用 Op 的参数。从概念上讲， DRR 表示在特定约束下 DAG 的等效性。图6 给出了 DRR 模式的示例，该模式将 图5 中定义的 Op 转换为由 compare 和 select 组成的通用低层实现。

<div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_6.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

DRR 被转换为 C++ 代码，可以使用通用图重写框架将其与直接在 C++ 中定义的更复杂的模式混合。通过这项功能， MLIR 可以使常见用例保持简洁，且不会限制框架的通用性。

### 4.3. Pass manager

MLIR pass 管理器以各种粒度组织并处理IR pass序列，保证pass的高效执行。现有编译系统中的 pass 管理通常是按照固定的粒度（例如，模块、函数或循环 pass 管理器）定义的。但在MLIR中，module 和 function 并无特殊，只是具有区域的 Op ， 并且有多种变体。因此，MLIR pass 管理器也不专门针对固定的 Op 集合，而是针对任意嵌套级别的任意 Op 。

**Parallel compilation**:

MLIR 的一个重要需求是利用多核加快编译。pass 管理器支持并发遍历和修改中间表示，这可以通过 Ops 的 "与上方隔离（isolated-from-above）" property 提供的不变量来实现，因为 SSA use-def 链无法跨越这些 op 的区域边界，因此具有这种行为的操作（例如 "std.func" 操作）定义了可以并行处理的区域树。

!!! warning "疑问"
    which is made possible by invariants provided by “isolated-from-above” property of operations, because the SSA use-def chains cannot cross the region boundaries of these ops

    什么意思 ?


这个需求也是 MLIR 不具有整个 module use-def 链的原因（这与 LLVM 相反）。全局对象通过符号表条目进行引用， 而常量则由具有关联属性的操作实现。

### 4.4. Round-trippable textual IR form

MLIR 中的 IR 和 Op 具有文本表示形式，可以完全反映内存中的 IR 表示，这对于调试、理解转换期间的 IR 以及编写测试用例至关重要。 图4 所示的原始 IR 表示冗长且难以理解，因此 MLIR 允许用户为 Op 定义定制的打印和解析格式，这使得示例可以如 图8 所示进行打印和解析，这更容易使用。

两种形式可以完全相互转换，并且可以使用文本形式作为输入和输出，分别测试每个编译器 pass 。 由于没有隐藏状态，因此运行单个 pass 的结果与在完整 pass pipeline 中运行相同 pass 的结果相同。这种方法对用户友好，因为可以手动创建 IR 格式，并可方便跟踪 IR 转换。

<div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_4.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

### 4.5. 文档

Dialect、Op 和 接口 都有从其对应 ODS 描述生成的文档。 除了摘要和更易读懂的描述之外，生成的文档还包括参数和结果类型约束。由于验证代码和文档使用相同的来源，因此文档可以与运行时行为保持同步。

> [MLIR-code docs](https://mlir.llvm.org/docs/)


### 4.6. Verifiers

验证器用于增强 IR 和 Ops 不变式的结构正确性，允许 pass 假定已验证的 IR 不变式是经过检查的，并且还可以用作调试工具。验证过程以 MLIR 总体结构属性检查开始，比如，检查类型必须完全匹配，值仅定义一次且遵守支配规则和可见性，符号名称在符号表中是唯一的，所有块均以终结符Op结尾，等等。之后，应用各个Op和属性的验证器。每个Op可以定义一组检查结构和语义有效性规则。例如，二元Op会检查是否有两个操作数，许多Op只接受特定类型的值，而许多Op需要附加特定的属性或区域。同样，dialect属性只能在特定的Op上被允许使用，或者通过这些属性对其所附加的Op做进一步的限制。例如，dialect属性可以要求Op仅使用dialect中定义的类型，即使Op本身更通用。验证失败被视为invariant violation并中止编译。


## 5. Evaluation: Applications of MLIR

MLIR 是一个旨在泛化多种编译器项目的系统，**因此主要评估指标是表明它正在被各种项目采用和使用**。 我们提供了社区活动的摘要，并更详细地描述了一些用例，以突出 MLIR 的通用性和可扩展性，并展示它如何很好地实现可定制性设计原则。

如今，MLIR 是一个不断发展的开源项目，社区跨越学术界和工业界。 例如，关于在高性能计算 (HPC) 中使用 MLIR 的学术研讨会有来自 16 所大学的个人参加，涉及来自 4 个不同国家的 4 个国家实验室。 MLIR 还得到了 14 家跨国公司的认可，在 LLVM 开发者会议上，超过 100 名行业开发人员参加了关于 MLIR 的圆桌会议。 社区采用和参与是可用性和需求的代理度量。 超过 26 种方言正在公共或私人开发中，不同公司的 7 个项目正在用 MLIR 替换自定义基础设施。 我们认为这表明了对 MLIR 的真正需求，并认可了它的可用性。

### 5.1. TensorFlow graphs

虽然讨论到的其他 IR 对于大多数编译器开发来说都很熟悉，但 MLIR 的一个关键用例是支持机器学习框架的开发。 **它们的内部表示通常基于具有动态执行语义的数据流图** [53]。

TensorFlow [1] 是此类框架的一个示例。 它的表示是一个高级数据流计算，其中节点是可以放置在各种设备上的计算，包括特定的硬件加速器。

MLIR 在 TensorFlow 中用于对这种内部表示进行建模并针对 图1 中显示的用例执行转换：从简单的代数优化到重定向图以在硬件加速器的数据中心集群上并行执行，从降低到适合移动部署的表示到 使用 XLA [57] 等工具生成高效的本机代码。 图 7 说明了 MLIR 中 TensorFlow 图的表示。

<div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_8.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>


### 5.2. Polyhedral code generation
MLIR 的最初动机之一是探索 accelerators 上的多面体代码生成。 仿射方言 ( affine dialect) 是一种简化的多面体表示，旨在实现渐进降低。 虽然对此处设计点的全面探索超出了本文的范围，但我们说明了仿射方言的各个方面以展示 MLIR 的建模能力并将仿射方言与过去的表示进行对比 [17、19、54、55、52] .

1. <u>**相似之处**</u>

    MLIR affine dialect 在结构化多维类型上运行，用于所有内存访问。 在默认情况下，这些结构化类型是单射的：保证不同的索引不会因构造而混淆，这是多面体依赖分析的常见先决条件。

    仿射建模分为两部分。 属性用于在编译时对仿射映射和整数集建模，Ops 用于对代码应用仿射限制。 也就是说，`affine.for` Op 是一个 "for" 循环，其边界表示为函数中需要不变的值的仿射映射。 因此循环具有静态控制流。 类似地， `affine.if` 是一个受仿射整数集限制的条件。 循环和条件的主体是使用 `affine.load` 和 `affine.store` 将索引限制为周围循环迭代器的仿射形式的区域。 这使得精确的仿射依赖分析成为可能，同时避免了从有损的低级表示中推断仿射形式的需要。

2. <u>**不同之处**</u>

    与现有多面体框架的差异很多，可以概括为以下4点：

    1. **丰富的类型(Rich types)**：MLIR 结构化内存引用类型包含一个连接缓冲区索引空间和实际地址空间的布局图。 这种关注点分离使循环和数据转换更好地组合：数据布局的更改不会影响代码，也不会污染依赖性分析。 之前已经探索过这种混合转换 [38]，但并不常见。

    2. **混合抽象(Mix of abstractions)**：MLIR 中的仿射循环体可以用对类型化 SSA 值的操作来表示。 因此，所有传统的编译器分析和转换仍然适用，并且可以与多面体转换交错。 相反，多面体编译器经常将这些细节完全抽象掉，这使得多面体编译器难以操作，例如向量类型。
    
    3. **更小的表示差距(Smaller representation gap)**：多面体模型的关键特征之一是它能够表示类型系统中循环迭代的顺序。 在这个系统中，大量的循环变换直接组合，可以使用简单的数学抽象进行推理[19]。 然而，多面体变换需要提升到一个通常与原始 [20, 10] 截然不同的表示。 此外，从变换的多面体到循环的转换在计算上是困难的 [7]。 基于 MLIR 的表示在较低级别的表示周围维护高级循环结构，从而消除了提升的需要。

    4. **编译速度**是 MLIR 的一个关键目标，如 4.3 节所述，但并不是大多数现有多面体方法的重点。 这些在很大程度上依赖于具有指数复杂度的算法：使用整数线性规划来自动导出循环顺序，以及使用多面体扫描算法将表示转换回循环。 MLIR 采用的方法明确不依赖于多面体扫描，因为在 IR 中保留了循环。

仿射方言的经验表明它对很多代码生成项目很有用，它的开发是 MLIR 设计实用化的重要探索。

### 5.3. Fortran IR (FIR)
由NVIDIA/PGI 主导的LLVM 前端"flang" 正在积极开发中。和swift, rust 及其他语言类似，flang 需要特殊的IR 来支持 高性能的 fortrance codebse 的转换。flang 使用MLIR来支持Fortran特定的优化。这些高级优化——高级循环优化，数组拷贝消除，调用规范，devirtualization——如果只是用LLVM 来实现建模是非常复杂的。

举个例子，FIR 可以对fortran 的虚拟派发表(virtual dispatch table)建模成一等概念。

<div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_9.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

在结构化的IR 中建模语言的高级语义的能力非常强大如，将分派表(dispatcher table)建模成一等公民语序是实现一个鲁棒的 devirtualization pass。尽管这也可用通过定制的编译器 IR 来实现，MLIR 的使用使得 flang 的开发者可以将他们的工程资源聚焦在设计他们的领域 IR 的表示，而不是实现基本的基础架构。

选择 MLIR 也解锁了复用非 Fortron 独有的方言的能力: 语言无关的OpenMP 方言可以在Fortron 和 C 的语言前端之间共享。相似的，基于MLIR通过共享和复用面向 GPU 的方言和 pass 使得采用 OpanACC 的异构 target 变得容易。 MLIR 从设计之处就支持可组合的方言的混合使得这种能力非常直接。

#### 5.4. Domain-specific compilers
上述 MLIR 的应用在大型编译工作流程中，但它也用于为特定的小型工作流程构建领域特定的编译器。 可重用和模块化的基础设施使这些专用路径变得可行并且构建起来相对便宜。

**优化 MLIR 模式重写:**

MLIR 有一个可扩展的模式重写系统，在第 4 节中描述。除了静态声明的模式之外，我们还拥有需要在运行时动态扩展重写模式的应用程序，允许硬件供应商在驱动程序中添加新的降低 . 解决方案是将 MLIR 模式重写表达为 MLIR 方言本身，使我们能够使用 MLIR 基础设施来动态构建和优化高效的有限状态机 (FSM) 匹配器和重写器。 这项工作包括在其他系统中看到的 FSM 优化，例如 LLVM SelectionDAG 和 GlobalISel 指令选择系统。

**Lattice regression compiler:**

格回归编译器格回归 [18] 是一种以快速评估时间和可解释性而闻名的机器学习技术。 编译器的前身是使用 C++ 模板实现的。 这允许使用元编程编写高性能代码，但在端到端模型上表达一般优化并不简单。 这种特殊的格回归系统用于具有数百万用户的应用程序，因此性能改进至关重要。

MLIR 被用作这个专门领域的新编译器的基础，它由专门的搜索方法驱动——有效地导致在编译期间解决机器学习问题。 最终的编译器是通过投入 3 个人月的努力开发的，在生产模型上实现了高达 8 倍的性能提升，同时还提高了编译过程中的透明度。

## 6. Consequences of the MLIR Design

MLIR设计有助于对新语言和编译抽象进行建模，同时有助于重用现有的、通用的相关编译方法。**MLIR对很多问题的有效解决方法是 "add new ops, new types"， 如果可能，将其收集到 "a new dialect" 中**。对于编译器工程而言，这是重大的设计转变，产生了新的机遇，挑战和见解。本节将探讨其中部分观点。

### 6.1. Reusable Compiler Passes

如果一种 IR 有表示多个抽象级别的能力，那么会自然而然地带来一个需求，即，编写可在多个抽象级别工作的 pass 。关于 MLIR 的一个常见问题是，既然 MLIR 具有可扩展的操作和类型系统，那么如何编写编译器 pass ？虽然编译器 pass 可能总是以保守、正确的方式处理未知结构，但MLIR的目标是生成高性能代码，主要有四种方法：

1. **Fundamental operation traits**

    一些 "bread and butter" 编译器 pass（如 死代码消除 和 公共子表达式消除 ）依赖于定义为 Op 特征的简单属性（例如 "has no side effect" 或 "is commutative" ）。ODS 中操作的定义允许操作的开发者指定这些特征，并且 pass 可以使用此信息来保持操作在许多不同抽象域都适用。

    MLIR 的可扩展性体现为某些结构属性，其中包括下述信息：是否知道某个操作是控制流终止符，是否知道某个操作包含的区域是 isolated-from-above 等等。这些信息可用于函数、闭包、模块和其他代码结构的建模和处理。

2. **Privileged operation hooks**

    While certain traits can be modeled with a single bit， 但是其它很多 traits 则需要 C++ 实现，例如常量折叠逻辑。 MLIR 对适用于大量pass的某些 hook 提供了最好的支持。这些 hook 可以被实现为 pre-operation ，也可以在 dialect 对象中实现。后一种方法对支持诸如 TensorFlow ops 的常量折叠之类 pass 很方便，在这种情况下，很容易实现对现有逻辑的委托。

    尽管常量折叠是非常重要的功能， 但更有意思的 hook 是 `getCanonicalizationPatterns` ，这个 hook 允许指定应用于操作的折叠模式。这使得重要的代数简化形式（例如x − x→0，min（x，y，y）→min（x，y）等）具有可扩展性，并可帮助将普通“规范化（Canonicalization）”pass应用到所有dialect 。这些都使得单一的可扩展系统可以包含像 "InstCombine"、 "DAGCombine"、 "PeepholeOptimizer"、 "SILCombine" 这类pass， 以及 LLVM 生态系统（和其它编译器）中的其它特殊用途 pass 。

3. **Optimization interfaces**
    MLIR 的主要目标是可扩展性，不仅在操作和类型方面，而且在转换方面也要有可扩展性。虽然规范化（canonicalization）和常量折叠是关键操作，但仍需要以某些方式对许多标准转换进行参数化设置，才能描述转换的特定属性，才能实现成本模型等。

    问题的解决方案是称为 "优化接口" 的子系统。考虑一下 MLIR 内联 pass ， 我们希望 inliner 可以处理 TensorFlow 图、Flang函数、函数语言的闭包等，但是inliner不知道调用方是什么，甚至不知道被调用方是什么。inliner需要了解的核心特性是：

    - 将操作内联到给定区域是否有效；
    - 如何处理内联后终止于块中间的终止符操作。

    为了了解这些属性，Inliner pass 定义了 图10 中的接口。各个操作和 dialect 可以向 MLIR 注册该接口在操作和 dialect 中的实现，并从通用 Innerer pass 中获益。如果某个操作或 dialect 无法提供接口，则相应的优化 pass 将会保守地对待该操作。这种设计让 dialect 的开发者能快速启动开发并运行 dialect 。随着时间的推移，通过将更多的精力投入接口开发，可以从系统中获得更多收益。

    <div class="autocb" style="text-align:center;"><img src="./mlir.assets\autocb_5.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>


4. **Dialect特定pass**

    最后，定义特定dialect可以定义专用pass，MLIR系统中的这些pass和在其它编译器系统中的pass一样都很有用。比如说，如果想让代码生成器根据特定的机器约束对机器指令进行自定义调度，就可以通过专用pass达到目的。这可当作开发新转换pass的起点，不需要考虑pass的通用性。

### 6.2. Mixing dialects together

MLIR 中一个最关键（也是最难理解）的部分是允许并鼓励将来自不同 dialect 的操作混合在一个程序中。尽管在某些情况下（例如，将主机和加速器计算保存在同一模块中），这样做很容易理解，但最有趣的情况是，在 MLIR 中可以将 dialect 直接混合（因为这样可以实现整个类的重用），这在其它系统中是见不到的。

考虑第 5.2 节中描述的 `affine dialect`。 仿射控制流和仿射映射的定义与仿射区域中包含的操作的语义无关。在我们的案例中，我们将 `affine dialect` 与 `std dialect` 结合起来，以目标无关的形式（如同LLVM IR）表示简单算术，也可以针对内部加速器，将 `affine dialect` 与多个目标相关机器指令 dialect 结合。也有人将 `affine dialect` 与其它问题领域的抽象相结合。

重用通用多面体转换（使用Op接口获取特定转换中操作的语义）的能力是分解编译器基础结构的一种有力方法。另一个例子是，可以在各种源语言IR中使用和重用 `OpenMP dialect` 。

### 6.3. Interoperability

本文的工作涉及与大量现有系统的互操作，例如，protobuff格式的机器学习图、包括LLVM IR在内的编译器IR、各种专有指令集等。任何一种表示形式不可避免都有各种缺陷，虽然这些缺陷在某个现有系统的适用场景下是合理的，但是MLIR的表达能力使MLIR成为一种更好的表示形式。因为importer和exporters的测试难度很大（测试用例通常是二进制格式），因此我们希望确保其复杂性最低。

问题的解决方案是尽可能定义与外部系统直接相对应的dialect，从而能以一种简单且可预测的方式来回转换该格式。一旦将IR导入MLIR格式中，就可以使用MLIR基础结构中所有转换，将导入的IR升级或降级为某种更适合的IR格式，并允许对这些转换pass进行类似于所有其它MLIR pass的测试。

这类dialect的例子很多，包括：a）LLVM dialect，可将LLVM IR映射为MLIR； b）TensorFlow的图表示形式，提出这种表示是为了简化TensorFlow中“切换和合并（switch and merge）”节点相关的分析和转换；c ）函数式控制流运算符。“functional while”和“functional if”在机器学习图中很常见，在这种情况下，将其代码主体作为区域而不是外联（out-of-line）函数更方便。

这种方法对我们来说效果很好，并且MLIR工具对于编写外来二进制文件格式的测试用例也很有用。

### 6.4. Unopinionated design provides new challenges

虽然 MLIR 允许人们定义几乎任意的抽象，但它对应该做什么提供的指导很少：在实践中什么更好或更差？ 我们有一些工程师和研究人员将技术应用于新问题领域的经验，并且已经意识到编译器 IR 设计和抽象设计的 “艺术” 在编译器和语言领域并没有得到很好的理解 —— 很多人 在既定系统的约束下工作，但相对较少的人有机会自己定义抽象。

这是一个挑战，但也是未来研究的另一组机会。 更广泛的 MLIR 社区正在通过这些抽象设计权衡建立大量的专业知识，我们预计随着时间的推移，这将成为一个丰富的研究领域。

### 6.5. Looking forward

MLIR 的设计与我们仍在学习的其他编译器基础架构有很大不同——即使在构建它并将其应用于许多不同的系统之后也是如此。 我们相信还有很多东西有待发现，还需要几年的研究才能完全理解设计要点并建立最佳实践。 例如，out-of-tree 方言的兴起，使用 MLIR 的源语言前端数量的增加，对抽象语法树的可能应用，以及对结构化数据（如 JSON、protocol buffers 等）的应用，这些都还在起步阶段，并且 正在/可能 会发现有趣的新挑战和机遇。

## 7. 相关工作
MLIR 是一个和多个领域重叠的项目。尽管把所有的这些基础设施组合在一起时一个足够新颖的系统，这些组件本身都可以在文献中找到相似点。关于IR设计本身的参考和讨论，请参考第二小节。

MLIR 和LLVM 类似，但是和 **LLVM 致力于标量优化和异构编译**不同， MLIR 旨在对作为一等值和操作的数据结构和算法建模，包括张量代数和算法，图表示和异构编译。MLIR 混搭(mix-and-match)优化将编译 pass 解耦成组件，重新定义lowering，cleanup roles. 这主要归功于模式重写基础设施，捕获成熟的变换并作为小的本地模式的组合并控制每个独立的算子上应用哪个模式重写。对重写逻辑的自动扩展，形式化和验证会是下一步工作的重点。从后端来看，MLIR 的 DDR 和LLVM 的指令选择逻辑和相似，支持多结果模式和规格限制的可扩展操作。

许多编程语言和模型旨在在处理硬件异质特性。最初， OpenMP 基于早期的 StarS 和 OpenACC 灯方案，提供同质的编程模型， 支持将任务和并行区 offload 到加速器。C++ AMP，HCC 和 SyCL 利用传统的Clang/LLVM flow 和现代C++ 提供硬件加速的高级抽象。不幸的是，所有的这些例子很快从高级构造变成依赖已经存在的宿主语言优化来降低抽象的痛点的运行时环境调用。在异构编译过程本身方面的关注就要少得多。并行中间表示扩展了LLVM 的地址部分，但是重点关注异构环境(settings)。截止当前最野心勃勃的项目是 liquid metal, 该项目通过 DLS 的编译流程的协同设计将所管理的对象语义转化为静态、矢量或者可配置的硬件，但是它的 Lime 编译器的大量努力都在于使 rond object 适配 square 硬件。MLIR 提供可扩展的操作集合和类型拥抱异质特性的高级语言嵌入，提供公共的基础设施，用于以最大可能复用不同目标的公共组件的的方式，逐步把这些构造过程(constructs)逐步的 lowering到硬件。

处理异质硬件也是元编程系统的长期目标，尤其是多阶段编程系统。LSM 是一个 SOTA 的框架和运行时代码生成器，该系统提供了核心组件库用于生成有效的代码和和基于scala 的内嵌 DSL。Delite 支持并行和异构执行，这大幅提升了DLS开发者的效率。我们相信这种通过提供嵌入式DSL 的高级抽象并通过通用的元编程构造实现优化的方案是MLIR的重要补充。

更进一步到语言的语法层面，ANTLR 是一系列的 parser 生成器，这些生成器简化了编译器前端的开发。MLIR 当前不包含通用的 parser 生成，不包含AST 构造，也不包含模型功能(modeling functionality). 将MLIR和ANTLR之类的库进行整合可以构建可复用的从用户输入到代码生成的可复用的编译器库。

XLA、GLOW、TVM 等系统更多的在机器学习的场景被提及，这些系统也强调异构的编译目标，但是这些系统从基于图的抽象到加速器的矢量抽象更为具体。所有的这些都可以将MLIR 作为基础设施，使用公共功能的同时，采用自己的代码生成策略。相似地，**Halide 和TVM 的循环嵌套元编程技术，earlier look nest metaprogramming，PolyMega 的全自动流(flow)，Tensor Comprehension，Stripe, Diesel，Tiramisu 和其底层的多面体编译技术可以作为基于MLIR 的编译框架的不同路径共存**。序列化和互操作格式如 ONNX 提供一套公共的算子(OP)集合，不同框架将自己的模型映射到这一套集合上以解决机器学习前端的多样性。ONNX 可以作为 MLIR 的方言(dialect)， 其他算子(OP)可以lower 到该方言(dialect)，或者从该方言(dialect)生成

