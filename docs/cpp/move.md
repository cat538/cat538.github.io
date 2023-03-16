# Move

## Lvalues and Rvalues
> Ref:
> 
> - [Move semantics in C++ and Rust: The case for destructive moves](https://radekvit.medium.com/move-semantics-in-c-and-rust-the-case-for-destructive-moves-d816891c354b)
> - [MS: How to write a move constructor and move assignment operator](https://learn.microsoft.com/en-us/cpp/cpp/move-constructors-and-move-assignment-operators-cpp?view=msvc-170)


在 C++ 中，每个表达式不仅有一个type，还有一个value category。**注意这两者是不同的概念，一个type为 rvalue reference (T&&)的值，它的value category可能是左值也可能是右值。** cpp中有三个主要的value category和两个混合类型value category。每个表达式都有一个主要的value category，它决定了语言将如何处理它与其他表达式的关系。

- **lvalue**: 简单地说， **lvalue** 是一个变量；**有名字**，**可以取内存地址**
- **xvalue**: 就像一个 **lvalue**，但是我们声明<u>这个变量拥有的资源可以转移给一个新的所有者</u>
- **prvalue**: 没有名称的临时值，**无法取地址**
- **glvalue** (mixed) is either an **lvalue** or an **xvalue**.
- **rvalue** (mixed) is either an **xvalue** or a **prvalue**.

<div class="autocb" style="text-align:center;"><img src="./move.assets\autocb_0.png" style="zoom: 50%;box-shadow: rgba(0, 0, 0, 0.5) 10px 10px 10px; border-radius: 10px;" /></div>

```c++
auto i = std::string{"value categories"};
// `i` is an lvalue
// `std::move(i)` is an xvalue
// `static_cast<std::string&&>(i)` is an xvalue
// `std::string{"value categories"}` is a prvalue (pure rvalue)
```

## Rvalue Reference
顾名思义，**rvalue reference** 是指向**rvalue**的引用。

通过右值引用，我们可以区分左值和右值：因为右值引用只能绑定右值(rvalue)，因此如果一个函数的参数是右值引用，那么传进到该函数的参数一定是右值：

```cpp
void foo(string&& ss){
  bar(std::move(ss));
}
```

例如上述代码中的`foo`，**这里最重要的一点是，我们能够知道`ss`是一个右值引用，因此任何能绑定到它的东西（即任何能传进`foo`这个函数的参数）都是右值**，这样我们就可以从`ss`中窃取资源（但是这里注意，具名右值引用的值类别是左值，因此需要`std::move`将`ss`作为右值对待，后面会再次提到）

最常见的是 move constructor，这里用 copy constructor 作为对比：

```cpp
struct MyData {
    std::string data1;
    std::string data2;
    MyData() noexcept = default;
    // this is (basically) what the compiler will generate for you
    // never write these by hand unless you're managing resources
    // copy constructor
    MyData(const MyData& other)
        : data1{other.data1}
        , data2{other.data1}
    {}
    // copy assignment
    MyData& operator=(const MyData& other) {
         data1 = other.data1;
         data2 = other.data2;
         return *this;
    }
    // move constructor
    MyData(MyData&& other) noexcept
        : data1{std::move(other.data1)}
        , data2{std::move(other.data1)}
    {}
    // move assignment
    MyData& operator=(MyData&& other) noexcept {
         data1 = std::move(other.data1);
         data2 = std::move(other.data2);
         return *this;
    }
};
```

管理资源的类，如 `std::vector<T>`、`std::string`，通常会在它们的移动构造函数中执行以下操作：不分配新内存，而是从正在构造的 rvalue 中获取已经分配的缓冲区，并留下一些有效的价值来代替它。move assignment 通常会简单地将分配的资源与右值交换，在赋值调用后它们将被移出的右值释放。

```cpp
template <typename T>
class almost_vector {
    T* buffer = nullptr;
    T* data_end = nullptr;
    T* buffer_end = nullptr;
public:
    almost_vector() noexcept = default;
    almost_vector(const almost_vector& other)
    {
        // allocate buffer, copy elements
    }
    almost_vector& operator=(const almost_vector& other) {
        // allocate new buffer, copy elements
        // swap the buffers
        // deallocate the old buffer
    }
    // the move constructor will do something like this
    almost_vector(almost_vector&& other) noexcept
    {
        std::swap(buffer, other.buffer);
        std::swap(data_end, other.data_end);
        std::swap(buffer_end, other.buffer_end);
    }
    // move assignment will do something like this
    almost_vector& operator=(almost_vector&& other) noexcept {
        std::swap(buffer, other.buffer);
        std::swap(data_end, other.data_end);
        std::swap(buffer_end, other.buffer_end);
        return *this;
    }
};
```

💡**rvalue reference**有不直观的一面：

- `std::move(data)` 仅仅只将data的类型强制转换为`T&&`；不要想象`std::move()`会销毁`data`或者修改data的内存等等
- **具名右值引用是左值**：`string&& s2 = std::move(s1)`，这里`s2` 的 value category 是 lvalue；如果有如下代码：

    ```c++
    auto s1 = string{"123"};
    string&& s2 = std::move(s1);
    auto s3 = s2; // 这里调用的是 copy constructor
    ```
  
    如代码中注释所示，第三行将会调用`std::string`的拷贝构造函数，因为这里的`s2`虽然值类型是右值引用，**但它的值类别是左值**，根据[Move constructors](https://en.cppreference.com/w/cpp/language/move_constructor)中 Note一节（如下），此时会选择copy constructor

    > If both copy and move constructors are provided and no other constructors are viable, overload resolution selects the move constructor if the argument is an rvalue of the same type (an xvalue such as the result of std::move or a prvalue such as a nameless temporary (until C++17)), and **selects the copy constructor if the argument is an lvalue** (named object or a function/operator returning lvalue reference).
- 右值引用只能绑定右值，左值引用(non-const)只能绑定左值，**const lvalue reference既可以绑定左值，又可以绑定右值**；[c++ - Why is it possible to pass an rvalue by const lvalue reference? - Stack Overflow](https://stackoverflow.com/questions/58062090/why-is-it-possible-to-pass-an-rvalue-by-const-lvalue-reference)

    ```c++
    string s{"abc"};
    string& s1 = std::move(s);  // error! std::move(s) is xvalue
    string& s2 = string{"123"}; // error! string{"123"} is prvalue
    string&& s3 = s;    // error! s is lvalue
    const string& s4 = string{"123"};  // ok
    const string& s5 = s;              // ok
    const string& s6 = std::move(s);   // ok
    ```


除了**lvalue reference**和**rvalue reference**之外，C++ 中还存在第三种引用：**the forwarding reference**。在模板函数中，`T&&` 成为转发引用而不是右值引用，另外，`auto&&` 始终是转发引用。转发引用保留它们初始化时使用的表达式的值类别，并且可以在传递给其他函数时保留。

```cpp
std::string baz(std::string);
template <typename T>
struct Templated {
    // t is an rvalue reference， 可以想一下为什么🎉
    void foo(T&& t) {}
    // u is a forwarding reference
    template <typename U>
    void bar(U&& u) {
        // forward to another function
        // x is a forwarding reference
        auto&& x = baz(std::forward<U>(u));
    }
};
```

更详细的内容见[Perfect forwarding](#perfect-forwarding)


## Move 的设计

一个有趣的问题是，object 被 move 之后，处于什么状态？

C++ 标准库选择将移出的变量保持在有效但未指定的状态(valid but unspecified state)；这意味着我们可以重用变量，我们只是不能依赖它的内容。 

移动的变量在 C++ 中移动后仍然可用。变量的析构函数将在这些变量到达其生命周期结束时运行，用户可以为它们赋值，或调用它们的任何成员函数。 

对于用户声明的类型，唯一真正的要求是移出变量的析构函数必须在运行时不会对程序的其余部分造成任何问题。类型的任何不变量都可能被破坏，并且在它们上调用任何函数都可能导致未定义的行为，我们通常不做这些事情只是约定俗成（和方便）的问题。

```cpp
auto s    = "123abc"s;
auto vec  = vector<string>{};
vec.emplace_back(std::move(s));

// 在move后，s仍然是可用的，这与rust非常不同
```

在Rust中 TODO:

## Forwarding Reference
如果一个类型占位符 P 需要被类型推导，那么`P&&` 将成为C++ 中 除了 左值引用和右值引用 之外的另一种引用： 转发引用 forwarding reference

例如，在模板函数中，`T&&` 是 forwarding reference 而不是右值引用； 类似地， auto&& 始终是一个 forwarding reference。

转发引用**保留它们初始化时使用的表达式的值类别**，并且可以在传递给其他函数时保留，这意味着，如果被绑定的 value 是一个 右值/右值引用， 则`T&&`成为一个右值引用，如果被绑定的value是一个左值/左值引用，则`T&&` 会成为一个左值引用（加上const 同理）。

引用折叠和 template argument deduction rules 构成了转发引用的语法规则：

### 引用折叠(Reference collapsing)

对右值引用的右值引用折叠为右值引用，所有其他组合形成左值引用：

```c++
typedef int&  lref;
typedef int&& rref;
int n;
 
lref&  r1 = n; // type of r1 is int&
lref&& r2 = n; // type of r2 is int&
rref&  r3 = n; // type of r3 is int&
rref&& r4 = 1; // type of r4 is int&&
```

因此对于以下代码：

```c++
auto s          = "123"s;
auto& sr        = s;
const auto& scr = s;

auto&& r1 = s;               // string&
auto&& r2 = sr;              // string&
auto&& r3 = scr;             // const string&
auto&& r4 = "123"s;          // string&&
auto&& r5 = std::move(s);    // string&&

// T&& 与 auto&& 都是 "转发引用"
template<typename T>
foo(T&& arg){
  static_assert(std::same_as_v<std::string, decltype(arg)>);
}
// T&& 与 auto&& 同理， C++20 之后可以更广泛使用auto代替模板参数， 即 "bar" 与 "foo" 等价
bar(auto&& arg){
  static_assert(std::same_as_v<std::string, decltype(arg)>);
}
```

### std::forawrd()
Forwarding Reference 的应用场景**仅在泛型编程中**：

在泛型编程中与参数列表绑定的可能是左值也可能是右值，那么这时候我们需要有一种方式能够保持参数的值语义转发
给其他的函数调用：

```c++
class BigObj {
 public:
  BigObj() = default;
  BigObj(unique_ptr<int[]> data, int len) : data_(std::move(data)), len_(len) {
    cout << "Create from raw\n";
  };
  BigObj(const BigObj& rhs) {
    cout << "Copy constructor: const BigObj&\n";
    len_  = rhs.len_;
    data_ = make_unique<int[]>(len_);
    memcpy(data_.get(), rhs.data_.get(), len_);
  };
  BigObj(BigObj&& rhs) noexcept {
    cout << "Move constructor: BigObj&&\n";
    len_ = rhs.len_;
    data_.swap(rhs.data_);
  };
  BigObj& operator=(const BigObj& rhs) {
    cout << "Copy assignment: const BigObj&\n";
    len_  = rhs.len_;
    data_ = make_unique<int[]>(len_);
    memcpy(data_.get(), rhs.data_.get(), len_);
    return *this;
  };
  BigObj& operator=(BigObj&& rhs) noexcept {
    cout << "Move assignment: BigObj&&\n";
    len_ = rhs.len_;
    data_.swap(rhs.data_);
    return *this;
  }

 private:
  unique_ptr<int[]> data_;
  int len_;
};

template <typename T, typename U>
concept same_rcv = same_as<remove_cvref_t<T>, U>;

class A {
 public:
  template <typename T, typename U, typename V>
    requires same_rcv<T, BigObj> && same_as<remove_cvref_t<U>, BigObj> &&
             same_as<remove_cvref_t<V>, BigObj>
  A(T&& v1, U&& v2, V&& v3) {
    v1_ = std::forward<T>(v1); // 使用std::forward 根据入参的value category 调用BigObj 的move or copy 构造
    v2_ = std::forward<U>(v2); // 使用std::forward 根据入参的value category 调用BigObj 的move or copy 构造
    v3_ = std::forward<V>(v3); // 使用std::forward 根据入参的value category 调用BigObj 的move or copy 构造
  }

 private:
  BigObj v1_;
  BigObj v2_;
  BigObj v3_;
};

int main() {
  auto len    = 1024;
  auto buffer = make_unique_for_overwrite<int[]>(len);

  auto obj1    = BigObj(std::move(buffer), len);  // Create from raw
  auto obj2    = obj1;  // Copy constructor! Not copy assignment operator!
  auto obj3    = obj1;  // copy constructor too
  auto obj4    = obj1;  // copy constructor too
  BigObj&& tmp = std::move(obj4);
  auto obj5 = tmp;  // Copy constructor !!!!! (具名右值引用是左值)

  auto a = A(obj1, std::move(obj2), obj3); // [copy, move, copy] constructor
}
```