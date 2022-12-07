# Rust-smart pointer

> [CS 242: Smart pointers (stanford-cs242.github.io)](https://stanford-cs242.github.io/f19/lectures/07-2-smart-pointers)

---

**Rust内存管理的基本原则:**

- Having several immutable references (`&T`) to the object (also known as **aliasing**).
- Having one mutable reference (`&mut T`) to the object (also known as **mutability**).

<img src="https://github.com/cat538/images-auto/raw/main/img/Rust-container-cheat-sheet.png" alt="Rust container cheat sheet"  />

## Raw pointers

尽管不经常使用，Rust有原始指针的基本类型： `*const T` 和 `*mut T`；这些指针与C语言中的指针完全一样，它们实际上只是内存地址，没有检查内存安全。

```rust
let x: i32 = 1;
let xptr: &i32 = &x;
let xraw: *const i32 = xptr as *const i32;

let mut y: i32 = 2;
let yptr: &mut i32 = &mut y;
let yraw: *mut i32 = yptr as *mut i32;
let yraw2: *mut i32 = yptr as *mut i32;
```

如上图所示，我们获得了`yraw`和`yraw2`两个可变指针指向同一块内存，这违背了Rust内存管理的基本原则。

但事实上创建指针可以在safe rust中使用，但是解引用只能在`unsafe`中了：

```rust
println!("{}", unsafe { *yraw });
unsafe { *yraw = 3; }
```



## Box

**资源分配在堆上，依靠`Deref`和`Drop`来管理堆上的资源，零运行时开销，类似C++的unique_ptr，是最常用的套娃。**

比如尝试定义递归类型（链表）时

```rust
pub enum List {
    Empty,
    Elem(i32, List),
}
```

因为编译期无法确定大小，因此无法编译通过；这时可以用`Box`包起来`Elem`

- `Box::new(v)` ： 创建，移动语义，独占所有权 - move，允许使用`*`转移本体所有权
- 不可变借用：`Box::borrow()`、`Box::as_ref()`、`Box::deref()`
- 可变借用：`Box::borrow_mut()`、`Box::as_mut()`、`Box::deref_mut()`



#### 相关接口：

- `Box::into_raw()`转换为裸指针`*mut T`，从而可以通过`unsafe`来修改本来不会被修改的——Box中的值
- `Box::from_raw()`为`into_raw()`的对应接口，(打印裸指针可以获得其地址)
- `Box::leak()`将Box转换为可变引用`&mut T`

#### 实现自己的Box

```rust
#[derive(Debug)]
struct MyBox<T> {
    ptr: *mut T,
}
impl<T> MyBox<T> {
    fn new(t: T) -> MyBox<T> {
        let ptr = unsafe {
            let layout = Layout::for_value(&t);
            let mut ptr = alloc(layout) as *mut T;
            ptr::copy(&t as *const T, ptr, layout.size());
            ptr
        };
        mem::forget(t);
        MyBox { ptr }
    }
}
```

- 首先声明一个`MyBox`包装裸指针`*mut T`

- 然后创建一个`Layout`实例——描述内存块大小和内存对齐规则

- 使用`alloc`在堆上分配内存——Rust对下层内存分配器(`jemalloc`)的轻度包装

- 然后使用 [`ptr::copy`](https://doc.rust-lang.org/std/ptr/fn.copy.html) (等价于`memcpy`)把内存复制到刚分配的堆内存上

- The `mem::forget` says “don’t try to call destructors on `t`, just forget about it.”：

  > If our `Box<T>` contains a type that has pointers somewhere else, e.g. a `Box<Box<i32>>`, we don’t want to memcpy the box bits but then have Rust still destruct the original `Box<i32>`, meaning our memcpy’d version now points to invalid data. Here, `mem::forget` ensures that the destructors don’t run until we call them in `Box::drop`. 

- 最后返回`MyBox`

通过`Deref` trait实现解引用：

> 直接解引用`*y`是不可以的，因为我们尚未在该类型实现这个功能。为了启用 `*` 运算符的解引用功能，需要实现 `Deref` trait。
>
> **没有 `Deref` trait 的话，编译器只会解引用 `&` 引用类型。**`deref` 方法向编译器提供了获取任何实现了 `Deref` trait 的类型的值，并且调用这个类型的 `deref` 方法来获取一个它知道如何解引用的 `&` 引用的能力。
>
> 当我们输入 `*y` 时，Rust 事实上在底层运行了`*(y.deref())`

```rust
impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        unsafe{&*self.ptr}
    }
}
```

>表达式`&*self.ptr`看起来很怪。在C语义下，`&*x == x`十分显然。但是在这里，这个表达式的意思是：解引用原始指针(必须在`unsafe`中)，然后使一个安全指针(即引用`&`)指向该原始值。


同理通过实现`DerefMut`trait实现返回可变引用的功能：

```rust
impl<T> DerefMut for MyBox<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe{&mut *self.ptr}
    }
}
```

> **Deref 强制转换**（*deref coercions*）是 Rust 在函数或方法传参上的一种便利。其将实现了 `Deref` 的类型的引用转换为原始类型通过 `Deref` 所能够转换的类型的引用。当这种特定类型的引用作为实参传递给和形参类型不同的函数或方法时，Deref 强制转换将自动发生。
>
> 例如：接收`&str`为参数的函数，可以将`&String`直接传入，由于`String`类型实现了`Deref`，编译器会自动转换
>

最后我们通过实现`Drop`trait来释放内存：

```rust
impl<T> Drop for MyBox<T> {
    fn drop(&mut self) {
        unsafe {
            let layout = Layout::for_value(&*self.ptr);
            mem::drop(ptr::read(self.ptr));
            dealloc(self.ptr as *mut u8, layout);
        }
    }
}
```

类似C++的析构函数；在这个函数中调用`dealloc`(等价于C中的`free`)释放堆上的内存



## Rc

> 有些情况单个值可能会有多个所有者。例如，在图数据结构中，多个边可能指向相同的节点，而这个节点从概念上讲为所有指向它的边所拥有。节点直到没有任何边指向它之前都不应该被清理。为了启用多所有权，Rust 有一个叫做 `Rc<T>` 的类型。其名称为 **引用计数**（*reference counting*）的缩写

**资源分配在堆上，依靠`Deref`和`Drop`来管理堆上的资源，使用引用计数算法，类似C++中shared_ptr。**

`Rc::clone()`将引用计数+1，不会真的复制对象

 `Rc`没有实现`DerefMut`，因此其管理的是不可变引用

循环引用可能会产生内存泄露：因为两个对象相互引用，最终都不会得到释放

## RefCell

`Rc`管理的是不可变引用，这意味着如果我们选择用其管理堆上内存，则我们尽管可以`clone()`出多个引用，但这些引用都不被允许改变堆上内存。但很多时候我们需要对其修改(仍然考虑图)

`RefCell`是一个实现了引用计数的指针，同时可以修改管理的内存。这似乎听起来违背Rust内存管理的基本原则；但事实上只是把这个检查从编译期推迟到了运行时：

```rust
let x: RefCell<i32> = RefCell::new(1);
{
  let xptr1: Ref<'_, i32> = x.try_borrow().unwrap();
  let xptr2: Ref<'_, i32> = x.try_borrow().unwrap();
  assert_eq!(*xptr1, 1);
  assert_eq!(*xptr2, 1);

  // If we have any immutable borrows active, we can't have a mutable borrow
  assert!(match x.try_borrow_mut() { Err(_) => true, Ok(_) => false });
}
{
  let mut xmut: RefMut<'_, i32> = x.try_borrow_mut().unwrap();
  *xmut = 3;

  // If we have a single mutable borrow active, no further borrows are permitted
  assert!(match x.try_borrow_mut() { Err(_) => true, Ok(_) => false });
  assert!(match x.try_borrow() { Err(_) => true, Ok(_) => false });
}
```

API：

```rust
let ref1 = RefCell::new(Node::new(1i32, None, None));
```

- `as_ptr()`，转成裸指针：`let p1: *mut Node<i32> = ref1.as_ptr();`

## Cell

Cell是一种提供内部可变性的容器，类似智能手机电池，看似不可换，打开盖子后是可以换的

**适合实现了Copy的类型，或者体积小的struct，因为get方法是直接按位复制的。**
**无运行时开销，运行时安全**



## 总结

其实智能指针只有不计数的`Box`和计数引用`Rc`两种；

因为Rust的自动解引用的特性，使用`Box`或`Rc`包装的对象时可以像直接使用其内部值一样直接调用方法、取值：

```rust
let b1 = Box::new(Node::new(1,None,None));
println!("{:?}", b1.value);
let rc1 = Rc::new(Node::new(2,None,None));
let rc2 = rc1.clone();
let rc3 = &rc2;
println!("{:?}", rc3.value);
```

