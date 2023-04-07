# MLIR-TOY Example

> **Ref**:
>
> - [MLIR的法宝：Dialects](https://zhuanlan.zhihu.com/p/102212806)
> - [【从零开始学深度学习编译器】十二，MLIR](http://giantpandacv.com/project/%E9%83%A8%E7%BD%B2%E4%BC%98%E5%8C%96/%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0%E7%BC%96%E8%AF%91%E5%99%A8/%E3%80%90%E4%BB%8E%E9%9B%B6%E5%BC%80%E5%A7%8B%E5%AD%A6%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0%E7%BC%96%E8%AF%91%E5%99%A8%E3%80%91%E5%8D%81%E4%BA%8C%EF%BC%8CMLIR%20Toy%20Tutorials%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E4%B8%80/)

## 1. Chapter1: Toy 语言和 AST

这个教程用的是kaleidoscope 的变形， 一个例子如下：

```py
# RUN: toyc-ch1 %s -emit=ast 2>&1 | FileCheck %s

# User defined function that operates 操作未知形状的参数
def multiply_transpose(a, b) {
  return transpose(a) * transpose(b);
}

def main() {
  # `a` has shape <2, 3>; 其 shape 从字面量推断而来
  var a = [[1, 2, 3], [4, 5, 6]];
  # b 与 a 完全相同, literal array is implicitly reshaped
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


上面提到 MLIR 被设计为可扩展的基础设施，没有封闭的属性集，类型集，操作集； MLIR 通过 方言 Dialects 的概念来支持这种可扩展性。 一个 Dialect 就是用户(或者MLIR预定义)在一个命名空间下提供的一组抽象， 通过 Dialect 来统一不同级别的IR。

在 MLIR 中， Operations 是抽象和计算的核心， MLIR 中的 instructions, functions, modules 都是使用 Operation 来表示。 以 [chapter1](#chapter1-toy-语言和-ast) 中的 `transpose` 操作为例，来看看MLIR表达式是由什么组成的： transpose(a)的 MLIR 表达式由操作结果名称、Dialect命名空间、操作名、参数列表、输入参数类型、输出类型和操作在源文件中的位置组成。

```py
%t_tensor = "toy.transpose"(%tensor) {inplace = true} : (tensor<2x3xf64>) -> tensor<3x2xf64> loc("example/file/path":12:1)
```

- `%t_tensor`：这个 Operation 定义的结果的名字，前面的%是避免冲突，见 https://mlir.llvm.org/docs/LangRef/#identifiers-and-keywords 。一个 Operation 可以定义 0 或者多个结果（在 Toy 语言中，只有单结果的 Operation），它们是 SSA 值。该名称在解析期间使用，但不是持久的（例如，它不会在 SSA 值的内存表示中进行跟踪）。
- `"toy.transpose"` ：Operation 的名字。它应该是一个唯一的字符串，Dialect 的命名空间前缀为 “.”。 这可以理解为 Toy Dialect 中的 transpose Operation。
(%tensor)：零个或多个输入操作数（或参数）的列表，它们是由其它操作定义或引用块参数的 SSA 值。
- `{ inplace = true }`：零个或多个属性的字典，这些属性是始终为常量的特殊操作数。 在这里，我们定义了一个名为 “inplace” 的布尔属性，它的常量值为 true。
- `(tensor<2x3xf64>) -> tensor<3x2xf64>`：函数形式表示的操作类型，前者是输入，后者是输出。<2x3xf64>号中间的内容描述了张量的尺寸2x3和张量中存储的数据类型f64，中间使用x连接。
- `loc("example/file/path":12:1)`：此操作的源代码中的位置。💡需要注意的是： MLIR 中的 loc 信息（源码位置信息）在实际应用中是不能去除的， 这与 在 LLVM等其它编译器中 中附加的 Debug 信息不同。

> In MLIR, every operation has a mandatory source location associated with it. Contrary to LLVM, where debug info locations are metadata and can be dropped, in MLIR, the location is a core requirement, and APIs depend on and manipulate it. Dropping a location is thus an explicit choice which cannot happen by mistake.
> 

接下来回到代码中， 执行以下 chapter2 的程序`./bin/toyc-ch2 ../mlir/test/Examples/Toy/Ch2/codegen.toy -emit=mlir -mlir-print-debuginfo` 得到MLIR 表达式如下

```c++
module {
  toy.func @multiply_transpose(%arg0: tensor<*xf64>, %arg1: tensor<*xf64>) -> tensor<*xf64> {
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

**那我们的 `codegen.toy` 自定义的语言是如何生成上面这个 MLIR 表达式的呢**：

> 在我看来，Dialects是将所有的IR放在了同一个命名空间中，分别对每个IR定义对应的产生式以及绑定相应的操作，从而生成一个MLIR的模型。整个的编译过程，从源语言生成AST，**借助Dialects遍历AST，产生MLIR的表达式**，此处可为多层IR通过Lowering Pass依次进行分析，最后经过MLIR分析器，生成目标语言。
>
> from: [MLIR的法宝：Dialects](https://zhuanlan.zhihu.com/p/102212806)

<div class="autocb" style="text-align:center;"><img src="./mlir-toy.assets\autocb_0.png" style="zoom: 45%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

就具体实现来说，从toy 语言的 AST(即 `toy::ModuleAST`) 到 MLIR 表达式(即`mlir::ModuleOp`) 的生成过程是通过 `dumpMLIR`来实现的。

1. dumpMLIR 实现在 `examples/toy/Ch2/toys.cpp` 中：

    ```c++
    int dumpMLIR() {
      mlir::MLIRContext context;
      // 1. Load our Dialect in this MLIR Context.
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

    - 将 toy AST 中的 `NumberExprAST` 转成 `ConstantOp`
    - 将 toy AST 中的 `BinaryExprAST` 转成 `AddOp` or `MulOp`
    - 将 toy AST 中的 `CallExprAST` 转成 `TransposeOp` 或其它 Op

    ```c++
    mlir::Value mlirGen(NumberExprAST &num) {
      return builder.create<ConstantOp>(loc(num.loc()), num.getValue());
    }

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

    最终的生成结果会按顺序被 insert 到 `mlir::OpBuilder::block` 中。 最终遍历生成的 MLIR Operation， 执行打印，就得到了上面打出来的 MLIR 表达式

在第一步的 `dumpMLIR` 中我们注意到有如下代码：

```c++
mlir::MLIRContext context;
context.getOrLoadDialect<mlir::toy::ToyDialect>();
```

这行代码涉及到 MLIR 的 Dialect 和 TableGen，对应上面的图片，以及前面所提到的： MLIR 通过 Dialect 来表示一组抽象。

### 2.1. 定义一个 Toy Dialect
接下这部分是 官方文档翻译：

> 💡**定义一个 Toy Dialect**
> 
> 为了有效地与MLIR交互，我们将定义一个新的toy方言。这种方言将模拟toy语言的结构，并为高级分析和转换提供一条简单的途径。
>
> ```c++
> /// 这是 Toy dialect的定义. 继承自 mlir::Dialect
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
> 从现在开始创建的任何新的 `MLIRContext` 都将包含toy方言的一个实例，并调用特定的 hooks 来解析属性和类型。
>
> ---
> 
> 💡**定义一个 Toy Dialect 上的 Operation**
> 
> 有了 toy dialect，我们就可以开始注册该 Dialect 上的操作了。这将允许提供语义信息给剩余系统调用。下面我们来介绍一下 `toy.constant` 操作的创建过程：
>
> ```c++
> %4 = "toy.constant"() {value = dense<1.0> : tensor<2x3xf64>} : () -> tensor<2x3xf64>
> ```
>
> 该操作没有参数，有一个名为value 的 [dense elements](https://mlir.llvm.org/docs/Dialects/Builtin/#denseintorfpelementsattr)属性用来表示constant value，返回一个 [RankedTensorType](https://mlir.llvm.org/docs/Dialects/Builtin/#rankedtensortype)。`ConstantOp` 继承自 CRTP `mlir::op`，该类还需要一些可选的 Traits 来约束其行为。这些特征可以提供额外的accessors, verification等功能。

>
> ```c++
> class ConstantOp : public mlir::Op<ConstantOp,
>                      /// ConstantOp takes no inputs.
>                      mlir::OpTrait::ZeroOperands,
>                      /// ConstantOp returns a single result.
>                      mlir::OpTrait::OneResult,
>                      /// result of getType is `Type`.
>                      mlir::OpTraits::OneTypedResult<Type>::Impl> {
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

不过在这里我们也可以看到，为了适配各种接口，定义一个 `ConstantOp` 已经十分复杂了。 MLIR 为了解决这种复杂性， 提供了一个基于 TableGen 的 Operation Definition Specification (ODS) Framework。 这种方式是 MLIR 推荐的定义 Dialect 的方式， 在 Toy 语言中我们的 toy Dialect，也是通过这种方法来定义的。
具体的文件在 `examples/toy/Ch2/include/toy/Ops.td` ， 