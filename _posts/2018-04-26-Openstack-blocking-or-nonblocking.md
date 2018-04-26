---
title: Openstack blocking or non-blocking
categories:
- 概念总结记录
tags:
- blocking
- non-blocking
---

在 Openstack 中支持 eventlet 的 wsgi server 是异步非堵塞的网络服务，

wsgi server 非堵塞是因为 wsgi server 在处理请求时如果遇到了 IO wait，eventlet 会调度别的 task 来继续做别的程序处理（比如 监听网络端口，处理其他的 IO 请求）。

wsgi server 异步是因为 wsgi server 在处理 task 的 IO wait 等待其他程序的 buffer（比如 write buffer） 时，调用了 eventlet poll（类似 python 的 [select](https://pymotw.com/2/select/) 函数），eventlet poll 收到了其他程序的 writeable return 后，会再去 call 之前等待的 task 进行 IO 操作。在此过程中其他程序会主动返回状态值，告知 wsgi server 自己现处的状态。

关于跟多细节可以参考 [Async I/O and Python](https://blogs.gnome.org/markmc/2013/06/04/async-io-and-python/)。

select 函数例子

```python
import errno
import select
import socket

sock = socket.socket()
sock.connect(('localhost', 1234))
sock.setblocking(0)

buf = buffer('foo\n' * 10 * 1024 * 1024)
print "starting"
while len(buf):
    try:
        buf = buf[sock.send(buf):]
    except socket.error, e:
        if e.errno != errno.EAGAIN:
            raise e
        print "blocking with", len(buf), "remaining"
        select.select([], [sock], [])
        print "unblocked"
print "finished"
```

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fqp15supltj30t70a7dgr.jpg)

## Openstack 使用协程和进程

Openstack 中使用了 eventlet 来实现 wsgi server 的非堵塞 (比如： nova/wsgi.py)，eventlet 是 python 实现协程的一个模块。那么，为什么 python 使用协程来实现 wsgi server 的非堵塞服务? 这里原因有以下几点：

1. 收到 python 线程 GIL 锁的限制，每个 python 进程只能有一个 GIL 锁，每个线程在运行时都要 acquire GIL 锁，所以同时最多只有一个线程在运行，且线程的上下文切换需要通过操作系统，开销相对协程来说比较大。
2. 多线程共享一个进程的 virtual memory address，当一个线程出现问题时，可能会导致整个进程出现问题
3. 协程虽然和 python 线程一样，在同一时刻只有一个协程在运行，但是协程的上下文切换不需要通过操作系统，时间开销比较小。
4. 协程有自己的栈和局部变量，多个协程可以共享全局变量。协程只有在碰到 IO 等待，eventlet.sleep 时才会被挂起，且不会出现操作一个全局变量的情况下被挂起（线程会出现这种状况），所以一般情况下多个协程对一个全局变量的操作是[不需要加锁的](https://zhang-shengping.github.io/cosz3_blog/%E6%A6%82%E5%BF%B5%E6%80%BB%E7%BB%93%E8%AE%B0%E5%BD%95/2018/04/23/Some-concepts-summary/)（但是我们还是看到 Openstack 中 协程的一些操作还是加了锁，这是为什么？）。

为了解释时间开销上的问题，这里引用网上的一个例子([来源](http://niusmallnan.com/_build/html/_templates/openstack/coroutine_usage.html))：

> 假设有一个操作系统，是单核的，系统上没有其他的程序需要运行，有两个线程 A 和 B ，A 和 B 在单独运行时都需要 10 秒来完成自己的任务，而且任务都是运算操作，A B 之间也没有竞争和共享数据的问题。现在 A B 两个线程并行，操作系统会不停的在 A B 两个线程之间切换，达到一种伪并行的效果，假设切换的频率是每秒一次，切换的成本是 0.1 秒(主要是栈切换)，总共需要 20 + 19 * 0.1 = 21.9 秒。如果使用协程的方式，可以先运行协程 A ，A 结束的时候让位给协程 B ，只发生一次切换，总时间是 20 + 1 * 0.1 = 20.1 秒。如果系统是双核的，而且线程是标准线程，那么 A B 两个线程就可以真并行，总时间只需要 10 秒，而协程的方案仍然需要 20.1 秒。



Openstack 使用了协程，减小了上下文切换的开销，同时也失去了使用多 CPU 的能力。为了弥补这个能力 Openstack 使用了多进程 + 协程的方式。

## 参考

[Openstack 中的协程](http://niusmallnan.com/_build/html/_templates/openstack/coroutine_usage.html)

[openstack中协程分析](https://blog.csdn.net/hzrandd/article/details/9772511)

[Can a multi-core processor run multiple processes at the same time?](https://superuser.com/questions/257406/can-a-multi-core-processor-run-multiple-processes-at-the-same-time)

[Can multi core cpu run multiple processes in parallel?](https://www.quora.com/Can-multi-core-cpu-run-multiple-processes-in-parallel)





