# THE CORA TENSOR COMPILER: COMPILATION FOR RAGGED TENSORS WITH MINIMAL PADDING

# 0. Abstract
DL 的输入数据的形状和大小通常会有所不同。 在许多情况下，此类数据可以使用形状不均匀的张量或参差不齐的张量来表示。 由于对参差不齐的张量的有效执行的支持有限且不可移植，当前的深度学习框架通常使用填充和掩码等技术使数据形状统一，然后将计算卸载到优化内核以实现密集张量代数。 然而，此类技术会导致大量计算浪费，从而导致性能损失。 

根据论文中的实验结果，CoRa编译器在不规则张量上的各种操作，以及变换模型的编码层，都能与手工优化的代码相媲美，甚至超过。CoRa编译器还可以在不同的CPU和GPU平台上进行移植和优化。可以在论文的第六节中看到更多细节。Nvidia GPU 上的 encoder 是 PyTorch 的 1.6倍； 64 核 ARM CPU 上达到了 TensorFlow 中的 multi-head attention module used in transformers的 1.37 倍。

