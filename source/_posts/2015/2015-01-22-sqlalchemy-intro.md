---
date: 2015-01-22 10:20:00
layout: post
title: SQLAlchemy入门
categories:
- Python
---

## 什么是SQLAlchemy ##
通俗的说法就是一个ORM框架

详细一点，就是它实现了一个从Python代码中的类与SQL数据库表结构之间的映射，也包括了对象和SQL数据库行内容的一个映射关系

通过在代码里直接操作这些对象，也比编写SQL去执行友好多了

安装的话直接使用pip安装就好了 pip install sqlalchemy

下面直接通过一个例子介绍如何使用它来操作数据库

## 连接数据库 ##

```python
from sqlalchemy import create_engine

engine = create_engine('mysql+mysqldb://username:password@localhost:3306/test?charset=utf8', echo=True)
```

这边是一个连接本地MySQL数据库的例子(参考形式： mysql+mysqldb://用户名:密码@IP地址:端口号/数据库名)

## 建立数据对象 ##

```python
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String(128))
    fullname = Column(String(128))
    password = Column(String(128))

    def __repr__(self):
        return "<User(name='%s', fullname='%s', password='%s')>" %(
            self.name, self.fullname, self.password)

```

在SQLAlchemy中使用对象映射是通过继承Base这个类来实现的，Base这个类通过调用declarative_base()来创建

在这个例子中表结构:  id(主键), name, fullname, password

在User类里创建相应的这几个属性，各个属性的值用Column来创建，参数描述了SQL数据库中的数据类型（比如Integer, String, 还有primary_key, autoincrement, unique等等)

\_\_tablename\_\_ 对应于SQL数据库的表名称

实现repr()方法，便于调试程序时清楚的看到结果

通过定义了User类之后，后续就可以对数据库进行操作了(包括创建表这一过程)

## 创建表 ##
```python
Base.metadata.create_all(engine)
```
通过上面的语句可以创建表，打开数据库就可以查看到与User类对应的表就已经被创建好了

如果定义了多个类，也会根据这些创建多个表

根据这个思路，在具体的项目中就可以直接编写好代码然后生成表了，相比与操作数据库，自己在代码里实现相应的数据对象还是非常方便的

## 创建会话 ##
```python
SessionMK = sessionmaker(bind=engine)
session = SessionMK()
```

sessionmaker这个函数将根据传入的engine创建一个会话的生成器(返回的是一个类)

具体的会话根据上面的会话生成器来创建实例化的会话对象

(这边用到的是抽象工厂模式吧?)

## 操作数据库 ##
```python
def session():
    SessionMK = sessionmaker(bind=engine)

    # Add new objects
    session = SessionMK()
    hello = User(name='hello', fullname='hello world', password='123')
    session.add(hello)

    # Query ...
    users = session.query(User).filter_by(name='hello')
    for user in users:
        print user
        user.password = 'xxx'

    # Dirty ..
    print session.dirty

    # Commit
    session.commit()
```

这边做的几件事情:
1. 新建一个hello条目插入表中
2. 新建一条查询，并修改hello的密码
3. 查看脏数据(可以看到被修改了哪些东西)
4. 执行commit提交修改

## 回滚 ##
```python
def rollback():
    session = SessionMK()

    user = session.query(User).filter_by(name='hello').first()
    user.name = 'test'

    another = User(name='another', fullname='xxx', password='456')
    session.add(another)

    users = session.query(User).filter(User.name.in_(['test', 'another']))
    for user in users:
        print user

    session.rollback()
    print ' --- rollback --- '

    users = session.query(User).filter(User.name.in_(['hello', 'test', 'another']))
    for user in users:
        print user
```
如果变更了一些东西后需要恢复到修改前，就需要执行回滚了(比如会用在错误处理上面)

上面的代码做了几件事情，首先修改hello的名字为test, 然后新加入一个叫another的用户，这时候创建查询找到这两个用户；之后进行回滚，之后用hello,test,another这三个名字去查询，这时候的结果就是名为hello的用户(数据库回滚到原来的状态)

