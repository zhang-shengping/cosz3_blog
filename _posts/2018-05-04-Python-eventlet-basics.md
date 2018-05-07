---
title: Eventlet Notes
categories:
- 使用总结记录
tags:
- eventlet
- non-blocking
- coroutine
---

![useful image]({{ site.url }}/cosz3_blog/assets/images/blog_pictures/blog-hawk-hill-view.jpg)

## Eventlet 基本使用：

Eventlet is built around the concept of green threads (i.e. coroutines, we use the terms interchangeably) that are launched to do network-related work. Green threads differ from normal threads in two main ways:

 <u>Green threads are so cheap they are nearly free</u>. You do not have to conserve green threads like you would normal threads. In general, there will be at least one green thread per network connection.
 <u>Green threads cooperatively yield to each other instead of preemptively being scheduled. The major advantage from this behavior is that shared data structures don’t need locks, because only if a yield is explicitly called can another green thread have access to the data structure</u>.It is also possible to inspect primitives such as queues to see if they have any pending data.

![](http://eventlet.net/doc/_images/threading_illustration.png)

## Greenthread Spawn（孵化 Greenthread）

### spawn

```python
eventlet.spawn(func, *args, **kw)
```

spawn 方法孵化出 （重复调用孵化多个）greenthread 去执行 func 方法。返回值是一个 [`greenthread.GreenThread`](http://eventlet.net/doc/modules/greenthread.html#eventlet.greenthread.GreenThread) object ， 可以从 [`greenthread.GreenThread`](http://eventlet.net/doc/modules/greenthread.html#eventlet.greenthread.GreenThread) object 中获取 func 的返回值。

### spawn_n

```python
eventlet.spawn_n(func, *args, **kw)
```

spawn_n 和 spawn 相似，但是这个方法得不到 func 的返回值或者异常。spawn 多个 greenthread 运行速度比较快。

### spawn_after

```python
eventlet.spawn_after(seconds, func, *args, **kw)
```

spawn_after 是 spawn 带有 delay 的版本，spawn_after 返回值是 [`GreenThread`](http://eventlet.net/doc/modules/greenthread.html#eventlet.greenthread.GreenThread) object，通过 [`GreenThread`](http://eventlet.net/doc/modules/greenthread.html#eventlet.greenthread.GreenThread) object 可以得到 func 的返回值和异常错误。调用  [`GreenThread.cancel()`](http://eventlet.net/doc/modules/greenthread.html#eventlet.greenthread.GreenThread.cancel) 可以停止相应的协程。

## 控制 Greenthread

### sleep

```python
eventlet.sleep(seconds=0)
```

睡眠暂定当前 greenthread，使得其他 greenthread 有机会被调用。

## GreenPool

```python
class eventlet.GreenPool
```

Pools 主要用来控制 spawn Greenthread 的个数，从而可以控制内存使用量。

```python
# client pattern
import eventlet
from eventlet.green import urllib2

urls = ["http://www.google.com/intl/en_ALL/images/logo.gif",
       "https://www.python.org/static/img/python-logo.png",
       "http://us.i1.yimg.com/us.yimg.com/i/ww/beta/y3.gif"]

def fetch(url):
    return urllib2.urlopen(url).read()

pool = eventlet.GreenPool()
for body in pool.imap(fetch, urls):
    print("got body", len(body))

# server pattern
import eventlet

def handle(client):
    while True:
        c = client.recv(1)
        if not c: break
        client.sendall(c)

server = eventlet.listen(('0.0.0.0', 6000))
# pool size 是 10000
pool = eventlet.GreenPool(10000)
while True:
    new_sock, address = server.accept()
    pool.spawn_n(handle, new_sock)
```

### 使用测试

在使用 Pool 时，可以用 pool.waitall() 方法等待其他所有的 greenthread 完成后再推出当前程序。尤其是当每个 greenthread 有屏幕打印时，如果主 greenthread 完成太早退出，就看不到屏幕打印记录。

```python
#!/usr/bin/env python
# encoding: utf-8

import eventlet
eventlet.monkey_patch()

from oslo_concurrency import lockutils

workers=4
jobs = 5
record = {}
pool = eventlet.greenpool.GreenPool()

synchronized = lockutils.synchronized_with_prefix('foo')

# @synchronized('bar')
def do_work(index):
    global record
    record[index] = record.get(index, 0) + 1
    print "worker=%s: %s" % (index, record[index])
    eventlet.greenthread.sleep(1)

def work(worker, jobs):
    for x in xrange(0, jobs):
        print "worker: %s jobs %s" %(worker, (x + 1))
        do_work(worker)
        # if sleep:
        #     eventlet.greenthread.sleep(0) # yield
            #return index

for i in xrange(workers):
    print 'worker: %s do %s jobs'% (i, jobs)
    pool.spawn(work, i, jobs)

# 这里使用 waitall
pool.waitall()
```

### GreenPile

```python
class eventlet.GreenPile
```

GreenPile 可以封装一个 GreenPool 来指定控制 GreenThread 数量。运行多个 Greenthread 后，可以用迭代 GreenPile 来得到所有 Greenthread 返回值。

```python
import eventlet
feedparser = eventlet.import_patched('feedparser')

pool = eventlet.GreenPool()

def fetch_title(url):
    d = feedparser.parse(url)
    return d.feed.get('title', '')

def app(environ, start_response):
    pile = eventlet.GreenPile(pool)
    for url in environ['wsgi.input'].readlines():
        pile.spawn(fetch_title, url)
    titles = '\n'.join(pile)
    start_response('200 OK', [('Content-type', 'text/plain')])
    return [titles]
```

## Patching Functions

首先需要知道 monkey patch 有什么作用： monkey patch 给其他的 python 模块打补丁，主要是为了防止堵塞。因为 eventlet 的 context swtich 都是由程序员自己控制（eventlet.sleep, IO 等），所以在产生堵塞时不会有系统来进行上下文切换，这时需要对一些 python 模块打补丁，进行自动切换。

### import_patched

```python
eventlet.import_patched(modulename, *additional_modules, **kw_additional_modules)
```

modulename 指定给某个 module 打 patch，从而使 module 实现 nonblocking。

### monkey_patch

```python
eventlet.monkey_patch(all=True, os=False, select=False, socket=False, thread=False, time=False)
```

给 system module 打 patch。`all` True 给所有 module 打 patch， `all` False 对映的其他`参数`为 True 时，给对映的包打 patch。

### **Eventlet 使用来解决 IO 堵塞的，但是有些 python 模块的 IO 堵塞并不能直接被绿化，这时提供了以下两种方式来解决这种情况：**

#### monkey_patch 使用测试

对 monkey_patch 的使用这里举个小例子。

首先对于 blocking 堵塞，个人理解可以分为两种类型：

1. 对所有协程堵塞：比如 time.sleep 会堵塞所有的协程。
   1. 针对当前协程堵塞（堵塞时，别的协程可以正常使用）：比如 eventlet.greenthread.sleep 不会堵塞所有协程。

那么如何将堵塞所有协程的 python module 修改成只堵塞当前协程的程序呢？，这里会就会使用到 monkey_patch。比较一下两个代码和输出。

不带 monkey_patch，运行到 time.sleep(10）后会停止等待，下一行输出。

```python
#!/usr/bin/env python
# encoding: utf-8

import eventlet
# 注释掉 monkey patch
# eventlet.monkey_patch()
import time

workers=4
jobs = 5
record = {}
pool = eventlet.greenpool.GreenPool()

def do_work(index):
    global record
    record[index] = record.get(index, 0) + 1
    data = "worker=%s: %s \n" % (index, record[index])
    print data

    # 堵塞
    time.sleep(10)

def work(worker, jobs):
    for x in xrange(0, jobs):
        # print "worker: %s jobs %s" %(worker, (x + 1))
        do_work(worker)

for i in xrange(workers):
    print 'worker: %s do %s jobs'% (i, jobs)
    pool.spawn(work, i, jobs)
pool.waitall()
```

加上 monkey_patch 后，运行到 time.sleep(10) 后，程序会立刻打印输出别的协程 data 变量值。

```python
#!/usr/bin/env python
# encoding: utf-8

import eventlet
# 加上 monkey_patch
eventlet.monkey_patch()
import time

workers=4
jobs = 5
record = {}
pool = eventlet.greenpool.GreenPool()

def do_work(index):
    global record
    record[index] = record.get(index, 0) + 1
    data = "worker=%s: %s \n" % (index, record[index])
    print data

    # 堵塞
    time.sleep(10)

def work(worker, jobs):
    for x in xrange(0, jobs):
        # print "worker: %s jobs %s" %(worker, (x + 1))
        do_work(worker)

for i in xrange(workers):
    print 'worker: %s do %s jobs'% (i, jobs)
    pool.spawn(work, i, jobs)
pool.waitall()
```

**monkeypatch 同时也改变 thread.get_ident() 输出，多个 coroutine 分别在不同的 ”thread“ 中？**

```python
#!/usr/bin/env python
# encoding: utf-8

import eventlet
# 加上 monkey_patch
eventlet.monkey_patch()
import time
import thread

workers=4
jobs = 5
record = {}
pool = eventlet.greenpool.GreenPool()

def do_work(index):
    global record
    record[index] = record.get(index, 0) + 1
    data = "worker=%s: %s \n" % (index, record[index])
    print data

    # 使用 monkeypatch 不堵塞
    time.sleep(10)

def work(worker, jobs):
    for x in xrange(0, jobs):
        # 打出 thread 号,这里会输出不同的 thread 号。
        print "worker in thread %s" % thread.get_ident()
        do_work(worker)

for i in xrange(workers):
    print 'worker: %s do %s jobs'% (i, jobs)
    pool.spawn(work, i, jobs)
pool.waitall()
```

**堵塞，多个 coroutine 分别在同一个的 thread 中**

```python
#!/usr/bin/env python
# encoding: utf-8

import eventlet
# 注释 monkey_patch
# eventlet.monkey_patch()
import time
import thread

workers=4
jobs = 5
record = {}
pool = eventlet.greenpool.GreenPool()

def do_work(index):
    global record
    record[index] = record.get(index, 0) + 1
    data = "worker=%s: %s \n" % (index, record[index])
    print data

    # 堵塞
    time.sleep(10)

def work(worker, jobs):
    for x in xrange(0, jobs):
        # 打出 thread 号,这里会输出相同的 thread 号。
        print "worker in thread %s" % thread.get_ident()
        do_work(worker)

for i in xrange(workers):
    print 'worker: %s do %s jobs'% (i, jobs)
    pool.spawn(work, i, jobs)
pool.waitall()
```

**不堵塞，多个 coroutine 分别在同一个的 thread 中**

```python
#!/usr/bin/env python
# encoding: utf-8

import eventlet
# 注释 monkey_patch
# eventlet.monkey_patch()
import time
import thread

workers=4
jobs = 5
record = {}
pool = eventlet.greenpool.GreenPool()

def do_work(index):
    global record
    record[index] = record.get(index, 0) + 1
    data = "worker=%s: %s \n" % (index, record[index])
    print data

    # 堵塞
    # time.sleep(10)
    eventlet.greenthread.sleep(1)

def work(worker, jobs):
    for x in xrange(0, jobs):
        # 打出 thread 号,这里会输出相同的 thread 号。
        print "worker in thread %s" % thread.get_ident()
        do_work(worker)

for i in xrange(workers):
    print 'worker: %s do %s jobs'% (i, jobs)
    pool.spawn(work, i, jobs)
pool.waitall()
```

根绝上面现象，产生了一个问题，如下：

问题 https://stackoverflow.com/questions/50152428/does-python-eventlet-monkeypatch-start-up-multi-threading-with-different-thread

这个问题暂时也没人回答，但是我看到了github 上一个相反的[问题](https://github.com/eventlet/eventlet/issues/172)，个人推测：monkeypatch 给多个模块打了 patch，这里也包含 thread 模块。在使用 thread 的时，多个协程的属性和行为可能被看成了不同的 ”threading“（表现为线程），但是他们其实还是协程。

为了验证，可以不对 thread 打 patch 来测试：

```python
#!/usr/bin/env python
# encoding: utf-8

import eventlet
# 只对 time 打 patch
eventlet.monkey_patch(all=False,time=True)
import time
import thread
import threading

workers=4
jobs = 5
record = {}
pool = eventlet.greenpool.GreenPool()

def do_work(index):
    global record
    record[index] = record.get(index, 0) + 1
    # data = "worker=%s: %s \n" % (index, record[index])
    # print data
    # eventlet.greenthread.sleep(1)

    # block
    time.sleep(5)

def work(worker, jobs):
    for x in xrange(0, jobs):
        # print same thread id
        print "thread.get_ident %s" % thread.get_ident()
        print "threading.current_thread %s" % threading.current_thread()
        do_work(worker)

for i in xrange(workers):
    print 'worker: %s do %s jobs'% (i, jobs)
    pool.spawn(work, i, jobs)
pool.waitall()

# -------------------------------------------------------------------------------
# 输出
[root@172-18-211-195 shengping]# python test_mky.py
worker: 0 do 5 jobs
worker: 1 do 5 jobs
worker: 2 do 5 jobs
worker: 3 do 5 jobs
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
thread.get_ident 139717144807232
threading.current_thread <_MainThread(MainThread, started 139717144807232)>
```



#### 使用 monkeypatch 无效怎么办（可以使用tpool）

在有些情况下 python 模块使用 C library 的 OS calls 做一些 IO socket 操作，这时使用 monkeypatch 也阻止不了  blocking。这里就需要是用 [eventlet.tpool](http://eventlet.net/doc/threading.html), tpool 会启动一个新的 thread （不是协程）来运行。

```python
>>> import thread
>>> from eventlet import tpool
>>> def my_func(starting_ident):
...     print("running in new thread:", starting_ident != thread.get_ident())
...
>>> tpool.execute(my_func, thread.get_ident())
running in new thread: True
```

[stackoverflow 上关于 tpool 的用处](https://stackoverflow.com/questions/43947405/how-is-eventlet-tpool-useful)

> The only thing I can guess is that when myfunc() is called directly, it not only blocks this coroutine but also prevents other coroutines from running, while calling tpool.execute() will block the current coroutine but somehow yields so that other coroutines can run. 



> Armed with that powerful terminology, *tpool.execute() turns blocking call into yielding one*.
>
> Combined with `eventlet.spawn(tpool.execute, fun, ...)` it would not block even the caller coroutine. Maybe you find this a helpful combination.

比如一次性大量的读写文件操作，就算是打了 monkeypatch 还是会有 [block 代码](https://stackoverflow.com/questions/17015403/python-eventlet-file-asyncnon-blocking-io?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)， [这时可以考虑使用 tpool](https://stackoverflow.com/questions/17015403/python-eventlet-file-asyncnon-blocking-io?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)。

## 参考

[eventlet design pattern](http://eventlet.net/doc/design_patterns.html#design-patterns)

[eventlet API](http://eventlet.net/doc/basic_usage.html)

[eventlet patch 的使用](http://eventlet.net/doc/patching.html)

[how to use tpool](https://stackoverflow.com/questions/43947405/how-is-eventlet-tpool-useful)

[tpool doc](http://eventlet.net/doc/threading.html)

[tpool 的一些 examples](https://programtalk.com/python-examples/eventlet.tpool.execute/)

[python eventlet - file asynchronousc(non-blocking) io](ihttps://stackoverflow.com/questions/17015403/python-eventlet-file-asyncnon-blocking-io?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)
