
# TORCH.FX

## 1. Intro
graph-mode 或者说 define-and-run-mode的深度学习框架(TensorFlow, Caffe等)可以看作是一种EDSL，它们对 host 语言(通常是py)进行元编程，构造出自己的图层级IR，在其上做 微分、device/host partitioning and placement、量化、优化等变换(transformation)。

eager mode 或者说 define-by-run的深度学习框架(PyTorch, TensorFlow Eager等)则允许直接使用host language 编写逻辑。 微分通过使用JIT的auto-differentiation来完成。但是其它类型的transformation，如量化、算子融合等，还是需要拿到程序的整体结构信息才好做。**因此动态图虽然易用性很强，但是图结构不能被提前感知和变换，需要一种方式(program capture systems)拿到程序的结构信息**。

有一些程序捕获系统是为了序列化整个程序(模型)而做，比如TorchScript，其对python语言大部分feature进行了建模，如控制流，用户自定义类型，可变状态等，这让TorchScript变得十分复杂。

相反地，如果只是需要做quantization or fusion的话，其实我们无需对Python所有语法、特性进行建模，而只需要用DAG的表示方式表示出模型的数据流，或者说程序的tensor计算图，无需关注算子的具体实现细节。

torch.fx就是这样一种用于捕获、变换图层级pytorch程序的系统。torch.fx 实现了一个图层级 IR，致力于简化程序的capture和变换。torch.fx 的 IR 只有 6 条指令， 完全使用py编写，在量化和算子融合等领域有很好的应用。

!!! warning "疑问"
    - The primary program transformation used in deep learning frameworks, program differentiation, is reformulated from an ahead-of-time transformation to a just-in-time transformation, in the form of autodifferentiation.
## 2. Background
不管是 graph-mode 还是 eager-mode 的框架在追踪捕获程序的时候必须要在以下几点中做考虑：

1. 捕获程序结构
2. 程序特化能力
3. IR 的设计

通常来说， 捕获更复杂的程序、使得变换后的程序有更好的效率，需要更复杂的 capture 框架以及更复杂的 IR 设计，也使得对程序编写 tansformation 更加困难。

### 2.1 捕获程序结构
最简单的捕获程序结构的方式就是 trace 程序的执行——对一段程序给定一些样例输入，记录程序执行过程中涉及的计算，从而确定程序的结构(PyTorch’s `jit.trace`)

稍微复杂一点的方式是使用符号追踪symbolic tracing(perform tracing
with abstract values rather than example inputs)。使用符号追踪可以确定控制流取决于输入的具体位置（而在样例输入trace中， 因为程序的执行取决于输入，如果输入覆盖不全，总是会有未到达的分支）

为了能覆盖到标准python程序执行，有一些 tracing system可以捕获更多结构，比如控制流（如tf.function）。

捕获程序结构的另一个思路是在 python 中嵌入另一套语言，例如 Tensorflow提供的 graph-building API 以及 PyTorch的 TorchScript 都可以看作这种 EDSL。用户调用给定的API来构建计算图 or 在特定的语法限制下来书写 torchscript。从而能够支持控制流等，torchscript甚至可以理解user-defined types（其基于 python AST）。相比于 tracing可能默默失败。这种嵌入语言的方法在遇到不支持的语法/运算时有明确报错信息。然而，embedded language的编译非常困难(相当于重新实现一门完备的语言)，并且还是没办法直接把用户写的python model直接提取出结构，用户还是需要修改他们的代码，以适应EDSL，毕竟它们只支持python 语法的一个子集。

!!! warning "疑问"
    - For instance, `tf.function` augments symbolic tracing with a Lightweight Modular Staging system that uses Python AST transforms to **convert imperative control flow constructs into higher-order Python functions, which can then be traced**.（怎么trace控制流？）

### 2.2 Specializing Programs
这里讲的是traced program 可能只是原本 program 的一个特化版本。比方一个循环神经网络会根据输入的序列长度进行不同轮数的循环，那么对于一个给定的序列长度比方说10，得到的trace就是一个10次循环——当更换一个输入序列长度时，就会得到另一个不同的trace。因此像是 TorchScript这种非侵入式的trace，就会产生shape specialized的问题：traced program is only valid for the value shapes used at trace time and may fail for other shapes。

为了解决特化的程序对某些输入失效的问题。DyNet 和 LazyTensor 使用 JIT trace；JAX使用。。。 TODO: 这部分没看懂🦊

!!! warning "疑问"
    - DyNet (Neubig et al., 2017) and LazyTensor (Suhan et al., 2021) perform tracing just-in-time
    - JAX’s jit combinator (Frostig et al., 2018) uses pure, functional Python programs as input. This enforces referential transparency on non-Tensor computation like shape expressions.

### 2.3. IR设计
ML 框架的 IR 格式各不相同，更丰富的 IR 可以捕获更多程序并且更具表现力，但代价是编写转换或高效运行代码的额外复杂性。

- 语言： Caffe， TensorFlow使用 Protocol Buffers format 来表示计算图。而PyTorch’s JIT， MXNet 使用 C++ 的数据结构实现它们的 IR，并有对应的 py 绑定。使用 native representations 会带来更好的运行时效率、更容易序列化等

- 控制流： 大部分神经网络（例如MLPs，CNN类的网络等）在不使用控制流的情况下也能很好的表达。**文中把这种没有控制流的程序叫做 Basic block programs** 。 Basic block programs 通常使用 DAG来表示。

    但是循环神经网络，常用于sequence processing domains，如传统RNN，GRU，LSTM等网络，则需要控制流。因为RNN类的神经网络循环次数往往依赖于输入序列长度。

    尽管如此，许多框架都支持在其 IR 中捕获和表示控制流。TorchScript为所有组件添加了控制流的支持。Jax使用高阶函数(`ax.lax.scan`)来允许函数式的控制流。MLIR represents control flow with basic blocks that end in tail calls. 控制流的表示为IR加入了复杂性，使得例如公共子表达式转换更难实现。

- state： python 语言的变量可变性和切片的语法使得在做程序变换时，必须要做别名分析等有效性检查，保证变换的安全性等。JAX 使用函数式的思想将 tracking state 的工作推到了框架之外。


## 3. 设计原则
1. 避免支持长尾分布，复杂的样例。主要关注经典模型的程序捕获和变换
2. 使用机器学习从业者已经熟悉的工具和概念，例如 Python 的数据结构和 PyTorch 中公开记录的算子
3. 使程序捕获过程具有高度可配置性，以便用户可以为长尾需求实现自己的解决方案

torch.fx 虽然不能像 TorchScript 这样的 IR 处理一些比较难的 Case（比如动态控制流），但是在神经网络这个领域里做得够用就可以了。最关键的是实现很简单，在这套IR上做转换非常方便。

## 4. torch.fx 概览
整体来看，torch.fx包括三个部分：

- torch.fx通过符号跟踪来捕获程序
- 并通过一个简单的 6 个指令的 IR 来表示它们
- 并基于这个 IR 重新生成 Python 代码来运行它

下面是一个使用 `torch.fx`的符号追踪打印出图层级 IR 并对relu算子进行重写的例子：
```python
import torch.nn.functional as F
import torch.fx as fx

def my_func(x):
    return torch.relu(x).neg()

traced : fx.GraphModule = symbolic_trace(my_func)
print(traced.graph)

def replace_activation(graph: fx.Graph, old_op, new_op):
    for n in graph.nodes:
        if n.op == "call_function" and n.target == old_op:
            # create IR to call new activate
            with graph.inserting_after(n):
                new_n = graph.call_function(new_op, n.args)
                n.replace_all_uses_with(new_n)
                graph.erase_node(n)

replace_activation(traced.graph, torch.relu, F.gelu)
traced.recompile()
print(traced.graph)
```

###  4.1. 捕获程序结构
torch.fx的符号跟踪机制使用一个 `Proxy` 数据结构来记录给定一个输入之后经过了哪些 Op。 `Proxy` 是一个 duck-typed 类型的 Python 类记录了在它之上的的属性访问和调用方法，是程序中真实 Op 的上层抽象.

PyTorch 的算子以及 Python 子集的某些函数都会被这个 `Proxy` 包装一次，然后在符号跟踪传入的是一个`nn.Module`时，会对这个`nn.Module`中的子`nn.Module`也进行 `Proxy` 包装，当然还包含输入数据。这样程序中的输入和其它 Op 都是 duck-typed 类型的 `Proxy` 对象，我们就可以执行这个程序了，也就是符号跟踪的过程。符号跟踪的过程通过一个Tracer类进行配置，它的方法可以被重写以控制哪些值被作为 `Proxy` 对象保留，哪些值被 unpack。（ `Proxy` 记录下来的 Op 可以进行 unpack，unpack 之后可以拿到真实的 Tensor, Parameter 和运算符等等）。通过 `Proxy` 和 Tracer 类的配合，torch.fx就可以完成 PyTorch 程序的符号跟踪，需要注意的是这里的符号跟踪的意思就是运行一遍这个被代理之后的`nn.Module`的 forward。

### 4.2. IR设计
torch.fx的 IR 由一个 Python 数据结构 Graph 来组织。这个 Graph 实际上是一个包含一系列Node的线性表。节点有一个字符串操作码opcode，描述节点代表什么类型的操作。 节点有一个关联的目标，它是调用节点( `call_module` 、 `call_function` 和 `call_method` ) 的调用目标。 最后，节点有 args 和 kwargs，在 trace 期间它们一起表示 Python 调用约定中的目标参数（每个 opcode 对应的 args 和 kwargs 的语义可以在附录 A.2 中找到）。 节点之间的数据依赖关系表示为 args 和 kwargs 中对其他节点的引用。

<div class="autocb" style="text-align:center;"><img src="./torch-fx.assets\autocb_0.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" title="6种opcode"/></div>

torch.fx 将程序的状态存储在 `GraphModule` 类中。 `GraphModule` 是转换程序的容器，暴露转换后生成的代码，并提供 `nn.Module` 类似的参数管理 APIs。 `GraphModule` 可以在任何可以使用普通的 `nn.Module` 的地方使用，以提供转换后的代码和 PyTorch 生态系统的其余部分之间的互操作性。

### 4.3. Source-to-Source 变换
torch.fx 流水线最后一步是 code generation。torch.fx 并不是退出 Python 生态系统并进入定制的运行时，而是从变换后的 IR 生成有效的 Python 源代码。 然后将此变换后的代码加载到 Python 中，生成一个可调用的 Python 对象，并作为forward方法安装在 GraphModule 实例上。 使用代码生成允许将 torch.fx 变换的结果安装在模型中并用于进一步的变换。 例如，在下面的代码中，我们拿到 trace 原始程序的结果并将其安装为新模块中的激活函数。

```python
class SampleModule(torch.nn.Module):
    def forward(self, x):
        return self.act(x + math.pi)
sm = SampleModule()
sm.act = traced # from previous code listing
traced : GraphModule = symbolic_trace(sm)
print(traced.code)
"""
def forward(self, x):
add = x + 3.141592653589793; x = None
gelu = torch.nn.functional.gelu(add); add = None
neg = gelu.neg(); gelu = None
return
"""
```


## 5. 设计上的选择

### 5.1. Symbolic Tracing
符号追踪相比于embedded language 更加灵活，也可以帮助我们 eliminate control flow，例如loop over `torch.nn.Sequential`。

符号追踪的问题在于它无法捕获 input-dependent control flow。 fx通过making the tracing process customizable解决这种比较少见(long-tail)的情况。

### 5.2. Configurable Program Capture
torch.fx 的符号追踪是可定制的。可以通过`Tracer` class 来定制 `fx.symbolic_trace`的行为。通过自定义追踪行为我们就可以屏蔽掉模型中不受支持的语言功能。

### 5.3. AoT Capture without Specialization
虽然 ahead-of-time tracing 限制了可以捕获的程序空间（例如，不支持任意控制流），但它提供了更可预测和更可观察的捕获、转换和代码生成过程。

与 example-based tracing 不同，符号跟踪不能特化程序流，因为进行数据相关的控制流决策所需的信息在跟踪时不存在。

比方说在控制流选择中常用到的 Tensor 的shape 信息 和 ndim 等信息，在符号追踪的过程中会以 Proxy 值被返回。当上下文的其它操作使用这些值时，会抛出异常。

### 5.4. 基于python的 IR 和转换
torch.fx 没有选择使用 protobuf 这种 cross-language 的格式，而是直接使用python 实现。方便python 用户使用。

transformations 的结果也是 python 代码。Transformed code 被封装在`GraphModule`中， 在PyTorch中可以像使用其它 `nn.Module`一样使用它。

代码生成进一步将 torch.fx 集成到 Python 生态系统中，而不是将转换后的代码隔离到定制且更难使用的运行时中（这个是在说谁？）。

### 5.5. 在 IR 中没有控制流
随着 Transformers 代替传统的循环神经网络，在dl中使用 host language 的控制流的机会越来越少。许多模型可以直接不需要控制流也能很好的表达。

然而一旦在IR 中加入了控制流，即使模型没有用到任何控制流，也会极大增加分析的复杂度。加入了控制流之后在进行转换时经常引入bug，相对地，对于 basic block IR，在执行 transformation 时只需要transfer function。

下面的代码展示了控制流引入的麻烦： `x`的shape 将会由输入的值确定，从而使得后续使用`x`的程序都变成动态shape。如果某个变换需要tensor确切的shape信息（如ASIC lowering），则无法继续进行。

```python
def loop_shapes(x, itr):
    # x is an input tensor of size [1, N]
    for _ in range(itr):
        x = torch.cat((x, x), dim=0)
    # Depending on the number of loop iterations, x may have an
    # arbitrary leading dimension i.e. x \in [*dynamic*, N]
    return x
```
文中提到对于包含控制流的大模型，torch.fx 仍然可以在 basic blocks 的子图上做transform。

### 5.6. 函数式的 Graphs but 有状态的 Modules

!!! warning "疑问"
    - Most models do not suffer from this restriction since most mutation is localized to the parameters of the model.（那什么model会因为状态不可变而受到影响呢）