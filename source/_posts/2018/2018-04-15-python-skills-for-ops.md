---
date: 2018-04-15 22:00:00
title: 写给运维的Python使用技巧
---

#### 前言
Python这门语言应用广泛，不论是大数据，AI，Web开发，运维等领域对Python的应用都有不同的要求，本次分享主要针对Python在运维脚本中的应用介绍一些实战技巧
#### PEP8 编码规范
PEP8 (Python 代码风格指南)，是通过编码风格上的一些要求，用于提高团队中的代码质量，通过规范约束加上静态代码检查(Flake8等)，也可避免一些比较低级的坑。

#### 代码编排
主要是终结 tab 跟空格的争论，PEP8要求全部使用空格，在编辑器中设置 tab 自动替换空格即可 (由于解析器的原因，tab跟空格混用可能会出现解释失败的情况)
其他主要影响代码美观性，如空格，空行，注释等等，可参考阅读官方写的示例就好:
https://www.python.org/dev/peps/pep-0008/

#### 变量命名
主要关注一下变量，方法等的命名方式，如下：
```python
# 常量大写
USER_CONSTANT = 'xxx'


# 类名: 首字母大小
class UserDetail(object):
    def __init__(self):
        # 变量名称, 小写加下划线
        self.user_name = 'user'
        # 私有属性: 双下划线前缀
        self.__encrpyped_password = ''

    # 函数/方法: 小写加下划线
    def query_roles(self):
        pass

    # 私有方法: 单下划线前缀
    def _encrypt_password(self):
        pass
```


#### 代码检查
Flake8 提供了 PEP8 编码规范及代码静态检查等，可集成到 Sublime Text 等工具中(Python Flake8 Lint 插件)，或者直接在命令行执行 flake8 xxx.py
静态检查的好处主要是检查，能够在执行代码前发现一些低级的错误，例如使用未定义方法，未定义的变量等
```python
example.py:35:5: F821 undefined name 'undefined_method'
example.py:36:9: F821 undefined name 'undefined_var'
```

#### 字符串处理
字符串可能是脚本中是最常用到的对象，指导思想就是不要自己写处理逻辑，尽量使用内建方法，正则表达式，模板格式化等方式

```python
# 使用占位符格式化
'ps -ef|grep %s' % 'java'
 
# 使用字典进行格式化
'ps -ef|grep %(keyword)s' % {'keyword': 'java'}
 
# 使用 format 方法格式化
'ps -ef|grep {keyword}'.format(keyword='java')
 
# 对于复杂一些的内容可以通过 Jinja2 来完成，例如生成一段 YML
template = jinja2.Template('\
- hosts: {{project.name}}-{{name}}\n\
  tasks:\n\
    - name: copy captain.jar\n\
      copy:\n\
        src: /home/ci/captain/captain.jar\n\
        dest: {{captain.path}}/\n\
    - name: restart captain\n\
      shell: bash {{captain.path}}/restart-captain.sh\n\
')
 
yml = template.render(model)
```
对于 Jinja2 的使用当然是强烈推荐的，在运维工作中掌握 Jinja2 的使用对于运维自动化的推进非常有帮助的


#### 序列解析及生成表达式

这一部分也是 Python 中处理序列数据时非常强大的一些功能，可以简洁优美地完成一些序列数据的变换，也就是通常所说的 Pythonic

```python
# 定义一个生成命令的函数
In [7]: def ps_cmd(x):
   ...:     return 'ps -ef|grep %s' % (x)
   ...:

In [8]: datas = ['lc-auth', 'lc-cmdb']

# 对所有元素执行方法并生成列表
In [9]: [ps_cmd(x) for x in datas]
Out[9]: ['ps -ef|grep lc-auth', 'ps -ef|grep lc-cmdb']

# 使用 map 方法
In [10]: map(ps_cmd,datas)
Out[10]: ['ps -ef|grep lc-auth', 'ps -ef|grep lc-cmdb']

# 对于这类序列的处理，还需要掌握 map, filter, reduce 这三个函数

# 也可以使用 lambda 表达式来处理
In [11]: map(lambda x: 'ps -ef|grep %s' % x, datas)
Out[11]: ['ps -ef|grep lc-auth', 'ps -ef|grep lc-cmdb']

# 包含 tuple 的列表处理, 将 tuple 转成字典存储结构
In [17]: [{'username': x, 'password': y} for x, y in userlist]
Out[17]:
[{'password': '000000', 'username': 'user'},
 {'password': '123456', 'username': 'admin'},
 {'password': '123456', 'username': 'ops'}]

# 转换成以用户名作为键值的字典
In [18]: userdatas = [{'username': x, 'password': y} for x, y in userlist]
In [20]: {item['username']:item for item in userdatas}
Out[20]:
{'admin': {'password': '123456', 'username': 'admin'},
 'ops': {'password': '123456', 'username': 'ops'},
 'user': {'password': '000000', 'username': 'user'}}

# 生成器，可以看到上面的例子中外面都是以 [] 来表达的，这样会直接生成一个列表，但如果改为 () 的话可以创建生成器
In [26]: (ps_cmd(x) for x in datas)
Out[26]: <generator object <genexpr> at 0x7f0ef1339be0>

In [27]: gen = (ps_cmd(x) for x in datas)

In [28]: gen
Out[28]: <generator object <genexpr> at 0x7f0ef1339c80>

In [29]: gen.next()
Out[29]: 'ps -ef|grep lc-auth'

# 生成器也可以通过 for 语句或者 map, filter, reduce 等方法来遍历
# 生成器与列表的区别主要是列表在创建时需要分配内存，而生成器不需要，用来减少内存开销，这一特性通过 yield 关键字来实现
# 上面创建生成器的语句 (ps_cmd(x) for x in datas) 等价于

In [31]: def gen(datas):
   ....:     for x in datas:
   ....:         yield ps_cmd(x)
   ....:
```

#### 错误处理

使用 traceback 追踪错误, 例如打印调用栈
```python
import traceback

try:
    do something
except Exception as ex:
    print ex
    traceback.print_exc()
```

了解标准库中异常的层级结构，知道异常的一些分类方法，合理地进行错误处理使程序更健壮
https://docs.python.org/2.7/library/exceptions.html#exception-hierarchy
简化列举一下
```python
Exception
  StandardError
  	IOError (I/O 包括文件系统，网络等)
  	OSError (系统相关)
  	LookupError (访问数组/字典)
  	TypeError (方法接收了不合法的类型等)
  	ValueError (编解码相关)
```

特殊的场景中也需要自定义异常，需要注意在自定义异常时也需要合理地定义异常的层级及父异常

#### 静态方法和类方法

静态方法和类方法的使用，静态方法的定义通过 staticmethod 装饰器，类方法通过 classmethod 装饰器

常见的一种用法是定义一个工具类, 将一系列的方法归类，和直接使用函数的差别主要是在这些方法有了一个类的命名空间，管理起来不那么混乱
```python
class AESUtils(object):
    @staticmethod
    def encrypt(text):
        pass

    @staticmethod
    def decrypt(text):
        pass
```

另外一种场景就是用于设置和该类相关的一些全局配置，环境变量等操作
类方法的使用和静态方法基本类似，区别在于接受的第一个参数是类的class对象

```python
class Query(object):
    DEFAULT_LIMIT = 100

    def __init__(self, offset, limit, keyword):
        self.offset = 0
        self.limit = 0
        self.keyword = keyword

    @classmethod
    def limit(cls, limit):
        # 改变类的全局选项
        cls.DEFAULT_LIMIT = limit

    @classmethod
    def from_keyword(cls, keyword):
        # 通过 cls 参数来创建一个查询对象
        return cls(0, DEFAULT_LIMIT, keyword)

query = Query.from_keyword('test')

```


#### 使用抽象类及接口
使用 abc 库定义抽象类，可以在Python实例化类的时候检查类是否实现了某些方法，这样可以在一定程度上避免运行时调用接口导致 AttributeError 异常
```python
class AbstractProcess(object):
    __metaclass__ = abc.ABCMeta
    
    @abc.abstractmethod
    def start(self):
        'start a process'
        return NotImplemented
 
    @abc.abstractmethod
    def stop(self):
        'stop a process'
        return NotImplemented


class SpringbootProcess(AbstractProcess):
    def __init__(self, name):
        self.name = name

# 这样就对 AbstractProcess 的子类进行了约束, 没有实现 start, stop 方法的类不能被实例化
Traceback (most recent call last):
  File "example.py", line 97, in <module>
    proc = SpringbootProcess('xxx')
TypeError: Can't instantiate abstract class SpringbootProcess with abstract methods start, stop
```

#### Mixin模式
Python中的类可以多重继承，但在面向对象原则上在继承的使用上必须保持单一继承关系，也就是一个类只能继承一个父类。
类比其他编程语言中的类可以实现多个接口，Python中的多重继承可以通过Mixin模式为类添加多种功能。
```python
# 序列化并且通过Redis读取
class RedisMixin:
    def redis_key(self):
        raise NotImplemented

    @classmethod
    def from_redis(cls, key):
        r = redis.Redis(host='192.168.100.101', port=6379)
        data = r.get(key)
        return pickle.loads(data)

    def to_redis(self):
        r = redis.Redis(host='192.168.100.101', port=6379)
        data = pickle.dumps(self)
        r.set(self.redis_key, data)


# 创建用户类, 并提供序列化到Redis的功能
class User(RedisMixin):
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def redis_key(self):
        return 'username:%s' % (self.name)

user = User('user', 'user@xiaopeng.com')
print(user.to_redis())

user = User.from_redis('username:user')
print(user.name, user.email)
```

#### 装饰器
装饰器的应用主要在一些开发框架中，例如 Web 框架中的路由
```python
class Application(object):
    def __init__(self):
        self.router_mapper = {}

    # 这里定义装饰器, 将URI注册到对象中
    def route(self, path):
        def decorator(callfunc):
            self.router_mapper[path] = callfunc
            return callfunc
        
        return decorator

    def dispatch(self, uri, kwargs={}):
        return self.router_mapper[uri](**kwargs)

if __name__ == '__main__':
    app = Application()

    @app.route('/hello')
    def hello():
        return 'hello'

    @app.route('/echo')
    def echo(message):
        return 'echo %s' % (message)

    print(app.dispatch('/hello'))
    print(app.dispatch('/echo', {'message': 'test'}))
```

大家也可以拓宽思路, 将装饰器其使用在脚本里或者开发一些公共的运维模块
