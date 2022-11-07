## Move Semantic
- rvalue 和 lvalue 都可以传递给参数类型为 `const T&`的函数

[MS: How to write a move constructor and move assignment operator](https://learn.microsoft.com/en-us/cpp/cpp/move-constructors-and-move-assignment-operators-cpp?view=msvc-170)

[c++ - Why is it possible to pass an rvalue by const lvalue reference? - Stack Overflow](https://stackoverflow.com/questions/58062090/why-is-it-possible-to-pass-an-rvalue-by-const-lvalue-reference)

[c++ - Confusion between rvalue references and const lvalue references as parameter - Stack Overflow](https://stackoverflow.com/questions/44738829/confusion-between-rvalue-references-and-const-lvalue-references-as-parameter)
这个提问讲了`const lvalue reference` 和 `rvalue reference`，即`const T&`和`T&&`之间的区别。

我在初学的时候遇到了同样的疑问，之所以会有这样的confusion是因为我认为`const T&`能够接收lvalue和rvalue如下：
```c++
void calculator(const Intvec& veccor) { 
  cout << "in veccor" << veccor.m_size << "\n"; 
}

calculator(Intvec(33)); //works
Intvec newvec(22);
calculator(newvec); //works
```

## Perfect forwarding
Perfect forwarding 完美转发