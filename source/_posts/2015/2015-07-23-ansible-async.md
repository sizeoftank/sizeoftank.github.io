---
date: 2015-07-23 22:00:00
layout: post
title: Ansible执行异步操作
---
Ansible是支持异步执行命令的，记一下通过Ansible异步执行命令的方法

### 使用方法
```python
import ansible.runner
import time
import os

target_host = '127.0.0.1'

runner = ansible.runner.Runner(
	module_name='command',
	host_list='ansible/hosts',
	module_args='sleep 3',
	forks=1,
	run_hosts=[target_host],
)
p = runner.run_async(30)

for i in range(0, 10):
	result=p[1].poll()
	time.sleep(0.5)
	print result

```

例子中只连接了我自己的机器127.0.0.1，使用run_async()可以进行异步调用，然后通过poll()来获取命令执行的情况，
这边只用了一个sleep 3的例子来演示，通过输出还是可以看到结果的变化的(前面的几次poll()该命令未结束，后面的几次
poll()可以看出该命令成功退出了)

另外Ansible对于整个script方式的调用还是不支持异步方式的(其他模块也没全部尝试)，后面再研究下有没有方法来实现
整个脚本的异步调用方式，虽然如此，不过command的异步调用在操作机器数量较多的情况下也还是很有用武之地的。