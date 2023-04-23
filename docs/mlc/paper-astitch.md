> Ref: 
> 
> - [机器学习访存密集计算编译优化框架AStitch](https://zhuanlan.zhihu.com/p/475898491)
>

## 2. Background
DLC 往往会将上层的 DSL 先翻译为中间层的IR，以方便编译器进行处理。比如，上层的一个LayerNorm算子，会被翻译为中间层的一个子图，该子图包含十多个算子，包括Add、Sub、Mul、Div、Reduce、Broadcast 等类型。在一个好的IR定义中，这些“原子”性质的算子可以组合表达任意用户定义的上层计算。