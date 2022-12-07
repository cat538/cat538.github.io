# Rust-modules

<img src="https://github.com/cat538/images-auto/raw/main/img/image-20211024205703750.png" alt="image-20211024205703750" style="zoom: 67%;" />

官方文档给出的关键字解释如下：

> - **包**（*Packages*）： Cargo 的一个功能，它允许你构建、测试和分享 crate。
> - **Crates** ：一个模块的树形结构，它形成了库或二进制项目。
> - **模块**（*Modules*）和 **use**： 允许你控制作用域和路径的私有性。
> - **路径**（*path*）：一个命名例如结构体、函数或模块等项的方式

## Package 和 Crate

*包*（*package*）是提供一系列功能的一个或者多个 crate

一个*包*（*package*） 会包含有一个 `Cargo.toml` 文件，阐述如何去构建这些 crate



**关于`package`和`crate`有以下几条规则：**

- 一个`package`包含0个或1个`lib crate`(库)
- 一个`package`包含任意多个`bin crate`(可执行)
- 一个`package`至少包含1个`lib crate`(库)或1个 `bin crate`(二进制)

另有两条默认规定：

- 默认`src/main.rs`是`bin crate`的根，且其名字就是package的名字
- 默认`src/lib.rs`是`lib crate`的根，且其名字就是package的名字

**注解：bin crate就是包含main函数的`.rs`文件**

另有关于 `bin crate`规则如下：

- 如果一个`package`有多个`bin crate`，可以放在`src/bin`下

![image-20211024195317131](https://github.com/cat538/images-auto/raw/main/img/image-20211024195317131.png)

如上图所示的`package`内有3个`bin crate`，可以看到直接`cargo r`会报错无法determine哪一个bin（不过实际上他们三个都被编译了），提示中看到有三个`bin crate`他们的名字分别是`hello, hi, myf`，这是因为我的package的名字是`myf`，而`src/bin`下两个文件分别叫做`hello hi`

## Crate 和 mod

使用`mod`关键字定义一个`module`，模块可以嵌套构成`mod`树

- 子`mod`可以使用父`mod`中的内容

- 子`mod`和父`mod`可以不在一个文件中，如图所示：

  ![image-20211024204721639](https://github.com/cat538/images-auto/raw/main/img/image-20211024204721639.png)

  在`main.rs`中使用了`mod parent`声明了一个名为`parent`的mod（该mod在`src/parent`目录下被定义），从而拼接了一个mod树

  `parent mod`有两个子`mod`分别是`son`和`daughter`

  - `mod`和文件映射关系:
    - `<module_name>.rs`
    - `<module_name>/mod.rs`

  **可以使用上述两种方式去跨文件定义一个`mod`**

- 综上所述`mod`树和文件树其实并没有太大关系，是通过`mod`关键字手动组成的一个`mod`树，在这个过程中需要记住：

  - 使用`pub`才能暴露给外部
  - 可以使用`crate`关键字从绝对路径（`mod tree` 根节点）寻找
  - 使用`super`关键字从父`mod`（`mod tree` 父节点）寻找
  - `lib.rs`和`main.rs`分别是两个root节点



