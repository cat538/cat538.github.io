# TVM-type system

本文 **基于 tlc-pack/relax dc7072efe290d7e8c69d8e216311510981fc82e1**

> Ref:
>
> - [深度学习编译器 TVM 代码串讲-知乎](https://zhuanlan.zhihu.com/p/446976730)
> - [TVM编译流程与中间表示分析-知乎](https://zhuanlan.zhihu.com/p/596526031)
> - [深入理解TVM：Object家族（二）-Wechat](https://mp.weixin.qq.com/s?__biz=Mzg5MzU4NTU5Nw==&mid=2247484141&idx=1&sn=c139df6c55494d8669f56e237ff513bb&chksm=c02dd33ff75a5a295884adb3147b9dff436ab359765fc64cecf479fbc6899d76009135feac87&scene=178&cur_album_id=1811050680510447621#rd)
> - [TVM-Doc: Pass Infrastructure](https://tvm.hyper.ai/docs/arch/arch/pass_infra)

## 1. Runtime

### 1.1. Object

> 对于类似 IR 这样的数据结构，天然就对其有 serialize/format/reflection 的需求。在 TVM 中，额外还有 python binding/hash 等需求，于是 TVM 要求**所有这样的数据结构继承自 `Object` 基类**，并注册其内部所有成员； 由 TVM 的类型注册系统和反射系统等为继承自 Object 的子类自动实现序列化，反射，hash等功能。

这种方式避免了为新增的 Class 单独实现 serialize/format/reflection/python binding/hash 等功能， 实现了代码重用。

`include/tvm/runtime/object.h `中定义了 `Object` 和 `ObjectRef` 两个类型， `ObjectRef` 可以视为 `shared_ptr<Object>`

删除部分函数和成员，`Object` 精简后的定义如下：

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
- `_type_child_slots_can_overflow` 标识是否可超过 `_type_child_slots` 定义的数量
- `_type_final` 表示是否没有子类，一般通过 `TVM_DECLARE_FINAL_OBJECT_INFO` 这个宏来设置， 而不是手动重写
- TODO: `_type_has_method_sequal_reduce`, `_type_has_method_shash_reduce` 等标识该类型的 hash 等功能是否实现，可以参考 TODO:

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

- `RuntimeTypeIndex` 是一个类型向外部暴露信息的接口， 外部通过这个函数来获得类型索引等相关信息。**一个应用实例**：在`include/tvm/runtime/object.h` 中的 `IsInstance` 功能为 Check if the object is an instance of `TargetType`， 其中使用 `RuntimeTypeIndex` 进行加速检查：

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

💡总体来说，通过同样的 `_type_key` 可以将 C++ 与 Python 的类型匹配上，**而设计 type_index 则是为了性能考虑**。具体的两边类型匹配可以在 `python/tvm/_ffi/_cython/object.pxi` 查阅

TODO: 额外的 serialize/format/reflection/python binding/hash 等功能则实现在 `node` 目录下。

### 1.2. PackedFunc
> TVM runtime 中另一个与 Object 同样底层的机制称为 FFI (Foreign Function Interface), 这个机制的目标是为了使得任意语言下定义的函数都可以被任意其他语言调用。而这个可以被任意语言调用的函数类型是 `PackedFunc`

PackedFunc 是类型擦除的（在后面会解释，其参数只能为一些特定类型），这使得我们可以从动态语言（如 Python）中调用 PackedFunc，而无需为每个创建的新类型函数添加额外的胶水代码。调用 PackedFunc 时，它会将输入参数打包到 stack 上的 `TVMArgs` ，并通过 `TVMRetValue` 获取结果。

这里比较有意思的是 `PackedFunc` 的 `operator()` 重载：

```c++
template <typename... Args>
inline TVMRetValue operator()(Args&&... args) const{
  const int kNumArgs = sizeof...(Args);
  const int kArraySize = kNumArgs > 0 ? kNumArgs : 1;
  TVMValue values[kArraySize];
  int type_codes[kArraySize];
  detail::for_each(TVMArgsSetter(values, type_codes), std::forward<Args>(args)...);
  TVMRetValue rv;
  (static_cast<PackedFuncObj*>(data_.get()))
      ->CallPacked(TVMArgs(values, type_codes, kNumArgs), &rv);
  return rv;
}
```

可以看到，函数利用了变长模板参数，并且为我们创建了 TVMArgs 和 TVMRetValue， 这样一来我们利用 `operator()` 就可以像调用普通函数一样调用 PackedFunc 了。

一个在 C++ 中构造、调用 PackedFunc 并注册到全局的示例如下:
```c++
#include <tvm/runtime/packed_func.h>
void MyAdd(TVMArgs args, TVMRetValue* rv) {
  int a = args[0];  // 自动将参数转换为所需的类型
  int b = args[1];  // 自动将参数转换为所需的类型
  *rv = a + b;      // 自动赋值返回给 rv
}
void CallPacked() {     // 在C++ 中 调用 PackedFunc
  PackedFunc myadd = PackedFunc(MyAdd);
  int c = myadd(1, 2);  // 返回 3
}
TVM_REGISTER_GLOBAL("myadd")  // 在 C++ 中注册一个全局 PackedFunc
  .set_body(MyAdd);
```
以上代码块中实例化了一个 PackedFunc `MyAdd` 。它有两个参数： `args` 代表输入参数， `rv` 代表返回值。

在 Python 中调用上面在 C++ 中注册的 PackedFunc `myadd`：

```py
import tvm
myadd = tvm.get_global_func("myadd")
print(myadd(1, 2))  # => 3
```

1. 这里首先通过 `get_global_func` (位于`python/tvm/_ffi/registry.py` 中) 构造了一个 `PackedFunc` 实例； 这个过程实际上是通过 python 端使用 ctypes 模块调用 C++端通过 `c_runtime_api` 暴露的 `TVMFuncGetGlobal` ， 在注册表中查找到名为`my_add`的函数， 拿到handle:
 
    ```python
    def _get_global_func(name, allow_missing=False):
        handle = PackedFuncHandle()
        check_call(_LIB.TVMFuncGetGlobal(c_str(name), ctypes.byref(handle)))
        if handle.value:
            return _make_packed_func(handle, False)
        if allow_missing:
            return None
        raise ValueError("Cannot find global function %s" % name)
    ```

2. 接下来的`myadd(1, 2)` 调用了 `my_add` 实例的 `__call__` 方法， 该方法实现在`PackedFunc` 的父类 `PackedFuncBase` 中(位于 `python/tvm/_ffi/_ctypes/packed_func.py`)
3. PackedFunc 的 `__call__` 方法事实上是使用 ctypes 模块调用 C++端通过 `c_runtime_api` 暴露的 `TVMFuncCall` ， 将结果通过传入的 `TVMValue` 返回：

    ```python
    class PackedFuncBase(object):
    def __init__(self, handle, is_global):
        self.handle = handle
        self.is_global = is_global
    def __call__(self, *args):
        values, tcodes, num_args = _make_tvm_args(args, [])
        ret_val, ret_tcode = TVMValue(), ctypes.c_int()
        if (
            _LIB.TVMFuncCall( # 调用 C++ 中 通过 c_runtime_api 暴露的 TVMFuncCall
                self.handle,
                values,
                tcodes,
                ctypes.c_int(num_args),
                ctypes.byref(ret_val),
                ctypes.byref(ret_tcode),
            )
            != 0
        ):
            raise get_last_ffi_error()
        return RETURN_SWITCH[ret_tcode.value](ret_val)
    ```

    TVMFuncCall 对应的 C++ 端注册：
    
    ```c++
    // include/tvm/runtime/c_runtime_api.h
    TVM_DLL int TVMFuncCall(TVMFunctionHandle func, TVMValue* arg_values, int* type_codes, int num_args,
                        TVMValue* ret_val, int* ret_type_code);
    ```

PackedFunc 的参数类型可以是 PackedFunc，而且 TVM 允许我们在 python 端进行全局注册， 因此可以将函数从 Python（需要先 wrap 为 `PackedFunc` ）传递给 C++，在C++中回调:

```c++
TVM_REGISTER_GLOBAL("callhello")
.set_body([](TVMArgs args, TVMRetValue* rv) {
  PackedFunc f = args[0];
  f("hello world");
});
```

```py
@tvm.register_func
def callback(msg):
    print(msg)
callhello = tvm.get_global_func("callhello")
callhello(f) # >>>"hello world"
```

类似地，我们可以从任意语言调用任意其它语言，这样就通过 PackedFunc 完成了 FFI， 接下来看一下 PackedFunc 是如何实现类型擦除， 以及具体保存了哪些信息， `PackedFunc` 源码如下(`include/tvm/runtime/packed_func.h`)：

```c++
class PackedFuncObj : public Object {
 public:
  void PackedFuncObj::CallPacked(TVMArgs args, TVMRetValue* rv) const {
    (*f_call_packed_)(this, args, rv);
  }
  static constexpr const char* _type_key = "runtime.PackedFunc";
  static constexpr const uint32_t _type_index = TypeIndex::kRuntimePackedFunc;
  TVM_DECLARE_FINAL_OBJECT_INFO(PackedFuncObj, Object);
 protected:
  template <class TPackedFuncSubObj>
  struct Extractor {
    static void Call(const PackedFuncObj* obj, TVMArgs args, TVMRetValue* rv);
  };
  PackedFuncObj() = delete;
  using FCallPacked = void(const PackedFuncObj*, TVMArgs, TVMRetValue*);
  explicit PackedFuncObj(FCallPacked* f_call_pack) : f_call_packed_(f_call_pack) {}
  
  FCallPacked* f_call_packed_;
};
```

其中涉及到的 PackedFunc 的参数类型 `TVMArgs` 简化定义如下（返回值类型 `TVMRetValue` 类似）：
```c++
class TVMArgs {
 public:
  const TVMValue* values;
  const int* type_codes;
  int num_args;
  inline TVMArgValue operator[](int i) const;
};
```

可以看到 `TVMArgs` 就是一个 `TVMValue` 数组； 而 `TVMValue` 是一个union类型， 定义位于 `c_runtime_api.h`：

```c++
typedef union {
  int64_t v_int64;
  double v_float64;
  void* v_handle;
  const char* v_str; // 字符串
  DLDataType v_type; // dlpack 数据类型； 包括整型, 浮点, Bfloat等
  DLDevice v_device; // CPU CUDA 等
} TVMValue;
```

其中的 `DLDataType` 和 `DLDevice` 定义在 `3rdparty/dlpack/include/dlpack/dlpack.h` 中。 [DLPack: Open In Memory Tensor Structure](https://github.com/dmlc/dlpack)

因此我们可以看到，PackedFunc 最终通过 TVMValue 实现了类型擦除，即参数只要是 TVMValue 中的一种类型，即可包装成`TVMArgs`作为参数传入 PackedFunc；常见的参数类型有：

- `int`, `float` and `string`
- `PackedFunc` 本身(通过 `v_handle` 传递)
- `Module` for compiled modules((通过 `v_handle` 传递))
- `DLTensor*`(见dlpack) for tensor object exchange
- TVM `Object` to represent any object in IR

💡PackedFunc 在 TVM 的 runtime 中扮演非常重要的角色：

- TVM 的所有编译器 pass 函数都以 `PackedFunc` 的类型暴露给前端
- 编译好的模块还将编译好的函数作为 `PackedFunc` 类型返回



### 1.3. Module
`runtime::Module` 定义在 `include/tvm/module.h` 中， 在 TVM stack中用来表达编译后的结果。 可以简单的视为 `<name, PackedFunc>` 的一个哈希表。 不过 `ModuleNode` 仅仅是一个接口， 在不同的 target 有不同的继承实现。 例如在编译时指定 target 为 llvm， 则生成的 runtime::Module 背后就是一个 `LLVMModuleNode` (定义在 `src/target/llvm/llvm_module.cc`中)，其它`target`也有相应的 `ModuleNode` 的子类。

`ModuleNode` 简化后定义如下：

```c++
class TVM_DLL ModuleNode : public Object {
 public:
  virtual ~ModuleNode() = default;
  virtual const char* type_key() const = 0; // LLVMModuleNode: "llvm"； CUDAModuleNode: "cuda"
  virtual PackedFunc GetFunction(const std::string& name, const ObjectPtr<Object>& sptr_to_self) = 0;
  virtual void SaveToFile(const std::string& file_name, const std::string& format);
  
  PackedFunc GetFunction(const std::string& name, bool query_imports = false);
  void Import(Module other);
  const PackedFunc* GetFuncFromEnv(const std::string& name);
  
  static constexpr const uint32_t _type_index = TypeIndex::kRuntimeModule;
  static constexpr const char* _type_key = "runtime.Module";
  TVM_DECLARE_FINAL_OBJECT_INFO(ModuleNode, Object); // NOTE! can still be sub-classed
 protected:
  std::vector<Module> imports_;   // modules this module depend on
 private:
  std::unordered_map<std::string, std::shared_ptr<PackedFunc>> import_cache_;
  std::mutex mutex_;
};
```

接下来是它的两个常用子类的例子：

- 子类 `LLVMModuleNode`
    
    ```c++
    class LLVMModuleNode final : public runtime::ModuleNode {
     public:
      ~LLVMModuleNode();
      const char* type_key() const final { return "llvm"; }
      PackedFunc GetFunction(const std::string& name, const ObjectPtr<Object>& sptr_to_self) final;
      void SaveToFile(const std::string& file_name, const std::string& format) final;
      void Init(const IRModule& mod, const Target& target);
      void LoadIR(const std::string& file_name);
     private:
      void LazyInitJIT();
      bool IsCompatibleWithHost(const llvm::TargetMachine* tm) const;
      void* GetGlobalAddr(const std::string& name, const LLVMTarget& llvm_target) const;
      void* GetFunctionAddr(const std::string& name, const LLVMTarget& llvm_target) const;
      std::unique_ptr<LLVMInstance> llvm_instance_; // 包含 `llvm::LLVMContext` 和 `llvm::Module`
      std::mutex mutex_;                    // JIT lock
      llvm::ExecutionEngine* ee_{nullptr};  // execution engine for JIT etc.
      llvm::Module* module_{nullptr};       // module_owning_ptr_.get()
      std::unique_ptr<llvm::Module> module_owning_ptr_; // EngineBuilder 会拿走该 Module 的所有权
      Array<String> function_names_;        // 该 module 内声明的函数名
    };
    ```

    注意其中的 `void Init(const IRModule&, const Target&)` ， 该函数是 tir IR 的 IRModule lower 到 LLVM 后端时会被调用的函数

- 子类 `CUDAModuleNode`

    ```c++
    lass CUDAModuleNode : public runtime::ModuleNode {
    public:
      ~CUDAModuleNode();
      const char* type_key() const final { return "cuda"; }
      PackedFunc GetFunction(const std::string& name, const ObjectPtr<Object>& sptr_to_self) final;
      void SaveToFile(const std::string& file_name, const std::string& format) final;
    private:
      std::string data_;  // the binary data
      std::string fmt_; // The format
      std::unordered_map<std::string, FunctionInfo> fmap_;  // function information table.
      std::string cuda_source_; // The cuda source.
      std::array<CUmodule, kMaxNumGPUs> module_;  // Internal modules per GPU
      std::mutex mutex_;  // internal mutex when updating the module
    };
    ```

TODO:

### 1.4. Container
TVM 中还重新实现了一些常用的 Container，例如`Map`, `Array`, `Optional`, `ADT` 等，定义在 `include/tvm/runtime/container` 中。

在我的理解中， 重新实现这些容器的目的是为了能够通过 FFI 传递这些容器。 因为这些容器派生自 `Object`, 因此能够作为 `TVMArgs` 或者 `TVMRetValue` 通过 PackedFunc 进行传递， 而 STL 中的容器则无法直接通过 PackedFunc 传递。下面是一个例子：

- Relay 中最常用到的类型就是 `TensorType` ， 在 C++ 端 其定义如下：
    ```c++
    class TensorTypeNode : public BaseTensorTypeNode {
    public:
      Array<PrimExpr> shape;
      DataType dtype;
      static constexpr const char* _type_key = "relay.TensorType";
      TVM_DECLARE_FINAL_OBJECT_INFO(TensorTypeNode, BaseTensorTypeNode);
    };

    class TensorType : public Type {
    public:
      TVM_DLL TensorType(Array<PrimExpr> shape, DataType dtype);
      TVM_DLL static TensorType Scalar(DataType dtype);
      TVM_DEFINE_OBJECT_REF_METHODS(TensorType, Type, TensorTypeNode);
    };
    ```

    这里可以看到 `TensorType` 的构造函数的两个参数类型分别是 `Array<PrimExpr>` 和 `DataType`

- 而在对应的 python 端， `TensorType` 定义在 `python/tvm/ir/tensor_type.py` 中：
    ```python
    @tvm._ffi.register_object("relay.TensorType")
    class TensorType(Type):
        def __init__(self, shape, dtype="float32"):
            self.__init_handle_by_constructor__(_ffi_api.TensorType, shape, dtype)
        @property
        def concrete_shape(self):
            """Get shape of the type as concrete tuple of int. """
            return tuple(int(x) for x in self.shape)
        def __str__(self):
            from tvm.relay import pretty_print  # pylint: disable=import-outside-toplevel
            return pretty_print(self)
    ```

因此如果我们需要在python端构造一个 TensorType：

```py
t1 = relay.TensorType((3, 4), "float32")
```

则我们首先需要把 python 端的元组转换成 `tvm.ir.runtime.Array`(具体逻辑位于python端的`_make_tvm_args`)， 再通过 PackedFunc 进行传递， 这个转换过程可以通过调试查看。

同样的，对于python中传递字典到 PackedFunc 中的情况，也需要先转成 `tvm.ir.runtime.Map` 等等诸如此类。

💡综上所述 Container 的实现使得 PackedFunc 可以传递 array，map 等常见的数据结构


## 2. IR

### 2.1. Type
**将 IR 视为一种相对高级的编程语言，有两个关键的基础概念，表达式 (Expr) 和 表达式的 类型 (Type)**。 `Type` 类主要表示TVM IR中的各种类型，包含bool、int8，float32等基础数据类型，以及张量Tensor和元组Tuple等类型。 `Expr` 包括简单的定义一个字面值，也包括定义一个复杂的函数。

在 TVM 中，有 **Relay**(定义在`include/tvm/relay/`中)， **Relax**(定义在`include/tvm/relax/`中)， **tir**(定义在`include/tvm/tir/`中) 等不同层级的IR。 这些 IR 共享同一套 IR 基础设施(主要定义在 `include/tvm/ir/`中)， 包括`type`和`expr`等；实现了工程上的代码重用，划分的相对清晰（不过从代码角度来说，这些IR之间并非完全隔离， 例如 Relay 中就需要重用 tir 中定义的 `Any` 类型）。

> **反应一个IR的抽象层级最明显的标志之一是IR所处理的 data type**，high-level IR 多用来处理Tensor数据类型， low-level IR 大多用来处理Buffer或指针类型，在TVM `Type`类中可以看到TVM各个层级IR需要的Type。

在 `include/tvm/type.h` 中定义了多个基础类型， 所有类型 Node 都继承自 `TypeNode`:

<div class="autocb" style="text-align:center;"><img src="./tvm-type.assets\autocb_1.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

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
      Array<PrimExpr> shape;          // tensor shape, represented by PrimExpr
      DataType dtype;                 // content data type
      TVM_DLL PrimExpr Size() const;  // return product of elements in the shape
      TVM_DECLARE_FINAL_OBJECT_INFO(TensorTypeNode, BaseTensorTypeNode);
    };
    ```

    `TensorType` 是 relay 中最常用到的类型； `TensorType` has **a fixed dimension, data type**

    `TensorTypeNode` 中有一个 `shape` field， 这表示shape是 TensorType的一部分；
    TODO:即 Tensor[(4, 4)]和Tensor[(Any, 4)]是不同的type (**`Any` 是`PrimExpr`的子类，用于在Relay中表示 dynamic shape**)； 在relax中引入了一个与之相对的 张量类型 `DynTensorType`(位于`include/tvm/relax/type.h`)， 具体可见 TODO:
    
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

### 2.2. Expr

表达式 expression 主要处理各种类型的数据，以及表示IR语句中控制结构、分支信息，其派生也要比Type类更加复杂一些。在TVM中， 表达式使用 `Expr` 类来表示， 其有两个直接子类： `RelayExpr` 和 `PrimExpr` 。 

此外继承自 Object 的 `Stmt` 在后文会介绍到，也是IR中的元素，与 Expr 的区别在于： `Stmt` 表示if判断、赋值，不处理Type类型的数据值，相当于陈述语句。

接下来关注 tir 中对应的 `PrimExpr` 和 Relay IR 中对应的 `RelayExpr`： 

- `PrimExprNode`: 定义在`include/tvm/ir.h`；其派生子类主要在 `tir` 模块中定义，可以相对直接地映射到 low-level code:

    ```c++
    class PrimExprNode : public BaseExprNode {
     public:
      DataType dtype; // POD 类型; 是一个对于 `DLDataType` 的封装
      static constexpr const char* _type_key = "PrimExpr";
      static constexpr const uint32_t _type_child_slots = 38;
      TVM_DECLARE_BASE_OBJECT_INFO(PrimExprNode, BaseExprNode);
    };
    ```

    `PrimExprNode` 里的 `DataType` 与Type一节里 `TypeNode` 中的 `runtime::DataType` 是同一个类型，即 一个对于 dlpack 中 `DLDataType` 类型的封装。
    
    因此，primitive expression 的evaluation结果type为 POD 类型——在 `ir.h` 中还为 `PrimExpr` 重载了四则运算，逻辑运算等运算符

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
      // 在type inference之前，该值可为 undefined
      // 序列化 时该字段将被丢弃
      mutable Type checked_type_ = Type(nullptr); // result of type checking.
      // Stores the result of structure information of the
      // expression that encapsulate both static shape and
      // runtime information such as shape.
      mutable Optional<ObjectRef> struct_info_ = Optional<ObjectRef>();
      inline const Type& checked_type() const;
      template <typename TTypeNode>
      inline const TTypeNode* type_as() const;
      // 该 filed 描述了一个 Expr 的求值结果被存放在哪里
      // first-order values (tuples, references, ADTs) 各字段必须存在相同的 virtual device 上
      // 对于函数类型， 表示函数调用返回值的存储device， 而不是函数本身的存储device
      // VirtualDevice 的 `target` field 描述了函数体该如何被编译
      // 函数调用返回值所在的device 与 函数 body的存储 device 相同
      // `src/relay/transforms/device_planner.cc` 中有详细内容
      // *type of virtual_device_ needs to be ObjectRef to avoid a circular import* ???
      mutable ObjectRef virtual_device_;
      VirtualDevice virtual_device() const;
      static constexpr const char* _type_key = "RelayExpr";
      static constexpr const uint32_t _type_child_slots = 22;
      TVM_DECLARE_BASE_OBJECT_INFO(RelayExprNode, BaseExprNode);
    };
    ```

    <div class="autocb" style="text-align:center;"><img src="./tvm-type.assets\autocb_0.png" style="zoom: 45%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

    1. RelayExpr 中有两种变量：全局变量 `GlobalVar` 和局部变量 `Var` ，在 relay IR 的 text-format 中使用不同的前缀表示(`@`、`%`)，局部变量一般用作函数的参数或者配合`let`表达式绑定使用

    2. Constant 表示常量张量类型。根据不同张量维度表示不同的常量，比如标量常量、数组常量，RelayExpr 中常量表达式使用 NDArray 表示； 这里可以对比 tir 中的常量表达式： 在tir 中， 常量表达式有 `FloatImm`, `IntImm` 等不同类型用于表示 scalar， 而在relay 中的常量表达式则是表示 tensor

💡<u>**从对于Let， Match， Constructor 等表达式的支持可以看出，Relay相比于传统的数据流图 添加了更多函数式的支持，更具体的信息可以参考 [relayIR](./tvm-relayIR.md)**</u>

#### 2.2.1. Let-Binding
关于为什么需要 Let-binding， 在这个 RFC 里的例子可能比官网写的更详细一些：
[TVM-Disc: Basic Block Normal Form](https://discuss.tvm.apache.org/t/basic-block-normal-form/5908)

简单来说就是：

- Graph form 的 IR 对于模式匹配和图重写非常友好，但是 graph-form 存在 **求值顺序不明确**， **无法直接表达scope语义** 的问题。graph-form 通常意味着子图之间的求值顺序可以任意重排；对于没有 effect 的 blocks(即子图) 之间 进行求值顺序重排是ok的，但如果一个 block 有effect（比如说会改变另一个block也用到的一个全局状态等），则该block的计算顺序，or 该block与其它相关 block 之间的依赖关系，应当被明确定义，而 graph-form 会因为缺失表达scope的能力，和缺失定义block之间计算顺序的能力，导致语义模糊(semantic ambiguity)。在[Introduction to Relay IR](https://tvm.apache.org/docs/arch/relay_intro.html) 中有两个例子，下面是RFC中的一个例子：

    <div class="autocb" style="text-align:center;"><img src="./tvm-type.assets\autocb_3.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

- 如果我们使用带有显式 let-binding 的 ANF， 那么我们能明确计算范围，以及需要副作用的值的顺序。但是使用 let-binding 在进行模式匹配的时候会相对困难一些。

TVM 的图级IR Relay 选择同时支持 let-binding 和 DAG 形式，两者之间可以相互转换。

- 在 `test_pass_to_graph_normal_form`， `test_pass_to_a_normal_form`， `test_pass_to_basic_block_normal_form`中有 不同 form 之间的转换 pass 具体应用


`Let`的具体实现代码如下：

```c++
class LetNode : public ExprNode {
 protected:
  // LetNode uses own deleter to indirectly call non-recursive destructor
  Object::FDeleter saved_deleter_;
  static void Deleter_(Object* ptr);
 public:
  Var var;    // The variable we bind to
  Expr value; // The value we bind var to
  Expr body;  // The body of the let binding
  static constexpr const char* _type_key = "relay.Let";
  TVM_DECLARE_FINAL_OBJECT_INFO(LetNode, ExprNode);
};
```

`tir` 中也支持`Let`:

```c++
class LetNode : public PrimExprNode {
 public:
  Var var;        // The variable
  PrimExpr value; // The value to be binded
  PrimExpr body;  // The result expression
  static constexpr const char* _type_key = "tir.Let";
  TVM_DECLARE_FINAL_OBJECT_INFO(LetNode, PrimExprNode);
};
```

- 在 `tests/python/relay/test_ir_parser.py` 中有 `Let` 的具体用例可以参考

### 2.3. Op
Relay 和 tir 的 Op 都是 RelayExpr (`include/tvm/ir/op.h`)：

```c++
class OpNode : public RelayExprNode {
 public:
  String name;
  String description;
  mutable FuncType op_type;
  Array<AttrFieldInfo> arguments;

  String attrs_type_key;
  uint32_t attrs_type_index{0};

  int32_t num_inputs = -1;  // input arguments to the operator; -1 means variable length

  int32_t support_level = 10; // The lower the more priority

  static constexpr const char* _type_key = "Op";
  TVM_DECLARE_FINAL_OBJECT_INFO(OpNode, RelayExprNode);
 private:
  // Program internal unique index of operator.
  // Used to help index the program.
  uint32_t index_{0};
  // whether this is a primitive op. -1 means unknown.
  mutable int is_primitive_{-1};
};
```

这里以 Relay 定义的 `bias_add` Op 的例子来理解，位于`src/relay/op/nn/nn.cc` 中:

```c++
// relay.nn.bias_add
TVM_REGISTER_NODE_TYPE(BiasAddAttrs);

bool BiasAddRel(const Array<Type>& types, int num_inputs, const Attrs& attrs,
                const TypeReporter& reporter) {
  ICHECK_EQ(types.size(), 3);
  const auto* data = types[0].as<TensorTypeNode>();
  if (data == nullptr) return false;

  const BiasAddAttrs* param = attrs.as<BiasAddAttrs>();
  ICHECK(param != nullptr);
  int axis = param->axis;
  if (axis < 0) {
    axis = data->shape.size() + axis;
  }
  if (axis >= static_cast<int>(data->shape.size()) || axis < 0) {
    reporter->GetDiagCtx().EmitFatal(Diagnostic::Error(reporter->GetSpan())
                                     << "The axis in bias_add must be in range for the shape; "
                                     << "attempted to access index " << param->axis << " of "
                                     << PrettyPrint(data->shape));
    return false;
  }

  // assign output type
  reporter->Assign(types[1], TensorType({data->shape[axis]}, data->dtype));
  reporter->Assign(types[2], types[0]);
  return true;
}

// Positional relay function to create dense operator used by frontend FFI.
Expr MakeBiasAdd(Expr data, Expr bias, int axis) {
  auto attrs = make_object<BiasAddAttrs>();
  attrs->axis = axis;
  static const Op& op = Op::Get("nn.bias_add");
  return Call(op, {data, bias}, Attrs(attrs), {});
}

TVM_REGISTER_GLOBAL("relay.op.nn._make.bias_add").set_body_typed(MakeBiasAdd);

RELAY_REGISTER_OP("nn.bias_add")
    .describe(R"code(Add bias to an axis of the input.

)code" TVM_ADD_FILELINE)
    .set_attrs_type<BiasAddAttrs>()
    .set_num_inputs(2)
    .add_argument("data", "nD Tensor", "Input data.")
    .add_argument("bias", "1D Tensor", "Bias.")
    .set_support_level(1)
    .add_type_rel("BiasAdd", BiasAddRel)
    .set_attr<TOpPattern>("TOpPattern", kBroadcast)
    .set_attr<FTVMCompute>("FTVMCompute", [](const Attrs& attrs, const Array<te::Tensor>& inputs,
                                             const Type& out_type) {
      const auto* param = attrs.as<BiasAddAttrs>();
      return tvm::Array<tvm::te::Tensor>{topi::nn::bias_add(inputs[0], inputs[1], param->axis)};
    });
```


### 2.4. IRModule
定义在 `include/tvm/ir/module.h` 中

```c++
class IRModuleNode : public Object {
 public:
  Map<GlobalVar, BaseFunc> functions;               // global-var => global-function
  Map<GlobalTypeVar, TypeData> type_definitions;    // global-type-var => ADT-type-data
  Map<String, GlobalVar> global_var_map_;           // string-name => global-var
  Map<String, GlobalTypeVar> global_type_var_map_;  // string-name => global-type-var
  SourceMap source_map; // source map for the module
  DictAttrs attrs;      // 存储该 module 的元信息

  std::unordered_map<int32_t, Constructor> constructor_tag_map_;  // constructor-tags => constructor
  std::unordered_set<String> import_set_; // files previously imported

  TVM_DLL void Add(const GlobalVar& var, const BaseFunc& func, bool update = false);
  TVM_DLL void AddTypeDef(const GlobalTypeVar& var, const TypeData& type, bool update = false);
  TVM_DLL GlobalVar GetGlobalVar(const String& str) const;
  TVM_DLL GlobalTypeVar GetGlobalTypeVar(const String& str) const;
  TVM_DLL void Import(const String& path);  // Import Relay code from path.
  static constexpr const char* _type_key = "IRModule";
  TVM_DECLARE_FINAL_OBJECT_INFO(IRModuleNode, Object);
};
```

TODO:

### 2.5. Schedule
TODO:

### 2.6. Pass

TVM 的Pass 基础设施定义了一个虚基类: `Pass`，以及 实现 Pass 管理的 `PassContext`(类似 LLVM 中的 PassManeger) 等， 它们的定义在 `include/tvm/ir/transform.h` 中

> 该文件实现了一个 pass 管理器。 Pass 管理器管理在给定的 AST 单元 上管理 IRModule -> IRModule 的转换 Passes。 该设计的灵感主要来自 LLVM 的 pass 管理器和执行 张量 -> 张量 转换的现代深度学习框架
> 
> 传统编译器pass 管理器的职责通常包括： 
> 
> - 组织优化 Pass 的执行顺序，但不一定是最佳顺序
> - 收集所需的分析信息并及时更新
> - 减少为编译器开发人员等实施新 Pass 所需的工作量
> 
> 与 LLVM 的 pass 管理器类似，我们将 Relay/Relax pass 管理器设计为以不同的粒度工作，即模块级别、功能级别，甚至 sequential passes that contains a host of passes。
> 
> 但是，我们还考虑了深度学习框架（例如 Pytorch 和 Gluon 等）的要求/约定，从而扩展了传统 Pass 管理器的功能。Relay/Relax  Pass 管理器中的每个 Pass 都执行 IRModule -> IRModule 转换。 所有不同类型的传递，包括 sequential-level pass object，本质上都是传递对象。 因此，这种设计有效地为用户提供了一个一致且方便的界面，即 Pass 。 它提供了一种简化 Relay/Relax pass 的开发和测试的方法。 例如，使用 Pass 管理器，外部用户将能够正确安排自定义 Pass ，而无需修改单个手工制作的 Pass 订单。
> 
> **将来我们需要描述 Pass 之间的约束。 例如，我们可能希望保留不同 Pass 之间的依赖关系，并在某个 Pass 完成时验证它们**
> 
> 我们还需要存储辅助信息并导入错误报告系统

其中 Pass 定义如下：

```c++
class PassNode : public Object {
 public:
  virtual ~PassNode() {}
  virtual PassInfo Info() const = 0;
  IRModule operator()(IRModule mod) const {
    return this->operator()(std::move(mod), PassContext::Current());
  }
  virtual IRModule operator()(IRModule mod, const PassContext& pass_ctx) const = 0;
  static constexpr const char* _type_key = "transform.Pass";
  TVM_DECLARE_BASE_OBJECT_INFO(PassNode, Object);
};
```

从描述中可以知道， Pass 做一个 IRModule to IRModule 的变换。我们需要注意两个类型， `PassInfo` 与 `PassContext` 。 这里首先来看 PassInfo， 该类型表示一个 Pass 的 metadata, 每个具体的 Pass 实现都要提供 PassInfo 信息

```c++
class PassInfoNode : public Object {
 public:
  int opt_level;  // 启用该 pass 的最小 opt_level
  String name;    // pass 名字
  bool traceable; // 该 pass 是否可被 trace
  Array<String> required; // 执行当前 pass 所需要的前置 pass
  PassInfoNode() = default;
  static constexpr const char* _type_key = "transform.PassInfo";
  static constexpr bool _type_has_method_sequal_reduce = false;
  TVM_DECLARE_FINAL_OBJECT_INFO(PassInfoNode, Object);
};
```

可以看到，每个 Pass 实现需要使用字符串命名，并设定好依赖的前置 Pass，从而确定多个 Pass 的执行顺序。

再看 PassContext

```c++
class PassContextNode : public Object {
 public:
  int opt_level{2};             // 默认 opt_level
  Array<String> required_pass;  // 需要的 pass 列表
  Array<String> disabled_pass;  // 禁用的 pass 列表
  mutable Optional<DiagnosticContext> diag_ctx; // 诊断信息相关
  Map<String, ObjectRef> config;  // Pass specific configurations
  Array<instrument::PassInstrument> instruments;  // pass instrument implementations
  mutable Array<ObjectRef> trace_stack; // Trace stack for relax pass infra
  Optional<Map<String, Bool>> make_traceable; // passes to be traced
  mutable int num_evals{0}; // Number of evaluations conducted in the pass pipeline
  Optional<ObjectRef> tuning_api_database;  // Database for tuning API
  
  static constexpr const char* _type_key = "transform.PassContext";
  TVM_DECLARE_FINAL_OBJECT_INFO(PassContextNode, Object);
};
```

顾名思义，是多个 Pass 执行过程中的共同上下文， 其中的 `instruments` 是提供给开发者的一个工具，开发者可以实现一些函数运行在每个Pass的运行前后或者其他时机，这些函数打包到一起称为 `PassInstrument` 注册到 `PassContext` 中。

通过 `PassContext::Current()` 可以获得一个 thread local 的当前生效的 `PassContext` ，也可以通过类似 Python `with` 的语法覆盖当前生效的 PassContext, 如下

```c++
auto new_ctx = PassContext::Create();
ctx->opt_level = 2;
With<PassContext> scope(ctx);
// pass context in effect.
```

为了方便，TVM 中实现了三个类别的 Pass：

1. Module-Level

    > Module级别 pass 旨在实现全局分析/优化，即过程间优化（IPO）等，类似于 LLVM 中的 module pass。
    > 
    > Relay 中一些需要 Module 全局图的典型 pass，如 A-normal form 转换和 lambda 提升等，都属于这个集合。在这个级别，用户甚至可以在 module 中添加和/或删除功能。此级别的 pass 可以完全控制给定的 relay 程序，包括添加和删除函数。

    ```c++
    class ModulePassNode : public PassNode {
     public:
      PassInfo pass_info;
      // `pass_func` 描绘了真正的优化。 例如: 我们可能需要在 module 级别进行无用代码消除， 那么
      // 我们可以在 `pass_func` 中实现算法并让它在 module 上运行。它将删除死代码，包括 module 中未使用的函数。
      runtime::TypedPackedFunc<IRModule(IRModule, PassContext)> pass_func;
      ModulePassNode() = default;
      // IRModule => IRModule: 返回更新后的 IRModule.
      IRModule operator()(IRModule mod, const PassContext& pass_ctx) const final;
      static constexpr const char* _type_key = "transform.ModulePass";
      TVM_DECLARE_FINAL_OBJECT_INFO(ModulePassNode, PassNode);
    };
    ```

    - Relay 中的 Module-level Pass：`InferType`, `ToBasicBlockNormalForm`, `ToANormalForm`, `ToGraphNormalForm`, `PartitionGraph`, `PartialEval`, `RemoveUnusedFunctions`, `RemoveStandaloneReshapes` 等
    - tir 中的 Module-level Pass：`VerifySSA`, `SplitHostDevice`, `ExtractPrimFuncConstants`, `MakePackedAPI` 等


2. Function-Level

    函数级 pass 用于对给定的 Relay/tir module 进行各种函数内的优化。
    它每次从 module 的函数列表中获取一个函数进行优化，并产生一个重写的 Relay Function 或 tir PrimFunc。
    大部分 pass 都可以归为这一类，比如 Relay 中常见的子表达式消除和推理简化，以及 tir 中的向量化和展平存储等。

    这个级别的 Pass 使用 `PrimFuncPassNode`(tir) 和 `FuncPassNode`(relay, relax) 来表示， 
    其中 relay 的 `FuncPassNode` 实现如下：

    ```c++
    class FunctionPassNode : public PassNode {
     public:
      PassInfo pass_info;
      // `pass_func` 描绘了真正的优化。 例如: 我们可以实现一个在 Relay 函数级别的 pass 
      // 作为 `pass_func` 并让它在 module 上运行，相同的 `pass_func` 将应用于 module 中每个函数。
      runtime::TypedPackedFunc<Function(Function, IRModule, PassContext)> pass_func;
      FunctionPassNode() = default;
      IRModule operator()(IRModule mod, const PassContext& pass_ctx) const final;
      static constexpr const char* _type_key = "relay.FunctionPass";
      TVM_DECLARE_FINAL_OBJECT_INFO(FunctionPassNode, PassNode);
    };
    ```

    - Relay 中的 Function-level Pass：`FuseOps`, `DynamicToStatic`(简单的一个例子), `ConvertLayout`, `DeFuseOps`, `DeadCodeElimination`, `FoldConstant`, `Inline`, `EliminateCommonSubexpr`(ECS), `SimplifyExpr` 等
    - tir 中的 Function-level Pass：`LowerIntrin`, `InjectPrefetch`, `StorageFlatten`, `StorageRewrite`, `LoopPartition`, `VectorizeLoop`, `UnrollLoop`, `RemoveNoOp`, `CommonSubexprElimTIR`(ECS), `Simplify`等

3. Sequential

    类似 pytorch 里面的 nn.Sequential, 包含了一堆可执行的Pass按照顺序执行。目前在 Relay 中只有少数 pass 被放入该组。
    
    例如， relay 的 `FoldScaleAxis` Pass 需要在内部调度 `ForwardFoldScaleAxis` 和 `BackwardFoldScaleAxis` 。 且应当先执行 `BackwardFoldScaleAxis` 。因此，这个 Pass 是 SequentialPass 的理想候选。


## 3. Target

已知 TVM 的 编译流程大体图下：

<div class="autocb" style="text-align:center;"><img src="./tvm-type.assets\autocb_4.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

在前面介绍了 runtime 的机制以及 IR 之后， 需要关注的就是 target， target 模块主要功能是 **Target 描述** 以及 **codgen module**，在 TVM 中使用 `Target` 来**描述代码生成的目标设备信息** (声明位于 `include/tvm/target/target.h`)：

```c++
class TargetNode : public Object {
 public:
  TargetKind kind;                  // target device 的种类
  Optional<ObjectRef> host;         // Target host 信息(必须为 Target 类型)
  String tag;                       // target 的tag，可以为空
  Array<String> keys;               // target 的 keys
  Map<String, ObjectRef> attrs;     // target 的属性合集
  Map<String, ObjectRef> features;  // Target features

  TVM_DLL Map<String, ObjectRef> Export() const;  // 导出为 JSON-like config
  static constexpr const char* _type_key = "Target";
  static constexpr const bool _type_has_method_sequal_reduce = true;
  static constexpr const bool _type_has_method_shash_reduce = true;
  TVM_DECLARE_FINAL_OBJECT_INFO(TargetNode, Object);
 private:
  mutable std::string str_repr_;  // 内部字符串表示
};
```

而 `TargetKind` 定义如下：

```c++
class TargetKindNode : public Object {
 public:
  String name;                    // 目标设备种类 的 字符串名称
  int default_device_type;        // 目标设备种类 的 device
  Array<String> default_keys;     // 目标设备种类的默认 keys
  PackedFunc preprocessor;        // preprocess on target creation
  FTVMTargetParser target_parser; // parse a JSON target during creation
  
  static constexpr const char* _type_key = "TargetKind";
  TVM_DECLARE_FINAL_OBJECT_INFO(TargetKindNode, Object);
 private:
  // 存储目标特定属性所需的 type_key 和 type_index
  struct ValueTypeInfo {
    String type_key;
    uint32_t type_index;
    std::unique_ptr<ValueTypeInfo> key;
    std::unique_ptr<ValueTypeInfo> val;
  };
  std::unordered_map<String, ValueTypeInfo> key2vtype_; // target-key's attr => type information
  std::unordered_map<String, ObjectRef> key2default_;   // target-key's attr => default value
  uint32_t index_;  // 用于属性注册表内部查找的索引
};
```

TODO: 一个例子
