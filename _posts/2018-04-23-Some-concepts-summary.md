---
title: Some Concepts
categories:
- 概念总结记录
tags:
- synchronous
- asynchronous
- blocking
- non-blocking
- thread
- process
- coroutine
---
Openstack 中使用到了许多异步和同步，堵塞和非堵塞的概念，在网络上找了些好文章这里针对这些概念和找到的资料做些记录总结。

# 概念

## 同步/异步/堵塞/非堵塞

关于同步异步，堵塞非堵塞，[知乎](https://www.zhihu.com/question/19732473) 上又个很通俗易懂的例子：

> 老张爱喝茶，废话不说，煮开水。
> 出场人物：老张，水壶两把（普通水壶，简称水壶；会响的水壶，简称响水壶）。
> 1 老张把水壶放到火上，立等水开。（同步阻塞）
> 老张觉得自己有点傻
> 2 老张把水壶放到火上，去客厅看电视，时不时去厨房看看水开没有。（同步非阻塞）
> 老张还是觉得自己有点傻，于是变高端了，买了把会响笛的那种水壶。水开之后，能大声发出嘀~~~~的噪音。
> 3 老张把响水壶放到火上，立等水开。（异步阻塞）
> 老张觉得这样傻等意义不大
> 4 老张把响水壶放到火上，去客厅看电视，水壶响之前不再去看它了，响了再去拿壶。（异步非阻塞）
> 老张觉得自己聪明了。
>
> 所谓同步异步，只是对于水壶而言。
> 普通水壶，同步；响水壶，异步。
> 虽然都能干活，但响水壶可以在自己完工之后，提示老张水开了。这是普通水壶所不能及的。
> 同步只能让调用者去轮询自己（情况2中），造成老张效率的低下。
>
> 所谓阻塞非阻塞，仅仅对于老张而言。
> 立等的老张，阻塞；看电视的老张，非阻塞。
> 情况1和情况3中老张就是阻塞的，媳妇喊他都不知道。虽然3中响水壶是异步的，可对于立等的老张没有太大的意义。所以一般异步是配合非阻塞使用的，这样才能发挥异步的效用。



当理解上面比较通俗的概念后，可以看下这个来自知乎上比较专业的简介：



> “阻塞”与"非阻塞"与"同步"与“异步"不能简单的从字面理解，提供一个从分布式系统角度的回答。
>
> 1. **同步与异步**
>
> 同步和异步关注的是**消息通信机制** (synchronous communication/ asynchronous communication)
> 所谓同步，就是在发出一个*调用*时，在没有得到结果之前，该*调用*就不返回。但是一旦调用返回，就得到返回值了。
> 换句话说，就是由*调用者*主动等待这个*调用*的结果。
>
> 而异步则是相反，**调用在发出之后，这个调用就直接返回了，所以没有返回结果**。换句话说，当一个异步过程调用发出后，调用者不会立刻得到结果。而是在*调用*发出后，*被调用者*通过状态、通知来通知调用者，或通过回调函数处理这个调用。
>
> 典型的异步编程模型比如Node.js
>
> 举个通俗的例子：
> 你打电话问书店老板有没有《分布式系统》这本书，如果是同步通信机制，书店老板会说，你稍等，”我查一下"，然后开始查啊查，等查好了（可能是5秒，也可能是一天）告诉你结果（返回结果）。
> 而异步通信机制，书店老板直接告诉你我查一下啊，查好了打电话给你，然后直接挂电话了（不返回结果）。然后查好了，他会主动打电话给你。在这里老板通过“回电”这种方式来回调。
>
> 2. **阻塞与非阻塞**
>
> 阻塞和非阻塞关注的是**程序在等待调用结果（消息，返回值）时的状态.**
>
> 阻塞调用是指调用结果返回之前，当前**线程会被[挂起](https://www.zhihu.com/question/42962803)**。调用线程只有在得到结果之后才会返回。
> 非阻塞调用指在不能立刻得到结果之前，该调用**不会阻塞当前线程**。
>
> 还是上面的例子，
> 你打电话问书店老板有没有《分布式系统》这本书，你如果是阻塞式调用，你会一直把自己“挂起”，直到得到这本书有没有的结果，如果是非阻塞式调用，你不管老板有没有告诉你，你自己先一边去玩了， 当然你也要偶尔过几分钟check一下老板有没有返回结果。
> 在这里阻塞与非阻塞与是否同步异步无关。跟老板通过什么方式回答你结果无关。

关于 [**Linux I/O 模型**的图解](https://www.ibm.com/developerworks/cn/linux/l-async/)

# Python 协程/线程/进程

## **进程**：

1. 由操作系统进行管理
2. 每个进程都有自己的虚拟地址空间（memory）
3. 可以被操作系统中断，跑其他的进程
4. 可以**并行 （parallel，not concurency）**运行在不同的 processor（core）上
5. 进程内存开销比较大，且进程间 context switch 时间开销比较高
6. **进程（process）是正在运行的程序的实例，但一个程序可能会产生多个进程**。每个进程都有自己的地址空间，内存，数据栈以及其他记录其运行状态的辅助数据，不同的进程只能使用消息队列、共享内存等进程间通讯（IPC）方法进行通信，而不能直接共享信息。

## **线程**：

1. 由操作系统进行管理

2. 在同一个进程中的所有线程分线一个进程的虚拟地址空间（memory）

3. 线程可以被操作系统中断，跑其他的线程

4. 可以**[并行 （parallel，not concurency 并发）](https://www.zhihu.com/question/33515481)**运行在不同的 processor（core）上

   1. **但是 Python 线程不可以并行， 因为Python interpreter 使用 GIL，每个 python 进程有一个 GIL，如果是多线程（在同一个进程中）则共享 GIL，导致python不可以利用物理机上的多个cpu，同时处理线程。所以 python 经常使用 多进程来代替 多线程并行。**
   2. ![](https://ws3.sinaimg.cn/large/006tNc79gy1fqmnzxgqzoj30ci09amxg.jpg)

5. 多线程间 context switch 时间开销，和memory 开销也会比较大。(For example, typically context switching involves entering the kernel and invoking the system scheduler.)

6. **线程（thread）是进程（process）中的一个实体，一个进程至少包含一个线程**

   **进程和线程的区别主要有：**

   > - 进程之间是相互独立的，多进程中，同一个变量，各自有一份拷贝存在于每个进程中，但互不影响；而同一个进程的多个线程是内存共享的，所有变量都由所有线程共享；
   > - 由于进程间是独立的，因此一个进程的崩溃不会影响到其他进程；而线程是包含在进程之内的，线程的崩溃就会引发进程的崩溃，继而导致同一进程内的其他线程也奔溃；

   ​

## **协程：**

1. 协程是由[线程来管理](https://stackoverflow.com/questions/15287101/is-a-greenthread-equal-to-a-real-thread?noredirect=1&lq=1)
2. 协程包含在线程中，线程包含在进程中
3. 协程的切换操作完全是 user 层面的，协程通过 yield 方式来让其他协程运行
4. 一个线程中的多个协程是不能 **并行（parallel）**的。（python 通常使用 multiprocess + coroutine 的方式使程序并行）
5. 每个协程有自己的栈，协程对全局变量操作**[不需要加锁](http://wsfdl.com/python/2014/10/08/%E4%B8%BA%E4%BB%80%E4%B9%88%E5%8D%8F%E7%A8%8B%E7%9A%84%E5%85%A8%E5%B1%80%E5%8F%98%E9%87%8F%E6%97%A0%E9%9C%80%E5%8A%A0%E9%94%81.html)**
6. 无需线程上下文切换的开销
7. 无需原子操作锁定及同步的开销


## 代码：



```python
import eventlet
import threading
import multiprocessing
import time

count = 0

def count_10000():
    global count
    for i in xrange(10000):
        # time.sleep(1)
        count += 1

def count_in_threads():
    threads = []
    for i in xrange(5):
        t = threading.Thread(target=count_10000)
        threads.append(t)
        t.start()

    # wait all threads to finish
    for t in threads:
        t.join()

def count_in_coroutines():
    pool = eventlet.GreenPool()
    for i in xrange(5):
        pool.spawn_n(count_10000)

    # wait all coroutines to finish
    pool.waitall()

def count_in_process():
    process = []
    for i in xrange(5):
        p = multiprocessing.Process(target=count_10000)
        process.append(p)
        p.start()

    for p in process:
        p.join()


# 如果使用共享资源，由 os 来控制 context switch，会带来不确定性，产生竞争。
# 从 CPU 的角度来看，count += 1 可以分为三个步骤：
#    读取数据 count （当两个 thread 读取数据一样时，产生竞争。）
#    count 加 1
#    写回数据 count
# count_in_threads()
# 打印 1 - 5000 之间都有可能
# print count

# ----------------------------------------------------------------

# count = 0
# 打印 5000

# 如果使用共享资源，由程序本身来控制 context switch，不会带来不确定性，不会产生竞争。
# 和线程不同，协程由应用程序负责调度，操作系统并不感知。操作系统在切换线程的时机是不确定的，但是应用程序切换
# 协程是有条件的。应用程序只有在以下场景才会切换协程：
#    sleep：如 eventlet.sleep()
#    IO：比如网络 IO，磁盘 IO 等。
# 所以协程在执行 count += 1 时不会被切换，保证了该操作的原子性
# 从 CPU 的角度来看，count += 1 可以分为三个步骤：
#    读取数据 count
#    count 加 1
#    写回数据 count
# count_in_coroutines()
# print count

# ----------------------------------------------------------
# count = 0
# 由 os 来控制 context switch，
# 打印 0
# 因为多进程不能使用 global 来分享资源
# 需要用 multiprocessing.Value 或者 multiprocessing.Queue
# 如果使用共享资源，由 os 来控制 context switch，会带来不确定性，产生竞争。
# In 'Threading' module, threads have shared memory, Threads can manipulate global 
# variables of main thread, instead of multiprocessing module, that runs another 
# subprocess in memory and it does not have shared memory like threading.

# count_in_process()
# print count
```



## 参考

[multiple process share global variable multiprocessing.Value](https://eli.thegreenplace.net/2012/01/04/shared-counter-with-pythons-multiprocessing)

[线程 count 和 协程 count 对比](http://wsfdl.com/python/2014/10/08/%E4%B8%BA%E4%BB%80%E4%B9%88%E5%8D%8F%E7%A8%8B%E7%9A%84%E5%85%A8%E5%B1%80%E5%8F%98%E9%87%8F%E6%97%A0%E9%9C%80%E5%8A%A0%E9%94%81.html)

[multiple processing doc](https://docs.python.org/2/library/multiprocessing.html)

[进程、线程和协程](https://github.com/ethan-funny/explore-python/tree/master/Process-Thread-Coroutine)

[知乎 GIL](https://zhuanlan.zhihu.com/p/20953544)
