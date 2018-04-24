---
date: 2015-01-31 0:20:00
layout: post
title: Python中的自省和反射
categories:
- Python
---
## WHAT ##
按照本人的理解，自省和反射机制是一个相似的概念，就拿反射来说，可能我在编程中遇到了这么一种情况：我(操作者)不知道需要操作的对象的数据结构和方法是怎么样的，但是我可以通过字符串的形式来令某个对象去调用某个方法，这就是一个简单的反射例子。 那么自省呢，我认为它是编程语言中的一个对象，它事先可以告诉我(或者说我可以通过某个方法了解到)它的结构是怎么样的，有哪些操作方法，然后我再去调用它，这就对编程者提供了一种很灵活的控制能力，也或者说它为实现反射提供了一种支持… 

## Python中的自省 ##
灵活的自省和反射机制也是Python的一个重要的优点之一，编程者可以通过一些内建函数和方法实现对Python虚拟机层面的一些操作，因为默认情况下我们是通过写好的代码来对对象进行操作的，而对于虚拟机来说，代码就是被编译解释的字符串，然而Python虚拟机提供了可以直接通过字符串来操作对象的接口，因而就具备了良好的自省和反射。

### 相关的内建函数 ###
```python
class Student:
    def __init__(self, ID, name, age):
        self.ID = ID
        self.name = name
        self.age = age

    def say(self):
        print 'I am a Student'
```
这边定义了一个Student类，后面来对它进行自省和反射的操作

```python
>>> ss = Student(0,'Li Lei',20)
>>> dir(Student)
['__doc__', '__init__', '__module__', 'say']
>>> dir(ss)
['ID', '__doc__', '__init__', '__module__', 'age', 'name', 'say']

>>> type(ss)
<type 'instance'>
>>> type(Student)
<type 'classobj'>

>>> isinstance(ss,Student)
True

```
这边创建了一个Student的实例，然后我两次分别通过dir()对类和实例进行查看，从输出结果可以看到我收集到了关于这个类和第一个实例的部分属性名，通过type()内建函数也可以看到它们一个是instance类型,一个是classobj类型，通过isinstance()也可以判断一个对象是否是某个类创建的实例

从上面的dir()看到的属性也只是一部分，还有些是隐藏的，比如\_\_class\_\_和\_\_dict\_\_

```python
>>> ss.__class__
<class student.Student at 0x7f55ef61b328>
>>> type(ss.__class__)
<type 'classobj'>
>>> ss.__dict__
{'age': 20, 'ID': 0, 'name': 'Li Lei'}
>>> ss.__class__.__dict__
{'__module__': 'student', 'say': <function say at 0x7f55ef625de8>, '__init__': <function __init__ at 0x7f55ef625d70>, '__doc__': None}
```
现在可以看出，\_\_class\_\_属性就是该对象的类（classobj)

实例(instance)和类(classobj)的属性都是通过\_\_dict\_\_这个字典来维护的

实例（instance)的\_\_dict\_\_里维护了学生的信息，也就是我在\_\_init\_\_()里定义的东西，这些东西在创建实例的时候被加到字典里

类(classobj)的\_\_dict\_\_维护了相关的方法，\_\_init\_\_和我自己定义的say()

这也就说明了对象方法的一个调用顺序

比如我执行ss.say()的时候，是从类(classobj)里找到这个方法，然后解释器再把自己通过self传入去执行

当然，我也有更奇葩的一种写法：
```python
>>> ss.__class__.__dict__['say']()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: say() takes exactly 1 argument (0 given)
```
这样就报错了，因为我没有将具体的实例(instance)作为参数传入
```python
>>> ss.__class__.__dict__['say'](ss)
I am a Student
```
现在将ss作为实例传进去，那么就执行成功了，这边的'say'我是通过字符串传入的，也就是完成动态调用的反射了

然后我要给Student加一个自杀的操作:
```python
>>> def selfkill(s):
...     print 'I going to kill myself, and my name is %s'%(s.name)
... 
>>> Student.__dict__['selfkill']=selfkill
>>> 
>>> ss.selfkill()
I going to kill myself, and my name is Li Lei
```

这边我新建了一个函数，打印一行信息，并且取了name这个属性，把它放到Student的字典里

然后我用ss.selfkill()就可以调用它了，这样就完成了动态添加属性的反射了

但是没有人会在编程时这么做(如果我的同事这样写代码，我会打死他的)

通用的做法是这样的:
```python
>>> hasattr(ss, 'say')
True
>>> getattr(ss, 'say')()
I am a Student
>>> hasattr(ss, 'sing')
False

>>> def wrapper(s):
...     def sing():
...         print 'My name is %s'%(s.name)
...     return sing
... 
>>> setattr(ss, 'sing', wrapper(ss))
>>> ss.sing()
My name is Li Lei
>>> hh = Student(2,"Han MeiMei",20)
>>> hasattr(hh,'sing')
False

```
1.通过hasattr(x, attr)函数检查某个对象x是否有某个属性attr

2.通过getattr(x, attr)来读某个x对象的属性attr，如果是函数方法，就得加上括号和参数了getattr(x,attr)()

3.在上面的代码中通过包装器wrapper(s)来创建sing()这个方法，包装器在这边的作用就相当与是帮助传入self参数

4.后面创建的另一个学生韩梅梅(hh)就没有sing()这个方法，因为刚刚的sing()是动态为李雷(ss)这个实例添加的，当然也可以为Student这个类加入一个属性，那么每个新创建的实例都能调用sing()这个方法了，这里也不介绍了

## 其他 ##
需要区分的几个概念，因为Python虚拟机需要维护一个用来记录用户自定义类(class)的classobject, 因而需要理清从虚拟机到用户代码的这四个层次关系:

用户类(class)的类型: classobject \[虚拟机(typeobject)\]

用户类(class)的类的实例: 一个由classobject创建的实例, Student类 \[虚拟机(typeobject)\]

用户类: Student \[用户代码\]

用户类的实例: 由Student创建的实例 \[用户代码\]

这部分内容可以阅读Python相关源代码的具体实现，看懂部分后就对这个机制有个很明了的理解啦



