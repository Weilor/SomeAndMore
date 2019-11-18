# Python并发学习笔记：从协程到GEVENT（一）

并发，让程序可以同时处理更多的事务，这大概是快速提高程序执行效率最优方案之一。但Python在这方面实现并不像C、Java那样简单。

直觉上，最容易实现并发的两种途径应该是多进程和多线程。这两种方法各有利弊。多进程可以充分利用现在处理器本身多核多进程的特点，提高了CPU的使用效率，而且各子进程间相互隔离，一个处理事务的子进程挂了，并不会连带把其他子进程也挂掉。但其缺点也是不容忽视的。进程开销很大，各进程间信息共享比较麻烦。多线程的情况则反过来了，各线程间共享同一块内存空间，共享信息会很简便，而且线程的实现是在同一进程内，系统开销比多进程小很多。但使用多线程就要考虑加锁的问题，锁的处理会大大增加代码的复杂程度，同时，共享内存资源容易导致整个进程内的线程被一个线程的崩溃而全部挂断。具体到Python，多线程还有一个巨大的GIL全局编译器锁横亘在面前。这种粗粒度的全局锁，事实上导致了Python线程调用是串行的，无论线程间是否在访问同一内存资源，只要一个线程获得了CPU时间，其他线程就会被阻塞。

除了多进程和多线程，Python还提供了一种特性可以实现程序某种程度上的“并发”，协程。协程最简单的例子应该是基于yield的一种实现：

```
def foo():
    print "foo start"
    r = ''
    while True:
        num = yield r
        print num
        r = 'OK'

if __name__ == "__main__":
    f = foo()
    f.next()
    print "begin send"
    for x in range(5):
        r = f.send(x)
        print r    
    f.close()
```

回显输出：

```
>>> 
foo start
begin send
0
OK
1
OK
2
OK
3
OK
4
OK
>>> 
```

这个例子可以看出协程和子程序调用的不同，更像是系统中断，主程序执行的过程中，子程序开始执行，到达yield时代码执行中断，返回到主程序，直到主程序send一个数字给子程序，周而复始。

当然，协程这样子写是不能支持并发的，但如果用协程来替代子线程，似乎有不小的优势。首先协程间的切换本身不涉及线程切换，所有的代码是运行在一个线程里的，也就不存在同步锁的问题；其次协程间的切换完全由用户决定，可以避免子线程长时间挂在一个地方阻塞其他子线程执行的问题。把协程和多进程结合起来，实现并发，提高多核CPU使用效率，这大概是个不错的选择。

协程的实现除了使用yield，还有一个非常著名的库，（可不是库吗，Python最大的优点就是各种各样现成的库了。）greenlet。

greenlet主要实现了控制Python栈切换的功能，这部分内容比较艰深，得慢慢啃，现在先不说了。但应用起来还是比较简明的，举个栗子：

```
Python 2.7.6 (default, Jun 22 2015, 17:58:13) 
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from greenlet import greenlet
>>> def foo1():
...   print "foo1.1"
...   gr2.switch()
...   print "foo1.2"
... 
>>> def foo2():
...   print "foo2.1"
...   gr1.switch()
...   print "foo2.2"
... 
>>> gr1 = greenlet(foo1)
>>> gr2 = greenlet(foo2)
>>> gr1.switch()
foo1.1
foo2.1
foo1.2
>>> 
```

总结一下的话，协程并不是真正意义上的并发，在我看来是一种手动触发的代码段转移执行，用来规避阻塞操作。这样，当代码运行到一个地方IO阻塞了，那就调用另一个代码段执行别的任务去，等到这边阻塞结束了，再重新回来执行。这种切换完全在于程序员自己的安排。

说回到greenlet的话，它其实提供了大量的接口，但这里就先不提了，因为还有一个对它进行了封装，对Web开发更好用的库，gevent。

gevent是基于greenlet封装的一个网络同步库，里面有好多好东西，效率又高又好用。下面先举几个例子，显示一下它的基本运行过程：

```
Python 2.7.6 (default, Jun 22 2015, 17:58:13) 
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 
>>> 
>>> import gevent
>>> def foo():
...     print "foo is runing"
...     gevent.sleep(0)
...     print "foo is ending"
... 
>>> def bar():
...     print "bar is runing"
...     gevent.sleep(0)
...     print "bar is ending"
... 
>>> gevent.joinall([gevent.spawn(foo), gevent.spawn(bar)])
foo is runing
bar is runing
foo is ending
bar is ending
[<Greenlet at 0x7f8100f71c30>, <Greenlet at 0x7f8100f71550>]
```

第一次看到这个例子的时候感觉还是很amazing的。当foo进入阻塞状态的时候，gevent自动调用bar，而foo阻塞结束后又被重新在中断的位置继续向下执行，执行结束后bar继续执行。这样，在不直接进行switch操作的前提下，gevent通过对事件的监控，自动对各协程的执行进行调度。

再补充一个例子，显示一下gevent下代码执行的“异步性”，例子里会有两个函数，一个是我们正常情况下实现的同步调用，另一个是使用了gevent的“异步”调用，看一下它们的回显有什么不同。

```
#!venv/bin/python

import gevent
import random

def task(num):
        gevent.sleep(random.randint(0, 2)*0.01)
        print "task %d is done!" % num

def synchronous():
        for i in xrange(10):
                task(i)

def asynchronous():
        threads = [gevent.spawn(task, i) for i in xrange(10)]
        gevent.joinall(threads)

if __name__ == "__main__":
        synchronous()
        print "=" * 20
        asynchronous()
```

执行后的结果是：

```
task 0 is done!
task 1 is done!
task 2 is done!
task 3 is done!
task 4 is done!
task 5 is done!
task 6 is done!
task 7 is done!
task 8 is done!
task 9 is done!
====================
task 1 is done!
task 4 is done!
task 7 is done!
task 8 is done!
task 9 is done!
task 0 is done!
task 2 is done!
task 6 is done!
task 3 is done!
task 5 is done!
```

虽然知道，内部实现里是把各个协程分别调用，但从实际效果上看已经有了异步的影子。

这里还有两个问题需要指出，一个是当gevent出错的时候，比如不能正常yield，一个是当协程执行超时的时候。

出错可以在主程序里对gevent的错误信号进行处理：

```
gevent.signal(signal.SIGQUIT, gevent.shutdown)
```

这样，当错误信号发生的时候，就会调用gevent的shutdown方法关闭gevent进程。

超时也是一个大问题，毕竟协程还是在一个线程里执行的，无论怎么“闪展腾挪”。这里gevent提供了一个Timeout定时器类来处理。用法如下：

```
import gevent
from gevent import Timeout

seconds = 10

timeout = Timeout(seconds)
timeout.start()

def wait():
    gevent.sleep(10)

try:
    gevent.spawn(wait).join()
except Timeout:
    print('Could not complete')
```

注意一点，这个定时器不会被重置，上面的代码里如果把wait方法里休眠的时间改为5，并执行两遍，定时器还是会超时的。

在协程间传递消息的话，gevent提供了一个数据结构，event，它可以异步的在各协程间传递，而且可以进行一对多的对话。举个栗子（今天举了好多栗子啊 ==！）：

```
Python 2.7.6 (default, Jun 22 2015, 17:58:13) 
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import gevent
>>> from gevent.event import Event
>>> evt = Event()
>>> 
>>> def server():
...     print "i am preparing data."
...     gevent.sleep(3)
...     evt.set()
... 
>>> def client():
...     print "client"
...     evt.wait()
...     print "get data"
... 
>>> gevent.joinall([gevent.spawn(server), gevent.spawn(client), gevent.spawn(client)])
i am preparing data.
client
client
get data
get data
[<Greenlet at 0x7f84b46caeb0>, <Greenlet at 0x7f84b46ca370>, <Greenlet at 0x7f84b46ca550>]
>>> 
```

还可以在event里传递参数，

```
Python 2.7.6 (default, Jun 22 2015, 17:58:13) 
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import gevent
>>> from gevent.event import AsyncResult
>>> a = AsyncResult()
>>> 
>>> def server():
...     print "i am preparing data."
...     gevent.sleep(2)
...     a.set("Hello,world.")
... 
>>> def client():
...     print "client"
...     print a.get()
... 
>>> gevent.joinall([gevent.spawn(server), gevent.spawn(client)])
i am preparing data.
client
Hello,world.
[<Greenlet at 0x7f04d3250c30>, <Greenlet at 0x7f04d3250d70>]
>>> 
```

除了event，还有一种数据结构可以用来在协程间通信，Queue队列。协程可以通过Queue的成员方法put把数据压入队列，通过get方法取出。注意，这两种方法是阻塞的，如果通过初始化Queue时指定maxsize的方法，规定了Queue实例最大的容量，那么当空余容量为0的时候put操作就会被阻塞，而当队列空的时候，get操作会返回一个Empty异常。参考下面的例子和回显：

> import gevent
> from gevent.queue import Queue, Empty
>
> tasks = Queue(maxsize=3)
>
> def worker(n):
> try:
> while True:
> task = tasks.get(timeout=1) # decrements queue size by 1
> print('Worker %s got task %s' % (n, task))
> gevent.sleep(0)
> except Empty:
> print('Quitting time!')
>
> def boss():
> """
> Boss will wait to hand out work until a individual worker is
> free since the maxsize of the task queue is 3.
> """
>
> for i in xrange(1,10):
> tasks.put(i)
> print('Assigned all work in iteration 1')
>
> for i in xrange(10,20):
> tasks.put(i)
> print('Assigned all work in iteration 2')
>
> gevent.joinall([
> gevent.spawn(boss),
> gevent.spawn(worker, 'steve'),
> gevent.spawn(worker, 'john'),
> gevent.spawn(worker, 'bob'),
> ])

> Worker steve got task 1
> Worker john got task 2
> Worker bob got task 3
> Worker steve got task 4
> Worker bob got task 5
> Worker john got task 6
> Assigned all work in iteration 1
> Worker steve got task 7
> Worker john got task 8
> Worker bob got task 9
> Worker steve got task 10
> Worker bob got task 11
> Worker john got task 12
> Worker steve got task 13
> Worker john got task 14
> Worker bob got task 15
> Worker steve got task 16
> Worker bob got task 17
> Worker john got task 18
> Assigned all work in iteration 2
> Worker steve got task 19
> Quitting time!
> Quitting time!

当然，get和put还有不阻塞的方式，get_nowait和put_nowait，这两个方法在无法成功执行的时候回分别抛出Empty异常和Full异常。