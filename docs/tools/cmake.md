# cmake

## 常用路径

- `CMAKE_SOURCE_DIR` : **顶级**cmakelists.txt的文件夹目录。
- `CMAKE_CURRENT_SOURCE_DIR` : 一般来说，一个工程会有多个cmakelists.txt文件，对应当前文件目录。
- `CMAKE_BINRAY_DIR` : 对应cmake的build的目录，主要是运行时生成的文件目录。
- `CMAKE_CURRENT_BINARY_DIR` : 对应build里的目录。
- `CMAKE_MODULE_PATH` : api(include/find_package)包含别的cmake文件时的搜索目录。
- `CMAKE_PREFIX_PATH` : api(find_libray/path)包含模块时的搜索目录。
- `CMAKE_INSTALL_PREFIX` : 调用install相关函数，要生成/保存的根目录路径。



- `target_compile_features`可以更细粒度的指定C++的特性，如`cxx_auto_type`，`cxx_lambda`等，如果某个子项目需要C++20，但是i项目整体是17，可以对项目设置`target_compile_features(<project name> INTERFACE cxx_std_20)`

## target_* 中的 PUBLIC  PRIVATE INTERFACE

这类命令常用的有`target_link_libraries`，`target_include_directories`，`target_compile_definitions`等：

```cmake
target_compile_definitions(say-hello PUBLIC VERSION=4)
```

这条命令在预编译阶段之前插入宏 `#define VERSION 4`，其中`PUBLIC`表示这个宏对于`say-hello`这个库以**及link了`say-hello`这个库的模块**可见；如果是`PRIVATE`，则这个宏仅对`say-hello`可见；而`INTERFACE`与`PRIVATE`相反，设置为`INTERFACE`后，对于外部可见，而对于模块内部不可见。

直观地来说`PUBLIC`会将某个属性向外"传播"，`PRIVATE`自己"独享"，`INTERFACE`只暴露给外部

- `target_include_directories` 设置为`PUBLIC`常用于库文件向外export 头文件
- `target_link_libraries` 
- `target_add_definitions`
- `target_compile_options`

## 第三方库引入

cmake引入第三方库可以分为三种方式：

1. header only，如Boost，fmt(有 header only 版本)，这种库直接把头文件加入到当前工程头文件目录即可
2. git module
3. FetchContent



## Windows使用cmake

在Windows平台下，`cl.exe`对应`gcc` or `clang`，`link.exe`对应`ld`；通常在安装VS后，电脑上会有多个`cl.exe`，他们分别对应不同host架构和target架构：

<img src="./cmake-study.assets/image-20220730190848346.png" alt="image-20220730190848346" style="zoom:50%;" />

但是如果从外部终端直接使用`cl.exe main.cpp`无法直接编译，因为`cl.exe`需要通过命令，或通过环境变量设置`include dir`。使用windows下这套编译链接工具最简单的方法是打开 X64 Native Tool Command Prompt for VS 2022，这个shell 预设置了这套工具运行所需要的环境变量（其它架构同理）。

如果想要从外部终端使用这套环境变量可以执行脚本 如：`vcvarsall.bat x64`(脚本的位置需要自己寻找)

### nmake

nmake是一个命令行工具，是Microsoft Visual Studio中的附带命令，可以使用cmake构建build system 时指定 使用nmake：`cmake -G "NMake Makefiles"`

![image-20220730194727781](./cmake-study.assets/image-20220730194727781.png)

关于其它的构建系统可以使用`cmake --help`查看：

<figure><img src="cmake-study.assets/image-20220730195434113.png" alt="image-20220730195434113" style="zoom: 50%;" /></figure>

如图所示可以看到现在默认的generators是 **MSBuild**，这意味着我在`/build`下使用`cmake ..`时默认会生成`.sln`等我们熟悉的VS工程文件。

### MSBuild

MSBuild 是 Visual Studio 中所有项目（包括 C++ C# 项目）的native build system。 在 Visual Studio IDE 中构建项目时，它会调用 msbuild.exe，该工具又会使用 `.vcxproj` 项目文件以及各种 `.targets` 和 `.props` 文件。Visual Studio 依赖MSBuild，但是MSBuild不依赖Visual Studio。MSBuild是nmake的替代品。MSBuild所管理的工程文件（`.sln`，`.vcxprj`）使用的是xml语法。

如前文所述，在Windows上，cmake默认会选择MSBuild作为默认的build system generator，在命令行使用MSBuild的步骤：

```
cd build
cmake ..
cmake --build . --config Release(默认是Debug)
```

我们经常看到的步骤是`cmake ..`之后`make`；这里使用`cmake --build .` 命令代替可以做到跨平台，这样即使是使用Ninja 或者 MSBuild 或者NMake Makefiles等生成的build tree 都可以被正确构建。

而我们常见的在`cmake ..`之后使用`make`其实是假设了使用`Unix Make`生成`build tree`，此时project files 主要是Makefile等，需要使用make去build。



### Ninja

```bash
 cd build
 cmake -G "Ninja" -DCMAKE_C_COMPILER=cl -DCMAKE_CXX_COMPILER=cl ..
 cmake --build . --config release
```

注意与nmake一样，这条命令执行必须要在有cl环境变量的shell中。一个直接的做法是使用 X64 Native Tool Command Prompt for VS 2022（其它版本，其它架构同理，下载VS后有对应的shell）；

<figure><img src="cmake-study.assets/image-20220730204145599.png" alt="image-20220730204145599" style="zoom:50%;" /></figure>