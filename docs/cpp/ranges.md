# Ranges

C++20 引入`ranges` library 来源于 `ranges-v3`，它是算法和迭代器库的扩展和泛化，
使它们可组合且不易出错。

`ranges` 库的核心在于创建和操作range *views*，*views*是一种轻量级的object(类比Rust中的*slice*)

## Range concepts
首先了解`ranges`提供的几个基础的concept:

1. *range*
    ```C++
    template< class T >
    concept range = requires( T& t ) {
        ranges::begin(t);
        ranges::end  (t);
    };
    ```
    其中`ranges::begin()`Returns an iterator to the first element of the argument.

    而`ranges::end()`Returns a sentinel indicating the end of a range.  
    > [Tutorial: C++20’s Iterator Sentinels](https://www.foonathan.net/2020/03/iterator-sentinel/)

    可以看到对于类型T，如果expression `ranges::begin(t);`和`ranges::end(t)`成立，那么T就是一个range。其含义为我们可以通过提供T的一个迭代器和一个sentinel在该范围内进行迭代。

2. *view*
    ```C++
    template<class T>
    concept view = ranges::range<T> && std::movable<T> && ranges::enable_view<T>;
    ```

## Views
views是ranges库的核心，ranges库功能有多丰富，取决于其提供多少种views。在`std::ranges`中预定义了一系列的views类型

在`std::ranges::views`中提供了一系列工厂方法以及adaptor来创建views / 将range 转换成views

- 工厂方法`views::repeat`，`views::iota`等对应返回`ranges::repeat_view`，`views::iota_view`等类型

另外还有一系列range adaptor object，他们可以通过管道符号进行链接形成链式调用。如`views::join`, `views::split`等。下面以`views::take`为例：

- `ranges::take_view` is a view consisting of the first N elements of another view
  
    `views::take` is a range adaptor object. The expression views::take(e, f) results in a view that represents the first f elements from e. The result is not necessarily a take_view.

    ```C++
    template< ranges::viewable_range R, class DifferenceType >
    requires 
    constexpr ranges::view auto take( R&& r, DifferenceType&& count );
    ```

## borrowed range

前文提到

> `ranges` 库的核心在于创建和操作range *views*，*views*是一种轻量级的object(类比Rust中的*slice*)

事实上，能够被range-ified algorithm操作的不止是*views*，还有一种类型即 *borrowed range*。

borrowed range specifies that a type is a [`range`](https://en.cppreference.com/w/cpp/ranges/range) and iterators obtained from an expression of it can be safely returned without danger of dangling

```C++
template<class R>
concept borrowed_range = ranges::range<R> 
    && (std::is_lvalue_reference_v<R> || 
        ranges::enable_borrowed_range<std::remove_cvref_t<R>>);
```

其中`ranges::enable_borrowed_range` 默认为false，标准库为一些类型做了specialization，如`std::string_view`,`std::span`, `std::ranges::subrange`等.

因为这些类型并没有其所限定范围内元素的ownership，因此当这些类型的rvalue作为参数传入algorithm的时候(如`ranges::min_element()`)，不会返回`std::ranges::dangling`
