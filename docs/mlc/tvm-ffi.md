# TVM-ffi
> 代码基于 tlc-pack/relax dc7072efe290d7e8c69d8e216311510981fc82e1
> 
> ref: 
> 
> - [深入理解TVM：Python/C++互调(下)](https://mp.weixin.qq.com/s?__biz=Mzg5MzU4NTU5Nw==&mid=2247483737&idx=1&sn=7b881ee096ac42b8975d6383256121a6&chksm=c02dd08bf75a599d8c03abe57f17c5f88fe5e66d14a9d3650d3399e9f99201553dcb79a9a7b7&scene=178&cur_album_id=1811050680510447621#rd)

这篇主要介绍 TVM 中 python 前端和 C/C++后端 交互的方法。

## import tvm 做了什么?
首先考虑一个问题：
```python
import tvm
```
这行代码做了什么工作？

根据python语言的定义， 这里的 `tvm` 是一个 package(一个package包含多个subpakcage 或 modules)。执行 `import`时，package 目录下若有`__init_.py`文件，则该文件中的所有代码都会被执行。因此 这行代码实际上隐式地执行了 `tvm/__init__.py`文件。因此接下来看该文件中做了哪些工作（这里重点关注 ffi 相关）：
在 `tvm/__init__.py`文件中 引入 ffi 部分的逻辑如下

```python
# tvm._ffi
from ._ffi.base import TVMError, __version__, _RUNTIME_ONLY

from ._ffi.runtime_ctypes import DataTypeCode, DataType
from ._ffi import register_object, register_func, register_extension, get_global_func
```

这里的`tvm/_ffi` 是一个 subpackage，`tvm/_ffi/__init__.py`中内容是：

```python
from . import _pyversion
from .base import register_error
from .registry import register_object, register_func, register_extension
from .registry import _init_api, get_global_func, get_object_type_index
```

1. `from ._ffi.base import ...`语句会从`tvm/_ffi/base.py` 做的工作是尝试load tvm动态库，其搜索几个常规路径，其逻辑如下：
    ```python
    def _load_lib():
        """Load libary by searching possible path."""
        lib_path = libinfo.find_lib_path()
        # The dll search path need to be added explicitly in
        # windows after python 3.8
        if sys.platform.startswith("win32") and sys.version_info >= (3, 8):
            for path in libinfo.get_dll_directories():
                os.add_dll_directory(path)
        lib = ctypes.CDLL(lib_path[0], ctypes.RTLD_GLOBAL)
        lib.TVMGetLastError.restype = ctypes.c_char_p
        return lib, os.path.basename(lib_path[0])
    ```
    可以看到，tvm使用python的`ctypes`模块加载动态库.

    在_load_lib函数执行完成后，`_LIB`和`_LIB_NAME`都完成了初始化，其中`_LIB`是一个`ctypes.CDLL`类型的变量，可以认为它是能够操作TVM动态链接库的export symbols的一个全局句柄，`_LIB_NAME`是库名称字符串。这样后续在python中，我们就能通过`_LIB`这个句柄和c++的代码进行交互。
    
2. 执行 `tvm/_ffi/registry.py` ，该module内定义了一系列与 c++ 代码交互的函数（如从py获取已注册func的方法：`get_global_func`，以及从py代码注册func的方法：`register_func` 等等）
    
    这里以`get_global_func`为例，看一下py是怎么拿到 c++中编写的函数的：
    `get_global_func`简单地调用 `_get_global_func` 返回值是一个`PackedFunc`对象，`_get_global_func`逻辑如下：

    ```python
    def _get_global_func(name, allow_missing=False):
        handle = PackedFuncHandle() # 类型为 ctypes.c_void_p 对应c中的一个 void*
        check_call(_LIB.TVMFuncGetGlobal(c_str(name), ctypes.byref(handle)))

        if handle.value:
            return _make_packed_func(handle, False)
        if allow_missing:
            return None
        raise ValueError("Cannot find global function %s" % name)
    ```

    其中`_LIB.TVMFuncGetGlobal` 函数定义位于`src/runtime/registry.cc`，该函数简化后逻辑如下：
    
    ```c++
    int TVMFuncGetGlobal(const char* name, TVMFunctionHandle* out) {
        const tvm::runtime::PackedFunc* fp = tvm::runtime::Registry::Get(name);
        tvm::runtime::TVMRetValue ret;
        ret = *fp;
        TVMValue val;
        int type_code;
        ret.MoveToCHost(&val, &type_code);
        *out = val.v_handle;
    }
    ```
    首先查表获得 name 对应的 `PackedFunc*` fp，然后用`TVMRetValue`类型将 fp wrap 起来，接着调用`TVMRetValue`的`MoveToCHost` 方法将 指针返回到py前端。

    python在从C++拿到 PackedFuncHandle 后，调用了`_make_packed_func` 将其包装成一个python对象，简化后逻辑如下:

    ```py
    def _make_packed_func(handle, is_global):
        obj = PackedFunc.__new__(PackedFunc)
        obj.is_global = is_global
        obj.handle = handle
        return obj
    ```

    这个过程告诉我们，如果想要C++的函数被 python 前端的`get_global_func`感知，需要实现将函数注册到一个表里，使得`tvm::runtime::Registry::Get`方法能从这张表中获取到。接下来我们关注 在c++中注册函数的整个过程。

## Registry
上面我们说到 tvm 可以通过 `get_global_func`之类 的方法获取到 c++ 中注册的函数，or 进行其它交互。这里有几个问题：

1. 具体来说怎么在c++中注册一个函数使其对于 py 前端可见？
2. python 前端具体会在什么时候 load 这些函数？

以一个例子来说明这两个问题：

比方说在使用`from_onnx()`时需要将onnx算子转换成RelayExpr，而构建 RelayExpr的实现在C++代码中: `src/relay/op/nn/nn.cc`，包括`MakeDense`, `MakeMatMul`, `MakeBiasAdd` 等。

   - 这些`Make*`函数 需要加上什么样的标记 or 通过什么阳的方式才能够被python 的`ctypes`模块感知并调用？
   - 我们每次在python中生成RelayExpr时都要先调用`get_global_func`拿到handle再进行一次调用吗？

看一下tvm是怎么做的。首先回答第二个问题，即python 如何拿到c++ 函数：

在`tvm/__init__.py`中有一行代码：
```py
from .runtime.object import Object
```
在`tvm/runtime/__init__.py`中有一行代码：
```py
from .object import Object
```
在`tvm/runtime/object.py`这个module中有一行代码：
```py
from . import _ffi_api, _ffi_node_api
```
而`tvm/runtime/_ffi_api.py`代码如下：
```py
"""FFI APIs for tvm.runtime"""
import tvm._ffi

# Exports functions registered via TVM_REGISTER_GLOBAL with the "runtime" prefix.
# e.g. TVM_REGISTER_GLOBAL("runtime.ModuleLoadFromFile")
tvm._ffi._init_api("runtime", __name__)
```
该module通过调用`tvm_ffi._init_api()`方法来为特定的python module 生成调用C++代码的 api。具体来说该函数通过调用`tvm/_ffi/registry`中的`list_global_func_names()`函数（底层调用 ctypes 暴露的 `TVMFuncListGlobalNames`这个C++API）得到C++中注册的所有函数

在整个 tvm project 中 搜索`tvm._ffi._init_api(` 发现有 88 个结果
，例如：
```py
tvm._ffi._init_api("relay.op._make", __name__)
tvm._ffi._init_api("relay.op.nn._make", __name__)
tvm._ffi._init_api("relay.op.dyn._make", __name__)
tvm._ffi._init_api("relay.op.dyn.nn._make", __name__)
```

💡**总结**：python 端通过`_init_api()`函数来为各个module生成对应的 C++函数接口。具体来说， `_init_api()`使用`ctypes`模块调用 动态库暴露给 python 的`TVMFuncListGlobalNames`方法——**该方法返回所有在C++代码中使用`TVM_REGISTER_GLOBAL`宏注册过的函数(的地址)**(在该版本代码库中有2236处查找结果)，`_init_api()`会根据注册名前缀进行筛选，为对应的python module 生成相应的api。

> `TVMFuncListGlobalNames`的实现位于`src/runtime/registry.cc`中
>
> 在`src/runtime/c_runtime_api.h`中导出符号

---

接下来我们关心第一个问题，即 tvm C++ 代码中的 **registry机制**，这也是其它语言前端与 C++代码交互的核心机制：

`TVM_REGISTER_GLOBAL`宏：
```c++
#define TVM_REGISTER_GLOBAL(OpName) \
  TVM_STR_CONCAT(TVM_FUNC_REG_VAR_DEF, __COUNTER__) = ::tvm::runtime::Registry::Register(OpName)
```

- `TVM_FUNC_REG_VAR_DEF` 展开为`static ::tvm::runtime::Registry& __mk_##TVM`，
- `__COUNTER__`是主流编译器都支持的一个内置宏，可以理解为一个每次+1的计数器
- 宏将会通过调用`Registry::Register(OpName)`返回一个`Registry&`


一个展开示例：
在`src/relay/op/nn/nn.cc`中：
```c++
TVM_REGISTER_GLOBAL("relay.op.nn._make.relu").set_body_typed([](Expr data) {
  static const Op& op = Op::Get("nn.relu");
  return Call(op, {data}, Attrs(), {});
});
```
会被替换为如下语句：
```c++
static ::tvm::runtime::Registry& __mk_TVM40 =
    ::tvm::runtime::Registry::Register("relay.op.nn._make.relu").set_body_typed([](Expr data) {
  static const Op& op = Op::Get("nn.relu");
  return Call(op, {data}, Attrs(), {});
});
```

其中`set_body`即将`Registry`的`func_`成员改为该lambda。

接下来我们看一下 `Registry` 结构，简化后：
```c++
// include/tvm/runtime/registry.h
class Registry {
 public:
  Registry& set_body(PackedFunc f);
  Registry& set_body_typed(FLambda f);
  Registry& set_body_method(R (T::*f)(Args...));

  static Registry& Register(const std::string& name);
  static const PackedFunc* Get(const std::string& name);
  static std::vector ListNames();

 protected:
  std::string name_;
  PackedFunc func_;
  friend struct Manager;
};
```
`name_`很好理解，在上面的例子中，即为`relay.op.nn._make.relu`；而`func_`的类型是`PackedFunc`，是对函数指针的一个包装，刚才例子中使用`set_body`方法传入的lambda会经过包装后赋值给`func_`，后续会对`PackedFunc`这一TVM的关键结构详细说明。

看一下刚才`TVM_REGISTER_GLOBAL`用到的`Register`方法，和`set_body`方法，定义如下：
```c++
Registry& Registry::Register(const std::string& name, bool can_override) {
  Manager* m = Manager::Global();
  std::lock_guard<std::mutex> lock(m->mutex);

  if (m->fmap.count(name)) {
    ICHECK(can_override) << "Global PackedFunc " << name << " is already registered";
  }

  Registry* r = new Registry();
  r->name_ = name;
  m->fmap[name] = r;
  return *r;
}

Registry& Registry::set_body(PackedFunc f) {
  func_ = f;
  return *this;
}
```
可以看到，其逻辑为获取`Manager`的单例，然后从`Manager`的成员`fmap`表中查询，如果为空，就new一个`Registry`实例，然后将该实例加入表中，并返回对该实例的引用，我们接着可以通过`.set_body()`来设置该实例的函数体。

首先看一下`Manager`的实现(位于`src/runtime/registry.cc`)：

```c++
struct Registry::Manager {
  std::unordered_map<std::string, Registry*> fmap;
  std::mutex mutex;

  Manager() {}

  static Manager* Global() {
    // We deliberately leak the Manager instance, to avoid leak sanitizers
    // complaining about the entries in Manager::fmap being leaked at program
    // exit.
    // 这段注释不理解？
    static Manager* inst = new Manager();
    return inst;
  }
};
```

💡到这里我们明白了：

1. 所有通过`TVM_REGISTER_GLOBAL` 宏注册的函数，都会存储到全局唯一的`Manager`的`fmap`实例中，该表的key是函数名，对于例子中即为`relay.op.nn._make.relu`， 表的value是一个`Registry*`，一个`Registry`实例中包含一个函数名和一个`PackedFunc`对象表示函数本身；
2. `TVM_REGISTER_GLOBAL` 宏所做的事情就是对于未注册的函数，new一个`Registry`实例，并把指针存到表中
3. 前面提到的`TVMFuncListGlobalNames` 就是遍历这张表

接下来的问题就是`PackedFunc`是什么？如何通过该结构实现对所注册函数的调用？

关于 `PackedFunc` 类型，可以参考 [tvm-type](./tvm-type.md) 这篇文章。

## 向 TVM 中添加注册自定义函数