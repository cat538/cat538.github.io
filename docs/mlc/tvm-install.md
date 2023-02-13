# TVM-install
## windows
主要参考官网的：[install from source](https://tvm.apache.org/docs/install/from_source.html#install-from-source)
以下自己总结值得注意的点：
1. 在windows上安装tvm首先要从源码安装llvm，注意这里**不能使用**
   从llvm官网下载的预编译的llvm，因为pre-built-binary中
   没有`llvm-config`等编译tvm所需要的东西。
   至于如何在windows从source安装llvm见`cpp/toolchain`文章
   
2. 使用Ninja 编译tvm:
    首先打开`x64 native dev prompt`(为了获取cl环境变量)，然后进入源码目录执行以下命令：
    
    ```powershell
    mkdir build
    cd build
    cp ../cmake/config.cmake . # 修改其中set(USE_LLVM ON)以及其它变量(参考官网)
    cmake -GNinja -DCMAKE_CXX_COMPILER=cl -DCMAKE_C_COMPILER=cl -DCMAKE_BUILD_TYPE=Release ..
    cmake --build .
    cmake --build . --target install # 该步骤可省略
    ```
    执行 `build` 后， 应该会得到编译生成的`tvm.dll`和`tvm_runtime.dll`(以及对应的两个导入库即.lib文件)

    执行`install`之后会显示安装的位置，默认应该是在`C:\Program Files(X86)`，可以自己放到合适的位置。

    不过，如果是简单使用 TVM 上层应用， `install` 步骤可省略。 **并不需要将编译出来的库路径加入系统环境变量，才能让ffi模块找到这些库**，因为 TVM 上层 ffi 会根据 环境变量`TVM_HOME`，即TVM源码目录 搜索其下 `build` 目录中的运行时依赖。

    
3. 而为了使用tvm，除了刚才的两个库之外，tvm project还提供了上层的软件包，包括python，rust，go等语言的binding。
   接下来还需要安装这些binding，以安装python包为例，官网提供了两种方式：

    1. 其中第一种即直接把源码的`python`目录加入到`PYTHONPATH`环境变量中，这样python在搜索lib时就能够搜索到这个库。这一步在linux上直接设置`$PYTHONPATH=$PYTHONPATH:<tvm-src>/python:`但是在windows系统上，不能够设置`%PYTHONPATH%=%PYTHONPATH%;<tvm-src>/python;`，而是需要设置`%PYTHONPATH%=D:\software\Python\Python38;%TVM_HOME%/python;`；

        **学习 TVM 经常需要改动 python 源码，这种方式更加推荐**


    2. 也可以安装预编译好的安装包，参照[https://tlcpack.ai/](https://tlcpack.ai/)即可：
    
        ```bash
        conda install tlcpack-nightly -c tlcpack
        ```
   