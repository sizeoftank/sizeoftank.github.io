---
author: chenwei
comments: true
date: 2012-06-10 01:42:13+00:00
layout: post
title: 进程间通信(IPC):共享内存
categories:
- Linux
---

共享内存就是允许两个或更多的进程可以访问一块共享的内存区域进行数据交换，是最快的IPC了。如果服务器端进程在共享内存区进行读写，就不允许客户端进程对共享内存进行访问了，这时候的IPC也就要对多个进程同步，使用信号量什么的来管理这一块共享区域了…

先来一个简单的应用：创建共享内存->映射共享内存 然后就可以进行读/写访问了…

头文件: sys/shm.h

**用来创建共享内存的函数：**

int shmget(key_t key,size_t size, int flag);

Returns: shared memory ID if OK, -1 on error.

如果成功则返回共享内存的ID

**用来映射共享内存的函数：**

void *shmat(int shmid, const void *addr, int flag);

Returns: pointer to shared memory segment if OK, -1 on error.

如果成功则返回一个指向共享内存区域的指针

**下面是一个简单的例程，通过命令行读取一个文件，执行fork后先由子进程读取文件中数据并写入共享内存中，子进程退出后，父进程从共享内存中读取数据输出到标准输出。**

```c
#include "apue.h"
#include <sys/shm.h>
#include <fcntl.h>
#define ARRAY_SIZE      40000
#define MALLOC_SIZE     40000
#define SHM_SIZE        100000
#define SHM_MODE        0600
#define BUFFSIZE        128
char array[ARRAY_SIZE];
int main(int argc,char *argv[])
{
        int shmid;
        char *ptr,*shmptr;
        pid_t pid;
        char writebuf[BUFFSIZE],readbuf[BUFFSIZE];
        int i,j,n;
        int fd;
        char *p,*q;
        if (argc!=2)
        {
                perror("usage: a.out filename");
                exit(1);
        }
        printf("array[] from %lx to %lxn", (unsigned long)&array[0],
                (unsigned long)&array[ARRAY_SIZE]);
        printf("statck around %lxn", (unsigned long) &shmid);  
        /* 分配堆空间 */
        if ((ptr = malloc(MALLOC_SIZE))==NULL)
                perror("malloc error");
        printf("malloced from %lx to %lxn", (unsigned long)ptr,
                (unsigned long)ptr+MALLOC_SIZE);
        /* 创建共享内存 */
        if ((shmid=shmget(IPC_PRIVATE,SHM_SIZE,SHM_MODE))<0)
                perror("shmget error");
        /* 映射共享内存 */
        if ((shmptr=shmat(shmid,0,0))==(void *)-1)
                perror("shmat error");
        printf("shared memory attached from %lx to %lxn",
                (unsigned long)shmptr, (unsigned long)shmptr+SHM_SIZE);
        if ((pid=fork())<0)
        {
                perror("fork error");
        }
        else if (pid==0)        /* 子进程 */
        {
                /* 向共享内存写数据后退出 */
                printf("子进程向共享内存写数据...n");
                if ((fd=open(argv[1],O_RDWR,0))==-1)
                {
                        printf("不能打开文件:%sn",argv[1]);
                        exit(1);
                }
                p = shmptr;
                while ((n=read(fd,readbuf,BUFFSIZE))>0)
                {
                        for (i = 0; i < n; i++)
                                *p++=readbuf[i];
                }
                *p=-1;
                printf("子进程写数据完毕n");
                exit(0);
        }
        else    /* 父进程 */
        {
                if (waitpid(pid,NULL,0)!=pid)   /* 等子进程退出 */
                        perror("waitpid error");
                /* 从共享内存中读数据 */
                printf("父进程读取数据...n");  
                p = shmptr;
                while (*p!=EOF) printf("%c",*p++);
        }
        if (shmctl(shmid,IPC_RMID,0)<0) /* 移除共享内存 */
                perror("shmctl error");
        exit(0);
}
```






 
