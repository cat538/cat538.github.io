## windows
主要参考官网的：[install from source](https://tvm.apache.org/docs/install/from_source.html#install-from-source)
以下自己总结的值得注意的点：
1. 在windows上安装tvm首先要从源码安装llvm，注意这里不能使用
   从llvm官网下载的预编译的llvm，因为pre-built-binary中
   没有`llvm-config`等编译tvm所需要的东西。
   至于如何在windows从source安装llvm见`cpp/config`文章
2. 使用Ninja 编译tvm:
    首先打开x64 native dev prompt(为了获取cl环境变量)，然后进入源码目录执行以下命令：
    ```powershell
    mkdir build
    cd build
    cp ../cmake/config.cmake . # 修改其中set(USE_LLVM ON)
    cmake -GNinja -DCMAKE_CXX_COMPILER=cl -DCMAKE_C_COMPILER=cl -DCMAKE_BUILD_TYPE=Release ..
    cmake --build .
    ninja install
    ```
    执行`ninja install`之后会显示安装的位置，默认应该是在`C:\Program Files(X86)`，可以自己放到合适的位置。
    到这里，应该会得到编译生成的`tvm.dll`和`tvm_runtime.dll`(以及对应的两个导入库即.lib文件)。

3. 而为了使用tvm，除了刚才的两个库之外，tvm project还提供了上层的软件包，包括python，rust，go等语言的binding。
   接下来还需要安装这些binding，以python为例，有以下几个需要注意的地方：

   1. 需要将第2步编译出来的库加入系统环境变量，才能让ffi模块找到这些库
   2. 安装python包，官网提供了两种方式，其中第一种即直接把源码的`python`目录加入到`PYTHONPATH`环境变量中，
   这样python在搜索lib时就能够搜索到这个库。这一步在linux上直接设置`$PYTHONPATH=$PYTHONPATH:<tvm-src>/python:`但是在windows系统上，不能够设置`%PYTHONPATH%=%PYTHONPATH%;<tvm-src>/python;`，而是需要设置`%PYTHONPATH%=D:\software\Python\Python38;<tvm-src>/python;`
4. 