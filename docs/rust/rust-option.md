---
title: RustOption
date: 2021-06-03 20:45:13
categories: 'Rust'
tags: ['Rust']
description: "学习Rust中的枚举类型Option..."
cover: "https://github.com/cat538/images-auto/raw/main/img/rustlogo.jpg"

---

## 所有权（ownership）、借用（borrow）和生命周期（lifetime）

### RAII

> Rust 强制实行 [RAII](http://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)（Resource Acquisition Is Initiallization，资源获取即初始化），所以任何对象在离开作用域时，它的析构函数（destructor）就被调用，然后它占有的资源就被释放。
>
> Rust 中的析构函数概念是通过 `Drop` trait 提供的。当资源离开作用域，就调用析构 函数。你无需为每种类型都实现 `Drop`trait，只要为那些需要自己的析构函数逻辑的 类型实现就可以了。

```rust
pub trait Drop {
    fn drop(&mut self);
}
```



### 所有权:

**所有权机制针对堆上数据。堆上的数据**在进行赋值（`let x = y`）或通过值来传递函数参数（`foo(x)`）的时候，资源的**所有权**（ownership）会发生转移。按照 Rust 的说法，这被称为资源的**移动**（move）。

**注意：**当所有权转移时，数据的可变性可能发生改变。

```rust
    let immutable_box = Box::new(5u32);
    println!("immutable_box contains {}", immutable_box);
    // 可变性错误
    //*immutable_box = 4;
    // *移动* box，改变所有权（和可变性）
    let mut mutable_box = immutable_box;
    println!("mutable_box contains {}", mutable_box);
    // 修改 box 的内容
    *mutable_box = 4;
    println!("mutable_box now contains {}", mutable_box);
```



另一个例子：

```rust
    let mut v = vec![1,2,3,4];
    let h = v[0];
	
    let mut v = vec![point,point,point];
    let h = v[0];
	// 这样是不合法的，因为v[0]所有权是v

	// 只能获得v[0]的不可变借用：
	let h = &v[0];
	// 或者获得v[0]的可变借用：
	let h = &mut v[0];
```



### 借用

> 多数情况下，我们更希望能访问数据，同时不取得其所有权。为实现这点，Rust 使用 了**借用**（borrowing）机制。对象可以通过引用（`&T`）来传递，从而取代通过 值（`T`）来传递。

**另外需要注意的一点是：**

一个数据可以有多个不可变借用，但是在使用数据的不可变借用的同时，不能使用数据的可变借用。

或者说，同一时间内只允许**一个**可变借用。仅当最后一次使用可变借用**之后**，原始数据才可以再次借用。如下例子：

```rust
struct Point { x: i32, y: i32, z: i32 }

fn main() {
    let mut point = Point { x: 0, y: 0, z: 0 };
    let borrowed_point = &point;
    let another_borrow = &point;
    // 数据可以通过引用或原始类型来访问
    println!("Point has coordinates: ({}, {}, {})",
                borrowed_point.x, another_borrow.y, point.z); 
    // 报错！`point` 不能以可变方式借用，因为当前还有不可变借用。
    // let mutable_borrow = &mut point;
    // TODO ^ 试一试去掉此行注释
    
    // 被借用的值在这里被重新使用
    println!("Point has coordinates: ({}, {}, {})",
                borrowed_point.x, another_borrow.y, point.z);
    // 不可变的引用不再用于其余的代码，因此可以使用可变的引用重新借用。
    let mutable_borrow = &mut point;
    // 通过可变引用来修改数据
    mutable_borrow.x = 5;
    mutable_borrow.y = 2;
    mutable_borrow.z = 1;
    // 报错！不能再以不可变方式来借用 `point`，因为它当前已经被可变借用。
    // let y = &point.y;
    // TODO ^ 试一试去掉此行注释

    // 报错！无法打印，因为 `println!` 用到了一个不可变引用。
    // println!("Point Z coordinate is {}", point.z);
    // TODO ^ 试一试去掉此行注释

    // 正常运行！可变引用能够以不可变类型传入 `println!`
    println!("Point has coordinates: ({}, {}, {})",
                mutable_borrow.x, mutable_borrow.y, mutable_borrow.z);

    // 可变引用不再用于其余的代码，因此可以重新借用
    let new_borrowed_point = &point;
    println!("Point now has coordinates: ({}, {}, {})",
             new_borrowed_point.x, new_borrowed_point.y, new_borrowed_point.z);
}
```



## Rust 的 Option枚举

Option是一个枚举类型，定义如下，

```rust
enum Option<T> {
    Some(T),
    None,
}
```

### 初始化Option：

使用`let x = Some(T)`，编译器可以自动推断x对应的枚举类型，如`let some_string = Some("a string");`

但使用`None`则不可以如此：

因为编译器只通过 `None` 值无法推断出 `Some` 成员保存的值的类型。

需要事先声明`Option`的类型，如`let absent_number: Option<i32> = None;`



### 解开Option：

- 可以使用 `match` 语句来解开 `Option`
- 可以使用`unwrap()`来获得`Option`中的`Some`，但如果使用这种方式，获得`None`的时候程序将会panic



### 使用`Option<T>` 为什么比使用空值要好？

**Option解决了空值引发的一系列问题？？？**

空值的问题在于当你尝试像一个非空值那样使用一个空值，会出现某种形式的错误？？？

而因为 `Option<T>` 和 `T`（这里 `T` 可以是任何类型）是不同的类型，编译器不允许像使用一个肯定有效的值那样使用 `Option<T>`。例如，这段代码不能编译，因为它尝试将 `Option<i8>` 与 `i8` 相加：

```rust
let x: i8 = 5;
let y: Option<i8> = Some(5);
let sum = x + y;
```

> 当在 Rust 中拥有一个像 `i8` 这样类型的值时，编译器确保它总是有一个有效的值。我们可以自信使用而无需做空值检查。只有当使用 `Option<i8>`（或者任何用到的类型）的时候需要担心可能没有值，而编译器会确保我们在使用值之前处理了为空的情况。
>
> 在对 `Option<T>` 进行 `T` 的运算之前必须将其转换为 `T`。通常这能帮助我们捕获到空值最常见的问题之一：假设某值不为空但实际上为空的情况。
>
> 为了拥有一个可能为空的值，你必须要显式的将其放入对应类型的 `Option<T>` 中。接着，当使用这个值时，必须明确的处理值为空的情况。只要一个值不是 `Option<T>` 类型，你就 **可以** 安全的认定它的值不为空。**也就是从语法的层面上强制程序员事先对可能出现空值的地方进行处理。**
