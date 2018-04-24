---
author: chenwei
comments: true
date: 2014-06-29 12:30:38+00:00
layout: post
title: Python解析HTML网页
categories:
- Python
---

最近在写个爬虫程序，用到用到了标准库中的HTMLParser模块。

**一、示例程序：**

其实这个框架用起来非常简单：

在handle_starttag方法中编写代码对开始标记进行处理 <a href="http://localhost/public_html">

在handle_endtag中编写代码对结束标记进行处理 </a>

在handle_data中编写代码对中间的数据进行处理：


```python
from HTMLParser import HTMLParser
class DemoParser(HTMLParser):
    def reset(self):
    	HTMLParser.reset(self)

    def handle_starttag(self, tag, attrs):
    	print tag, attrs

    def handle_endtag(self, tag):
    	print tag

    def handle_data(self, data):
    	print data

def test():
    html = '''
        <a href="http://localhost/public_html"> Test </a>
        '''
    p = DemoParser()
    p.feed(html)

if __name__ == '__main__':
    test()
```

** 二、将页面解析成标签树：**

这里说一下如何借助HTMLParser这个模块将网页解析成标签树的形式，实现的思路比较简单，就是创建一个节点类（HTMLNode)，记录标签(TAG)、属性(ATTRS)，数据(DATA)这些信息，更进一步地可以利用动态添加属性的机制，直接从ATTRS创建每个属性和相应的值

在节点类(HTMLNode)之中将树本身的相关的信息(孩子链表、深度、父节点等信息)定义成私有成员不允许外部直接访问，再实现相应的对树操作的方法，这样完成了对这个数据结构的封装。

```python
class HTMLNode:
    def __init__(self, tag , attrs, data=None, root=None, depth=0):
    	# define children, depth, root as private.
    	self.__children = []
    	self.__depth = depth
    	# top node of a tree, set root refer to itself.
    	# otherwise, set root refer to it's parent.
    	if root == None:
    		self.__root = self
    	else:
    		self.__root = root
    	# define tag, attrs, data as public.
    	self.tag  = tag
    	self.attrs = attrs
    	self.data = data

    def lchild(self, i):
    	# index children list from left to right.
    	if len(self.__children) > 0:
    		return self.__children[i]

    def rchild(self, i):
    	# index children list from right to left.
    	if len(self.__children) > 0:
    		n = len(self.__children)
    		return self.__children[n-i-1]

    def add(self, node):
    	# add a node to children list.
    	node.__root = self
    	node.__depth = self.__depth + 1
    	self.__children.append(node)

    def parent(self):
    	return self.__root

    def num(self):
    	return len(self.__children)

    def depth(self):
    	return self.__depth

    def trvs(self, visit_in_func, visit_out_func=None):
    	visit_in_func(self)
    	for child in self.__children:
    		child.trvs(visit_in_func, visit_out_func)
    	if visit_out_func != None:
    		visit_out_func(self)
    	return
```


其中 trvs()方法递归去访问节点和节点的子树，这边传入了两个函数，一个在进入时调用，一个在退出前调用，对HTML进行处理的时候具体去实现这两个函数对这颗标签树进行访问。

写个一个简单的测试代码，用于以增加缩进量的形式格式化输出HTML代码便于阅读（不过这边没有对内容进行转码，转换后的HTML文件再打开就不对了，只能用来阅读它的结构）

```python
# visit_in and visit_out function use to test.
def visit_in(n):
    if n.tag == None: return
    if n!=None:
    	print '%s<%s '%('    '*(n.depth()-1),n.tag),
    	for i in range(0, len(n.attrs)):
    		if i == len(n.attrs)-1:
    			print '%s=\"%s\">'%(n.attrs[i][0],n.attrs[i][1]),
    		else:
    			print '%s=\"%s\" '%(n.attrs[i][0],n.attrs[i][1]),
    	if n.data != None:
    		if n.tag == 'script':
    			repr_code_lines = n.data.split('\n')
    			for code_line in repr_code_lines:
    				print '\n%s%s'%('    '*(n.depth()), code_line),
    			print
    		else:
    			while n.data.find('  ')!=-1:
    				n.data = n.data.replace('  ',' ')
    			n.data = n.data.replace('\r\n', ' ')
    			n.data = n.data.replace('\n', ' ')
    			n.data = n.data.replace(' ', '')
    			n.data = n.data.replace('\t\t','\t')
    			n.data = n.data.replace('\t','')
    			if n.data != '':
    				print '\n%s%s'%('    '*(n.depth()),n.data)
    	if n.num() != 0:
    		if n.data == None:
    			print
    		elif n.data == '':
    			print

def visit_out(n):
    if n.tag == None: return
    if n.num() != 0 or n.data != None:
    	print '%s</%s>'%('    '*(n.depth()-1), n.tag)
    else:
    	print '</%s>'%(n.tag)

```


下面来看如何建立这棵树，算法比较简单，先创建一个根节点，然后定义一个pre对象保持始终引用当前节点的对象，如果遇到新的开始标记，就将该标记加入到孩子链表中，然后pre对象引用的是孩子链表中的最右边节点；如果遇到了结束标记，就将pre对象返回上层的父节点

编码的时候发现两个问题，一个是有一些标签可以不书写结束标记的，那么就需要特殊处理，否则会出错，我的解决方法时将这些标签定义成terminated(终结的)如果遇到这些标签时直接跳出返回上层节点；另一个是发现有些标签不匹配，这边是出现了多出了结束标记的情况，我的解决方法是用了个辅助堆栈进行检查，在处理结束标记时，先检查栈顶元素，如果栈顶元素作为开始标记不和当前结束标记匹配，就丢弃这个结束标记。

```python
class HTMLTreeParser(HTMLParser):
    terminated = ( 'meta','link','br', 'img', 'input', 'frame', 'col', 
                    'hr', 'base','option', 'area','param')

    def reset(self):
    	HTMLParser.reset(self)
    	self.tree = HTMLNode(None,None)
    	self.pre = self.tree
    	self.stack = []
    	return

    '''
    def inc(self,key):
    	for i in range(0, len(self.count)):
    		if self.count[i][0] == key:
    			self.count[i][1] = self.count[i][1] + 1
    			return
    	self.count.append([key, 1])

    def dec(self, key):
    	for i in range(0, len(self.count)):
    		if self.count[i][0] == key:
    			self.count[i][1] = self.count[i][1] - 1
    			return
    	self.count.append([key, 0])
    '''

    def handle_starttag(self, tag, attrs):
    	self.pre.add(HTMLNode(tag,attrs))
    	self.pre = self.pre.rchild(0)
    	# some terminated tags have not endtag </xx>.
    	if tag in HTMLTreeParser.terminated:
    		self.pre = self.pre.parent()
    	else:
    		self.stack.append(tag)
    	return

    def handle_endtag(self, tag):
    	if tag in HTMLTreeParser.terminated:
    		return
    	else:
    		# use stack to check whether endtag match to starttag.
    		if len(self.stack) == 0 or self.stack[len(self.stack)-1] != tag:
    			return
    		else:
    			self.stack.pop()
    	self.pre = self.pre.parent()
    	return

    def handle_data(self, data):
    	self.pre.data = data
    	return

    def parse_html(self, html):
    	self.feed(html)
    	return self.tree

```


测试的时候编写代码打开有道词典的页面：
```python
def test():
    params = {'keyfrom':'dict.index','q': 'test'}
    url = 'http://dict.youdao.com/search?'+urllib.urlencode(params)
    opener = urllib2.build_opener()
    opener.addheaders = [('User-Agent',
        'Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:31/0) Gecko/20100101 Firefox/31.0')]
    f = opener.open(url)
    html=f.read()
    hp=HTMLTreeParser()
    T=hp.parse_html(html)
    T.trvs(visit_in, visit_out)
```
