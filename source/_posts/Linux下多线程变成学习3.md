---
title: Linux下多线程编程学习3
date: 2016-08-20 01:37:56
tags: 多线程编程
categories: [notes, pthread]
---
上一篇记录了如何用条件变量和互斥锁来实现生产者消费者问题，根据我仅存不多的一点记忆，当年上操作系统课程的时候，课本上是用的信号量（semaphore），所以打算研究研究信号量，搞一个信号量版的生产者消费者代码。

什么是信号量？一个比较普遍的理解是用于实现多线程同步的一种机制，信号量是非负整型变量，除了初始化之外，它只能通过两个标准原子操作：P和V（P来源于Dutch proberen"测试"，V来源于Dutch verhogen"增加"）。信号量与互斥锁有什么联系和区别？一个最重要的区别就是互斥锁是用于实现互斥的，而信号量则用于同步；通常同步都是在互斥的基础上的，实现了互斥之后，通过同步实现对资源的有序访问（一个线程完成了某个动作后告知其他线程，其他线程在执行某些操作，并不一定锁定资源，是一种流程上的概念，听起来有点像条件变量）。

在Linux下使用信号量需要用到一个库semaphore.h，下面是一些重要函数的介绍：
#### 创建信号量
主要用到`int sem_init (sem_t *sem, int pshared, unsigned int value);`这个函数的作用是对由sem指定的信号量进行初始化，设置好它的共享选项，并指定一个整数类型的初始值value。pshared参数控制着信号量的类型。如果pshared的值是０，就表示它是当前线程的局部信号量；否则，其它进程就能够共享这个信号量。不过Linux pthread并未实现多线程共享信号量，所以pshared的值为非零时会出错。销毁信号量使用`int sem_destroy(sem_t * sem)`，调用这个函数时，确保没有线程在等到sem了，不然会返回-1出错。
#### PV操作
P对应semaphore中的`int sem_wait(sem_t * sem)`或者`int sem_trywait(sem_t * sem)`即，尝试将信号量减1，这个跟条件变量里面的wait类似，前者是阻塞版后者是非阻塞版本；V对应的是`int sem_post(sem_t * sem)`信号量加1,表示资源个数增加一个。

<!-- more -->

#### 其他
读取信号量的值：`int sem_getvalue(sem_t * sem, int * sval)`，需要注意的是得到信号量的值是保存在sval中并不是函数的返回值，若读取成功则函数返回0,否则-1.


#### 一个用信号量和互斥锁实现的生产者消费者问题
```
#include <cstdlib>
#include <cstdio>
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
int buff_size = 0;
sem_t empty,full;
pthread_mutex_t mutex;


void *produce(void *arg){
  while(1){
    sem_wait(&empty);  //缓冲区未满
    pthread_mutex_lock(&mutex); //尝试加锁
    buff_size++;
    printf("Add one, current is %d\n", buff_size);
    pthread_mutex_unlock(&mutex);
    sem_post(&full);  //缓冲区数据加1,若此前是空的话，会给消费者发送一个信号，使其苏醒
    sleep(1);
  }
  pthread_exit(NULL);
}

void *consume(void *arg){
  while(1){
    sem_wait(&full);
    pthread_mutex_lock(&mutex);
    buff_size--;
    printf("Delete one, current is %d\n", buff_size);
    pthread_mutex_unlock(&mutex);
    sem_post(&empty);
    sleep(2);
  }
  pthread_exit(NULL);
}

int main(){
    pthread_t producer;
    pthread_t consumer;
    pthread_mutex_init(&mutex,NULL);
    //此时缓冲区为0, 设置可以装载10个数据
    sem_init(&full,0,0);
    sem_init(&empty,0,10);
    pthread_create(&producer,NULL,produce,NULL）；
    pthread_create(&consumer,NULL,consume,NULL);
    pthread_join(consumer,NULL);
    pthread_join(producer,NULL);
    pthread_exit(NULL);
    return 0;
}
```
