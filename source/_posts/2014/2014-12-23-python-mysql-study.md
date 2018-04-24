---
date: 2014-12-23 12:20:00
layout: post
title: Python操作MySQL数据库
categories:
- Python
---

项目地址: http://sourceforge.net/projects/mysql-python/

项目文档: http://mysql-python.sourceforge.net/MySQLdb.html#functions-and-attributes

简单的案例(获取MySQL版本)：
```python
import MySQLdb

db = MySQLdb.connect('localhost','username','123','xxx')
cursor = db.cursor()
cursor.execute("SELECT VERSION()")
data = cursor.fetchone()
print "DataBase version : %s" % (data)
db.close()
```

SQL操作：

在连接数据库成功后就可有使用cursor来执行SQL语句
```python
cursor.execute("select column_xxx,column_xxx from table_name where xxx=xxx“)
```
像这种查询类语句，后面还得加上一条fetch操作
```python
result = cursor.fetchall()
```
这一步返回一个查询结果的列表，处理数据时遍历这个列表即可

如果使用期间对数据庫有修改，还要加上将本次修改提交
```python
db.commit()
db.close()
```

批量执行:

例如现在有在大量数据需要插入，使用cursor.execute()单条语句执行的效率是非常之低的，
因此就需要使用cursor.executemany()来批量提交

cursor.executemany(query, args)

其中query是需要执行的SQL，args是值列表，其中表中的每个元素是一个包含数据库各个字段的值的tuple

```python
# 生成SQL语句
# 这样: insert into table (name, age, salary) values (%s, %s, %s)
Columns = '''name, age, salary'''
SQL = '''insert into %s'''%(TABLE)+'('+Columns+')'+''' values '''
SQL = SQL + '('
for i in range(0, len(Columns.split(','))):
    SQL = SQL + r'%s' + ','
SQL = SQL.rstrip(',')
SQL = SQL + ')'
```

从文本导入数据的例子
```python
# 处理行数据(每个字段用制表符分隔)
valuelist = []
for line in fileinput.input(filename):
    line = line.rstrip('\n')
    valueitem = []
    # 先将文本数据放入list，再转为tuple
    for item in line.split('\t'):
        valueitem.append('\'%s\''%(item.replace('\'','')))

    if len(valueitem) != len(Columns.split(',')):
        print 'Bad Item:', valueitem
    else:
        # 将单个元素放入待插入的list中
        valuelist.append(tuple(valueitem))

cursor.executemany(SQL, valuelist)
```

上面提供的代码是不完整的，还需设置一个单次批量插入多少数据的数值，然后进行计数，执行语句，重置等操作~
