# TVM-Relax

> Ref:
>
> - [Relax: Co-Designing High-Level Abstraction Towards TVM Unity](https://discuss.tvm.apache.org/t/relax-co-designing-high-level-abstraction-towards-tvm-unity/12496)\
> - [Relax: TVM 的下一代图层级 IR](https://zhuanlan.zhihu.com/p/523395133)
> - [tlcpack/relax-wiki](https://github.com/tlc-pack/relax/wiki/Relax-Architecture-Overview)


## Dynamic Shape
Relax 引入了一个新的 动态 tensor 类型 `DynTensorTypeNode`， 对比原本TVM中的 静态 tensor类型 `TensorTypeNode`

`include/tvm/ir/tensor_type.h`:

```c++
class TensorTypeNode : public BaseTensorTypeNode {
 public:
  Array<PrimExpr> shape;
  DataType dtype;
  TVM_DLL PrimExpr Size() const; // product of shape
  static constexpr const char* _type_key = "relay.TensorType";
  TVM_DECLARE_FINAL_OBJECT_INFO(TensorTypeNode, BaseTensorTypeNode);
};
```

`include/tvm/relax/type.h`:

```c++
class DynTensorTypeNode : public BaseTensorTypeNode {
 public:
  int ndim; // number of dim; -1 denote tensor with unknwon number of dim
  DataType dtype; // content data type, use void to denote the dtype is unknown
  inline bool IsUnknownNdim() const { return ndim == kUnknownNDim; }
  inline bool IsUnknownDtype() const { return dtype.is_void(); }
  static constexpr const char* _type_key = "relax.DynTensorType";
  TVM_DECLARE_FINAL_OBJECT_INFO(DynTensorTypeNode, BaseTensorTypeNode);
};
```

> Dynamic shape 是 TVM-Relay 的一大短板，核心原因是 relay 把Tensor的shape作为type的信息之一存进去了（即 Tensor[(m, n)]和Tensor[(m, 4)]是不同的type，且不可分析。relax引入了一个新的type叫DynTensor，其中包含的信息是dtype和shape的纬度，但shape本身的表达式是独立存储的。也就是Tensor[(m, n)]和Tensor[(_, _)]都是同一个type， 但是Tensor[(_, _)]和Tensor[(_, _, _)]是不同类型。这样从原生上支持了symbolic shape。
