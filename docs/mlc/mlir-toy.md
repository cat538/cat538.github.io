# MLIR-TOY Example

> **Ref**:
>
> - [MLIR的法宝：Dialects](https://zhuanlan.zhihu.com/p/102212806)
> - [【从零开始学深度学习编译器】十二，MLIR](http://giantpandacv.com/project/%E9%83%A8%E7%BD%B2%E4%BC%98%E5%8C%96/%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0%E7%BC%96%E8%AF%91%E5%99%A8/%E3%80%90%E4%BB%8E%E9%9B%B6%E5%BC%80%E5%A7%8B%E5%AD%A6%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0%E7%BC%96%E8%AF%91%E5%99%A8%E3%80%91%E5%8D%81%E4%BA%8C%EF%BC%8CMLIR%20Toy%20Tutorials%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E4%B8%80/)

## 1. Chapter1: Toy 语言和 AST

这个教程用的是kaleidoscope 的变形， 一个例子如下：

```py
# RUN: toyc-ch1 %s -emit=ast 2>&1 | FileCheck %s

# User defined function that 操作 **任意** 形状的参数（类比 function template）
def multiply_transpose(a, b) {
  return transpose(a) * transpose(b);
}

def main() {
  # `a` shape <2, 3>; 其 shape 从字面量推断而来
  var a = [[1, 2, 3], [4, 5, 6]];
  # b 与 a 完全相同, literal array is **implicitly reshaped**
  var b<2, 3> = [1, 2, 3, 4, 5, 6];

  # This call will specialize `multiply_transpose` with <2, 3> for both
  # arguments and deduce a return type of <2, 2> in initialization of `c`.
  var c = multiply_transpose(a, b);
  # A second call to `multiply_transpose` with <2, 3> for both arguments will
  # reuse the previously specialized and inferred version and return `<2, 2>`
  var d = multiply_transpose(b, a);
  # A new call with `<2, 2>` for both dimension will trigger another
  # specialization of `multiply_transpose`.
  var e = multiply_transpose(b, c);
  # call `multiply_transpose` with 不匹配的 shape will 导致一个 inference error
  var f = multiply_transpose(transpose(a), c);
}
```

执行`.\bin\toyc-ch1 ..\mlir\test\Examples\Toy\Ch1\ast.toy -emit=ast`

上面 Toy 程序产生的 AST(去掉源码位置信息):

```c++
Module:
Function
    Proto 'multiply_transpose' 
    Params: [a, b]
    Block {
    Return
        BinOp: * 
        Call 'transpose' [ var: a ]
        Call 'transpose' [ var: b ]
    } // Block
Function
    Proto 'main' 
    Params: []
    Block {
    VarDecl a<> 
        Literal: <2, 3>[ <3>[ 1.000000e+00, 2.000000e+00, 3.000000e+00], <3>[ 4.000000e+00, 5.000000e+00, 6.000000e+00]] 
    VarDecl b<2, 3> 
        Literal: <6>[ 1.000000e+00, 2.000000e+00, 3.000000e+00, 4.000000e+00, 5.000000e+00, 6.000000e+00] 
    VarDecl c<> 
        Call 'multiply_transpose' [ var: a var: b ]
    VarDecl d<> 
        Call 'multiply_transpose' [ var: b var: a ]
    VarDecl e<> 
        Call 'multiply_transpose' [ var: b var: c ]
    VarDecl f<> 
        Call 'multiply_transpose' [ 
            Call 'transpose' [ var: a ]
            var: c 
        ]
    } // Block
```



## 2. Chapter2. 生成初级 MLIR
首先说了编译器领域的软件碎片化问题：

> 像 LLVM 这种编译器， 提供了a fixed set of predefined types and (usually low-level / RISC-like) instructions。 但尽管不同的前端可以重用LLVM，生成LLVMIR这一统一的中间表示，但是 像C++ 这样的语言需要自己实现 high level AST(例如 clang AST, Rust AST)， 完成语言特定的类型检查、分析(如C++的模板特化，Rust的生命周期检查)等。 
>
> 最终每一门高级语言都要去设计实现一套自己的AST， 尽管这些AST在很多地方相似，尽管再这些AST上的优化很类似，但是彼此之间无法重用，造成软件碎片化。 
>
> MLIR 天生为可扩展、可重用而设计， MLIR中有很少的内置类型和指令， 即所谓的 (little builtin, everything customizable) 设计原则。它的愿景是提供一套构建IR的基础设施， 即使用MLIR 来构建IR， 这样不同的IR 之间有差异，但是能共用很多部分，能利用已经充分成熟的编译器领域的成果和这么多年积累下来的最佳实践，而不是去重复"发明"一些"新技术"出来。
>
> the system should encourage one to design reusable abstractions and assume they will be used outside of their initial scope.


上面提到 MLIR 被设计为可扩展的基础设施，没有封闭的属性集，类型集，操作集； MLIR 通过 方言 Dialects 的概念来支持这种可扩展性。 **一个 Dialect 就是用户(或者MLIR预定义)在一个命名空间下提供的一组抽象**， 通过 Dialect 来统一不同级别的IR。


首先看一个 Dialect 的例子： 执行 chapter2 的程序: `./bin/toyc-ch2 ../mlir/test/Examples/Toy/Ch2/codegen.toy -emit=mlir -mlir-print-debuginfo` 得到第一章中程序示例的 **Toy Dialect 表示**如下（下面的代码删除掉了 debug-info ）

```c++
module {
  toy.func @multiply_transpose(%arg0: tensor<*xf64>, %arg1: tensor<*xf64>) -> tensor<*xf64> {
    // 1. 转置两个输入矩阵
    %0 = toy.transpose(%arg0 : tensor<*xf64>) to tensor<*xf64>
    %1 = toy.transpose(%arg1 : tensor<*xf64>) to tensor<*xf64>
    // 2. 相乘
    %2 = toy.mul %0, %1 : tensor<*xf64>
    toy.return %2 : tensor<*xf64>
  }
  toy.func @main() {
    // var a<2, 3> = [[1, 2, 3], [4, 5, 6]];
    %0 = toy.constant dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>
    %1 = toy.reshape(%0 : tensor<2x3xf64>) to tensor<2x3xf64>
    // var b<2, 3> = [1, 2, 3, 4, 5, 6];
    %2 = toy.constant dense<[1.000000e+00, 2.000000e+00, 3.000000e+00, 4.000000e+00, 5.000000e+00, 6.000000e+00]> : tensor<6xf64>
    %3 = toy.reshape(%2 : tensor<6xf64>) to tensor<2x3xf64>
    // var c = multiply_transpose(a, b);
    // var d = multiply_transpose(b, a);
    %4 = toy.generic_call @multiply_transpose(%1, %3) : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64>
    %5 = toy.generic_call @multiply_transpose(%3, %1) : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64>
    // print(d);
    toy.print %5 : tensor<*xf64>
    toy.return
  }
}
```

在 MLIR 中， `Operations` 是抽象和计算的核心， MLIR 中的 instructions, functions, modules 都使用 `Operation` 来表示。 以上面 MLIR toy dialect 中的 `transpose` 操作为例，来看看 MLIR 表达式是由什么组成的： `transpose(a)` 的 MLIR 表达式由操作结果名称、Dialect命名空间、操作名、参数列表、输入参数类型、输出类型和操作在源文件中的位置组成。

```py
%t_tensor = "toy.transpose"(%tensor) {inplace = true} : (tensor<2x3xf64>) -> tensor<3x2xf64> loc("example/file/path":12:1)
```

- `%t_tensor`：这个 Operation 定义的结果的名字，前面的%是避免冲突，见 https://mlir.llvm.org/docs/LangRef/#identifiers-and-keywords 。一个 Operation 可以定义 0 或者多个结果（在 Toy 语言中，只有单结果的 Operation），它们是 SSA 值。该名称在解析期间使用，但不是持久的（例如，它不会在 SSA 值的内存表示中进行跟踪）。
- `"toy.transpose"` ：Operation 的名字。它应该是一个唯一的字符串，Dialect 的命名空间前缀为 `.`。 这可以理解为 Toy Dialect 中的 transpose Operation。
- `(%tensor)`：零个或多个输入操作数（或参数）的列表，它们是由其它操作定义或引用块参数的 SSA 值。
- `{ inplace = true }`：零个或多个属性的字典，这些属性是始终为常量的特殊操作数。 在这里，我们定义了一个名为 `inplace` 的布尔属性，它的常量值为 true。
- `(tensor<2x3xf64>) -> tensor<3x2xf64>`：函数形式表示的操作类型，前者是输入，后者是输出。<2x3xf64>号中间的内容描述了张量的尺寸2x3和张量中存储的数据类型f64，中间使用x连接。
- `loc("example/file/path":12:1)`：此操作的源代码中的位置。**需要注意的是**： MLIR 中的 loc 信息（源码位置信息）在实际应用中是不能去除的， 这与 在 LLVM等其它编译器中 中附加的 Debug 信息不同。

> In MLIR, every operation has a mandatory source location associated with it. Contrary to LLVM, where debug info locations are metadata and can be dropped, in MLIR, the location is a core requirement, and APIs depend on and manipulate it. Dropping a location is thus an explicit choice which cannot happen by mistake.
> 

💡<u>**那我们的 `codegen.toy` 中的toy语言源码是怎么样转换成上面这个 Toy Dialect 的呢**</u>：

<div class="autocb" style="text-align:center;"><img src="./mlir-toy.assets\autocb_0.png" style="zoom: 45%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

简单来说，我们遍历 toy 语言的 AST， 将对应的表达式 Expr 转换成 Toy dialect 中的 operation。就具体实现来说，从toy 语言的 AST(即 `toy::ModuleAST`) 到 MLIR 表达式(即`mlir::ModuleOp`) 的生成过程在函数 `dumpMLIR` 中实现。

> 💡在继续接下来的内容之前， 先来看一下 MLIR Toy Example 的代码结构，以第二章为例：
>
> - <u>**include**</u>
>     - **toy**
>         - `Dialect.h`:  Toy Dialect 的定义， 以及 Toy Dialect 上的 Operation 的定义；实际上该文件只是简单的包含 由 tablegen 生成的代码
>         - `MLIRGen.h`:  声明了 **从 Toy AST 生成 Toy Dialect 的接口 `mlirGen`**
>         - `Ops.td`:     **整个 Example 中最核心的部分**， 声明了 Toy Dialect， 以及 Toy Dialect 上的 Operation； 将由 mlir-tablegen 生成c++ 代码
>         - `AST.h`:      Toy 语言的 AST 定义， 来自 <u>Kaleidoscope</u>， 不需要关心
>         - `Lexer.h`:    一个 Tokenizer 的实现，来自 <u>Kaleidoscope</u>， 不需要关心
>         - `Parser.h`:   一个递归下降 Parser，来自 <u>Kaleidoscope</u>， 不需要关心
> 
> - <u>**mlir**</u>
>     - `Dialect.cpp`:  **从 Toy AST 生成 Toy Dialect 的具体实现**， 主要在 `MLIRGenImpl` class 中实现， 这个类暴露一个 `mlir::ModuleOp mlirGen(ModuleAST &moduleAST)` ；内部实现对每一种 Toy AST(如 ExprAST, FunctionAST 等) 的转换
>     - `MLIRGen.cpp`:  主要是 Toy Dialect 上的 Operation 的具体实现， 目前不是特别懂TODO； 还实现了每种 Operation 的 print 具体实现， 用于dump
> - <u>**parser**</u>
>     - `AST.cpp`:  Toy 语言的 AST 定义， 来自 <u>Kaleidoscope</u>， 不需要关心
> - `toyc.cpp`: toy 语言编译器的 cli， 实现了 `emit-MLIR` 和 `emit-AST` 等
> 

1. dumpMLIR 实现在 `examples/toy/Ch2/toys.cpp` 中：

    ```c++
    int dumpMLIR() {
      mlir::MLIRContext context;
      // 1. 将我们自定义的 toy Dialect 加载到这个 MLIR Context
      context.getOrLoadDialect<mlir::toy::ToyDialect>();

      // 2. 从 cli 读取的 inputFilename 是 '.toy' 文件时， 调用 mlirGen
      auto moduleAST = parseInputFile(inputFilename);
      mlir::OwningOpRef<mlir::ModuleOp> module = mlirGen(context, *moduleAST);

      // 3. dump
      module->dump();
      // 输入是 .mlir 等其它情况时；省略
      ...
    }
    ```

2. 从上面可以看到，关键在于 `mlirGen` 方法，其实现在 文件 `examples/toy/Ch2/mlir/MLIRGen.cpp` 的 `MLIRGenImpl::mlirGen` 方法中:

    ```c++
    /// 使用 RTTI 判断表达式类型， Dispatch 到对应的 codegen 方法
    mlir::Value mlirGen(ExprAST &expr) {
      switch (expr.getKind()) {
      case toy::ExprAST::Expr_BinOp:
        return mlirGen(cast<BinaryExprAST>(expr));
      case toy::ExprAST::Expr_Var:
        return mlirGen(cast<VariableExprAST>(expr));
      case toy::ExprAST::Expr_Literal:
        return mlirGen(cast<LiteralExprAST>(expr));
      case toy::ExprAST::Expr_Call:
        return mlirGen(cast<CallExprAST>(expr));
      case toy::ExprAST::Expr_Num:
        return mlirGen(cast<NumberExprAST>(expr));
      default:
        emitError();
        return nullptr;
      }
    }
    ```

3. codeGen 的具体过程就是遍历整个AST， 生成对应的 MLIR Operation， 插入到 `OpBuilder::block` 中。 下面是三个例子：

    ```c++
    // 1. 将 toy AST 中的 `NumberExprAST` 转成 `ConstantOp`
    mlir::Value mlirGen(NumberExprAST &num) {
      return builder.create<ConstantOp>(loc(num.loc()), num.getValue());
    }
    // 2. 将 toy AST 中的 `BinaryExprAST` 转成 `AddOp` or `MulOp`
    mlir::Value mlirGen(BinaryExprAST &binop) {
      mlir::Value lhs = mlirGen(*binop.getLHS());
      if (!lhs) return nullptr;
      mlir::Value rhs = mlirGen(*binop.getRHS());
      if (!rhs) return nullptr;
      switch (binop.getOp()) {
      case '+':
        return builder.create<AddOp>(loc(binop.loc()), lhs, rhs);
      case '*':
        return builder.create<MulOp>(loc(binop.loc()), lhs, rhs);
      }
    }
    // 3. 将 toy AST 中的 `CallExprAST` 转成 `TransposeOp` 或其它 Op
    mlir::Value mlirGen(CallExprAST &call) {
      // 首先为 operands 进行 Codegen
      SmallVector<mlir::Value, 4> operands;
      for (auto &expr : call.getArgs()) {
        auto arg = mlirGen(*expr);
        operands.push_back(arg);
      }
      // Builtin calls 
      if (call.getCallee() == "transpose") {
        return builder.create<TransposeOp>(loc(call.loc()), operands[0]);
      }
      // 否则 this is a call to a user-defined function.
      ...
    }
    ```

    过程中生成的 Operation 会按顺序被 insert 到 `mlir::OpBuilder::block` 中。 最终遍历生成的 MLIR Operation， 执行打印，就得到了上面打出来的 MLIR 表达式。 这里的问题是 `AddOp` `ConstantOp` 等 Op 是 toy dialect 中的内容， 因此我们需要实现定义这些数据结构， 那么具体该如何利用 MLIR 这个框架，去定义我们自己的 dialect 呢？


### 2.1. 定义一个 Toy Dialect
接下这部分是 官方文档翻译：

> 💡**定义一个 Toy Dialect**
> 
> 为了有效地与MLIR交互，我们将定义一个新的toy方言。这种方言将模拟toy语言的结构，并为高级分析和转换提供一条简单的途径。
>
> ```c++
> /// Toy dialect 继承自 mlir::Dialect
> /// 并且 在构造函数中 registers 一系列自定义的 属性、操作、类型
> /// 它也可以通过重写一些 virtual methods 改变一些 general behavior
> /// (在下一个 chapter 中将会讨论)
> class ToyDialect : public mlir::Dialect {
>  public:
>   explicit ToyDialect(mlir::MLIRContext *ctx);
> 
>   static llvm::StringRef getDialectNamespace() { return "toy"; }
> };
> ```
>
> 现在可以在全局注册表中注册该方言：
>
> ```c++
> mlir::registerDialect<ToyDialect>();
> ```
>
> **从现在开始创建的任何新的 `MLIRContext` 都将包含toy方言的一个实例**，并调用特定的 hooks 来解析属性和类型。
>
> ---
> 
> 💡**定义一个 Toy Dialect 上的 Operation**
> 
> 有了 toy dialect，我们就可以开始注册该 Dialect 上的操作了。这将允许提供语义信息给剩余系统调用。以 `toy.constant` 操作的创建过程为例：
>
> ```c++
> %4 = "toy.constant"() {value = dense<1.0> : tensor<2x3xf64>} : () -> tensor<2x3xf64>
> ```
>
> `toy.constant` 操作没有参数，有一个名为value 的 [dense elements](https://mlir.llvm.org/docs/Dialects/Builtin/#denseintorfpelementsattr)属性用来表示constant value，返回一个 [RankedTensorType](https://mlir.llvm.org/docs/Dialects/Builtin/#rankedtensortype)。`ConstantOp` 继承自 CRTP `mlir::op`，该类还需要一些可选的 Traits(`mlir::OpTrait`) 来约束其行为。这些特征可以提供额外的accessors, verification等功能。
>
>
> ```c++
> class ConstantOp : public mlir::Op<ConstantOp,
>                      /// ConstantOp takes no inputs.
>                      mlir::OpTrait::ZeroOperands,
>                      /// ConstantOp returns a single result.
>                      mlir::OpTrait::OneResult,
>                      /// result of getType is `Type`.
>                      mlir::OpTrait::OneTypedResult<Type>::Impl> {
> 
>  public:
>   /// Inherit the constructors from the base Op class.
>   using Op::Op;
> 
>   /// 为这个 operation 提供一个 unique name
>   /// MLIR 使用这个名字在全局注册该 Operation
>   static llvm::StringRef getOperationName() { return "toy.constant"; }
> 
>   /// 返回 constant value by fetching it from the attribute.
>   mlir::DenseElementsAttr getValue();
> 
>   /// Operations 除了 traits 还可提供额外的 verification；
>   /// 这里我们将确保 specific invariants of the constant
>   /// operation are upheld. 例如， 返回类型必须是 TensorType.
>   LogicalResult verify();
>
>   /// 提供从input values 构建该 operation 的接口
>   /// builder 将会使用这个接口，从而能够简单地生成该 operation 实例:
>   ///   ``mlir::OpBuilder::create<ConstantOp>(...)``
>   /// This method populates the given `state` that MLIR uses to create
>   /// operations. This state is a collection of all of the discrete elements
>   /// that an operation may contain.
>   /// Build a constant with the given return type and `value` attribute.
>   static void build(mlir::OpBuilder &builder, mlir::OperationState &state,
>                     mlir::Type result, mlir::DenseElementsAttr value);
>   /// Build a constant and reuse the type from the given 'value'.
>   static void build(mlir::OpBuilder &builder, mlir::OperationState &state,
>                     mlir::DenseElementsAttr value);
>   /// Build a constant 广播 given 'value'.
>   static void build(mlir::OpBuilder &builder, mlir::OperationState &state,
>                     double value);
> };
> ```
>
> 并且我们在Toy Dialect构造函数中注册此操作：
>
> ```c++
> ToyDialect::ToyDialect(mlir::MLIRContext *ctx)
>    : mlir::Dialect(getDialectNamespace(), ctx) {
>   addOperations<ConstantOp>();
> }
> ```

不过在这里我们也可以看到，为了适配各种接口，定义一个 `ConstantOp` 已经十分复杂了。 MLIR 为了解决这种复杂性， 提供了一个基于 TableGen 的 Operation Definition Specification (ODS) Framework， 用户通过ODS 声明 dialect 及其 Operations， MLIR 框架通过`mir-tblgen` 工具将该声明式语言自动转化为C++ 代码， 简化了用户自定义 dialect 的流程。 

这种方式是 MLIR 推荐的定义 Dialect 的方式， 在 Toy 语言中我们的 toy Dialect，也是通过这种方法来定义的。具体的文件在 `mlir/examples/toy/Ch2/include/toy/Ops.td` ， 

```c++
def Toy_Dialect : Dialect {
  let name = "toy";
  let cppNamespace = "::mlir::toy";
  let emitAccessorPrefix = kEmitAccessorPrefix_Prefixed;
}

// Inherits from the base `Op` class in `OpBase.td`
// And provides:
//   * Toy operation 的 parent dialect(即 toy dialect).
//   * Toy operation 的 mnemonic(助记词)， or the name without the dialect prefix.
//   * A list of traits for the operation.
class Toy_Op<string mnemonic, list<Trait> traits = []> :
    Op<Toy_Dialect, mnemonic, traits>;
```

其中 `ConstantOp` 的定义如下：

```rust
// ConstantOp(具体的 Operation) 继承 'Toy_Op'.
// 这里为 constant operation 提供了一个助记词 和一个 traits 列表.
// Constant operation 有 trait 'NoSideEffect' 因为它是 pure operation, so that may be removed if dead.
def ConstantOp : Toy_Op<"constant", [NoSideEffect]> {
  // summary and description. 可被用于自动生成文档
  let summary = "constant";
  let description = [{
    Constant operation turns a literal into an SSA value. The data is attached
    to the operation as an attribute. For example:

    ```mlir
      %0 = toy.constant dense<[[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]>
                        : tensor<2x3xf64>
    ```
  }];

  // The constant operation 接收一个属性作为唯一的输入
  let arguments = (ins F64ElementsAttr:$value);

  // The constant operation 返回一个 TensorType 类型的值
  let results = (outs F64Tensor);

  // Indicate that the operation has a custom parser and printer method.
  let hasCustomAssemblyFormat = 1;

  // Add custom build methods for the constant operation. These method populates
  // the `state` that MLIR uses to create operations, i.e. these are used when
  // using `builder.create<ConstantOp>(...)`.
  let builders = [
    // Build a constant with a given constant tensor value.
    OpBuilder<(ins "DenseElementsAttr":$value), [{
      build($_builder, $_state, value.getType(), value);
    }]>,

    // Build a constant with a given constant floating-point value.
    OpBuilder<(ins "double":$value)>
  ];

  // https://mlir.llvm.org/docs/Tutorials/Toy/Ch-2/#verifying-operation-semantics
  let hasVerifier = 1;
}
```

可以注意到这里 与上面的 C++ 代码相比缺少了 `ZeroOperands` and `OneResult` traits;

接下来我们使用 `mlir-tblgen` 工具自动将 `.td` 描述文件转成 C++ 代码：

1. `mlir-tblgen -gen-dialect-decls ../mlir/example/toy/Ch2/include/toy/Ops.td -I ../mlir/include`

    生成了`ToyDialect` 的c++代码定义：

    ```c++
    /*===- TableGen'erated file -------------------------------------*- C++ -*-===*\
    |*                                                                            *|
    |* Dialect Declarations                                                       *|
    |*                                                                            *|
    |* Automatically generated file, do not edit!                                 *|
    |*                                                                            *|
    \*===----------------------------------------------------------------------===*/

    namespace mlir {
    namespace toy {

    class ToyDialect : public ::mlir::Dialect {
      explicit ToyDialect(::mlir::MLIRContext *context);

      void initialize();
      friend class ::mlir::MLIRContext;
    public:
      ~ToyDialect() override;
      static constexpr ::llvm::StringLiteral getDialectNamespace() {
        return ::llvm::StringLiteral("toy");
      }
    };
    } // namespace toy
    } // namespace mlir
    MLIR_DECLARE_EXPLICIT_TYPE_ID(::mlir::toy::ToyDialect)
    ```

    这些代码在 `Dialect.cpp` 中 被包含进去: `#include "toy/Dialect.cpp.inc"`

2. `mlir-tblgen -gen-op-defs ../mlir/example/toy/Ch2/include/toy/Ops.td -I ../mlir/include`

    类似，生成 Operations 的定义， 以 `ConstantOp`为例， 生成的代码在 `toy/Ops.cpp.inc` 和 `toy/Ops/h/inc` 中，太长不放在这里



## 3. Chapter 3: High-level 分析和转换
MLIR 中使用 表达式匹配 和 重写 来完成 MLIR 分析/转换。这个教程中分别介绍使用 C++ 模式匹配和重写以及基于 [DRR 框架](https://mlir.llvm.org/docs/DeclarativeRewrites/) 来定义重写规则，然后使用 ODS 框架来自动生成代码。

这里以 toy 语言中的一个连续两次转置的程序为例：

```py
def transpose_transpose(x) {
  return transpose(transpose(x));
}
```

不做转换的情况下生成的 toy dialect 如下：

```go
func @transpose_transpose(%arg0: tensor<*xf64>) -> tensor<*xf64> {
  %0 = toy.transpose(%arg0 : tensor<*xf64>) to tensor<*xf64>
  %1 = toy.transpose(%0 : tensor<*xf64>) to tensor<*xf64>
  toy.return %1 : tensor<*xf64>
}
```

如果将这个程序使用 C++ 写， 大概长这个样子：

```c++
void sink(void *);
void double_transpose(int A[N][M]) {
  int B[M][N];
  for(int i = 0; i < N; ++i) {
    for(int j = 0; j < M; ++j) {
       B[j][i] = A[i][j];
    }
  }
  for(int i = 0; i < N; ++i) {
    for(int j = 0; j < M; ++j) {
       A[i][j] = B[j][i];
    }
  }
  sink(A);
}
```

这个程序至少对于现在的 clang 而言， 无法将两次连续的转置消除。我们接下来看一下如何利用 MLIR 中的 模式匹配和重写机制实现 IR 的转换

### 3.1. 使用 C++ 写 pass
可以直接写 C++ 来重写 IR，我们可以通过实现 `RewritePattern` 插入 MLIR Canonicalizer pass：

```c++
/// 折叠 `transpose(transpose(x))` -> `x`
struct SimplifyRedundantTranspose : public mlir::OpRewritePattern<TransposeOp> {
  /// We register this pattern to match every toy.transpose in the IR.
  /// The "benefit" is used by the framework to order the patterns and process
  /// them in order of profitability.
  SimplifyRedundantTranspose(mlir::MLIRContext *context)
      : OpRewritePattern<TransposeOp>(context, /*benefit=*/1) {}

  /// 这个方法尝试匹配固定的模式(即两个连续的转置)，并进行重写. 
  matchAndRewrite(TransposeOp op,
                  mlir::PatternRewriter &rewriter) const override {
    // Look through the input of the current transpose.
    mlir::Value transposeInput = op.getOperand();
    TransposeOp transposeInputOp = transposeInput.getDefiningOp<TransposeOp>();

    // Input defined by another transpose? If not, no match.
    if (!transposeInputOp)
      return failure();

    // Otherwise, we have a redundant transpose. Use the rewriter.
    rewriter.replaceOp(op, {transposeInputOp.getOperand()});
    return success();
  }
};
```

上述代码位于 `examples/toy/Ch3/mlir/ToyCombine.cpp`， 除了重写规则之外， 还需要将该规则添加到 规范化框架 (Canonicalization Framework) :

```c++
/// 将我们的 SimplifyRedundantTranspose patterns 加入到 "canonicalization" 重写集合中
/// 从而执行 Canonicalize pass 时，会执行到这里
void TransposeOp::getCanonicalizationPatterns(RewritePatternSet &results,
                                              MLIRContext *context) {
  results.add<SimplifyRedundantTranspose>(context);
}

```

将表达式重写规则添加到了规范化框架后，我们还需要修改一下定义 `TransposeOp` 的 `.td` 文件，启用规范化框架(添加 `let hasCanonicalizer=1`)，同时给 `TransposeOp` 的定义添加一个 `NoSideEffect` 的 trait， 现在 `Transpose` 操作的定义如下：

```rust
def TransposeOp : Toy_Op<"transpose", [NoSideEffect]> {
  let summary = "transpose operation";
  let arguments = (ins F64Tensor:$input);
  let results = (outs F64Tensor);
  let assemblyFormat = [{
    `(` $input `:` type($input) `)` attr-dict `to` type(results)
  }];

  // Enable registering canonicalization patterns with this operation.
  let hasCanonicalizer = 1;

  let builders = [
    OpBuilder<(ins "Value":$input)>
  ];
  let hasVerifier = 1;
}
```

我们还需要更新 `toyc.cpp`， 以添加优化 pass。在 MLIR 中，优化以类似于 LLVM 的方式通过 PassManager 运行：

```c++
mlir::PassManager pm(module->getName());
pm.addNestedPass<mlir::toy::FuncOp>(mlir::createCanonicalizerPass());
```

然后运行 `./bin/toyc-ch3 ../mlir/test/Examples/Toy/Ch3/transpose_transpose.toy -emit=mlir -opt` 得到

```go
module {
  // 注意对比这里优化后 与 之前未优化
  toy.func @transpose_transpose(%arg0: tensor<*xf64>) -> tensor<*xf64> {
    toy.return %arg0 : tensor<*xf64>
  }
  toy.func @main() {
    %0 = toy.constant dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>
    %1 = toy.generic_call @transpose_transpose(%0) : (tensor<2x3xf64>) -> tensor<*xf64>
    toy.print %1 : tensor<*xf64>
    toy.return
  }
}
```

这里需要注意， 如果我们不修改 `.td` 中 `TransposeOp` 的 trait（添加一个 `NoSideEffect`）， 则会出现以下结果：

```go
toy.func @transpose_transpose(%arg0: tensor<*xf64>) -> tensor<*xf64> {
  %0 = toy.transpose(%arg0 : tensor<*xf64>) to tensor<*xf64>
  toy.return %arg0 : tensor<*xf64>
}
```

这是因为， 我们的模式用函数输入替换了最后一个变换，并留下了现在死掉的转置输入。 Canonicalizer 知道清理死操作；然而，MLIR 保守地假设操作可能有副作用。而在为 `TransposeOp` 添加一个 `NoSideEffect` 可以解决这个问题。 这个 trait 在 `include/mlir/Interfaces/SideEffectInterfaces.td` 中定义 TODO: 之后可以研究一下 在 MLIR 中怎么定义 并使用 trait

### 3.2. 使用 DRR 写 pass
MLIR 还提供了一种基于 DDR 规则的方式来自动生成 模式匹配 和 重写函数，代码生成的部分仍然基于 ODS 框架实现。Declarative, rule-based pattern-match and rewrite (DRR) 它是一种基于 DAG 的声明式重写器，提供基于 tablegen 的模式匹配和重写规则的句法：

```c++
class Pattern<
    dag sourcePattern, list<dag> resultPatterns,
    list<dag> additionalConstraints = [],
    dag benefitsAdded = (addBenefit 0)>;
```

接下来以消除 MLIR 表达式中冗余的张量 reshape 操作为例:

```py
def main() {
  var a<2,1> = [1, 2];
  var b<2,1> = a;
  var c<2,1> = b;
  print(c);
}
```

对应的 toy dialect 如下：

```go
module  {
  func @main() {
    %0 = toy.constant dense<[1.000000e+00, 2.000000e+00]> : tensor<2xf64>
    %1 = toy.reshape(%0 : tensor<2xf64>) to tensor<2x1xf64>
    %2 = toy.reshape(%1 : tensor<2x1xf64>) to tensor<2x1xf64>
    %3 = toy.reshape(%2 : tensor<2x1xf64>) to tensor<2x1xf64>
    toy.print %3 : tensor<2x1xf64>
    toy.return
  }
}
```

a，b，c 完全相同， 我们有机会在 dialect 中消除冗余的 `reshape` 。 下面我们要基于 DDR 框架来定义匹配和消除规则。

类似于 `SimplifyRedundantTranspose` 的针对 冗余 `reshape` 的优化可以使用 DRR 更简单地表示如下：

```rust
// Reshape(Reshape(x)) -> Reshape(x)
def ReshapeReshapeOptPattern : Pat<(ReshapeOp(ReshapeOp $arg)),
                                   (ReshapeOp $arg)>;
```

当转换以参数和结果的某些属性为条件时，DRR 还提供了一种添加参数约束的方法。一个例子是当 reshape 的参数和结果的类型是一样的，说明这个 reshape 是无用的，直接返回输入参数即可， 即 `Reshape(x) = x`

```rust
// Reshape(x) = x, where input and output shapes are identical
def TypesAreIdentical : Constraint<CPred<"$0.getType() == $1.getType()">>;
def RedundantReshapeOptPattern : Pat<
  (ReshapeOp:$res $arg), (replaceWithValue $arg),
  [(TypesAreIdentical $res, $arg)]>;
```
即当 `0.getType()` 与 `1.getType()` 相同时，使用操作数 `$arg` 代替。

同样需要将这个函数加入到

DRR代码位于 `mlir/examples/toy/Ch3/mlir/ToyCombine.td`， 生成的代码位于 `Ch3/ToyCombine.inc`

执行 `./bin/toyc-ch3 ../mlir/test/Examples/Toy/Ch3/trivial_reshape.toy -emit=mlir -opt`

得到结果如下：

```go
module {
  toy.func @main() {
    %0 = toy.constant dense<[[1.000000e+00], [2.000000e+00]]> : tensor<2x1xf64>
    toy.print %0 : tensor<2x1xf64>
    toy.return
  }
}
```

## 4. Chapter 4: 使用 Interfaces 泛化 Transformation
上面实现的重写规则有一个明显的问题：我们为 Toy Dialect 实现的 Pass 在其它的 Dialect 中没办法重用，因为是针对 Toy dialect 一些 Operation 的特化操作，如果为每种 Dialect 实现每种转化会导致大量重复代码。所以，这一节看如何在 MLIR 中 利用 Interfaces 实现泛化。

首先是 `mlir/test/Examples/Toy/Ch4/codegen.toy` 中的例子

```rust
def multiply_transpose(a, b) {
  return transpose(a) * transpose(b);
}

def main() {
  var a<2, 3> = [[1, 2, 3], [4, 5, 6]];
  var b<2, 3> = [1, 2, 3, 4, 5, 6];
  var c = multiply_transpose(a, b);
  var d = multiply_transpose(b, a);
  print(d);
}
```

其生成的 Toy Dialect：

```go
module {
  toy.func private @multiply_transpose(%arg0: tensor<*xf64>, %arg1: tensor<*xf64>) -> tensor<*xf64> {
    %0 = toy.transpose(%arg0 : tensor<*xf64>) to tensor<*xf64>
    %1 = toy.transpose(%arg1 : tensor<*xf64>) to tensor<*xf64>
    %2 = toy.mul %0, %1 : tensor<*xf64>
    toy.return %2 : tensor<*xf64>
  }
  toy.func @main() {
    %0 = toy.constant dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>
    %1 = toy.reshape(%0 : tensor<2x3xf64>) to tensor<2x3xf64>
    %2 = toy.constant dense<[1.000000e+00, 2.000000e+00, 3.000000e+00, 4.000000e+00, 5.000000e+00, 6.000000e+00]> : tensor<6xf64>
    %3 = toy.reshape(%2 : tensor<6xf64>) to tensor<2x3xf64>
    %4 = toy.generic_call @multiply_transpose(%1, %3) : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64>
    %5 = toy.generic_call @multiply_transpose(%3, %1) : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64>
    toy.print %5 : tensor<*xf64>
    toy.return
  }
}
```

💡注意到这里 `multiply_transpose` 的签名输入输出类型 都是 `tensor<*xf64>` 即**动态形状**， 或者叫 generic tensor

在这一节的优化之后(`./bin/toyc-ch4 ../mlir/test/Examples/Toy/Ch4/codegen.toy -emit=mlir -opt`)， 生成的 IR 将会变为：

```go
module {
  toy.func @main() {
    %0 = toy.constant dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>
    %1 = toy.transpose(%0 : tensor<2x3xf64>) to tensor<3x2xf64>
    %2 = toy.mul %1, %1 : tensor<3x2xf64>
    toy.print %2 : tensor<3x2xf64>
    toy.return
  }
}
```

💡先是利用到了上一节中的优化消除掉了冗余的 `reshape`；并且此时 `multiply_transpose` 被内联（原定义被删除）； 最重要的是**所有的 tensor 形状都变成了 static， IR 中 不再具有动态形状**； 并且 `%4` and `%5` 相同， 因此只被计算了一次；


Toy IR 目前操作 generic tensors， 这意味着除了使用常量初始化的 tensor， 我们不知道其它 tensor 的形状。这使 优化 和 代码生成 变得复杂。我们可以通过在 IR 中传播编译时已知的静态形状 来缓解这个问题。形状传播的难题在于如何处理 user-defined 函数调用参数和返回值都没有给出形状信息（即上面例子中的 `multiply_transpose`）：每个调用点都可以推断出不同的形状。一种可能是根据参数类型**执行符号推理**，但随着引入更多控制流，符号推理将很难泛化。另一种方法是函数特化，其中每个具有新参数形状的调用点都内联被调用函数并对其进行特化。我们对 Toy 采取的方法是内联所有函数调用，然后**执行过程内形状传播**。

我们可以编写专为 Toy Dialect 设计的内联算法，但这可能会变得相当复杂。撇开成本建模不谈，从头开始写固定模式的代码就已经很复杂了。MLIR 提供了 **Dialect 可以 接入 的通用内联算法**。在 Toy 中，我们需要做的就是为内联器提供 [Interfaces](https://mlir.llvm.org/docs/Interfaces/)。

1. 第一步是在 Toy Dialect 中定义对内联操作的约束。此信息通过 [dialect interface](https://mlir.llvm.org/docs/Interfaces/#dialect-interfaces) 提供。 dialect interface 本质上是包含一组 virtual hooks 的类，方言通过重写这些方法提供具体实现。在这个内联例子中， 我们用到的 Interface 是 `DialectInlinerInterface`

    ```c++
    /// 这个类定义了用于 Toy Dialect 执行 inline 操作的接口
    /// 这里的实现做了简化，并且只重写了必要的方法
    struct ToyInlinerInterface : public DialectInlinerInterface {
      using DialectInlinerInterface::DialectInlinerInterface;

      /// 在我们的例子中 simply return true, 因为 Toy `Call` operation 总是可内联的
      bool isLegalToInline(Operation *call, Operation *callable, bool wouldBeCloned) const final {
        return true;
      }

      /// 在我们的例子中 simply return true, 因为 Toy operation 总是可内联的
      bool isLegalToInline(Operation *, Region *, bool, IRMapping &) const final {
        return true;
      }

      /// 同上
      bool isLegalToInline(Region *dest, Region *src, bool wouldBeCloned, IRMapping &valueMapping) const final {
        return true;
      }

      /// 当一个 terminator operation 被 inline 后执行此 hook， 保证控制流的正确性（不会提前返回）
      /// Toy 中唯一的 terminator Operation是 `toy.return`
      void handleTerminator(Operation *op, ArrayRef<Value> valuesToRepl) const final {
        // 处理 "toy.return"
        auto returnOp = cast<ReturnOp>(op);

        // 简单的将 return 语句替换成 return 语句的 operands
        assert(returnOp.getNumOperands() == valuesToRepl.size());
        for (const auto &it : llvm::enumerate(returnOp.getOperands()))
          valuesToRepl[it.index()].replaceAllUsesWith(it.value());
      }
    };
    ```

    此外，内联器只会 删除 私有可见的未使用函数定义。因此还需要在 `mlirGen` 中设置函数（`main`除外）为 private

    ```c++
    mlir::toy::FuncOp mlirGen(FunctionAST &funcAST) {
      ...
      if (funcAST.getProto()->getName() != "main")
        function.setPrivate();
      ...
    }
    ```

2. 然后我们直接在 Toy Dialect 上注册我们的 Dialect interface；在文件 `mlir/examples/toy/Ch5/mlir/Dialect.cpp` 中:

    ```c++
    void ToyDialect::initialize() {
      addOperations<
    #define GET_OP_LIST
    #include "toy/Ops.cpp.inc"
          >();
      addInterfaces<ToyInlinerInterface>();
    }
    ```

    这里的 `addInterfaces<ToyInlinerInterface>()` 就是注册内联 Pass 的过程，其中`ToyInlinerInterface` 就是我们定义的表达式变形规则。

3. 接下来，我们需要提供一种方法让内联器知道 `toy.generic_call` 代表一个调用， `toy.func` 代表一个函数。 MLIR 提供了 [operation interface](https://mlir.llvm.org/docs/Interfaces/#attributeoperationtype-interfaces)， 可用于将 operation 标记为 `call-like` 或 `callable-like` (与 dialect interface 不同， operation interface 提供更细粒度的信息)， 这里为 GenericCallOp 添加 `CallOpInterface` ， 为 FuncOp `CallableOpInterface` 
  
    要添加此接口，我们只需将定义 include 到我们的 operation 描述文件 (`Ops.td`) 中：

    ```py
    include "mlir/Interfaces/CallInterfaces.td"

    ...

    def FuncOp : Toy_Op<"func",
        [DeclareOpInterfaceMethods<CallableOpInterface>]> {
      ...
    }

    def GenericCallOp : Toy_Op<"generic_call",
        [DeclareOpInterfaceMethods<CallOpInterface>]> {
      ...
    }
    ```

4. 还需要在 Dialect 定义中添加 cast 操作并设置调用的接口。为什么需要添加 cast 操作呢？这是因为在函数调用时，输入张量的类型是确定的。但在函数定义的时候，输入张量的类型是 generic 的。即我们需要`tensor<2x3xf64> -> tensor<*xf64>` 。下面在 `mlir/examples/toy/Ch5/include/toy/Ops.td` 中添加 cast 操作：

    ```rust
    def CastOp : Toy_Op<"cast", [
        DeclareOpInterfaceMethods<CastOpInterface>,
        NoSideEffect,
        SameOperandsAndResultShape
      ]> {
      let summary = "shape cast operation";
      let description = [{ ... }];

      let arguments = (ins F64Tensor:$input);
      let results = (outs F64Tensor:$output);

      let assemblyFormat = "$input attr-dict `:` type($input) `to` type($output)";
    }
    ```

    我们使用了 `DeclareOpInterfaceMethods` 在 `CallOpInterface` 的声明中声明所用的接口方法。 `DeclareOpInterfaceMethods` 这个 trait 说明程序会识别 cast 操作。

    接下来还需要重写 cast op 的 `areCastCompatible` 方法（在 `mlir/examples/toy/Ch5/mlir/Dialect.cpp`中）：

    ```c++
    bool CastOp::areCastCompatible(TypeRange inputs, TypeRange outputs) {
      if (inputs.size() != 1 || outputs.size() != 1)
        return false;
      // The inputs must be Tensors with the same element type.
      TensorType input = inputs.front().dyn_cast<TensorType>();
      TensorType output = outputs.front().dyn_cast<TensorType>();
      if (!input || !output || input.getElementType() != output.getElementType())
        return false;
      // The shape is required to match if both types are ranked.
      return !input.hasRank() || !output.hasRank() || input == output;
    }
    ```
    这个方法用来判断是否需要进行类型转换，如果 inputs 和 outputs 的类型是兼容返回真，否则需要进行类型转换（cast）返回假。

    另外我们还需要重写 `ToyInlinerInterface` 上的 hook ， 即 `materializeCallConversion` 函数：

    ```c++
    struct ToyInlinerInterface : public DialectInlinerInterface {
      ....
      /// 尝试在函数调用的地方插入 cast 节点. 如果不能转换 返回 nullptr
      Operation *materializeCallConversion(OpBuilder &builder, Value input,
                                          Type resultType,
                                          Location conversionLoc) const final {
        return builder.create<CastOp>(conversionLoc, resultType, input);
      }
    };
    ```
    这个函数是内联 Pass 的入口。

5. 将 Inline Pass 添加到优化 pipline 中，在 `mlir/examples/toy/Ch5/toyc.cpp` 中：

    ```c++
    if (enableOpt) {
        mlir::PassManager pm(&context);
        // Apply any generic pass manager command line options and run the pipeline.
        applyPassManagerCLOptions(pm);

        // 将所有的 call 插入 main 并且删除之
        pm.addPass(mlir::createInlinerPass());
    ...
    }
    ```

此时得到的 toy dialect 如下：

```go
func @main() {
  %0 = "toy.constant"() {value = dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>} : () -> tensor<2x3xf64>
  %1 = "toy.constant"() {value = dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>} : () -> tensor<2x3xf64>
  %2 = "toy.cast"(%1) : (tensor<2x3xf64>) -> tensor<*xf64>
  %3 = "toy.cast"(%0) : (tensor<2x3xf64>) -> tensor<*xf64>
  %4 = "toy.transpose"(%2) : (tensor<*xf64>) -> tensor<*xf64>
  %5 = "toy.transpose"(%3) : (tensor<*xf64>) -> tensor<*xf64>
  %6 = "toy.mul"(%4, %5) : (tensor<*xf64>, tensor<*xf64>) -> tensor<*xf64>
  toy.print %6 : tensor<*xf64>
  toy.return
}
```

现在 MLIR 表达式只有一个主函数，之前的 `transpose` 函数被内联了，并且可以看到 `toy.cast` 实现的功能。

### 4.2. 形状传播 pass
现在我们已经内联了所有函数，剩下的是一个包含静态和动态形状操作混合的主函数。我们现在可以编写一个简单的形状推理过程在单个函数内播形状。我们可以将其写成针对 Toy Dialect pass，然而经验法则告诉我们，最好尽可能通用地表达转换，以便将来可以扩展到其他方言。

与 Operation 类似，我们也可以使用 ODS 框架来定义 Operation Interfaces

1. 使用 ODS 框架定义 Shape 推断 Operation Interface， 代码在 `mlir/examples/toy/Ch5/include/toy/ShapeInferenceInterface.td`

    ```rust
    def ShapeInferenceOpInterface : OpInterface<"ShapeInference"> {
      let description = [{
        Interface to access a registered method to infer the return types for an
        operation that can be used during type inference.
      }];

      let methods = [
        InterfaceMethod<"Infer and set the output shape for the current operation.",
                        "void", "inferShapes">
      ];
    }
    ```

    其中的 `method` 表示需要为该接口提供的方法 ， 三个字段分别表示 desc， return type， method name
2. 将 定义的 `ShapeInferenceOpInterface` 添加到必要的 Toy Operation 定义中

    这里我们给 `AddOp`, `MulOp`, `CastOp` 和 `TranposeOp` 加上 ShapeInferenceOpInterface：

    ```Rust
    def CastOp : Toy_Op<"cast", [
        DeclareOpInterfaceMethods<CastOpInterface>,
        // 这里添加 ShapeInferenceOpInterface
        DeclareOpInterfaceMethods<ShapeInferenceOpInterface>, 
        NoSideEffect,
        SameOperandsAndResultShape
      ]> {...}
    ```

    所有添加了 `ShapeInferenceOpInterface` 的 Op 都需要实现接口`method`字段中定义的的 `inferShapes` 方法：

    ```c++
    // cast op 的形状推断函数
    void CastOp::inferShapes() { getResult().setType(getOperand().getType()); }

    // transpose op 的 形状推断函数
    void TransposeOp::inferShapes() {
      auto arrayTy = getOperand().getType().cast<RankedTensorType>();
      SmallVector<int64_t, 2> dims(llvm::reverse(arrayTy.getShape()));
      getResult().setType(RankedTensorType::get(dims, arrayTy.getElementType()));
    }
    ```


3. 实现形状推导（即形状传播） Pass

    `ShapeInferencePass` 将对函数进行操作：它将在每个函数上独立运行。 MLIR 还支持在任何 单独的 Operation 上运行的通用 `OperationPass`，但这里我们的 module仅包含函数，因此无需泛化到所有操作。

    ```c++
    struct ShapeInferencePass 
     : public mlir::PassWrapper<ShapeInferencePass, OperationPass<toy::FuncOp>> {
      MLIR_DEFINE_EXPLICIT_INTERNAL_INLINE_TYPE_ID(ShapeInferencePass)
    
    void runOnOperation() override {
      auto f = getOperation();

      // 0. 将所有需要 形状推导 的Op(返回 dynamic shape 的 Op) 加入 worklist
      llvm::SmallPtrSet<mlir::Operation *, 16> opWorklist;
      f.walk([&](mlir::Operation *op) {
        if (returnsDynamicShape(op))
          opWorklist.insert(op);
      });

      // 在 worklist 上迭代， 直到所有的 operations 都被推导（worklist 为空）
      while (!opWorklist.empty()) {
        // 1. 找到下一个需要推导的 operation —— 这个 operation 的所有参数形状应该都已经被推导
        auto nextop = llvm::find_if(opWorklist, allOperandsInferred);

        // 2. 没有找到说明已经收敛， 结束算法
        if (nextop == opWorklist.end()) break;

        // 3. 从 worklist 中删除 该 op， 并调用该 op 的 inferShapes 方法 推导形状
        Operation *op = *nextop;
        opWorklist.erase(op);
        
        // 3.1. op 实现了 `ShapeInference` 接口上的方法 `inferShapes` 在这里被调用
        if (auto shapeOp = dyn_cast<ShapeInference>(op)) {
          shapeOp.inferShapes();
        } else {
          op->emitError("operation doesnt have shape inference interface");
          return signalPassFailure();
        }
      }
      
      // 4. 跳出循环后， 如果 operation worklist 不为空, 意味着算法失败.
      if (!opWorklist.empty()) {
        f.emitError("Shape inference failed, ")
            << opWorklist.size() << " operations couldn't be inferred\n";
        signalPassFailure();
      }
    }
    }
    ```

4. 把形状推导 Pass 加到优化 pipline

    同样的，我们在最后把 shapeinference pass 添加到我们的 passmanager 中：

    ```c++
    optPM.addPass(mlir::toy::createShapeInferencePass());
    ``` 

    MLIR 代码中还添加了一个 CSE pass， 这样就得到了本章开始的最终结果。

