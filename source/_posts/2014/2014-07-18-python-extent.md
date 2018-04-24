---
author: chenwei
comments: true
date: 2014-07-18 16:52:41+00:00
layout: post
title: 使用C/C++对Python进行扩展
categories:
- Python
---


Python是一门极其灵活和简洁的语言，但其执行效率并不理想，相反C/C++的写法相对来说就复杂多了，但是C/C++却有很好的执行效率。而Python提供了可扩展的底层API给编程人员，可以通过它编程实现功能后，给Python上层调用，这样就将Python的灵活性和C/C++的效率结合起来了~

**一些比较有用的资料**

[http://web.mit.edu/people/amliu/vrut/python/ext/intro.html ](http://web.mit.edu/people/amliu/vrut/python/ext/intro.html%20)

[http://www.ibm.com/developerworks/cn/linux/l-pythc/#icomments ](http://www.ibm.com/developerworks/cn/linux/l-pythc/#icomments%20)

[https://docs.python.org/2/c-api/index.html ](https://docs.python.org/2/c-api/index.html%20)

**C/C++代码如何导出到Python**

1. **加入Python的头文件**
```c
#include <python2.7/Python.h>
```
2. **导出方法表和模块**
```cpp
//定义模块的方法表，这边有两个例子函数
//get_args(): 获取Python中的参数并在终端打印
//isorder(): 判断一个Python的列表是否是升序排列的
static PyMethodDef PyDemoMethods[] = {
    { "get_args", get_args, METH_VARARGS, "None."},
    { "isorder", isorder, METH_VARARGS, "None."},
    { NULL, NULL, 0, NULL }
};

//编译后会生成Python可以import的模块PyDemo
PyMODINIT_FUNC initPyDemo(void)
{
    PyObject *m = Py_InitModule("PyDemo", PyDemoMethods);
    if (m == NULL)
        return;
}
```

**编译和链接**

C/C++ 的源码在Linux编译成.so文件给Python调用

g++ -shared -fpic pydemo.cpp -o PyDemo.so

**在C/C++中获取并打印Python的参数**

其中在C/C++中的函数都需要遵循这个函数原型进行扩展

```cpp
static PyObject* get_args(PyObject *self, PyObject *args)
```

其中PyObject是Python中的对象，Python是一门完全面向对象的语言，它的所有对象在C语言中都是以PyObject结构体来表现的

参数args提供了一个变长参数表给C/C++语言
```cpp
PyArg_ParseTuple(args, "ids", &i, &d, &s)
```

这行代码用来获取Python中的参数，"ids" 用来描述有三个参数，分别是整型(i)、双精度浮点数(d)和字符串(s)

其中C/C++语言的变量作为参数时需要加上取地址符&

对于字符串类型，在C/C++中声明一个指针就行了，不需要对其分配空间，因为在参数解析后其会指向原先运行时数据所在的内存

```cpp
static PyObject* get_args(PyObject *self, PyObject *args)
{
    int i;
    double d;
    char *s;
    if (!PyArg_ParseTuple(args, "ids", &i, &d, &s)){
    	return NULL;
    }
    cout << i << endl;
    cout << setiosflags(ios::fixed) << setprecision(2) << d << endl;
    cout << s << endl;
    Py_INCREF(Py_None);
    return Py_None;
}
```

**判断一个Python列表是否升序**

上面一个例子只是简单的获取参数并打印，现在看下如何对Pyhton中的List对象进行操作

```cpp
static PyObject* isorder(PyObject *self, PyObject *args)
{
    PyObject* pList = NULL;  //声明一个对象指针
    Py_ssize_t i;            //下标i需要用这个结构体进行定义

    if (!PyArg_ParseTuple(args, "O", &pList)){ //Python中的参数作为对象传入("O")
    	return NULL;
    }
    if (!PyList_Check(pList))
    	return NULL;

    //遍历列表比较相邻对象
    for (i = 1; i < PyList_Size(pList); i++){
    	if (PyObject_Compare(PyList_GetItem(pList, i-1), PyList_GetItem(pList, i)) != -1){
    		Py_INCREF(Py_False);
    		return Py_False;
        }
    }

    Py_INCREF(Py_True);
    return Py_True;
}
```

其中上面两个例子返回None和布尔对象时都用了Py_INCREF这个宏~

是Python内部对于这些对象有引用计数的机制，在使用这些对象进行返回时，就需要使用这个宏更新它的计数

**六、在Python中调用:**

```python
import PyDemo
def test():
    print(PyDemo.get_args(1, 4.7, "hello"))
    l = [1,2,0,4]
    print(PyDemo.isorder(l))
```