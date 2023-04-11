# 程序中循环的优化

> **Ref**:
>
> - [知乎-编译器领域的多面体模型（Polyhderal Model)](https://zhuanlan.zhihu.com/p/310142893)
> - https://en.wikipedia.org/wiki/Polytope_model

## Polyhedron Model

> **From Wiki**:
> 
> 嵌套循环程序是可以应用多面体模型（也称为多面体方法）进行优化的典型例子。多面体方法将嵌套循环中的每个循环迭代视为称为多面体的数学对象内的格点，执行仿射变换或更一般的非仿射变换，例如在多面体上 tiling，然后将变换后的多面体转换为等效但优化的（取决于targeted optimization goal），通过多面体扫描进行循环嵌套。