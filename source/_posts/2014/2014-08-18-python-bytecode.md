---
author: chenwei
comments: true
date: 2014-08-18 08:07:15+00:00
layout: post
title: 使用dis模块解析Python字节码
categories:
- Python
---

相应的模块文档看这里 : [https://docs.python.org/3/library/dis.html](https://docs.python.org/3/library/dis.html)

Python源码在编译执行时，为了加快速度，在源代码中import的代码模块将会编译成.pyc文件去运行，这个.pyc文件中也就是Python的字节码，通常在使用Python时都能在代码目录中看到这些文件。

对于命令行执行的时候可以使用 python -m dis xxx.py 来看到相应的字节码

下面看在代码中如何使用dis模块将其他的源码解释成相应的字节码进行分析：

dis.dis()这个方法来的参数可以是一个模块、类、方法或者是一个函数，如果是一段字符串源码要先通过compile()内建函数进行编译

```python
import dis
def myfunc(n):
    s = 0
    for i in range(0, n):
        s = s+i
    return s

#例1: 分析myfunc()函数的字节码
print "Example 1:"
dis.dis(myfunc)

print 'Example 2:'
s = '''
    a = 2
    b = 3
    if a < b:
        print 'true'
    else:
        print 'false'
    '''
#例2: 分析上面代码段的字节码
dis.dis(compile(s,'', mode='exec'))
```

函数：compile(source, filename, mode)

1. source参数可以制定从字符串进行编译

2. filename参数可以制定从文件进行编译，如果不是从文件来的源码可以留空字符串

3. mode参数用来设定编译的类型

编译后可以看到字节码的结果：

```python
Example 1:
    4           0 LOAD_CONST               1 (0)
                3 STORE_FAST               1 (s)

    5           6 SETUP_LOOP              33 (to 42)
                9 LOAD_GLOBAL              0 (range)
                12 LOAD_CONST               1 (0)
                15 LOAD_FAST                0 (n)
                18 CALL_FUNCTION            2
                21 GET_ITER
           >>   22 FOR_ITER                16 (to 41)
                25 STORE_FAST               2 (i)

    6           28 LOAD_FAST                1 (s)
                31 LOAD_FAST                2 (i)
                34 BINARY_ADD
                35 STORE_FAST               1 (s)
                38 JUMP_ABSOLUTE           22
           >>   41 POP_BLOCK

    7     >>    42 LOAD_FAST                1 (s)
                45 RETURN_VALUE
Example 2:
    2           0 LOAD_CONST               0 (2)
                3 STORE_NAME               0 (a)

    3           6 LOAD_CONST               1 (3)
                9 STORE_NAME               1 (b)

    4          12 LOAD_NAME                0 (a)
               15 LOAD_NAME                1 (b)
               18 COMPARE_OP               0 (<)
               21 POP_JUMP_IF_FALSE       32

    5          24 LOAD_CONST               2 ('true')
               27 PRINT_ITEM
               28 PRINT_NEWLINE
               29 JUMP_FORWARD             5 (to 37)

    7     >>   32 LOAD_CONST               3 ('false')
               35 PRINT_ITEM
               36 PRINT_NEWLINE
          >>   37 LOAD_CONST               4 (None)
               40 RETURN_VALUE
```