---
date: 2020-10-21 22:00:00
title: CentOS中XGBoost安装失败问题
categories:
- 个人笔记
---

执行安装命令后报出各种编译错误
```
pip install xgboost
```

查看官方文档要求如下

1. A recent C++ compiler supporting C++11 (g++-5.0 or higher)
2. CMake 3.13 or higher.


通常使用 yum groupinstall 'Development Tools' 安装的gcc为<5.0的版本, 导致代码编译不了, 升级方式如下

```
sudo yum install centos-release-scl
sudo yum install devtoolset-7-gcc*
scl enable devtoolset-7 bash
which gcc
gcc --version
```

安装 cmake 版本 >= 3.13 

```
sudo yum install epel-release
sudo yum install cmake3
```


先替换 bash 中的 gcc 版本再执行 pip install
```
scl enable devtoolset-7 bash
source .../venv/bin/activate
pip install xgboost
```