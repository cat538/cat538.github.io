# THE CORA TENSOR COMPILER: COMPILATION FOR RAGGED TENSORS WITH MINIMAL PADDING

# 0. Abstract
DL 的输入数据的形状和大小通常会有所不同。 在许多情况下，此类数据可以使用形状不均匀的张量或参差不齐的张量来表示。 对参差不齐的张量的有效执行的支持有限且不可移植，当前的深度学习框架通常使用填充和掩码等技术使数据形状统一，然后将计算交给优化内核以实现密集张量代数算子。 然而，此类技术会导致大量计算浪费，从而导致性能损失。 

CoRa编译器在不规则张量上的各种操作，以及变换模型的编码层，都能与手工优化的代码相媲美，甚至超过。CoRa编译器还可以在不同的CPU和GPU平台上进行移植和优化。可以在论文的第六节中看到更多细节。Nvidia GPU 上的 encoder 是 PyTorch 的 1.6倍； 64 核 ARM CPU 上达到了 TensorFlow 中的 multi-head attention module used in transformers的 1.37 倍。

## 1. Intro
我们使用 DL 处理的数据大小通常会有所不同。 图像可以有不同的分辨率，文本句子和文档可以有不同的长度，音频可以有不同的持续时间。 因此，使用相同的模型进一步处理显示形状或形状动态变化的此类数据（Shen 等人，2020 年），作为同一小批量的一部分非常重要。 图1 显示了对此类数据的逐元素操作示例，其中张量 A 的内部维度的切片具有可变大小。 这样的张量和算子分别称为 ragged tensors and ragged operators 。 注意**动态形状如何转化为循环 L2 的变量边界，循环 L2 在可变大小的张量切片上迭代**。

<div class="autocb" style="text-align:center;"><img src="./paper-cora.assets\autocb_0.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

过去的工作开发了手动优化的内核来加速一些重要的 ragged 程序，例如可变维度的批量矩阵乘法、三角矩阵乘法 和 广泛使用的 transformer 模型。 然而，这种手动优化的内核需要大量的开发工作，因此仅适用于少数算子。 此外，它们不能跨不同的硬件移植，这是由于 DL 硬件的快速创新而产生的问题。

虽然一些 DL 框架最近开始为 ragged 算子提供支持（TensorFlow 团队，2022 年；PyTorch 团队，2022 年），但它非常有限（TensorFlow 社区；PyTorch 社区；PyTorch 团队），如第 8 节所述。 因此，框架通常依赖于供应商库中实现的高效密集张量代数内核，例如 cuDNN和 oneDNN (Intel) 或由 TVM（Chen 等人，2018a）等张量编译器生成。 因此，填充（如图 1 右上角所示）和 masking 通常用于消除参差不齐的张量中的形状动态，并允许使用供应商库或密集张量编译器 (Hug-gingFace, 2020)。

!!! note "note"
    Masking involves setting some tensor elements to a special value so that these elements are ignored in computations.


然而，填充和掩码会导致计算浪费，因为填充或掩码数据点在执行后会被丢弃。 图2 绘制了包含和不包含填充的 transformer 模型 的 encoder 层的正向传递所涉及的相对计算量（以 FLOPs 进行分析计算）。 我们看到填充导致层的计算需求显着增加，尤其是在批量较大的情况下，增加了大模型的本就很大的计算量。

因此，当前用于高效 ragged 算子执行的解决方案并不令人满意。 因此，我们提出了一种基于编译器的解决方案，该解决方案可以为参差不齐的算子轻松且更便携地生成符合要求的代码。 稀疏 和 密集 张量编译器已得到充分研究，由于以下挑战，将这些技术应用于参差不齐的张量并不简单：

