
# 属性(Attribute)

> 参考: [Rust Reference-Attributes](https://rustwiki.org/zh-CN/reference/attributes.html)

写这一篇的出发点是了解rust中的条件编译。

> [rust - How do I use conditional compilation with `cfg` and Cargo? - Stack Overflow](https://stackoverflow.com/questions/27632660/how-do-i-use-conditional-compilation-with-cfg-and-cargo)
>
> rust中的条件编译通过**属性(Attributes)**来实现。属性是根据名称、约定、语言和编译器版本解释的通用、free-form的元数据。

- 内部属性(*Inner attributes*)：形式为`#![]`例如`#![allow(unused_variables)]`；
对其作用域内所有内容生效。

- 外部属性(*Outer attributes*)：形式为`#[]`例如`#[test]` 把一个函数标记为测试function；对其后紧跟的一项内容生效。

> - The outer attribute is placed outside something - i.e. before the struct, or function, or module. This is usually what you want. 
> - The inner attribute is placed inside something. This is the only way to place attributes on the crate (by writing them in the root of it), and AFAIK this use-case is the only one where you'll really want them.


属性可以分为如下几类：

1. 内置属性(Built-in attributes)
   
    - 条件编译(Conditional compilation)

        - `cfg` — 控制条件编译。
        - `cfg_attr` — 选择性包含属性。

    - 测试(Testing)
        - `test` — 将函数标记为测试函数。
        - `ignore` — 禁止测试此函数。
        - `should_panic` — 表示测试应该产生 panic。

    - 诊断(Diagnostics)
        - `allow、warn、deny、forbid` — 更改默认的 lint检查级别。
        - `deprecated` — 生成弃用通知。
        - `must_use` — 为未使用的值生成 lint 提醒。
    
    ...

2. 宏属性(Macro attributes)

3. 派生宏的辅助属性(Derive macro helper attributes)

4. 外部工具属性(Tool attributes)
   
    >注意: rustc 目前能识别的工具是 “clippy” 和 “rustfmt”。
    
    这种属性格式：第一个字段是外部工具的名称，后续的一个或多个字段由工具决定。例如：

    ```rust
    // Tells the rustfmt tool to not format the following element.
    #[rustfmt::skip]
    struct S {}
    // Controls the "cyclomatic complexity" threshold for the clippy tool.
    #[clippy::cyclomatic_complexity = "100"]
    pub fn f() {}
    ```

    其中`rustfmt`和`clippy`是两个官方提供的工具`rust-clippy`和`rust-fmt`，前者用于检查代码：

    > 在编程实践中，将有助于将代码写的容易维护，质量合乎一定规范的做法，称为Linting，在Rust中，相应的工具是`clippy`。

    后者用于格式化代码

