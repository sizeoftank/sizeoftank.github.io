---
date: 2015-01-14 10:20:00
layout: post
title: 在Flask中进行单元测试小记
categories:
- Python
---

最近在开发Flask的项目，打算利用单元测试来提高效率和质量

Unittest: [https://docs.python.org/2/library/unittest.html](https://docs.python.org/2/library/unittest.html)

Python本身提供了unittest这个模块，完全足够来应对一般的单元测试了

```python
#coding=utf-8
import sys
from flask import Flask
import unittest
import views
import models
from __init__ import make_app

app = make_app('config.cfg')
app.config['TESTING'] = True

class TestXXX(unittest.TestCase):
    
    def setUp(self):
        pass

    def tearDown(self):
        pass

    def test_one(self):
        print '\n +++ test_one +++ \n'

    def test_two(self):
        print '\n +++ test_two +++ \n'

if __name__ =='__main__':
    unittest.main()
```

按照上面的示例代码，大概说下单元测试的编写过程：

STEP1: 编写一个测试类(以Test作为开头)，然后在其内部添加测试方法（以test作为开头)，这样，之后在启动unittest.main()之后，unittest就会自动调用刚刚编写的测试方法进行测试，并返回测试结果

STEP2: import项目中的需要被测试的模块，考虑一些初始化的东西，比如

```python
app = make_app('config.cfg')
app.config['TESTING'] = True
```

在这边我就初始化了一个Flask的测试app，做了一个TESTING的配置项目，这样在项目的其他模块中，就可以根据这来编写测试时需要执行的代码，比如打印一些测试信息等
    
STEP3: setUp() 和 tearDown() 分别是启动和停止单元测试是调用的方法，比如也可以将一些初始化动作放在里面
    
STEP4: 将来需要增加新的测试可以直接在测试类中添加新的方法，如果是不同模块的测试，则最好编写不同的测试类，以便分开测试

运行单元测试(假设测试代码的文件名是test.py)：

可以直接在控制台 python test.py 这样就运行所有单元测试

也可有加载单个测试模块来进行 python -m unittest test.TestXXX

至于如何更有效的利用单元测试这个手段，本文不多介绍，项目中各个单元很好地模块化是必须的
