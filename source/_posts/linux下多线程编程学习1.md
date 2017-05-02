---
title: Linux下多线程编程学习1
date: 2016-08-17 04:40:18
tags: 多线程编程
categories: [notes, pthread]
---

*在共享内存多处理架构下，线程可以用来实现并行。早期，很多硬件厂商都设计了自己的一套线程标准，以方便程序员进行并行程序开发。在Unix或Unix-like系统中，IEEE POSIX 1003.1c标准定义了一套C语言的线程编程接口，即 POSIX threads或者叫Pthreads。接下来的学习就是借助pthread库进行学习的。*

关于线程的定义，网上有很多种说法，比较普遍的是：一种轻量级的进程(light process)，是可以被操作系统调度的一个独立的指令序列。既然提到是轻量级进程，那就必须得对比一波。阮一峰有一篇博客用了一个比喻来形容线程和进程之间的关系，非常形象。他把计算机比作是一座工厂，工程里面有很多的车间（进程），但是由于电力（CPU）吃紧，每次只能一个车间开工（这是对于单个CPU的计算机来说的，任意时刻只能有一个进程在运行，其他都处于非运行状态）。而线程则是车间里面的工人，要么负责不同的工作，要么协同完成同一个工作。车间的公共资源是所有工人共享的（地址空间，文件描述符等），而对于工人来说，每个人又有自己的独立的工具箱（程序计数器，栈等）。这个比喻已经很形象地说明了进程，线程和CPU之间的关系了。那么从更加严肃的层面上他们有什么区别和联系呢:
- 进程是可以独立运行的一段程序，是操作系统进行资源分配和调度的一个基本单位；线程是进程的各异实体，基于进程而存在的，是CPU调度和分配的基本单位，其本身不占系统资源（暂用计数器和栈等）
- 一个线程属于某个进程，而一个进程可以拥有多个线程（至少拥有一个主线程）；进程是系统资源分配的单位，进程中的所有线程共享系统资源；线程是CPU分配的单位，CPU上运行的实际是线程；

<!-- more -->

聊完线程与进程的关系，接下来就直接进入pthread库的学习了,在linux环境下使用pthread库直接包含pthread.h这个库就好了，在编译的时候加上-lphtread编译参数。

### 创建线程
用到的函数是`int pthread_create(pthread_t *thread, const pthread_attr_t *attr,void *(*start_routine) (void *), void *arg)`，返回0表示创建成功，其他表示出错。原型有点长，但是这个函数非常简单：thread参数表示指向线程标识符的指针；attr用来设置线程属性，可以指定线程属性对象，一般使用默认值NULL；start_routine是线程执行任务的函数起始地址（该函数的原型为`void *func(void *arg)`；arg表示传入start_routine函数的参数，传入是将参数转换为void *类型，所以理论上说可以给它传入任何值，传入多个参数的话需要自定义一个结构将多个参数包含在里面。

### 终止线程
用到的函数是`void pthread_exit(void *retval);`当线程设置了joinable属性时，retval是传给其他调用pthread_join函数的返回值。一般使用NULL就好了。

### 一个简单的实例
创建4个线程，分别向标准输出不同的字符
```
#include <stdio.h>
#include <pthread.h>

void * hello(void *arg){
  int pid = *(int *)arg; //获取到参数的值
  printf("This is a message from thread %d\n", pid);
}

int main(){
  pthread_t threads[4];
  int pids[4];
  for(int i = 0; i < 4; i++){
    pids[i] = i;
    pthread_create(&threads[i], NULL, hello, (void *)&pids[i]);
  }
  pthread_exit(NULL); //等各个线程都结束了，进程再结束
}
```
上面就是一个多线程最简单的例子了，每个线程一创建就开始执行hello函数了，通过给他们传入不同的id，使得他们的输出不一样。下面再来个传入多个参数的例子(用一个自定义的结构来保存多个参数，然后传入该结构的地址)。
```
#include <stdio.h>
#include <pthread.h>
#include <string.h>


struct p_data{
  int pid;
  char msg[128];
};

void *hello(void *arg){
  struct *p_data = (struct p_data *)arg;
  printf("This is a message from thread %d, thread_info: %s\n", arg -> pid, arg -> msg);
}

int main(){
  pthread_t threads[4];
  struct p_data[4];
  for(int i = 0; i < 4; i++){
    p_data[i].pid = i;
    sprintf(p_data[i].msg, "I am thread %d", i);
    pthread_create(&thread[i], NULL, hello, (void *)&p_data[i]);
  }
  pthread_exit(NULL);
}
```
### joinable与detached
pthread有两种释放线程资源的方式pthread_join（默认方式）和pthread_detach，当设置为join方式时，线程可以被其他线程杀死和回收资源，若无其他线程对其调用pthread_join，是不会释放资源的（直到进程结束）；当设置为detach方式时，不能被其他线程杀死或释放资源，知道线程结束自动释放。根据pthread_detach的man手册：

*Either pthread_join(3) or pthread_detach() should be  called  for  each thread  that  an application creates, so that system resources for the thread can be released.(But note that the resources  of  all  threads are freed when the process terminates.)*

所以，在线程中既没有调用pthread_join也没调用pthread_detach的话，会导致内存泄露，当创建的线程越来越多时，内存就不够用了。一般的处理方法有三种：
- 如果是采用默认的方式的话，需要在主线程显式调用pthread_join，等待子线程结束
- 若是在创建线程时设置了PTHREAD_CREATE_DETACHED，则系统会自动回收资源
- 线程自己调用pthread_detach(pthread_self())，pthread_self()返回本线程的ID

介绍一下以上三种方式会用到的函数，首先是显式调用join函数：`int pthread_join(pthread_t thread, void **retval);`，调用者会一直等待，直到指定的线程结束，若retval不为NULL的话，其值等于等待的线程的退出状态，比如调用pthread_exit()时的retval。需要注意一点，如果有多个线程同时对某个线程调用pthread_join的话，结果是未定义的。举个这种方式的例子，改写一下上面的代码：

```
#include <stdio.h>
#include <pthread.h>
#include <string.h>


struct p_data{
  int pid;
  char msg[128];
};

void *hello(void *arg){
  struct *p_data = (struct p_data *)arg;
  printf("This is a message from thread %d, thread_info: %s\n", arg -> pid, arg -> msg);
}

int main(){
  pthread_t threads[4];
  struct p_data[4];
  for(int i = 0; i < 4; i++){
    p_data[i].pid = i;
    sprintf(p_data[i].msg, "I am thread %d", i);
    pthread_create(&thread[i], NULL, hello, (void *)&p_data[i]);
  }
  for(int i = 0; i < 4; i++){
    pthread_join(&threads[i], NULL);//不需要用到线程的退出状态的话，填上NULL即可
  }
  pthread_exit(NULL);
}
```

第二种是线程自己调用detach函数设置自动释放资源：

```
#include <stdio.h>
#include <pthread.h>
#include <string.h>


struct p_data{
  int pid;
  char msg[128];
};

void *hello(void *arg){
  pthread_detach(pthread_self());
  struct *p_data = (struct p_data *)arg;
  printf("This is a message from thread %d, thread_info: %s\n", arg -> pid, arg -> msg);
}

int main(){
  pthread_t threads[4];
  struct p_data[4];
  for(int i = 0; i < 4; i++){
    p_data[i].pid = i;
    sprintf(p_data[i].msg, "I am thread %d", i);
    pthread_create(&thread[i], NULL, hello, (void *)&p_data[i]);
  }
  pthread_exit(NULL);
}
```

最后一种方式是在创建线程的时候就设置PTHREAD_CREATE_DETACHED属性（初始化一个属性变量，然后在Pthread_create的时候传入）：

```
#include <stdio.h>
#include <pthread.h>
#include <string.h>


struct p_data{
  int pid;
  char msg[128];
};

void *hello(void *arg){
  struct *p_data = (struct p_data *)arg;
  printf("This is a message from thread %d, thread_info: %s\n", arg -> pid, arg -> msg);
}

int main(){
  pthread_t threads[4];
  pthread_attr_t attr;
  pthread_attr_init (&attr);
  pthread_attr_setdetachstate (&attr, PTHREAD_CREATE_DETACHED);
  struct p_data[4];
  for(int i = 0; i < 4; i++){
    p_data[i].pid = i;
    sprintf(p_data[i].msg, "I am thread %d", i);
    pthread_create(&thread[i], &attr, hello, (void *)&p_data[i]);
  }
  pthread_exit(NULL);
}
```
