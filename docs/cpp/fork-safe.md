# Fork Safe

## 问题所在

fork和多线程一起使用有时会引起死锁。

`fork`的行为：

- 把父进程整个虚拟地址空间copy到子进程中(COW)，包含**互斥变量、条件变量和其它线程对象的状态**。

- 子进程在单线程状态下被生成。

    > The child process is created with a single thread—the one that called fork(). 

上述两个特点就引起了`fork safe`问题

```cc
// compile: g++ ‐‐std=c++11 ‐lpthread mutex_deadlock.cpp
// execute: ./a.out
#include <pthread.h>
#include <time.h>
#include <unistd.h>
using namespace std;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
void* doit(void*) {
    pthread_mutex_lock(&mutex);
    struct timespec ts = {20, 0};
    nanosleep(&ts, 0);
    pthread_mutex_unlock(&mutex);
    return 0;
}
int main(void) {
    pthread_t t;
    pthread_create(&t, nullptr, doit, nullptr);
    if (fork() == 0) {
        doit(nullptr);
        return 0;
    }
    pthread_join(t, 0);
    return 0;
}
```

首先 `pthread_create()` 创建一个子线程。在子线程里，`doit()` 函数对全局互斥量加锁，sleep 20 秒后释放。而后，与子线程同时进行的，在主线程中，我们调用 `fork()` 函数，创建一个子进程。并且，在子进程里，我们也调用 `doit()` 函数，尝试获取互斥量。结果是子进程无法获得mutex，形成死锁。

这是因为主线程`fork`出子进程的时候，把`mutex`也一并复制(上锁状态)，但是`fork`只复制主线程，不复制子线程，当主进程的子线程释放锁的时候，并不会对子进程的mutex造成影响，子进程的mutex仍然处于上锁状态，且没有机会得到释放，从而造成死锁。

像这里的`doit`函数那样的，在多线程里因为fork而引起问题的函数,我们把它叫 做”fork-unsafe函数”.反之,不能引起问题的函数叫做”fork-safe函数”.虽然在一些商用的UNIX里，源于OS提供的函数(系统调用),在文档里有`fork-safety`的记载，但是在 Linux(`glibc`)里当然不会被记载.即使在POSIX里也没有特别的规定，所以那些函数是fork-safe的，几乎不能判别.不明白的话，作为unsafe考虑的话会比较好一点吧.

`malloc`函数就是一个典型的例子。`malloc`是`thread safe`，其分配内存的时候会先尝试获得并加锁。因此如果子线程中进行`malloc`时，主线程恰好进行了`fork`，则会造成死锁。

## 应对方法

当然也有相应的应对措施，那就是在`fork`之后马上调用`exec`函数。把原本子进程应该做的事情写成一个单独的程序，编译成可执行程序后由exec函数来调用。因为`exec`族的函数会清空内存，因此避免了上述问题。但是?

TODO: