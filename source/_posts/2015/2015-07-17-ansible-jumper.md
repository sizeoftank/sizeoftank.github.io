---
date: 2015-07-17 17:30:00
layout: post
title: Ansible通过跳板机管理服务器
---
记录一下使用ansible通过跳板机来管理机器的方式(也就是ansible放在一个非跳板机的情况下，通过跳板机来管理所有机器的方法）

首先确保下面两个条件

1. 确保跳板机能够直接通过ssh访问到需要被管理的机器
2. 确保ansible这台机器能够访问跳板机

假设现在跳板机IP为 192.168.0.100

需要通过它管理 192.168.0.* 网内的机器

### SSH代理

编写一个ssh_config文件 (默认路径是在~/.ssh目录下，根据自己的环境选择存放位置）

```python

Host jumper      #跳板机的访问规则
    User root
    HostName 192.168.0.100
    IdentityFile ~/.ssh/id_rsa #访问跳板机的密钥
    PasswordAuthentication no

Host 192.168.0.*  #其他机器的访问规则
    User root
    ForwardAgent yes
    ProxyCommand ssh root@192.168.0.100 -p 22 nc %h %p  #这边连接跳板机做代理
    IdentityFile ~/.ssh/jumper_id_rsa  #对应跳板机的密钥位置(需要将跳板机的密钥拷贝到这台机器上)    

```

配置OK了做两个测试：

1. ssh jumper -F ssh_config 测试能否登上跳板机
2. ssh 192.168.0.120 -F ssh_config 测试能否登上其他机器

### Ansible 配置

上面已经将ssh配置好了，这里给ansible做配置(ansible.cfg)

ansible配置文件查找优先级是这样的：
   
   当前目录的ansible.cfg > ~/.ansible.cfg > /etc/ansible/ansible.cfg
   
根据特定的环境选择放哪里吧

```python
[default]
transport = ssh

[ssh_connection]
ssh_args = -F ssh_config #ssh_config文件路径
scp_if_ssh = True

```

这么配置好就可以用ansible通过跳板机直接管理其他机器了

transport这个配置一定要设置成ssh, ansible默认是在原生ssh和paramiko之间进行自动选择的(通常会使用paramiko进行连接机器)

而paramiko里没有支持ProxyCommand做代理这一项，因此无法穿过跳板机直接访问别的机器，这一点本人郁闷了很久，最后查看ansible源码才发现必须要在这里transport其配成ssh。