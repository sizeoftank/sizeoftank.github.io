---
author: chenwei
comments: true
date: 2014-05-08 03:07:09+00:00
layout: post
title: Sublime Text 插件开发
categories:
- Others
---

官方API文档： [http://www.sublimetext.com/docs/api-reference](http://www.sublimetext.com/docs/api-reference)


**1.目录结构(Linux为例)：**

~/.config/sublime-text-3/Packages/ 下

为插件创建一个文件夹，比如Extend

Tool->New Plugin新建源码，保存到刚刚的目录下


**2.示例代码：**
```python
import sublime, sublime_plugin
class HelloWorld(sublime_plugin.TextCommand):
    def run(self, edit):
        print("Hello, World!")
```

保存之后可以通过view.run_command("hello_world")在控制台看到程序打印的结果了

需要注意定义Class的时候需要以陀峰形式命名，即单词首字母大写

然后对应的执行命令是全小写加下划线的形式

**3.添加一个快捷命令： **

新建Default.sublime-commands文件保存到插件目录

```json
[
    {"caption" : "HelloWorld", "command" : "hello_world"}
]
```

然后通过Ctrl-Shift-P里面输入HelloWorld就可以运行刚刚的插件

**4.加上菜单：**

新建Main.sublime-menu文件保存到插件目录

```json
[
    {
        "caption" : "Extend-Demo" , "id" : "extend_demo",
        "children":
        [
            {"caption" : "HelloWorld", "command" : "hello_world"}
        ]
    }
]
```

这样子才菜单栏上会多出一个Extend-Demo出来

然后在子菜单里有个HelloWorld可以执行刚刚的插件

完毕.
