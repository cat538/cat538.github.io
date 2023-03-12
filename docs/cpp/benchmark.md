# Benchmark

这里介绍两个库，`Google Benchmark`以及`Catch2`。
其中前者使用最广泛，后者是单元测试框架，但是也加入了benchmark功能。

在了解Catch2的benchmark之前，首先回顾一下Google的Benchmark：
## Google benchmark

- 使用`BENCHMARK`macro注册benchmark函数：

    ```C++
    #include <benchmark/benchmark.h>
    #include <stdint.h>
    std::uint64_t FibonacciRecur(std::uint64_t number) {
      return number < 2 ? 1 : FibonacciRecur(number - 1) + FibonacciRecur(number - 2);
    }
    std::uint64_t FibonacciIter(std::uint64_t number) {
      int pre{1}, cur{1}, tmp{};
      for (size_t i = 1; i < number; i++) {
        tmp = cur;
        cur += pre;
        pre = tmp;
      }
      return cur;
    }
    static void BM_FibRecur(benchmark::State& state){
      for (auto _ : state) FibonacciRecur(state.range(0));
    }
    static void BM_FibIter(benchmark::State& state){
      for (auto _ : state) FibonacciIter(state.range(0));
    }

    BENCHMARK(BM_FibRecur)->Arg(20)->Arg(25)->Arg(30);
    BENCHMARK(BM_FibIter)->DenseRange(10, 30, 5);
    specific step[]
    ```

    - 其中`DenseRange`：Run this benchmark once for all values in the range [start..limit] with specific step
    - 其中 `Arg()` ：向目标函数传入的参数，注意在`BM_*`函数中要使用 `state.range(x)` 来指明第`x`个参数；传入多个参数时(如三个)类似： `Arg({1, "2", 3})`

- 防止编译器优化：

    使用`benchmark::DoNotOptimize()`

    ```C++
    std::uint64_t FibonacciIter(std::uint64_t number) {
      int pre{1}, cur{1}, tmp{};
      for (size_t i = 1; i < number; i++) {
          benchmark::DoNotOptimize(tmp = cur);
          benchmark::DoNotOptimize(cur += pre);
          benchmark::DoNotOptimize(pre = tmp);
      }
      return cur;
    }
    ```

- 使用 `Unit` 指定读数单位：

    ```c++
    BENCHMARK(FibonacciIter)
      ->Unit(benchmark::kMillisecond)
      ->RangeMultiplier(1 << 1)
      ->Range(1ll, 1ll << 7);
    ```