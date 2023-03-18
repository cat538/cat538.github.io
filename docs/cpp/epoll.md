# Epoll

> **Ref**:
> 
> - 原文：[The method to epoll’s madness](https://copyconstruct.medium.com/the-method-to-epolls-madness-d9d2d6378642)
> - [epoll 原理是如何实现的？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/486578358/answer/2126762689)

epoll是Linux特有的构造。它允许**单个**进程监视**多个**文件描述符，并在文件描述符可能出现I/O时获得通知——边缘触发或水平触发通知。在深入了解epoll的本质之前，我们先来看看它的语法。

## The syntax of epoll 

与`poll`不同，epoll本身不是一个系统调用。它是一种内核数据结构，允许进程在多个文件描述符上进行I/O复用。

![img](https://miro.medium.com/max/610/1*k-PycQSwivn-jIoEhXsvxg.png)

这个数据结构可以通过三个系统调用来创建、修改和删除：

1. [**epoll_create**](http://man7.org/linux/man-pages/man2/epoll_create.2.html)

   epoll实例是通过系统调用`epoll_create`创建的，它将一个文件描述符返回给epoll实例。`epoll_create`的签名如下:

   ```c
   #include <sys/epoll.h>
   int epoll_create(int size);
   ```

   `size`参数向内核表明一个进程希望监视的文件描述符的最大值，这有助于内核决定epoll实例的大小；但是从Linux 2.6.8 开始这个值被忽略掉，因为epoll数据结构会随着文件描述符的添加或删除而动态调整大小。但为了向前兼容`size`仍然需要大于0。

   `epoll_create`让内核产生一个epoll 实例并返回一个文件描述符，这个特殊的描述符就是epoll实例的句柄，后面的两个接口都以它为中心(即`epfd`形参)。进程可以通过这个文件描述符向epoll实例添加、删除或修改它希望监视I/O的其他文件描述符。

   ![img](https://miro.medium.com/max/677/1*o21hEWChu-cNHDj49xgCkg.jpeg)

   还有一个类似的系统调用`epoll_create1`，签名如下:

   ```c
   int epoll_create1(int flags);
   ```

   `flags`参数可以是`0`或`EPOLL_CLOEXEC`

   - 当设置为0时，`epoll_create1`的行为与`epoll_create`相同。

   - 当设置了`EPOLL_CLOEXEC`标志时，由当前进程`fork`出的任何子进程都将在`execs`之前关闭epoll文件描述符，因此子进程将不再有访问epoll实例的权限。

   **需要注意的是**：与epoll实例关联的文件描述符需要通过`close()`系统调用来释放。多个进程可能持有同一个epoll实例的描述符，因为，比方说进程调用没有设置`EPOLL_CLOEXEC`的`epoll_create1()`，`fork`将把描述符复制到子进程中。当所有这些进程都将它们的epoll描述符放弃时(通过调用close()或退出)，内核将销毁epoll实例。

   调用`epoll_create`时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个**红黑树**用于存储以后`epoll_ctl`传来的`fd`外，还会再建立一个list链表，用于存储准备就绪的事件.

2. [**epoll_ctl**](http://man7.org/linux/man-pages/man2/epoll_ctl.2.html)

   进程可以通过调用`epoll_ctl()`向epoll实例添加它想要监视的文件描述符 / 删除它不再想监视的描述符 / 修改监视条件(类似`select`对应的`FD_SET`和`FD_CLR`)。注册到epoll实例的所有文件描述符统称为**epoll set**或**interest list**

   ![img](https://miro.medium.com/max/677/1*Abjr5spvjK56w1p7PszWVw.jpeg)

   在上图中，进程483向epoll实例注册了文件描述符fd1、fd2、fd3、fd4和fd5。这是该特定epoll实例的**epoll set**或称作**interest list**。随后，当任何已注册的文件描述符变为ready状态时，就认为它们在就绪列表(**ready list**)中——**ready list**是**interest list**的子集：

   ![img](https://miro.medium.com/max/677/1*24HukCwzdkH0Vb8n-RHlFw.jpeg)

   `epoll_ctl`签名如下:

   ```c
   #include <sys/epoll.h>
   int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
   ```

   - `epfd`，是`epoll_create`返回的文件描述符，它在内核中标识epoll实例

   - `fd`，是我们想要添加到**epoll set / interest list**中的文件描述符

   - `op`，指要对文件描述符fd执行的操作。一般来说，支持三种操作：

     - **Register** **fd** ，`EPOLL_CTL_ADD`向`interest list`添加一个需要监视的描述符
     - **Delete/ Deregister fd**，`EPOLL_CTL_DEL`从`interest list`中删除一个描述符。如果一个文件描述符被添加到多个epoll实例中，那么关闭它将从添加它的**所有**epoll兴趣列表中删除它。
     - **Modify**，`EPOLL_CTL_MOD`对`interest list`中一个描述符监听事件进行修改

     ![img](https://miro.medium.com/max/677/1*oZMBCl_zVPqyOfbGLSkcjA.jpeg)

     

   - `event`，是一个指向`epoll_event`结构体的指针，该结构体存储了我们实际想要监视fd的事件。

   ![img](https://github.com/cat538/images-auto/raw/main/img/1KDk1AVzQJegkcWKJQURYfw.jpeg)

   `epoll_event`结构如下：

   ```c
   typedef union epoll_data {
       void *ptr; /* 指向用户自定义数据 */
       int fd; /* 注册的文件描述符 */
       uint32_t u32; /* 32-bit integer */
       uint64_t u64; /* 64-bit integer */
   } epoll_data_t;
   
   struct epoll_event {
       uint32_t events; /* 描述epoll事件 */
       epoll_data_t data; /* 见上面的结构体 */
   };
   ```

   - `data`域是唯一能给出描述符信息的字段，所以在调用`epoll_ctl`加入一个需要检测的描述符时，必须要在此域写入描述符相关信息
   - `events`域是`bit mask`，描述一组`epoll`事件，在`epoll_ctl`调用中解释为：描述符所期望的epoll事件，可多选。选项列在枚举`EPOLL_EVENTS`中，常用的有：`EPOLLIN`描述符处于可读状态，`EPOLLOUT`描述符处于可写状态，`EPOLLONESHOT`第一次进行通知，之后不再监测，`EPOLLET`将epoll event通知模式设置成edge triggered

   在使用 epoll_ctl 注册每一个`fd`的时候，内核会做如下三件事情：

   1. 分配一个红黑树节点对象 epitem，
   2. 添加等待事件到`fd`的等待队列中，并给内核中断处理程序注册一个回调函数，告诉内核，如果这个句柄的中断到了，就把它放到`ready list`链表里
   3. 将`epitem`插入到 epoll 对象的红黑树里

3. [**epoll_wait**](http://man7.org/linux/man-pages/man2/epoll_wait.2.html)

   线程通过调用`epoll_wait`来获知epoll实例的**interest list**上发生的事件，`epoll_wait`系统调用会阻塞，直到any of descriptors 注册的事件发生(或计时器超时)。

   `epoll_wait`的签名如下:

   ```c
   #include <sys/epoll.h>
   int epoll_wait(int epfd, struct epoll_event *evlist, int maxevents, int timeout);
   ```

   - `epfd`，是`epoll_create`返回的文件描述符，标识epoll实例
   - `evlist`是`epoll_event`结构的数组。`evlist`由调用进程分配，当`epoll_wait`返回时，这个数组将被修改，以表示兴趣列表中处于就绪状态的文件描述符子集的信息(这称为就绪列表**ready list**)。
   - `maxevents`，是`evlist`的长度
   - `timeout`，此参数的行为与`poll`或`select`相同。这个值指定了`epoll_wait`系统调用将阻塞多长时间:
   
     - 设置为0时，`epoll_wait`不阻塞，在检查`epfd`的**interest list**中哪些文件描述符处于就绪态后立即返回
   
     - 设置为-1时，`epoll_wait`将一直阻塞，直到
   
       1. `epfd`**interest list**中指定的1个或多个描述符就绪，或者
   
       2. 调用被信号处理程序中断
   
     - 设置为一个正数时，`epoll_wait`将阻塞,直到
   
       1. 兴趣列表中指定1个或多个描述符epfd准备好，或
       2. 调用中断信号处理器，或
       3. 达到指定的超时的时间(单位：毫秒)
   - 返回值：
   
     1. 发生错误返回-1
     2. 如果调用在**epoll set**中的任何文件描述符准备就绪之前超时，则返回代码为0
     3. 如果**epoll set**中的一个或多个文件描述符准备好了，那么返回的代码是一个正整数，表示`evlist`数组中文件描述符的总数。然后检查`evlist`以确定哪些事件发生在哪些文件描述符上

   `epoll_wait`做的事情不复杂，当它被调用时它观察`ready list`链表里有没有数据即可。有数据就返回，没有数据就创建一个等待队列项，将其添加到 eventpoll 的等待队列上，然后把自己阻塞掉就完事。

![epoll](https://github.com/cat538/images-auto/raw/main/img/epoll.jpg)

TODO: