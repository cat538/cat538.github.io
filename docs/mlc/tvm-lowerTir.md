# TVM-lowerTir

这篇文章用一个例子追踪 tir 是如何经过 target translation(以LLVM为例) 被翻译为一个 `runtime.Module`

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

## 1. 理解 tir 元张量函数(`PrimFunc`) 的编译过程

`tvm.build` 将代码从 IRModule 转换成 runtime.Module； 

- 其中 IRModule 定义在 `include/tvm/ir/module.h` 中； 可以简单视作一个 `<name, BaseFunc>` 的哈希表(即可同时包含`PrimFunc` 和 `Func`)
- 其中 runtime.Module 定义对应在 `include/tvm/runtime/module.h` 中； 可以简单视作一个 `<name, PackedFunc>` 的哈希表。 `ModuleNode` 是一个接口， 对应有多种不同的实现， 如 `CUDAModuleNode`， `LLVMModuleNode` 等


以使用te声明 计算过程 为例，展示一个 `te.tensor.ComputeOp` 是如何被编译成 `runtime.Module` 的：

1. 使用te声明计算过程
2. 使用 `te.create_schedule` 创建一个Schedule 对象， 接着使用 `te.lower()` 将一个 `te.Schedule` 转成一个IRModule；

    或者使用 `te.create_prim_func` 创建一个 `tvm.tir.PrimFunc` 对象(即一个元张量函数)， 然后使用 IRModule的构造函数把这个 PrimFunc 放到 这个IRModule 中

3. 接着调用 `tvm.build()` 对 IRModule进行构建， 在这一步中， IRModule 最终将被转换成一个 `runtime.Module`； 

    这里需要注意 `tvm.build` 可以接受 `Schedule`, `PrimFunc`, `IRModule` 等多种类型（实际上都会被转成 IRModule 然后再编译）作为输入， 如果传入的是一个 `Schedule` 的话， 则需要传入函数相应的输入参数列表

💡因此接下来主要看 位于 `python/driver/build_module.py` 中的 `tvm.build` 方法

---

`tvm.build()` 包括以下几个主要步骤

1. 调用 `python/driver/build_module.py` 中 `lower()`， 这是一步 IRModule 到 IRModule 的变换； 具体来说，`lower()` 中会调用 `ffi.lower_module(inp, simple_mode)`，这里对应到 C++ 代码 `src/dirver/driver_api.cc` 中的 `LowerModule` 方法：

    ```c++
    IRModule LowerModule(IRModule mod, bool simple_mode) {
        Array<transform::Pass> pass_list = CreatePassList(simple_mode);
        return LowerWithPassList(std::move(mod), pass_list);
    }
    TVM_REGISTER_GLOBAL("driver.lower_module").set_body_typed([](IRModule mod, bool simple_mode) {
        return LowerModule(std::move(mod), simple_mode);
    });
    ```

    可以看到 在 `LowerModule` 逻辑中， 首先创建了一个pass 列表，然后把列表中的pass应用到该 IRModule 中：

    ```c++
    IRModule LowerWithPassList(IRModule mod, Array<tvm::transform::Pass> pass_list) {
        auto optimize = tvm::transform::Sequential(pass_list);
        mod = optimize(std::move(mod));
        return mod;
    }
    ```

    在这段逻辑中， 首先构造了一个 `Sequential` 对象， 接着调用了 `SequentialNode::operator()`

2. 在第一步的 lower 之后，`tvm.build()` 接收的inputs参数 `Union[te.Schedule, PrimFunc, IRModule, Mapping[str, IRModule]]` 被转成了一个 IRModule， 接下来检查 target 相关信息，会根据 `tvm.build()` 的输入设置相应的 target, target_host 等变量


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

          bool overrides_host_target =
            target->GetTargetDeviceType() == target_host->GetTargetDeviceType();
          bool non_host_target_kind = target->kind != target_host->kind;
              
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
      // the build function.
      std::string build_f_name = "target.build." + target->kind->name;
      const PackedFunc* bf = runtime::Registry::Get(build_f_name);
      return (*bf)(mod, target);
    }
    ```

    可以看到 通过使用字符串拼接 拿到 `build_f_name` (例如 target 是 llvm， 这里得到就是 "target.build.llvm")， 接着用这个名字查找到已注册的函数，进行 build

4. 接下来以 llvm 后端为例 看一下如何生成的 llvm IR

    ```c++
    TVM_REGISTER_GLOBAL("target.build.llvm")
    .set_body_typed([](IRModule mod, Target target) -> runtime::Module {
      auto n = make_object<LLVMModuleNode>();
      n->Init(mod, target);
      return runtime::Module(n);
    });
    ```

    可以看到这里构造了一个 `LLVMModuleNode` 实例， 调用了它的 `Init` 方法， 因此可知该方法就是将IRModule lower 为 LLVMModuleNode 的核心函数。 关于 `LLVMModuleNode` 的定义， 可参考 [TVM-type](./tvm-type.md)，下面是 `Init` 方法的具体实现：

    ```c++
    void LLVMModuleNode::Init(const IRModule& mod, const Target& target) {
      // LLVMInstance 实例中包括一个 llvm::Module 和一个 llvm::LLVMContext 
      llvm_instance_ = std::make_unique<LLVMInstance>();
      With<LLVMTarget> llvm_target(*llvm_instance_, target);
      llvm::TargetMachine* tm = llvm_target->GetOrCreateTargetMachine();
      // 会根据 llvm_target 创建 CodeGenCPU or CodeGenNVPTX or CodeGenAMDGPU
      std::unique_ptr<CodeGenLLVM> cg = CodeGenLLVM::Create(llvm_target.get());
      std::vector<PrimFunc> funcs;
      std::string entry_func;
      relay::Runtime runtime =
          mod->GetAttr<relay::Runtime>(tvm::attr::kRuntime).value_or(relay::Runtime::Create("cpp"));
      bool system_lib = runtime->GetAttr<Bool>("system-lib").value_or(Bool(false));
      bool target_c_runtime = runtime->name == "crt";
      // 1. 遍历 IRModule 中的 PrimFunc, 函数名保存到 LLVMModuleNode 实例的 `function_names_` 中
      for (auto kv : mod->functions) {
        if (!kv.second->IsInstance<PrimFuncNode>()) 
          continue;
        auto f = Downcast<PrimFunc>(kv.second);
        auto global_symbol = f->GetAttr<String>(tvm::attr::kGlobalSymbol);
        function_names_.push_back(global_symbol.value());
        if (f->HasNonzeroAttr(tir::attr::kIsEntryFunc))
          entry_func = global_symbol.value();
        funcs.push_back(f);
      }
      // 2. 调用 CodeGenLLVM 的 Init 初始化相关信息， 将 PrimFunc 添加到 CodeGenLLVM 实例中
      cg->Init("TVMMod", llvm_target.get(), system_lib, system_lib, target_c_runtime);
      cg->SetFastMathFlags(llvm_target->GetFastMathFlags());
      cg->AddFunctionsOrdered(funcs.begin(), funcs.end());
      if (entry_func.length() != 0)
        cg->AddMainFunction(entry_func);
      // 3. 调用 Finish 方法完成 lower
      module_owning_ptr_ = cg->Finish();
      module_ = module_owning_ptr_.get();
      llvm_target->SetTargetMetadata(module_);
    }
    ```

5. 从上一步中可知， `cg->Finish()`  真正完成了 lower 的动作

    TODO: