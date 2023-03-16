# TVM-lowering
> Ref:
>
> - [TVM 自底向上（二）：TIR 的概念和编译原理](https://zhuanlan.zhihu.com/p/533161438)
> - [【从零开始学深度学习编译器】四，解析TVM算子](https://mp.weixin.qq.com/s?__biz=MzA4MjY4NTk0NQ==&mid=2247494373&idx=1&sn=6eb14998e8cfe45a144ba1b8c61346ac&chksm=9f835073a8f4d96555ed6382f25d8b5793e7b56da0f403867b7a38c48bbeedc23701c2e59721&scene=178&cur_album_id=1799364124980609027#rd)
> - [【从零开始学深度学习编译器】六，TVM的编译流程详解](http://giantpandacv.com/project/%E9%83%A8%E7%BD%B2%E4%BC%98%E5%8C%96/%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0%E7%BC%96%E8%AF%91%E5%99%A8/%E3%80%90%E4%BB%8E%E9%9B%B6%E5%BC%80%E5%A7%8B%E5%AD%A6%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0%E7%BC%96%E8%AF%91%E5%99%A8%E3%80%91%E5%85%AD%EF%BC%8CTVM%E7%9A%84%E7%BC%96%E8%AF%91%E6%B5%81%E7%A8%8B%E8%AF%A6%E8%A7%A3/)
> - [TVM 学习指南（个人版）](https://mp.weixin.qq.com/s/NM5yvxW2JSbR06RmrR3ubw)


## 1. Lower Tir

> TIR一般认为是 target ir，TensorIR是后面又提的新概念，做为tvm script的一部分。

这节用一个例子追踪 tir 表示的 PrimFunc 是如何经过 target translation 被翻译为特定target的一个 `runtime.Module`

使用的例子代码如下：

```py
import tvm
from tvm import te
SIZE = 1024

# 1. 使用 TE 声明计算
# mm
k = te.reduce_axis((0, SIZE), 'k')
A = te.placeholder((SIZE, SIZE), name='A')
B = te.placeholder((SIZE, SIZE), name='B')
C = te.compute((SIZE, SIZE), lambda x, y: te.sum(A[x, k] * B[k, y], axis=k), name='MM')
# relu
D_IN = te.placeholder((SIZE, SIZE), name='RELU_IN')
D_OUT = te.compute((SIZE, SIZE), lambda i, j: te.max(D_IN[i, j], 0), "RELU_OUT")

# 2. 从 TE 创建 PrimFunc 
te_mm: tvm.tir.PrimFunc = te.create_prim_func([A, B, C]).with_attr({"global_symbol": "mmult"})
te_relu: tvm.tir.PrimFunc = te.create_prim_func([D_IN, D_OUT]).with_attr({"global_symbol": "relu"})

# 3. 将创建出来的 PrimFunc 添加到一个 IRModule 中
ir_m: tvm.IRModule = tvm.IRModule({'mm': te_mm})
gv_relu = tvm.ir.GlobalVar("relu")
ir_m.update_func(gv_relu, te_relu)
print(ir_m.get_global_vars())
# 这里也可以使用 先构建 Schedule， 再调用 `tvm.lower` 的方法 得到一个 IRModule
# te_sch: te.Schedule = te.create_schedule(C.op) # C.op: te.tensor.ComputeOp
# ir_m: tvm.IRModule = tvm.lower(te_sch, [A, B, C], simple_mode=True,name='mmult')

# 4. Lower and Build
''' 在这一步将 IRModule ==> runtime.Module'''
rt_m: tvm.runtime.Module = tvm.build(ir_m, target='llvm', name='func_set')

print(f"\033[31mtvm.build\033[0m: {type(ir_m)} \033[31m==>\033[0m {type(rt_m)}")

# print IRModule， 其中包含一个 PrimFunc
# print(f"\033[31mAfter lowering, the IRModule\033[0m:\n{ir_m}")
ir_m.show()

# # print source code
# print(f"\033[31msource code\033[0m:\n{rt_m.get_source()}")
```


`tvm.build` 将代码从 IRModule 转换成 runtime.Module； 

- 其中 IRModule 定义在 `include/tvm/ir/module.h` 中； 可以简单视作一个 `<name, BaseFunc>` 的哈希表(即可同时包含`PrimFunc` 和 `Func`)
- 其中 runtime.Module 定义对应在 `include/tvm/runtime/module.h` 中； 可以简单视作一个 `<name, PackedFunc>` 的哈希表。 `ModuleNode` 是一个接口， 对应有多种不同的实现， 如 `CUDAModuleNode`， `LLVMModuleNode` 等


以使用te声明 计算过程 为例，展示一个 `te.tensor.ComputeOp` 是如何被编译成 `runtime.Module` 的：

1. 使用te声明计算过程
2. 使用 `te.create_schedule` 创建一个Schedule 对象， 接着使用 `te.lower()` 将一个 `te.Schedule` 转成一个IRModule；

    或者使用 `te.create_prim_func` 创建一个 `tvm.tir.PrimFunc` 对象(即 MLC 课程 中所说的一个**元张量函数**)， 然后使用 IRModule 的构造函数把这个 PrimFunc 放到IRModule 中

3. 接着调用 `tvm.build()` 对 IRModule进行构建： 在这一步中， IRModule 最终将被转换成一个 `runtime.Module`； 

    这里需要注意 `tvm.build` 可以接受 `Schedule`, `PrimFunc`, `IRModule` 等多种类型（实际上都会被转成 IRModule 然后再编译）作为输入， 如果传入的是一个 `Schedule` 的话， 则需要传入函数相应的输入参数列表

💡**因此接下来主要看 位于 `python/driver/build_module.py` 中的 `tvm.build` 方法**

---

`tvm.build()` 包括以下几个主要步骤

```py
def build(
    inputs: Union[te.Schedule, PrimFunc, IRModule, Mapping[str, IRModule]],
    args: Optional[List[Union[Buffer, tensor.Tensor, Var]]] = None,
    target: Optional[Union[str, Target]] = None,
    target_host: Optional[Union[str, Target]] = None,
    runtime: Optional["tvm.relay.backend.Runtime"] = None,
    name: Optional[str] = "default_function",
    binds: Optional[Mapping[tensor.Tensor, Buffer]] = None,
):
    if isinstance(inputs, tvm.IRModule):
      input_mod = lower(inputs)
    if not isinstance(inputs, (dict, container.Map)):
      target = Target.current() if target is None else target
      target = target if target else "llvm"
      target_input_mod = {target: input_mod}
    else:
      target_input_mod = inputs

    annotated_mods = {}
    for tar, mod in target_input_mod.items():
      annotated_mods[tar] = mod.with_attr("runtime", runtime)
    
    # 省略 get target_host 的代码
    target_host = "llvm"

    annotated_mods, target_host = Target.canon_target_map_and_host(annotated_mods, target_host)
    rt_mod_host = _driver_ffi.tir_to_runtime(annotated_mods, target_host)
    annotated_mods, target_host = Target.canon_target_map_and_host(annotated_mods, target_host)

    to_return = rt_mod_host
    return OperatorModule.from_module(to_return, ir_module_by_target=annotated_mods, name=name)
```

1. 调用 `python/driver/build_module.py` 中 `lower()`， 这是一步作用是执行 IRModule（or schedule等） 到 IRModule 的变换，执行 tir 层级的 target 无关优化。 具体来说，`lower()` 中会调用 `ffi.lower_module(inp, simple_mode)`，这里对应到 C++ 代码 `src/dirver/driver_api.cc` 中的 `LowerModule` 方法：

    ```c++
    IRModule LowerModule(IRModule mod, bool simple_mode) {
        Array<transform::Pass> pass_list = CreatePassList(simple_mode);
        return LowerWithPassList(std::move(mod), pass_list);
    }
    TVM_REGISTER_GLOBAL("driver.lower_module").set_body_typed([](IRModule mod, bool simple_mode) {
        return LowerModule(std::move(mod), simple_mode);
    });
    ```

    可以看到 在 `LowerModule` 逻辑中， 首先创建了一个pass 列表， 包含了`tir::transform`中的一些变换， 然后调用 `LowerWithPassList` 把列表中的 pass 应用到该 IRModule 中：即首先将 `pass_list` 构造成一个 `Sequential` 对象， 然后调用 `SequentialNode::operator()` ， 

2. 在第一步的 lowering 之后，`tvm.build()` 接收的inputs参数 `Union[te.Schedule, PrimFunc, IRModule, Mapping[str, IRModule]]` 被转成了一个 经过初步优化的 IRModule ， 并执行了 tir 级别的 target independent 优化。 **接下来就是执行代码生成的工作**， 首先检查 target 相关信息，会根据 `tvm.build()` 的输入设置相应的 target, target_host 等变量

3. 在拿到 target 等规范格式的信息之后，会执行`rt_mod_host = _driver_ffi.tir_to_runtime(annotated_mods, target_host)` ，这里调用的 `tir_to_runtime` 对应到C++中的 `TIRToRuntime`:

    ```c++
    TVM_REGISTER_GLOBAL("driver.tir_to_runtime")
        .set_body_typed([](const Map<Target, IRModule>& inputs_arg, Target host_target) {
            return TIRToRuntime(inputs_arg, host_target);
        });
    ```

    比较关键的 `TIRToRuntime` 逻辑如下：

    ```c++
    runtime::Module TIRToRuntime(const Map<Target, IRModule>& inputs_arg, const Target& target_host_arg) {
      std::vector<runtime::Module> device_modules;
      Map<Target, IRModule> inputs = inputs_arg;
      Target target_host = target_host_arg;
      // 1. 检查并设置target_host
      // ...
      // 2. 遍历输入的 {Target: IRModule}
      //  SplitMixedModule 将一个 IRModule 分为 host 与 device 两部分
      //  根据 host 与 device 信息更新 mhost_all
      IRModule mhost_all = IRModule(Map<GlobalVar, BaseFunc>(),{},{},{},(*inputs.begin()).second->attrs);
      for (const auto& it : inputs) {
        if (it.second.defined()) {
          const auto& [target, ir_module] = it;
          auto& [host_mod, device_mod] = SplitMixedModule(ir_module, target, target_host);

          bool overrides_host_target = target->GetTargetDeviceType() == target_host->GetTargetDeviceType();
          bool non_host_target_kind  = target->kind != target_host->kind;
          if (overrides_host_target && non_host_target_kind)
            device_modules.push_back(codegen::Build(host_mod, target));
          else
            mhost_all->Update(host_mod);

          if (device_mod->functions.size() != 0)
            device_modules.push_back(codegen::Build(device_mod, target));
        }
      }
      // 3. Build host module
      runtime::Module mhost = codegen::Build(mhost_all, target_host);
      for (const auto& it : device_modules)
        if (it.operator->())
          mhost.Import(it);
      return mhost;
    }
    ```

    这里比较关键的 `codegen::build` 在`src/target/codegen.cc` 中， 实现如下：

    ```c++
    runtime::Module Build(IRModule mod, Target target) {
      auto target_attr_map = tvm::TargetKind::GetAttrMap<FTVMTIRToRuntime>("TIRToRuntime");
      if (target_attr_map.count(target->kind))
        return target_attr_map[target->kind](mod, target);
      
      std::string build_f_name = "target.build." + target->kind->name;
      const PackedFunc* bf = runtime::Registry::Get(build_f_name);
      return (*bf)(mod, target);
    }
    ```

    可以看到 通过使用字符串拼接 拿到 `build_f_name` (例如 target 是 llvm， 这里得到就是 "target.build.llvm"， cuda 就是 "target.build.cuda")， 接着用这个名字查找到已注册的函数，进行 build

4. 接下来分别以 llvm 后端（不设置runtime） 和 cuda 后端为例 看一下如何生成的 llvm IR or CUDA 代码

    ```c++
    // `src/target/llvm/llvm_module.cc`
    TVM_REGISTER_GLOBAL("target.build.llvm")
    .set_body_typed([](IRModule mod, Target target) -> runtime::Module {
      auto n = make_object<LLVMModuleNode>();
      n->Init(mod, target);
      return runtime::Module(n);
    });
    // `src/target/opt/build_cuda_on.cc`
    TVM_REGISTER_GLOBAL("target.build.cuda").set_body_typed(BuildCUDA);
    ```

    首先看CUDA的 tir => bin 编译流程：
    
    ```c++
    runtime::Module BuildCUDA(IRModule mod, Target target) {
      // Step 1: Initialize CodeGen for CUDA  
      bool output_ssa = false;
      CodeGenCUDA cg;
      cg.Init(output_ssa);
      
      // Step 2: Add all tir::PrimFunc in IRModule to compile list
      for (auto kv : mod->functions) {
        auto f = Downcast<PrimFunc>(kv.second);
        cg.AddFunction(f);
      }

      // Step 3: Lower IRModule to CUDA source code
      std::string code = cg.Finish();

      // Step 4: Compile CUDA source code using NVCC and create runtime::Module
      std::string fmt = "ptx";
      std::string ptx = NVRTCCompile(code, cg.need_include_path());
      return CUDAModuleCreate(ptx, fmt, ExtractFuncInfo(mod), code);
    }
    ```

    **并行线程执行（Parallel Thread eXecution，PTX）代码是编译后的GPU代码的一种 IR ， 它可以再次编译为原生的GPU指令**。PTX提供了一个稳定的编程模型和指令集，这个ISA能够跨越多种GPU，并且能够优化代码的编译等等。
    
    接下来看 LLVM，值得注意的是，如果 target="llvm"，由于 LLVM IR 仍然只是一种中间表示，还需要根据 target 当中更详细的硬件参数，找到目标编译硬件，然后调用相应的 CodeGen（省略部分辅助代码）：
    
    ```c++
    void Init(const IRModule& mod, const Target& target) {
      // Step 1: Initialize CodeGen for LLVM with different target
      InitializeLLVM();
      tm_ = GetLLVMTargetMachine(target);
      std::unique_ptr<CodeGenLLVM> cg = CodeGenLLVM::Create(tm_.get());

      // Step 2: Add all tir::PrimFunc in IRModule to compile list
      std::vector<PrimFunc> funcs;
      for (auto kv : mod->functions) {
        if (!kv.second->IsInstance<PrimFuncNode>()) {
          // (@jroesch): we relax constraints here, Relay functions will just be ignored.
          DLOG(INFO) << "Can only lower IR Module with PrimFuncs, but got " << kv.second->GetTypeKey();
          continue;
        }
        auto f = Downcast<PrimFunc>(kv.second);
        funcs.push_back(f);
      }

      // Step 3: Lower IRModule to LLVM IR code
      module_ = cg->Finish();
    }
    ```
    
    可以看到这里构造了一个 `LLVMModuleNode` 实例， 调用了它的 `Init` 方法， 因此可知该方法就是将 IRModule lower 为 LLVMModuleNode 的核心函数。 关于 `LLVMModuleNode` 的定义， 可参考 [TVM-type](./tvm-type.md)，下面是 `Init` 方法的具体实现：


5. 从上一步中可知， TIR 能 lower 成目标源代码，关键是 CodeGen。上面提到的 CodeGenCUDA，以及 CodeGenLLVM 是完成 TIR lower 为 C++ 代码的相关结构。 其中 `CodeGenCUDA`继承自 `CodeGenC`，以其为例子说明（`tvm/src/target/source/codegen_c.[h, cc]`）：
    
    ```c++
    class CodeGenC : public ExprFunctor<void(const PrimExpr&, std::ostream&)>,
                 public StmtFunctor<void(const Stmt&)>,
                 public CodeGenSourceBase {
    public:
      // expression
      void VisitExpr_(const VarNode* op, std::ostream& os) override;         // NOLINT(*)
      void VisitExpr_(const LoadNode* op, std::ostream& os) override;        // NOLINT(*)
      void VisitExpr_(const BufferLoadNode* op, std::ostream& os) override;  // NOLINT(*)
      void VisitExpr_(const LetNode* op, std::ostream& os) override;         // NOLINT(*)
      ...
      // statment
      void VisitStmt_(const LetStmtNode* op) override;
      void VisitStmt_(const StoreNode* op) override;
      void VisitStmt_(const BufferStoreNode* op) override;
      void VisitStmt_(const ForNode* op) override;
      void VisitStmt_(const WhileNode* op) override;
      void VisitStmt_(const IfThenElseNode* op) override;
      ...
    }
    ```

    可以看到， `CodeGenC` 会遍历到两种 TIR Node：Expression（表达式） 和 Statement（语句）。Expression（表达式）中包含了常见的变量声明、运算、判断、函数调用，而 Statement（语句）中包含了控制流（if-else，Loop 等）、内存管理、赋值等操作。

    例如，遇到四则运算的 Expression，CodeGenC 直接翻译为 " a OP b "的代码：

    ```c++
    template <typename T>
    inline void PrintBinaryExpr(const T* op, const char* opstr,
                                std::ostream& os, CodeGenC* p) {
      // If both a and b are scalars
      if (op->dtype.lanes() == 1) {
        // If OP is an alphabet string, then lower it as "OP(a, b)"
        if (isalpha(opstr[0])) {
          os << opstr << '(';
          p->PrintExpr(op->a, os);
          os << ", ";
          p->PrintExpr(op->b, os);
          os << ')';
        }
        // If OP is a symbol, like + - * / %, then lower it as "a OP b"
        else {
          os << '(';
          p->PrintExpr(op->a, os);
          os << ' ' << opstr << ' ';
          p->PrintExpr(op->b, os);
          os << ')';
        }
      }
      // If both a and b are vectors
      else {
        p->PrintVecBinaryOp(opstr, op->dtype, op->a, op->b, os);
      }
    }

    void CodeGenC::VisitExpr_(const AddNode* op, std::ostream& os) {  // NOLINT(*)
      PrintBinaryExpr(op, "+", os, this);
    }
    void CodeGenC::VisitExpr_(const SubNode* op, std::ostream& os) {  // NOLINT(*)
      PrintBinaryExpr(op, "-", os, this);
    }
    void CodeGenC::VisitExpr_(const MulNode* op, std::ostream& os) {  // NOLINT(*)
      PrintBinaryExpr(op, "*", os, this);
    }
    void CodeGenC::VisitExpr_(const DivNode* op, std::ostream& os) {  // NOLINT(*)
      PrintBinaryExpr(op, "/", os, this);
    }
    ```

    如果遇到选择 SelectNode，CodeGenC 则翻译为 "(c ? a : b)" 的代码：

    ```c++
    void CodeGenC::VisitExpr_(const SelectNode* op, std::ostream& os) {
      os << "(";
      PrintExpr(op->condition, os);
      os << " ? ";
      PrintExpr(op->true_value, os);
      os << " : ";
      PrintExpr(op->false_value, os);
      os << ")";
    }
    ```

## 2. Lower TE
上一节中我们使用 TE(Tensor Expression) 声明计算过程， 构造了一个 PrimFunc。 在TVM 中， PrimFunc 还可以通过编写 TVMScript， 或者通过 relay/relax lower 而来。 

TE是位于 Relay IR / TOPI 和 TIR 之间概念， 是TVM 提供的一种用于编写 Primfunc 的 EDSL， 其抽象程度比 TIR 更高，无法直接被编译为硬件源代码，必须先 lower 为 TIR 的 Primitive Function 再进行编译。

在这一节中我们关注使用 te 描述的计算过程是如何被转换成一个 PrimFunc 的，即对应到 [Lower Tir](#1-lower-tir) 中提到的 `te.create_prim_func`， 也就是官方给出的 编译流程中的红框中的部分：

<div class="autocb" style="text-align:center;"><img src="./tvm-lowering.assets\autocb_0.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

测试代码与第一节中代码相同，只不过此时我们关注更上层的部分：

```py
import tvm
from tvm import te
SIZE = 1024

# 1. 使用 TE 声明计算
# mm
A = te.placeholder((SIZE, SIZE), name='A')  # te.tensor.Tensor
B = te.placeholder((SIZE, SIZE), name='B')  # te.tensor.Tensor
k = te.reduce_axis((0, SIZE), 'k')          # tir.expr.IterVar
C = te.compute((SIZE, SIZE), lambda x, y: te.sum(A[x, k] * B[k, y], axis=k), name='MM') # te.tensor.Tensor

# 2. 从 TE 创建 PrimFunc 
te_mm: tvm.tir.PrimFunc = te.create_prim_func([A, B, C]).with_attr({"global_symbol": "mmult"})

# 3. 将创建出来的 PrimFunc 添加到一个 IRModule 中
ir_m: tvm.IRModule = tvm.IRModule({'mm': te_mm})

# 这里也可以使用 先构建 Schedule， 再调用 `tvm.lower` 的方法 得到一个 IRModule
te_sch: te.Schedule = te.create_schedule(C.op) # C.op: te.tensor.ComputeOp
ir_m: tvm.IRModule = tvm.lower(te_sch, [A, B, C], simple_mode=True,name='mmult')

print(ir_m.get_global_vars())
```

在关注 lowering 之前首先看一下一个 TE 的构成

1. 首先是 `te.placeholder` 将返回一个 `te.tensor.Tensor`。 其数据结构定义位于 `include/tvm/te/operation.h`中， 具体如下：

    ```c++
    class PlaceholderOpNode : public OperationNode {
     public:
      Array<PrimExpr> shape;
      DataType dtype;

      int num_outputs() const final{return 1;};
      Array<PrimExpr> output_shape(size_t i) const final{return shape;};
      Array<Tensor> InputTensors() const final{return {};};

      static constexpr const char* _type_key = "PlaceholderOp";
      TVM_DECLARE_BASE_OBJECT_INFO(PlaceholderOpNode, OperationNode);
    };
    ```

    placeholder 通常用于计算图的 Input 节点使用，没有前序节点，可能也是te中最常用的Op。 除了`PlaceholderOp` 之外， TE 中还有一些其他的 Op 定义，例如后面会提到的 `ComputeOp`， 以及 `ExternOp` 等， 这些 Op 是 TE AST 的组成部分。

2. 接下来是 `te.compute` ， compute 从一个或者多个前序节点接收数据，并按初始化的时候传入的 lambda 表达式计算 Tensor 内的数据。其API 第一个传入参数是 Tensor 的 shape，第二个参数是一个 lambda 表达式，表明 Tensor 的数据是如何计算来的，返回一个`te.tensor.Tensor` 。 其 C++ 部分数据结构定义具体如下：

    ```c++
    class TVM_DLL ComputeOpNode : public BaseComputeOpNode {
    public:
      Array<PrimExpr> body; //compute expression
      // override functions
      int num_outputs() const final;
      Array<Tensor> InputTensors() const final;

      static constexpr const char* _type_key = "ComputeOp";
      TVM_DECLARE_FINAL_OBJECT_INFO(ComputeOpNode, BaseComputeOpNode);
    };
    ```
    
    可以看到其关键成员只有一个 body，是一个 PrimExpr 数组，因此接下来看 python 端是怎样把一个 lambda 表达式表达的计算转换成了 PrimExpr：

    ```py
    def compute(shape, fcompute, name="compute", tag="", attrs=None, varargs_names=None):
      out_ndim = len(shape)

      argspec = inspect.getfullargspec(fcompute)
      arg_names = argspec.args
      
      # 1. 将传入的 shape 转成 tir.IterVar 列表
      dim_var = [tvm.tir.IterVar((0, s), x, 0) for x, s in zip(arg_names, shape[:out_ndim])]
      
      # 2. 构造计算的 AST
      body = fcompute(*[v.var for v in dim_var])

      # 3. 调用 C++ API 获取 compute_op_node
      body = convert(body) # 将List[PrimExpr] 转成 Array<PrimExpr>
      op_node = _ffi_api.ComputeOp(name, tag, attrs, dim_var, body)

      return op_node.output
    ```

    这里的关键点在于 `IterVar` 继承了 `ExprOp`（位于 `python/tvm/tir/expr.py`）， 而 `ExprOp` 重载了四则运算，移位运算，比较运算等操作符，因此可以直接被 lambda 表达式操作，调用重载的运算符返回表示运算之后的表达式，例如：

    ```py
    Z = te.compute((n,), lambda i: X[i]*Y[i])
    ```

    首先对于 `X[i]`和`Y[i]` 会分别构造一个 `tir.expr.ProducerLoad` 表达式，接着这两个表达式的乘法会调用到 `ExprOp` 重载的乘法运算符：

    ```py
    def multiply(lhs, rhs, span=None):
      return _ffi_api._OpMul(lhs, rhs, span)
    ```

    返回一个 `tir.Mul` Expr；其它的运算类似。**值得注意的是** TE 中也提供了控制类 PrimExpr 的封装，例如 `te.if_then_else`；以及 `te.extern_primfunc` 用其它PrimFunc 来创建 PrimExpr

3. 在构造表示计算的 AST 之后，我们把这个从lambda构造出来的 PrimExpr 数组传入了`_ffi_api.ComputeOp`， 返回一个 ComputeOp 类型， `ComputeOp` 是 `te::Operation` 的一个子类，因此可以从其`output`方法拿到计算结果 `Tensor` 作为 `te.compute` 的返回结果

💡**在这里总结一下 TE 的构成关键元素**： 即 `Tensor` 和 `Operation` 两个类型。前者表征一个张量的形状、元素类型、source operation等；后者表示产生 Tensor 的一种操作，包括表示输入的操作 PlaceholderOp， 表示计算逻辑的操作 ComputeOp 等。

<div class="autocb" style="text-align:center;"><img src="./tvm-lowering.assets\autocb_1.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>


---

接下来我们开始关注 `te.create_prim_func` 是如何 将 TE lower 到 PrimFunc 的，对应到`src/te/operation/create_prim_func.cc` ：

```c++
PrimFunc CreatePrimFuncWithConstants(const Array<te::Tensor>& arg_list,
                                     const Array<runtime::NDArray>& constants,
                                     const Optional<Array<tir::Var>>& tir_var_list,
                                     std::optional<DataType> index_dtype_override) {
  // Infomations used in CreatePrimFunc and its sub-functions.
  CreateFuncInfo info(arg_list);
  // Root body stmts.
  Array<Stmt> root_stmts;
  // Analyzer
  arith::Analyzer analyzer;

  // Step 1. 后续遍历计算图： placeholder被放到前面，最终computeOp在最后
  Array<te::Operation> order = CollectOrderedOps(arg_list);

  // Step 2. 这段逻辑是处理 TE 中后来引入的 ExternOp，在本例中无需考虑
  InitializeBufferBinds(order, &info);

  // Step 3. 如果是 placeholder，创建对应的 tir.Buffer；如果是compute，创建对应的 tir.Stmt；如果是 extern，则创建一个 tir.Stmt 节点
  for (const te::Operation& op : order) {
    RewriteStageToBlock(op, &info, &root_stmts, &analyzer);
  }

  // 创建并返回 PrimFunc
  auto func = GenerateAndCompletePrimFunc(arg_list, root_stmts, &info, tir_var_list);

  return func;
}
```

回顾下 PrimFunc 的结构：

```c++
class PrimFuncNode : public BaseFuncNode {
 public:
  Array<tir::Var> params;
  tir::Stmt body;
  Type ret_type;
  Map<tir::Var, Buffer> buffer_map;

  static constexpr const char* _type_key = "tir.PrimFunc";
  TVM_DECLARE_FINAL_OBJECT_INFO(PrimFuncNode, BaseFuncNode);
};
```

`GenerateAndCompletePrimFunc` 会将 `arg_list`(`Array<te::Tensor>`) 转成一个 `Array<tir::Var>` 作为函数的 params； 而函数 body 和 `buffer_map` 已从建立的计算图中得到；因此到这里我们就完成了从 TE 到 `PrimFunc` 的转换。

💡总结整个过程，最核心的部分在于后续遍历 TE 表示的计算图（Operation作为节点，通过 `InputTensors` 方法拿到所有入边，做 DFS），转换成对应的 `tir.Buffer` 和 `tir.Stmt` 即 tir 中的 AST


## 3. Lower Relay
这一节关注一个更高层次的抽象， TVM 中图级别 IR Relay 的 lowering

<div class="autocb" style="text-align:center;"><img src="./tvm-lowering.assets\autocb_2.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

从这张图可以看到， Relay 要到 TIR 有2条路径，第一条就是直接到 TIR， 比如 PrimExpr 派生的节点 IntImmNode 可以直接映射到 TIR ，另外一条就是 Relay 里面类似 Conv 的 Op 的计算逻辑是用 TOPI 来表达的， TOPI 是 TVM 中用 TE 来表示常用算子的预定义库。

### 3.1. TOPI
考虑这个例子：

```py
n = te.var("i", dtype="int32")
m = te.var("j", dtype="int32")
A = te.placeholder((n, m), name="A")
k = te.reduce_axis((0, m), "k")
B = te.compute((n,), lambda i: te.sum(A[i, k], axis=k), name="B")

# 使用 TOPI: 这一句与上面两句等价
C = topi.sum(A, axis=1) 

# 可以利用上一节的代码， 将 TE lower 到 tir 查看表示：
prim_func1 = te.create_prim_func([A,B])
prim_func2 = te.create_prim_func([A,C])
print(f"\033[31m{'='*20}prim_func1{'='*20}\033[0m\n{prim_func1}")
print(f"\033[31m{'='*20}prim_func2{'='*20}\033[0m\n{prim_func2}")

# Build 之后运行结果当然也是相同的：
prim_func1 = te.create_prim_func([A,B])
prim_func2 = te.create_prim_func([A,C])

lib1, lib2 = tvm.build(prim_func1), tvm.build(prim_func2)

a_np = np.random.rand(128, 1024).astype("float32")
expected_res = np.sum(a_np, axis=1)
a_nd1, a_nd2 = tvm.nd.array(a_np), tvm.nd.array(a_np)
test_res1 = tvm.nd.empty((128,), dtype="float32")
test_res2 = tvm.nd.empty((128,), dtype="float32")

lib1(a_nd1, test_res1)
lib2(a_nd2, test_res2)
np.testing.assert_allclose(expected_res, test_res1.numpy(), rtol=1e-5)
np.testing.assert_allclose(expected_res, test_res2.numpy(), rtol=1e-5)
print("🎉\033[32mTest Pass...\033[0m")
```

可以看到，对于求和这样常见的简单操作，如果只使用 te 的话，也需要先定义一个 reduce_axis, 再用 lambda 风格去声明，比较繁琐，更不必说在神经网络中大量使用的其它更复杂一点的算子，如果用 te 去写的话，会比较麻烦。 因此 TVM 提供了一些使用 TE 定义的现成的算子，把他们放到一起（以及一些 schedule 操作），称作 TOPI(Tensor operator inventory)

TOPI 中包含了如 卷积， softmax， 矩阵乘 等常见的算子，可以帮助我们更好的表达 DL 中常见的计算过程。

在看具体实现前可以首先观察一下 TOPI 的目录结构，包含了 `nn.cc`，`elemwise.cc`，`schedule.cc`等，分别对应了机器学习常见算子，element wise 运算（如上面例子中的 `topi.sum`）， 算子调度；在这些文件中，使用 `TVM_REGISTER_GLOBAL` 注册了 TOPI 中的具体实现。


💡这里我们以 `topi.matmul` 为例看一下实现。


`topi.matmul` 对应 C++ 注册在`src/topi/nn.cc` 中，实现在`include/tvm/topi/nn/dense.h`中，简化后如下：

```c++
inline tvm::te::Tensor matmul(const tvm::te::Tensor& A, const tvm::te::Tensor& B,bool trans_a = false, bool trans_b = false,std::string name = "T_matmul", std::string tag = kMatMul) {
  tvm::Array<tvm::PrimExpr> output_shape{A->shape[trans_a ? 1 : 0], B->shape[trans_b ? 0 : 1]};
  auto k = tvm::te::reduce_axis(tvm::Range{0, A->shape[trans_a ? 0 : 1]}, "k");
  auto l = [&](tvm::tir::Var i, tvm::tir::Var j) {
    return tvm::sum((trans_a ? A[k][i] : A[i][k]) * (trans_b ? B[j][k] : B[k][j]), {k});
  };
  return tvm::te::compute(output_shape, l, name, tag);
}
```

可以看到其实 `dense` 跟我们手写的没有什么不同，但是对于 logsoftmax 之类有实现技巧的算子，TOPI 有很好的实现并且有相应的调度可以使用。

> TOPI 中有些算子在C++ 和 python 端分别实现了一次，这样做的原因见： https://discuss.tvm.apache.org/t/why-topi-is-implemented-in-both-c-and-python/50/7
>
> 简单来说，当一个算子地调度方式稳定之后，会被放在C++部分， 正在dev地放在python实现；而对于一些简单的算子，就保留了在 C++ 和 py 两部分的实现


### 3.2. Relay
本节使用如下例子：

```py
import numpy as np
import tvm
from tvm import relay

# 1. 定义 Relay Var
data = relay.var("data", shape=(1,784),dtype="float32")
weight1 = relay.var("weight1",shape=(128,784),dtype="float32")
bias1 = relay.var("bias1",shape=(128,),dtype="float32")
weight2 = relay.var("weight2",shape=(10,128),dtype="float32")
bias2 = relay.var("bias2",shape=(10,),dtype="float32")

# 2. 使用 Relay Op 定义网络结构
dense1 = relay.nn.dense(data,weight1)
bias_add1 = relay.nn.bias_add(dense1, bias1)
relu1 = relay.nn.relu(bias_add1)
dense2 = relay.nn.dense(relu1,weight2)
bias_add2 = relay.nn.bias_add(dense2, bias2)
relu2 = relay.nn.relu(bias_add2)

# 3. 构建网络
func = relay.Function([data,weight1,bias1,weight2,bias2],relu2)
print(f"\033[31m{'='*20}relay func{'='*20}\033[0m\n{func}")

# 4. 构建 IRModule
ir_mod = tvm.IRModule.from_expr(func)

# 5. 调用 relay.build 进行构建
# relay.build 目前暂时支持第一个参数传入 `relay.Function`， 但是之后将只支持 IRModule
with tvm.transform.PassContext(opt_level=2):
  rt_lib = relay.build(ir_mod=ir_mod,target="llvm")

```

这里以 `dense` 算子为例， 首先关注 `dense1 = relay.nn.dense(data,weight1)` ：

```py
def dense(data, weight, units=None, out_dtype=""):
  return _make.dense(data, weight, units, out_dtype)
```

在 C++ 端使用 RELAY_REGISTER_OP 机制统一管理 Relay Op， `dense`算子的注册位于 `src/relay/op/nn/nn.cc` 中：

```c++
Expr MakeDense(Expr data, Expr weight, IndexExpr units, DataType out_dtype) {
  auto attrs = make_object<DenseAttrs>();
  attrs->units = units;
  attrs->out_dtype = out_dtype;
  static const Op& op = Op::Get("nn.dense");
  return Call(op, {data, weight}, Attrs(attrs), {});
}

TVM_REGISTER_GLOBAL("relay.op.nn._make.dense").set_body_typed(MakeDense);

RELAY_REGISTER_OP("nn.dense")
    .describe(/*省略*/)
    .set_attrs_type<DenseAttrs>()
    .set_num_inputs(2)
    .add_argument("data", "nD Tensor", "Input data.")
    .add_argument("weight", "2D Tensor", "Weight matrix.")
    .set_support_level(1)
    .set_attr<FInferCorrectLayout>("FInferCorrectLayout", DenseInferCorrectLayout)
    .add_type_rel("Dense", MatmulRel<DenseAttrs>)
    .set_attr<TOpPattern>("TOpPattern", kOutEWiseFusable);
```

可以看到最终返回的就是一个 `CallNode`， Op 是 `dense`。

同时注意到 `RELAY_REGISTER_OP("nn.dense")` 只是设置了输入输出 type 关系，以及其它的一些相关属性， 没有指定 `dense` 这个算子具体的计算和调度， 那么一个 Relay 算子的计算和调度在哪里指定呢？

答案是在 python 端， 以`dense`为例， 其调度和计算的指定位于 `python/tvm/relay/op/nn/_nn.py` 中， 该文件会在 `import relay` 时被自动加载：

```py
# dense
reg.register_strategy("nn.dense", strategy.dense_strategy)
```

`strategy.dense_strategy` 定义如下：

```py
@override_native_generic_func("dense_strategy")
def dense_strategy(attrs, inputs, out_type, target):
  """dense generic strategy"""
  logger.warning("dense is not optimized for this platform.")
  strategy = _op.OpStrategy()
  strategy.add_implementation(
    wrap_compute_dense(topi.nn.dense),
    wrap_topi_schedule(topi.generic.schedule_dense),
    name="dense.generic",
  )
  return strategy
```

其中 `topi.nn.dense` 和 `topi.generic.schedule_dense` 就是 上一节 TOPI 中预定义好的 TE 表示的算子库中的算子。 而 `_op.OpStrategy` 对应 C++ 中的数据结构位于 `include/tvm/relay/op_strategy.h` 中：

```c++
class OpImplementationNode : public Object {
 public:
  FTVMCompute fcompute;
  FTVMSchedule fschedule;
  String name;
  int plevel;

  static constexpr const char* _type_key = "relay.OpImplementation";
  TVM_DECLARE_FINAL_OBJECT_INFO(OpImplementationNode, Object);
};
```

其中 `FTVMCompute` 是 `runtime::TypedPackedFunc<Array<te::Tensor>(const Attrs& attrs, const Array<te::Tensor>& inputs, const Type& out_type)>;` ， FTVMSchedule 是 `runtime::TypedPackedFunc<te::Schedule(const Attrs& attrs, const Array<te::Tensor>& outs, const Target& target)>;`；

💡**这样就将 TOPI 和 Relay Op 建立了联系**

接下来关注 `relay.build`

---

先暂时忽略 `PassContext(opt_level=2)`， 直接看 `relay.build`：

```py
def build(ir_mod,target=None, target_host=None,
          executor=Executor("graph"), runtime=Runtime("cpp"),
          workspace_memory_pools=None, constant_memory_pools=None,
          params=None, mod_name="default",
):
  raw_targets = Target.canon_multi_target_and_host(Target.target_or_current(target), target_host)
  target_host = raw_targets[0].host

  # 如果当前 dispatch context 是 fallback context (the default root context),
  # 从 TopHub 中 load pre-tuned parameters
  if isinstance(autotvm.DispatchContext.current, autotvm.FallbackContext):
    tophub_context = autotvm.tophub.context(list(raw_targets))
  else:
    tophub_context = autotvm.utils.EmptyContext()

  with tophub_context:
    bld_mod = BuildModule()
    graph_json, runtime_mod, params = bld_mod.build(
      mod=ir_mod, target=raw_targets, params=params,
      executor=executor, runtime=runtime,
      workspace_memory_pools=workspace_memory_pools,
      constant_memory_pools=constant_memory_pools,
      mod_name=mod_name,
    )
    # build 之后 bld_mod 中包含 IRModule（其中为 PrimFunc）
    func_metadata = bld_mod.get_function_metadata()
    devices = bld_mod.get_devices()
    lowered_ir_mods = bld_mod.get_irmodule()
    executor_codegen_metadata = bld_mod.get_executor_codegen_metadata()
    # 这里只看 graph_executor 去掉了aot 相关
    if executor.name == "graph":
      executor_factory = _executor_factory.GraphExecutorFactoryModule(
        ir_mod, raw_targets,
        executor, graph_json, runtime_mod,
        mod_name, params, func_metadata,
      )
    return executor_factory
```

函数返回一个 `tvm.relay.backend.executor_factory.ExecutorFactoryModule` ， 是relay的 graph executor factory（目前有 graph 和 aot 两种）

1. TopHub:

    > 首先 Relay 会寻找是否有 AutoTVM 预先 Fintune 的记录，如果没有那么就使用autotvm.FallbackContext这个环境上下文信息，如果有那么接下来的所有操作都在 tophub_context 的 scope 之下 (with tophub_context:)。值得一提的是 Relay 考虑了异构情景下的代码生成，用户可以指定多个生成代码的目标 (target)。


2. 在 `tophub_context`中，创建了一个 `BuildModule` ，然后调用了 `build` 函数生成一个硬件可执行的更底层的 IR，以及包含各种必需运行时库的tvm.Module和优化后的计算图的参数。这里还有一个_graph_executor_factory.GraphExecutorFactoryModule函数，它的功能就是将上面的 IR，运行时库以及参数打包成一个tvm.Module，这样用户只需要把这个tvm.Module存下来，下次就可以省去编译过程直接在硬件上执行了。

3. `BuildModule()` 对应在 C++ 中的 `src/relay/backend/build_module.cc` 中:

    ```c++
    class RelayBuildModule : public runtime::ModuleNode {
     public:
      RelayBuildModule() = default;

      PackedFunc GetFunction(const std::string& name, const ObjectPtr<Object>& sptr_to_self) final {
        if (name == "get_graph_json") {
          return PackedFunc(
              [sptr_to_self, this](TVMArgs args, TVMRetValue* rv) { *rv = this->GetGraphJSON(); });
        } else if (name == "build") {
          return PackedFunc([sptr_to_self, this](TVMArgs args, TVMRetValue* rv) {
            this->Build(args[0], args[1], args[2], args[3], args[4], args[5], args[6], args[7]);
          });
        } else if (name == "get_irmodule") {
          return PackedFunc([sptr_to_self, this](TVMArgs args, TVMRetValue* rv) {
            *rv = this->executor_codegen_->GetIRModule();
          });
        } else if (name == "optimize") {
          return PackedFunc([sptr_to_self, this](TVMArgs args, TVMRetValue* rv) {
            *rv = this->Optimize(args[0], args[1]);
          });
        }
      }

     protected:
      // return The updated Relay IR module after optimization.
      IRModule Optimize(IRModule relay_module, const Array<Target>& raw_targets);

      // Compile a Relay IR module to runtime module. 结果保存在 `this->ret_`
      void BuildRelay(IRModule relay_module, const String& mod_name);

     protected:
      std::unique_ptr<ExecutorCodegen> executor_codegen_;
      Executor executor_;     // Executor to build for
      Runtime runtime_;       // Runtime to codegen for
      WorkspaceMemoryPools workspace_memory_pools_;
      ConstantMemoryPools constant_memory_pools_;
      std::unordered_map<std::string, runtime::NDArray> params_; // parameters
      BuildOutput ret_; // building output
      CompilationConfig config_;
    };
    ```


## 4. Lower Relax