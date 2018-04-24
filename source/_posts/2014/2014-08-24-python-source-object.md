---
author: chenwei
comments: true
date: 2014-08-24 09:47:09+00:00
layout: post
title: Python源码分析：对象
categories:
- Python
---

首先要知道Python中的所有数据类型都是以对象来实现的，然而这个PyObject相当于是一个最顶层的抽象对象，它用来维护和操作Python中所有类型对象共同的信息和功能部分
描述对象的代码在object.h和object.c中可以看到~

```cpp
typedef struct _object {
    PyObject_HEAD
} PyObject;
```


如这段代码中看到的，定义了一个对象的结构体
实际上并不会去以PyObject来声明一个对象(相当于是面向对象语言中的抽象类型）
但是任何对象的指针都可以转换成PyObject *，这相当与是用C语言手纯手写出来的继承关系

Python内部所有对象的实现的源码都位于Objects目录下：object.c, intobject.c, listobject.c, typeobject.c 等等
看Objects目录下面的代码会发现：
每个对象的结构体里面都含有PyObject_HEAD这个宏
实际上这个宏展开后是

```cpp
/* PyObject_HEAD defines the initial segment of every PyObject. */
#define PyObject_HEAD                   \
    _PyObject_HEAD_EXTRA                \
    Py_ssize_t ob_refcnt;               \
    struct _typeobject *ob_type;
```

其中ob_refcnt记录了对象的引用计数信息(Python的使用了引用计数算法来实现垃圾收集）

ob_type是一个_typeobject结构体(PyTypeObject，属于内部的特殊类型对象)，其中保存了对象的类型名称、需要分配的内存大小，对象的操作方法表(tp_methods)，以及一系列关于该对象的操作行为的函数指针（比如dealloc, print, compare等等)
那么Python在创建对象时就会用到这个全局变量了，把这个全局变量指针赋值给ob_type这个成员，至此一个Object就标记上了相应的‘身份’了，至于具体的每个对象如何去创建和删除，就可以去阅读相应类型的源码了(Objects目录下）

例如listobject.c中会创建一个PyList_Type全局变量(类似的在其他类型对象的.c文件中也会创建相应的全局变量)

```cpp
PyTypeObject PyList_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "list",
    sizeof(PyListObject),
    0,
    (destructor)list_dealloc,                   /* tp_dealloc */
    (printfunc)list_print,                      /* tp_print */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_compare */
    (reprfunc)list_repr,                        /* tp_repr */
    0,                                          /* tp_as_number */
    &list_as_sequence,                          /* tp_as_sequence */
    &list_as_mapping,                           /* tp_as_mapping */
    (hashfunc)PyObject_HashNotImplemented,      /* tp_hash */
    0,                                          /* tp_call */
    0,                                          /* tp_str */
    PyObject_GenericGetAttr,                    /* tp_getattro */
    0,                                          /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC |
    Py_TPFLAGS_BASETYPE | Py_TPFLAGS_LIST_SUBCLASS,         /* tp_flags */
    list_doc,                                   /* tp_doc */
    (traverseproc)list_traverse,                /* tp_traverse */
    (inquiry)list_clear,                        /* tp_clear */
    list_richcompare,                           /* tp_richcompare */
    0,                                          /* tp_weaklistoffset */
    list_iter,                                  /* tp_iter */
    0,                                          /* tp_iternext */
    list_methods,                               /* tp_methods */
    0,                                          /* tp_members */
    0,                                          /* tp_getset */
    0,                                          /* tp_base */
    0,                                          /* tp_dict */
    0,                                          /* tp_descr_get */
    0,                                          /* tp_descr_set */
    0,                                          /* tp_dictoffset */
    (initproc)list_init,                        /* tp_init */
    PyType_GenericAlloc,                        /* tp_alloc */
    PyType_GenericNew,                          /* tp_new */
    PyObject_GC_Del,                            /* tp_free */
};
```


通过PyObject*指针访问这个成员不仅可以访问到对象的类型，并可以通过它访问ob_type之后对对象进行具体的操作，而具体操作的实现则由每个对象类型具体的代码中去做；实际上就相当于面向对象中抽象类和每个子类的关系，但是在C语言里面用了指针去实现的话阅读起来就显得复杂一些。

比如析构函数的例子：

```
void
_Py_Dealloc(PyObject *op)
{
    destructor dealloc = Py_TYPE(op)->tp_dealloc;
    _Py_ForgetReference(op);
    (*dealloc)(op);
}
```

在这边就是使用 Py_TYPE(op)->tp_dealloc来获取具体的析构函数指针（Py_TYPE宏是用来访问成员ob_type的)
​
变长对象：
```cpp
typedef struct {
    PyObject_VAR_HEAD
} PyVarObject;
```


跟上面PyObject不同的是不同的变长对象占用的内存大小是不一样的，比如不同的list对象里元素个数是不一样的
```cpp
#define PyObject_VAR_HEAD               \
    PyObject_HEAD                       \
    Py_ssize_t ob_size; /* Number of items in variable part */
#define Py_INVALID_SIZE (Py_ssize_t)-1
```

PyObject_VAR_HEAD这个宏里面就是在PyObject_HEAD的基础上加上ob_size来描述变长对象的长度。

对比intobject和listobject的代码发现：
定义PyIntObject对象使用的是PyObject_HEAD这个宏，说明它是定长的（这边要注意区分普通整型数int和Python中的大整数的区别，大整数是基于变长对象来实现的），定义PyListObject对象使用的是PyObject_VAR_HEAD这个宏，说明它是变长的
