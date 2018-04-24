---
date: 2014-12-21 03:20:00
layout: post
title: Python多线程编程学习之基础篇
categories:
- Python
---
主要介绍Python使用threading模块进行多线程编程的例子，以及一个简单的线程池的实现思路。

具体可以参考该模块的[官方文档](https://docs.python.org/2/library/threading.html#threading.Condition)

##线程的创建和使用、锁、条件变量##

**线程创建**

Python的threading模块提供了两种创建线程的方法

方法一是直接使用threading.Thread()来创建，在这种创建方式中，创建Thread()对象时需要指定该线程准备执行的函数，然后用
\*args与\*\*kwargs来传递参数，其中\*args传递一个tuple类型的参数，\*\*kwargs传递字典类型的参数表
```python
def thread_print_args(*args, **kwargs):
    for i in range(0, 10):
        print args, kwargs
        time.sleep(0.1)

def thread_create_test():
    threading.Thread(None, thread_print_args, args = ('a1','a2'), kwargs = {'name': 'test'}).start()
    threading.Thread(None, thread_print_args, args = ('b1','b2'), kwargs = {'name': 'hello'}).start()
```

方法二是通过自定义线程类继承threading.Thread类来创建，需要重载run()方法，在run()方法中设置线程要跑的函数就行了
```python
class ThreadPrint(threading.Thread):
    def run(self):
        for i in range(0, 10):
            print self.name
            time.sleep(0.1)

def thread_create_test():
    ThreadPrint().start()
    ThreadPrint().start()
```

可以参看threading.py文件中的源码，其实run()方法就是线程调度运行时要执行的函数，默认的Thread类里也是
通过self.\_\_target来调用相应的函数，只是在构造对象时创建了这个属性

另外，不论使用哪种方式创建了线程，线程在run()方法执行完毕之后自动退出


**使用互斥锁**

互斥锁的用途是用于控制线程间共有资源的访问，实现原子操作，也不多介绍了

在Python中互斥锁被封装在threading.Lock这个类中，具有请求和释放两个方法

这边的例子是创建2个线程使用互斥锁对全局的count进行累加的操作，两个线程有不同的访问间隔:
```python
mutex = threading.Lock()
count = 0
class ThreadLockTest(threading.Thread):
    def __init__(self, interval):
        threading.Thread.__init__(self)
        self.interval = interval
    def run(self):
        global count
        while count < 100:
            print "%s acquires..." % (self.name)
            mutex.acquire()
            print "%s enter..." % (self.name)
            count = count + 1
            print '%s : %d' %(self.name, count)
            time.sleep(self.interval)
            mutex.release()
            print "%s exit..." % (self.name)


def thread_lock_test():
    ThreadLockTest(0.1).start()
    ThreadLockTest(0.3).start()
```
该例子也只是简要地说明锁的使用方式，在实际的多线程编程中使用锁时，还得注意死锁的避免，一个简单的原则就是：保证线程都按相同的次序来持有锁变量


**使用条件变量**

条件变量是用于线程的同步的，简单地说就是去操作线程'挂起等待-唤醒'这一过程

Python中条件变量封装在threading.Condition这个类中，wait()方法用于阻塞等待，notify()方法用于唤醒，需要注意到的是对于条件变量的访问是原子的，因而条件变量也含有了acquire(),release()这两个方法，在使用wait()，notify()之前就需要先请求锁

```python
condition = threading.Condition()
class ThreadCondTest(threading.Thread):
    def run(self):
        print "%s runs.." %(self.name)
        condition.acquire()
        print "%s wait.." %(self.name)
        condition.wait()
        condition.release()
        for i in range(0, 5):
            print "Hi, I'm runs now"

def thread_cond_test():
    ThreadCondTest().start()
    time.sleep(3)
    condition.acquire()
    condition.notify()
    condition.release()
```
在这个例子中，线程创建启动后马上将自己阻塞等待，直到主程序time.sleep(3)之后将其唤醒后继续执行后面的代码

##一个简单线程池的实现思路##

实际的多线程编程中，为了避免大量的创建和销毁线程，就需要建立一个线程池来辅助管理和回收利用这些线程来达到程序优化的目的；然而在Python中，相比与Linux下用C/C++来实现一个线程池的话，编程的复杂度就简单得多了；通常一个完整的线程池由线程池需要有线程队列和工作队列两个部分组成，这边简要介绍一个线程队列的实现思路作抛砖引玉用，当然如果要设计成通用的模块和API的话还是有大量的工作要做的

Python中的Queue模块已经提供了一个多线程环境下的队列应用(内部用互斥锁已经实现了原子访问)，可以直接拿起来用，就不必考虑设计队列时的互斥和同步问题了

```python
MAX_THREADS = 4

class TestThread(threading.Thread):
    def __init__(self, poll, cond, num):
        threading.Thread.__init__(self)
        self.poll = poll
        self.task = None
        self.task_args = None
        self.cond = cond
        self.threadID = num

    def run(self):
        while True:
            print 'Thread: %s wait now...'%(self.getName())
            self.cond.acquire()
            self.poll.threads.put(self)
            self.cond.wait()
            self.cond.release()
            print 'Thread: %s run now...'%(self.getName())

            if self.task != None:
                print 'Call Function : %s(%s)' %(self.task, self.task_args)
                self.task(self.task_args)
                del self.task, self.task_args


class ThreadPoll():
    def __init__(self, n):
        self.threads = Queue.Queue(n)
        self.maxsize = n
        self.__init__threads()

    def __init__threads(self):
        for i in range(0, self.maxsize):
            thr = TestThread(poll = self, cond = threading.Condition(), num = i)
            thr.start()

    def dispatch(self, func, arg):
        thr = self.threads.get()
        thr.task = func
        thr.task_args = arg
        thr.cond.acquire()
        thr.cond.notify()
        thr.cond.release()
        return thr.threadID

def thread_print(s):
    for i in range(0, 10):
        print 'Thread: %s' % (s)
        time.sleep(0.1)

def thread_poll_test():
    threadpoll = ThreadPoll(MAX_THREADS)
    threadpoll.dispatch(thread_print, 'Test-1')
    threadpoll.dispatch(thread_print, 'Test-2')
    threadpoll.dispatch(thread_print, 'Test-3')
    threadpoll.dispatch(thread_print, 'Test-4')

    threadpoll.dispatch(thread_print, 'Test-1')
    threadpoll.dispatch(thread_print, 'Test-2')
    threadpoll.dispatch(thread_print, 'Test-3')
    threadpoll.dispatch(thread_print, 'Test-4')
```

TestThread类是线程池使用的线程类，其中task属性用于设置线程将要完成的任务，默认为空，使用时通过线程池的dispatch()来自动分派，
run()函数里是一个无穷循环，初始化时首先将自己阻塞挂起，之后放回到线程池中，统一由线程池进行管理。

ThreadPoll类是线程池的管理器，提供dispatch()来分派将要进行的任务给正在等待的线程，初始化时先创建N个线程，它们都统一阻塞自己放回到线程池中，如果有一个新的任务要做时，就通过dispatch()去分派，先是从队列中取出一个线程，给他重新设置task属性和参数，之后将其唤醒，那么该线程就会去执行这个工作，待它执行完之后又回到主循环，将自己放回线程池和进入等待状态。

进一步的工作还需要考虑工作队列模块，例如当一个新的task来的时候，如果线程池中没有空闲的线程池，那么就得先将该任务放进工作队列缓冲区中，而线程池中一旦有空闲线程，就得从工作队列中拉出一个task来处理




