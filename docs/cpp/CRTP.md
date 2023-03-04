# CRTP

C++ CRTP 是一个 C++ idiom，它的全称是 Curiously Recurring Template Pattern（奇异递归模板模式）。它的名字是由 James Coplien 在 1995 年提出的，是早期 C++ 模板代码中的一个构造。CRTP 的特点是一个类 X 继承自一个以 X 为模板参数的类模板。

- CRTP的优点是可以避免使用虚拟表来实现多态，从而提高性能；CRTP的缺点是不能在运行时动态地改变接口的实现

- virtual动态派发的优点是可以在运行时动态地改变接口的实现，从而实现更强大的多态；virtual动态派发的缺点是需要为每个对象增加一个虚拟指针3，从而增加内存开销，并且不能内联虚拟函数3，从而降低性能。

## 实现细节

父类声明为模板，子类继承父类时，**将自己的类型作为模板参数传递给父类模板**；在父类中声明公共API，但是具体的实现通过 **在父类中`static_cast<Derived*>(this)` 进行静态dispatch** 转发给子类

```c++
template <class Derived>
struct Base { void name() { (static_cast<Derived*>(this))->impl(); } };
struct D1 : public Base<D1> { void impl() { std::puts("D1::impl()"); } };
struct D2 : public Base<D2> { void impl() { std::puts("D2::impl()"); } };

template<typename T>
void SayMyName(Base<T>* obj){
  obj->name();
}

int main() {
  D1 d1;
  D2 d2;
  SayMyName(&d1);
  SayMyName(&d2);
}
```

在 `D1 : public Base<D1>` 中， `Base<D1>` 是先于 D1 而存在的，所以当 `Base<D1>::name()` 被申明时，编译器并不知道 `D1` 的存在的，但由于此时 `Base<D1>::name()` 并不会被实例化。只有当 `Base<D1>::name()` 被调用时，才会被实例化，而此时编译器也已经知道了 `D1::impl()` 的声明了。


对比传统 virtual 做法：

```c++
struct Base { virtual void name(); };
struct D1 : public Base { void name() override { std::puts("D1::impl()"); } };
struct D2 : public Base { void name() override { std::puts("D2::impl()"); } };

void SayMyName(Base* obj){
  obj->name();
}

int main() {
  D1 d1;
  D2 d2;
  SayMyName(&d1);
  SayMyName(&d2);
}
```

- virtual 调用过程：call `SayMyName` 时， 传入的指针被转换成了 `obj` —— Base类型指针， 在调用 `obj->name` 时， 会通过查找虚函数表， 根据 RTTI 信息将调用派发到真正的运行时类型的对应方法上
 
- CRTP call `SayMyName` 时， 传入的指针则是被转换成了 `Base<T>` 类型的指针， 这里的 `T` 是子类类型， 在调用 `obj->name` 时， 会直接根据模板参数调用子类中对应方法，省去了查虚函数表的开销

## 应用实例

TODO: