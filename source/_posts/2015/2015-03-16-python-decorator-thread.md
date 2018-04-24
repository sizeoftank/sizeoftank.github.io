---
date: 2015-03-16 00:30:00
layout: post
title: 使用装饰器包装多线程程序
categories:
- Python
---
Python中的装饰器的实质上是一个函数，它以其他函数作为参数来传递

在使用装饰器语法来定义一个函数的时候

```python
@Decorator
def function()
    ...
```

实际上这个装饰器会对function函数进行一次调用

拆开来就是这样子了

```python
def function()
    ...
Decorator(function)
```

那么它有什么作用呢，现在看一个在多线程使用的例子

这里主要是利用装饰器对线程进行初始化，通过消息队列来传递参数

那么我用到这个装饰器来定义一个函数时，那么它就是由线程去执行的，我就不用关心线程的初始化动作了

```python
queue = Queue.Queue(10)
@Thread(queue, 2)
def testThread(a, b):
    print 'a+b=%s'%(a+b)
```

上面的代码我定义了一个测试线程，首先我创建了一个消息队列，然后描述线程的行为

在这里Thread(queue, 2)的意义是线程通过queue来获取参数，程序会创建2个线程去执行下面的代码

```python
def Thread(Q, threads = 1, interval = 1):
    def decorator(callfunc):
        def process():
            while True:
                kwargs = Q.get()
                print 'Thread %s Run...' % (thread.get_ident())
                json = callfunc(**kwargs)
                time.sleep(interval)
        for i in range(0, threads):
            thread.start_new_thread(process, ())
    return decorator
```

现在看装饰器的代码

有三个参数Q，thread，interval分别意味着消息队列，线程数，执行间隔

这些变量可以简单地用来描述一段代码令其多线程执行

1.这边返回值是一个名为decorator的函数，也就是所谓的装饰器(他的参数是被装饰的那个函数callfunc)

2.然后还需对其处理一下，定义一个process的嵌套函数，这是一个无穷循环(也就是线程里执行的函数)

3.先通过Q从消息队列里抓取参数，如果有参数就调用callfunc去执行(在这里应该加入一个线程的终止条件的，我已经省略了), 在这里线程程序就完成了一次函数调用了，然后用sleep将自己挂起一定时间，当然也可以设置为0

4.在装饰器的最后，需要对线程进行创建，创建后消息队列是空的，那么每个线程就自动将自己设置成挂起状态了

```python
@Thread(queue, 2)
def testThread(a, b):
    print 'a+b=%s'%(a+b)

def producer():
    for i in range(0, 10):
        queue.put({'a': i, 'b': i*2})
    time.sleep(10)

if __name__ == '__main__':
    producer()
```

回到主程序的代码，现在已经定义了testThread是一段由两个线程去执行它的代码

后面定义了一个生产者程序，往消息队列里放东西，这样在生产者运行后，消费者线程也就从消息队列中获取到参数去处理了




