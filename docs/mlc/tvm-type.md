# TVM-type

本文 **基于 tlc-pack/relax dc7072efe290d7e8c69d8e216311510981fc82e1**

> Ref:
>
> - [深度学习编译器 TVM 代码串讲-知乎](https://zhuanlan.zhihu.com/p/446976730)
> - [TVM编译流程与中间表示分析-知乎](https://zhuanlan.zhihu.com/p/596526031)
> 

## 1. Object

对于类似 IR 这样的数据结构，天然就对其有 serialize/format/reflection 的需求。在 TVM 中，额外还有 python binding/hash 等需求，于是 TVM 要求**所有这样的数据结构继承自 `Object` 基类**，并注册其内部所有成员

这种方式避免了为新增的 Class 额外单独实现 serialize/format/reflection/python binding/hash 等功能。

`include/tvm/runtime/object.h `中定义了 `Object` 和 `ObjectRef` 两个类型， `ObjectRef` 可以视为 `shared_ptr<Object>`

删除部分函数和成员，精简后的定义如下：

```c++
class TVM_DLL Object {
 public:
  typedef void (*FDeleter)(Object* self);
  uint32_t type_index() const { return type_index_; }
  static uint32_t RuntimeTypeIndex() { return TypeIndex::kRoot; }
  static uint32_t _GetOrAllocRuntimeTypeIndex() { return TypeIndex::kRoot; }

  static constexpr const char* _type_key = "runtime.Object";
  static constexpr bool _type_final = false;
  static constexpr uint32_t _type_child_slots = 0;
  static constexpr uint32_t _type_index = TypeIndex::kDynamic;

  static constexpr bool _type_has_method_visit_attrs = true;
  static constexpr bool _type_has_method_sequal_reduce = false;
  static constexpr bool _type_has_method_shash_reduce = false;
  // Override the copy and assign constructors to do nothing.
  // This is to make sure only contents, but not deleter and ref_counter
  // are copied when a child class copies itself.
  Object() {}
  Object(const Object& other) {}
  Object(Object&& other) {}
  Object& operator=(const Object& other) { return *this; }
  Object& operator=(Object&& other) { return *this; }

 protected:
  uint32_t type_index_{0};
  RefCounterType ref_counter_{0};
  FDeleter deleter_{nullptr};
};
```

一个自定义的类型需要继承 `Object` ， 并且往往需要考虑重写 `Object` 中的一些成员变量，其中一部分如下：

- `_type_key` 表示该类型的全局唯一字符串标识符；
- `_type_index` 用于设置类的全局唯一 `uint32_t` 标识符；在 `TypeIndex` 结构体中定义了一系列枚举，values包括 `kRoot = 0`, `kRuntimeModule = 1`， `kRuntimeNDArray = 2`， `kDynamic = kStaticIndexEnd` 等； `Object` 的值为 `kRoot`； 如果一个类型的 `_type_index` 被设置为 `kDynamic`， 则 TVM 的类型注册系统(接下来会提到)会在运行时为该类型(由该类型的 `_type_key` 标识)分配一个对应的 index。 以两个`Object`的子类做对比：

    `include/tvm/runtime/module.h` 中声明的 `ModuleNode` 类型:
    ```c++
    class TVM_DLL ModuleNode : public Object {
     public:
      // 该类型重写了 `Object` 中的 `_type_index` 为 `TypeIndex::kRuntimeModule`
      // 运行时通过 ModuleNode::RuntimeTypeIndex() 获取类型索引时，会返回 TypeIndex::kRuntimeModule 对应的值
      static constexpr const uint32_t _type_index = TypeIndex::kRuntimeModule; // 注意这里的重写
      static constexpr const char* _type_key = "runtime.Module";
      TVM_DECLARE_FINAL_OBJECT_INFO(ModuleNode, Object);
    };
    ```

    `include/tvm/ir/expr.h` 中声明的 `BaseExprNode` 类型:
    ```c++
    class BaseExprNode : public Object {
     public:
      // 该类型未重写 `Object` 中的 `_type_index`， 因此该类型的 `_type_index` 为 `TypeIndex::kDynamic`
      // TVM_DECLARE_BASE_OBJECT_INFO 宏中生成的方法 `RuntimeTypeIndex` 在第一次被调用时将会触发
      // TVM 的 *类型注册系统* 在运行时为该类型分配一个 类型索引
      // 该类型的 类型索引 应当通过 RuntimeTypeIndex 方法来访问
      //
      // 其它自定义类型同理， 通常设置 `_type_index` 为 `KDynamic`(default) 
      // 由TVM 的 *类型注册系统* 为该类型分配 类型索引
      static constexpr const char* _type_key = "BaseExpr";
      static constexpr const uint32_t _type_child_slots = 62;
      TVM_DECLARE_BASE_OBJECT_INFO(BaseExprNode, Object);
    };
    ```

    TVM 中的 `TypeNode`， `BaseExprNode` 以及这些类型的子类如 `PrimTypeNode`， `BaseTensorTypeNode`， `PrimExprNode`， `RelayExprNode`等大部分类型没有预留 `_type_index`， 因此都由 TVM类型注册系统在运行时管理 其类型名对应的类型索引(为 `_type_key` 分配一个 `_type_index`)

    而 `PackedFuncObj`， `ModuleNode` 等类型有预留 `_type_index`；所有预留的10种类型索引在 `TypeIndex` 结构体中可查

- `_type_child_slots` 表示该类型为子类预留的index个数
- `_type_final` 表示是否没有子类，一般通过 `TVM_DECLARE_FINAL_OBJECT_INFO` 这个宏来设置， 而不是手动重写
- `_type_child_slots_can_overflow` 标识是不是可以超过_type_child_slots定义的数量
- `_type_has_method_sequal_reduce` 默认为 false, 
- `_type_has_method_shash_reduce`  默认为 false, 

并且在 定义过类之后要 在 TVM 的类型系统中使用 `TVM_REGISTER_OBJECT_TYPE` 宏进行注册

在 TVM 的类型系统中注册自定义类型的一个例子：

```c++
// Create a base object
class BaseObj : public Object {
 public:
  int field0;
  // _type_index 默认为 TypeIndex::kDynamic (因为 Object 中即为该值) 这一行可以不写
  static constexpr const uint32_t _type_index = TypeIndex::kDynamic;
  static constexpr const char* _type_key = "test.BaseObj";
  TVM_DECLARE_BASE_OBJECT_INFO(BaseObj, Object);
};

class LeafObj : public BaseObj {
 public:
  // fields
  int child_field0;
  // object properties
  static constexpr const char* _type_key = "test.LeafObj";
  TVM_DECLARE_FINAL_OBJECT_INFO(LeafObj, Object);
};

// The following code should be put into a cc file.
TVM_REGISTER_OBJECT_TYPE(BaseObj);
TVM_REGISTER_OBJECT_TYPE(LeafObj);

// Usage example.
void TestObjects() {
  // create an object
  ObjectRef leaf_ref(make_object<LeafObj>());
  // cast to a specific instance
  const LeafObj* leaf_ptr = leaf_ref.as<LeafObj>();
  ICHECK(leaf_ptr != nullptr);
  // can also cast to the base class.
  ICHECK(leaf_ref.as<BaseObj>() != nullptr);
}
```

TVM 提供了 

1. `TVM_DECLARE_BASE_OBJECT_INFO(TypeName, ParentType)` 
2. `TVM_DECLARE_FINAL_OBJECT_INFO(TypeName, ParentType)` 

两个宏来简化生成重复代码：

其中 `TVM_DECLARE_BASE_OBJECT_INFO` 主要是为一个名称为 `TypeName` 的类生成 `RuntimeTypeIndex()` 和 `_GetOrAllocRuntimeTypeIndex()` —— 重写 父类 `Object` 中对应的这两个 static 方法。

具体展开内容如下：

```c++
static_assert(!ParentType::_type_final, "ParentObj marked as final");                       
static uint32_t RuntimeTypeIndex() {                                                        
  static_assert(TypeName::_type_child_slots == 0 || ParentType::_type_child_slots == 0 ||   
                      TypeName::_type_child_slots < ParentType::_type_child_slots,            
                  "Need to set _type_child_slots when parent specifies it.");                 
  if (TypeName::_type_index != ::tvm::runtime::TypeIndex::kDynamic) {                       
    return TypeName::_type_index;                                                           
  }                                                                                         
  return _GetOrAllocRuntimeTypeIndex();                                                     
}                                                                                           
static uint32_t _GetOrAllocRuntimeTypeIndex() {                                             
  static uint32_t tindex = Object::GetOrAllocRuntimeTypeIndex(                              
    TypeName::_type_key, TypeName::_type_index, ParentType::_GetOrAllocRuntimeTypeIndex(),
    TypeName::_type_child_slots, TypeName::_type_child_slots_can_overflow);               
  return tindex;                                                                            
}
```

- `_GetOrAllocRuntimeTypeIndex` 简单地调用父类 `Object::GetOrAllocRuntimeTypeIndex` 方法， 该方法又会调用 `TypeContext` 单例的：`TypeContext::Global()->GetOrAllocRuntimeTypeIndex`； 

    TypeContext 类声明和实现都位于 `src/runtime/object.cc`中， 其负责管理分配， 验证 TVM 中的类型， 其通过`Global`暴露全局唯一单例，其 `GetOrAllocRuntimeTypeIndex` 方法负责为一个类型 分配一个 index， 并建立该类型 `_type_key` 与 index 之间的双向索引。

    TypeContext 使用一个 `std::vector<TypeInfo> type_table_;` 存储 TVM 中所有注册过的类型信息， 一个类型的index即 其类型信息在 该表中的索引下标； 一个类型对应的 `TypeInfo` 包括了该类型的 name， index， parent_index， num_slots 等信息

    TypeContext 的 `GetOrAllocRuntimeTypeIndex` 方法在被调用时，检查表中对应索引项是否已经初始化，如果没有初始化（说明该类型尚未注册到该Context中），则构造相应项

- `RuntimeTypeIndex` 是一个类型向外部暴露信息的接口， 外部通过这个函数来获得类型索引等相关信息。一个应用实例：在`include/tvm/runtime/object.h` 中的 `IsInstance` 功能为 Check if the object is an instance of `TargetType`， 其中使用 `RuntimeTypeIndex` 进行加速检查：

    ```c++
    template <typename TargetType>
    inline bool Object::IsInstance() const {
      if (std::is_same<TargetType, Object>::value) return true;
      if (TargetType::_type_final) {
          return type_index_ == TargetType::RuntimeTypeIndex();
      } else {
          // quick check using type_index
          uint32_t begin = TargetType::RuntimeTypeIndex();
          if (TargetType::_type_child_slots != 0) {
            uint32_t end = begin + TargetType::_type_child_slots;
            if (type_index_ >= begin && type_index_ < end) return true;
          } else {
            if (type_index_ == begin) return true;
          }
          if (!TargetType::_type_child_slots_can_overflow) return false;
          if (type_index_ < TargetType::RuntimeTypeIndex()) return false;
          // slow path using type hierarchy
          return DerivedFrom(TargetType::RuntimeTypeIndex());
      }
    }
    ```

💡总体来说，通过同样的 type_key 可以将 C++ 与 Python 的类型匹配上，**而设计 type_index 则是为了性能考虑**。具体的两边类型匹配可以在 `python/tvm/_ffi/_cython/object.pxi` 查阅

TODO: 额外的 serialize/format/reflection/python binding/hash 等功能则实现在 `node` 目录下，有兴趣可自行查阅。

## 2. PackedFunc
TVM 中另一个与 Object 同样底层的机制称为 FFI (Foreign Function Interface), 这个机制的目标是为了使得任意语言下定义的函数都可以被任意其他语言调用。而这个可以被任意语言调用的函数类型是 `PackedFunc`， 一个示例如下:

```c++
#include <tvm/runtime/packed_func.h>
void MyAdd(TVMArgs args, TVMRetValue* rv) {
  // 自动将参数转换为所需的类型。
  int a = args[0];
  int b = args[1];
  // 自动赋值返回给 rv
  *rv = a + b;
}
// 在C++ 中 调用 PackedFunc
void CallPacked() {
  PackedFunc myadd = PackedFunc(MyAdd);
  // 返回 3
  int c = myadd(1, 2);
}

// 在 C++ 中注册一个全局 PackedFunc
TVM_REGISTER_GLOBAL("myadd")
  .set_body(MyAdd);
```

以上代码块中定义了一个 PackedFunc `MyAdd` 。它有两个参数： `args` 代表输入参数， `rv` 代表返回值。该函数是类型擦除的，这意味着函数签名不限制传入或返回的输入类型。调用 PackedFunc 时，它会将输入参数打包到stack上的 TVMArgs，并通过 TVMRetValue 获取结果。

由于 C++ 中的模板技巧，我们可以像调用普通函数一样来调用 PackedFunc。其类型擦除的性质，使得可以从动态语言（如 Python）中调用 PackedFunc，而无需为每个创建的新类型函数添加额外的胶水代码。以下示例在 C++ 中注册 PackedFunc，并在 Python 中调用。


```py
import tvm

myadd = tvm.get_global_func("myadd")
# 打印 3
print(myadd(1, 2))
```

PackedFunc 的关键在于 TVMArgs 和 TVMRetValue 结构。我们限制了可传递的可能类型列表。以下是常见的类型：

- `int`, `float` and `string`
- `PackedFunc` 本身
- `Module` for compiled modules
- `DLTensor*`(见dlpack) for tensor object exchange
- TVM `Object` to represent any object in IR

由于一个 PackedFunc 可以将另一个 PackedFunc 作为参数，因此可以将函数从 Python（作为 PackedFunc）传递给 C++:

```c++
TVM_REGISTER_GLOBAL("callhello")
.set_body([](TVMArgs args, TVMRetValue* rv) {
  PackedFunc f = args[0];
  f("hello world");
});
```

```py
import tvm

def callback(msg):
    print(msg)

# 转换成 PackedFunc
f = tvm.convert(callback)
callhello = tvm.get_global_func("callhello")
# 打印 hello world
callhello(f)
```

- TVM 的所有编译器 pass 函数都以 `PackedFunc` 的类型暴露给前端
- 编译好的模块还将编译好的函数作为 `PackedFunc` 类型返回

`PackedFunc` 源码如下：

```c++
class PackedFuncObj : public Object {
 public:
  static constexpr const uint32_t _type_index = TypeIndex::kRuntimePackedFunc;
  static constexpr const char* _type_key = "runtime.PackedFunc";
  TVM_ALWAYS_INLINE void PackedFuncObj::CallPacked(TVMArgs args, TVMRetValue* rv) const {
    (*f_call_packed_)(this, args, rv);
  }
  TVM_DECLARE_FINAL_OBJECT_INFO(PackedFuncObj, Object);
 protected:
  PackedFuncObj() = delete;
  /*! \brief Internal struct for extracting the callable method from callable type. !*/
  template <class TPackedFuncSubObj>
  struct Extractor {
    static void Call(const PackedFuncObj* obj, TVMArgs args, TVMRetValue* rv);
  };
  using FCallPacked = void(const PackedFuncObj*, TVMArgs, TVMRetValue*);
  explicit PackedFuncObj(FCallPacked* f_call_pack) : f_call_packed_(f_call_pack) {}
  FCallPacked* f_call_packed_;
};
```

其中涉及到的 PackedFunc 的参数类型 `TVMArgs` 定义如下：
```c++
class TVMArgs {
 public:
  const TVMValue* values;
  const int* type_codes;
  int num_args;
  TVMArgs(const TVMValue* values, const int* type_codes, int num_args)
      : values(values), type_codes(type_codes), num_args(num_args) {}
  inline int size() const; // size of the arguments
  inline TVMArgValue operator[](int i) const;
};
```

其中的 `TVMValue` 类型定义位于 `c_runtime_api.h`：

```c++
typedef union {
  int64_t v_int64;
  double v_float64;
  void* v_handle;
  const char* v_str; // 字符串
  DLDataType v_type; // 对于 dlpack 数据类型的支持
  DLDevice v_device;
} TVMValue;
```
其中的 `DLDataType` 和 `DLDevice` 定义在 `3rdparty/dlpack/include/dlpack/dlpack.h` 中。 [DLPack: Open In Memory Tensor Structure](https://github.com/dmlc/dlpack)


## 3. Type and Expr

**将 IR 视为一种相对高级的编程语言，有两个关键的基础概念，类型 (Type) 和表达式 (Expr)**。 `Type` 类主要表示TVM IR中的各种类型，包含bool、int8，float32等基础数据类型，以及张量Tensor和元组Tuple等类型。 `Expr` 包括简单的定义一个字面值，也包括定义一个复杂的函数。

### 3.1. Type

> **反应一个IR的抽象层级最明显的标志是IR所处理的 data type**，high-level IR 多用来处理Tensor数据类型， low-level IR 大多用来处理Buffer或指针类型，在TVM `Type`类中可以看到TVM各个层级IR需要的Type。

在 `include/tvm/type.h` 中定义了多个基础类型， 所有类型 Node 都继承自 `TypeNode`:

<div class="autocb" style="text-align:center;"><img src="./tvm-type.assets\autocb_1.png" style="zoom: 100%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

`include/tvm/ir/type.h` 中 `TypeNode` 定义如下：

```c++
class TypeNode : public Object {
 public:
  mutable Span span; // points to the original source code. Reserved debug information
  static constexpr const char* _type_key = "Type";
  static constexpr const bool _type_has_method_sequal_reduce = true;
  static constexpr const bool _type_has_method_shash_reduce = true;
  static constexpr const uint32_t _type_child_slots = 14; // 预留了14个子类
  TVM_DECLARE_BASE_OBJECT_INFO(TypeNode, Object);
};
```
介绍该类型几个典型的子类：

- `PrimTypeNode` 定义位于 `include/tvm/ir/type.h` 表示可以直接映射到 low-level IR 的基本数据类型，代码如下

    ```c++
    class PrimTypeNode : public TypeNode {
     public:
      runtime::DataType dtype;  // corresponding dtype field
      static constexpr const char* _type_key = "PrimType";
      TVM_DECLARE_FINAL_OBJECT_INFO(PrimTypeNode, TypeNode);
    };
    ```

    其中 `runtime::DataType` 位于 `include/tvm/data_type.h` ， 是一个对于 `DLDataType` 的封装， 提供了额外的类型检查方法等。其可选的值包括：

    ```c++
    enum TypeCode {
      kInt = kDLInt,
      kUInt = kDLUInt,
      kFloat = kDLFloat,
      kHandle = TVMArgTypeCode::kTVMOpaqueHandle,
      kBFloat = kDLBfloat,
      kCustomBegin = 129
    };
    ```

- `TensorTypeNode` 定义位于 `include/tvm/ir/tensor_type.h` 是一种 *Polymorphic tensor type*

    ```c++
    class BaseTensorTypeNode : public TypeNode {
     public:
      static constexpr const char* _type_key = "relay.BaseTensorType";
      static constexpr const uint32_t _type_child_slots = 1;
      TVM_DECLARE_BASE_OBJECT_INFO(BaseTensorTypeNode, TypeNode);
    };

    class TensorTypeNode : public BaseTensorTypeNode {
     public:
      static constexpr const char* _type_key = "relay.TensorType";
      Array<PrimExpr> shape;    // shape of the tensor, represented by PrimExpr
      DataType dtype;           // content data type
      TVM_DLL PrimExpr Size() const; // return product of elements in the shape
      TVM_DECLARE_FINAL_OBJECT_INFO(TensorTypeNode, BaseTensorTypeNode);
    };
    ```

    `TensorType` 是 relay 中最常用到的类型； `TensorType` has **a fixed dimension, data type**

    可以看到 `TensorTypeNode` 中有一个 `shape` field， 这表示shape是 TensorType的一部分；即 
    
- `FunctypeNode` 定义位于 `include/tvm/ir/type.h` **可以看作C++中的 template function**

    ```c++
    class FuncTypeNode : public TypeNode {
     public:
      Array<Type> arg_types;    // type of arguments
      Type ret_type;            // type of return value

      // The following fields are used in polymorphic(template) functions
      // For normal functions, the following two fields will be empty.
      /* The type parameters of the function */
      Array<TypeVar> type_params;
      /* potential constraint the type need to obey; For further purposes. */
      Array<TypeConstraint> type_constraints;
      static constexpr const char* _type_key = "FuncType";
      TVM_DECLARE_FINAL_OBJECT_INFO(FuncTypeNode, TypeNode);
    };
    ```


同时Relay中还提供了描述Relay函数的输入和输出类型之间关系的类型关系特性，允许用户扩展类型推断，方便算子的shape推理。

### 3.2. Expr

无论是Relay IR还是Tensor IR都共享TVM IR基础设施，TVM IR 基础设施中除了 Type 系统， 另一个重要的部分就是表达式 expressions 。 表达式主要处理各种类型的数据，以及表示IR语句中控制结构、分支信息，其派生也要比Type类更加复杂一些。在TVM中， 表达式使用 `Expr` 类来表示， 其有两个直接子类： `RelayExpr` 和 `PrimExpr` 。 

此外继承自 Object 的 Stmt 在后文会介绍到，也是IR中的元素，与 Expr 的区别在于：Stmt 表示if判断、赋值，不处理Type类型的数据值，相当于陈述语句。

接下来关注Tensor IR 中对应的 `PrimExpr` 和 Relay IR 中对应的 `RelayExpr`： 

- `PrimExprNode`: 主要在 `tir` 模块中定义，可以相对直接地映射到 low-level code:

    ```c++
    class PrimExprNode : public BaseExprNode {
     public:
      DataType dtype; // POD 类型; 是一个对于 `DLDataType` 的封装
      static constexpr const char* _type_key = "PrimExpr";
      static constexpr const uint32_t _type_child_slots = 38;
      TVM_DECLARE_BASE_OBJECT_INFO(PrimExprNode, BaseExprNode);
    };
    ```

    `PrimExprNode` 里的 `DataType` 与前文提到的 `TypeNode` 中的 `runtime::DataType` 是同一个类型，即 一个对于 dlpack 中 `DLDataType` 类型的封装；
    因此，在 TVM 中， primitive expression 的类型为 POD 类型

    其子类包括:

    1. `BinaryOpNode`(其子类有 `Add`, `Mod`, `Mul`, `Min` 等二元op), `CmpOpNode`, `AndNode`, `OrNode`, `NotNode` 等基本的算数运算和逻辑运算表达式；
    2. `FloatImmNode`, `IntImmNode`, `StringImmNode` 等常量表达式
    3. `CastNode`, `BroadCastNode`, `LoadNode`, `BufferLoadNode`, `ProducerLoadNode` 等数据操作表达式
    4. `LetNode`, `VarNode`, `SelectNode`, `ReduceNode`, `CallNode` 等表达式
    5. ...

    <div class="autocb" style="text-align:center;"><img src="./tvm-type.assets\autocb_2.png" style="zoom: 60%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

- `RelayExprNode`: 主要在 `relay` 模块中定义， `RelayExpr` 是所有的 non-primitive expressions 的基类

    ```c++
    class RelayExprNode : public BaseExprNode {
     public:
      // This can be undefined before type inference.
      // This value is discarded during serialization.
      mutable Type checked_type_ = Type(nullptr); // result of type checking.

      // Stores the result of structure information of the
      // expression that encapsulate both static shape and
      // runtime information such as shape.
      mutable Optional<ObjectRef> struct_info_ = Optional<ObjectRef>();

      inline const Type& checked_type() const;

      template <typename TTypeNode>
      inline const TTypeNode* type_as() const; // check if `checked_type_` is backed by TTypeNode

      // this describes where the result of evaluating the expression should be stored
      // first-order values (tuples, references, ADTs) must be stored on the same virtual device
      // 对于函数类型， 表示函数调用返回值的存储device， 而不是函数本身的存储device
      // VirtualDevice's Target field describes how the body of the function should be compiled
      // 函数调用返回值所在的device 与 函数 body的存储 device 相同
      // *type of virtual_device_ needs to be ObjectRef to avoid a circular import* ???
      mutable ObjectRef virtual_device_;

      // 如果 `virtual_device_` 未设置， 返回 VirtualDevice::FullyUnconstrained()
      // 对于函数类型， 返回值是函数调用返回值的存储device， 而不是函数本身的存储device
      // 函数调用返回值所在的device 与 函数 body的存储 device 相同
      // `src/relay/transforms/device_planner.cc` 中有详细内容
      VirtualDevice virtual_device() const;

      static constexpr const char* _type_key = "RelayExpr";
      static constexpr const uint32_t _type_child_slots = 22;
      TVM_DECLARE_BASE_OBJECT_INFO(RelayExprNode, BaseExprNode);
    };
    ```

    <div class="autocb" style="text-align:center;"><img src="./tvm-type.assets\autocb_0.png" style="zoom: 45%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

RelayExpr中的变量值表示分为：全局变量 `GlobalVar` 和局部变量 `Var` ，在IR中使用不同的前缀区分(`@`、`%`)，本地变量Variable一般用作函数的参数或者配合let表达式绑定使用。

Constant表示一种常量张量类型，根据不同张量维度表示不同的常量，比如标量常量、数组常量，TVM中常数使用NDArray表示。


## Relay

`relay/type.h` 中内容如下：

```c++
namespace tvm {
namespace relay {
using Any = tvm::tir::Any;
using Kind = TypeKind;
using Type = tvm::Type;
using TypeVar = tvm::TypeVar;
using GlobalTypeVar = tvm::GlobalTypeVar;
using TupleType = tvm::TupleType;
using TypeConstraint = tvm::TypeConstraint;
using FuncType = tvm::FuncType;
using IncompleteType = tvm::IncompleteType;
using RelayRefType = tvm::RelayRefType;
using TensorType = tvm::TensorType;
using TypeCall = tvm::TypeCall;
using TypeRelation = tvm::TypeRelation;
using TypeRelationFn = tvm::TypeRelationFn;
using TypeReporter = tvm::TypeReporter;
}  // namespace relay
}  // namespace tvm

```

### Dynamic Shape Tensor

Relay 通过引入 `Any` 添加了对于 dynamic shape Tensor 的支持

## Relax
Relax 种引入了一个新的 动态 tensor 类型 `DynTensorTypeNode`， 对比原本TVM中的 静态 tensor类型 `TensorTypeNode`

`include/tvm/ir/tensor_type.h`:

```c++
class TensorTypeNode : public BaseTensorTypeNode {
 public:
  static constexpr const char* _type_key = "relay.TensorType";
  Array<PrimExpr> shape;    // shape of the tensor, represented by PrimExpr
  DataType dtype;           // content data type
  TVM_DLL PrimExpr Size() const; // return product of elements in the shape
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
>
> !!! warning "疑问"
      这里的不可分析是什么意思？
>

