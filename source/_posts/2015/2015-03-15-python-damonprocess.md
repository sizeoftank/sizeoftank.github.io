---
date: 2015-03-15 11:30:00
layout: post
title: 使用Python创建守护进程
categories:
- Python
---

在使用Python编写服务端程序中需要将进程放在系统后台运行

在Unix/Linux环境中通常的做法

```python
def daemon():
    try:
        if os.fork() > 0:
            os._exit(0)
    except OSError, error:
        print 'fork #1 failed: %d (%s)' % (error.errno, error.strerror)
        os._exit(1)
    os.chdir('/')
    os.setsid()
    os.umask(0)
    try:
        pid = os.fork()
        if pid > 0:
            print 'Daemon PID %d' % pid
            os._exit(0)
    except OSError, error:
        print 'fork #2 failed: %d (%s)' % (error.errno, error.strerror)
        os._exit(1)
```


如果考虑系统中只允许一个这样的进程存在，就可以考虑使用文件锁来实现一个单例进程

```python
lockfile = 'Server.pid'
fp = open(lockfile, 'w')
try:
    fcntl.lockf(fp, fcntl.LOCK_EX | fcntl.LOCK_NB)
except IOError:
    print 'Another instance is running'
    sys.exit(0)
```



