---
title: Linux下多线程编程学习2
date: 2016-08-19 10:11:06
tags: 多线程编程
categories: [notes, pthread]
---

上一篇文章介绍了pthread的一些基本的用法，现在要说一下多线程编程中的线程安全问题。线程安全指某个函数、函数库在多线程环境中被调用时，能够正确地处理多个线程之间的共享变量，使程序功能正确完成。pthread用于实现线程安全的机制主要有两个，互斥锁(Mutex Variables)和条件变量(Condition Variables)。

### 互斥锁（Mutex Variables）
互斥锁是一把用于保护共享数据资源的一把“锁”，确保任意时间点只有一个线程能够对其执行锁操作，以保证线程同步。一个比较通俗的比喻就是一群人（线程）排队用洗手间，而洗手间同一时间只能允许一个人使用，获得了洗手间的钥匙(Mutex)的人就可以使用洗手间了，当没用使用洗手间时，钥匙会根据一定的调度策略分配给某一个排队的人，当他用完之后，再以一定的策略传给下一个排队的人，如此循环。在pthread库中，使用互斥锁的大致流程为：
- 创建并初始化
- 多个线程尝试锁定Mutex
- 某一个线程成功获得Mutex
- 执行对共享数据资源的操作
- 取消对Mutex的锁定

#### 创建与销毁锁
创建Mutex有两种方式，一种是通过函数调用来初始化`int pthread_mutex_init(pthread_mutex_t *mutex,const pthread_mutexattr_t * attr);`其中attr可以为Mutex设置属性值，填NULL则采用默认值跟第二种创建方式等价，属性值调用`int pthread_mutexattr_init(pthread_mutexattr_t *attr);`进行初始化，`int pthread_mutexattr_destroy(pthread_mutexattr_t *attr);`进行销毁；另外一种是直接用一个宏定义初始化`pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;`，这种方法不能设置Mutex的属性，采用默认设置。销毁Mutex采用`int pthread_mutex_destroy(pthread_mutex_t *mutex);`。

<!-- more -->

#### Mutex操作
主要有三个：
- 加锁`int pthread_mutex_lock(pthread_mutex_t *mutex);`，这个函数是阻塞调用，若其他线程正占用锁的话，会一直等待
- 尝试加锁`int pthread_mutex_trylock(pthread_mutex_t *mutex);`，非阻塞调用，当锁被其他线程占用时，立即返回
- 释放锁`int pthread_mutex_unlock(pthread_mutex_t *mutex);`，释放之后，其他线程可以多Mutex进行加锁操作

#### 实例代码
```
#include <pthread.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

pthread_mutex_t mutex;
int data = 0;

void *modify(void *arg){
  int num = *(int *)arg;
  pthread_mutex_lock(&mutex);
  data = num;
  printf("current num is %d\n", num);
  pthread_mutex_unlock(&mutex);
  pthread_exit(NULL);
}

int main(){
  pthread_t threads[4];
  int nums[4];
  pthread_attr_t attr;  //线程属性设置
  //初始化Mutex
  pthread_mutex_init(&mutex);
  //设置属性为joinable，需要主线程显式调用pthread_join函数才能士气释放资源
  pthread_attr_init(&attr);
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);
  for(int i = 0; i < 4; i++){
    nums[i] = i;
    pthread_create(&threads[i], &attr, modify, (void *)&nums[i]);
  }
  pthread_attr_destroy(&attr);
  //调用join
  for(int i = 0; i < 4; i++){
    pthread_join(&threads[i], NULL);
  }
  pthread_mutex_destroy(&mutex);
  pthread_exit(NULL);
}
```
### 条件变量(Condition Variables)
条件变量是另外一种实现线程同步的方法，互斥锁是限制线程对数据的访问，而条件变量是要求数据达到一定的条件来实现线程同步的，两种机制常常搭配使用。他们搭配使用的流程大致如下所示(此处以两个线程竞争为例）：

主线程声明并且初始化全局变量（如缓冲区的大小，这个是需要同步的变量），条件变量和互斥变量。
- 线程1(读线程）仅在缓冲区大小不为零时工作，其首先锁定互斥变量并检查缓冲区的大小若缓冲区有数据则直接读取数据，否则调用pthread_cond_wait()来阻塞等待线程2(写线程)的信号（虽然是阻塞等带，当线程2需要对互斥变量加锁的时候线程1会自动的对互斥变量解锁）， 最后当收到线程2发来的信号后，线程1苏醒，然后会自动对互斥变量加锁，接着读取数据，若读取数据前缓冲区是满的，则读取后给线程2发信号，告知其缓冲区目前未满，释放锁
- 线程2(写线程)仅在缓冲区未满的时候工作，首先锁定互斥变量并且检查缓冲区大小，若未满则写入数据，否则调用pthread_cond_wait()来等待线程1将数据读取出去，收到线程1的信号后，线程2苏醒，然后加锁，写入数据，若写入数据之前缓冲区的空的，则在写入数据后告知线程1目前缓冲区有数据了，解锁

#### 创建与初始化
这个与互斥锁基本一样，有两种初始化方式，可以给条件变量设置属性等。可以通过使用宏定义直接赋值的方式`pthread_cond_t myconvar = PTHREAD_COND_INITIALIZER;`此时，其属性为默认值，也可以采用调用函数来初始化`int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr);`用这种方式时，可以通过设置attr来为条件变量设置属性；通过调用`int pthread_cond_destroy(pthread_cond_t *cond);`来销毁条件变量。

#### 条件变量操作
主要有两种操作，一种是等待信号，一种是发送信号，很简单。
- 等待信号，`int pthread_cond_wait(pthread_cond_t *cond,pthread_mutex_t *mutex);`，阻塞等待条件变量满足一定的条件，若在阻塞中有其他线程需要用到Mutex，会自动释放锁
- 发送信号，`int pthread_cond_signal(pthread_cond_t *cond);`，告知在等待该信号的线程，条件满足了，如果有多个线程在等待，则使用广播形式告知，`int pthread_cond_broadcast(pthread_cond_t *cond);`

#### 实例代码（生产者消费者问题）
生产者和消费者对同一个有限大小的缓冲区进行操作
```
#include <pthread.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>


#define MAXSIZE 10   //缓冲区大小

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;  //对缓冲区操作的锁
pthread_cond_t  empty = PTHREAD_COND_INITIALIZER;   //缓冲区是否未满
pthread_cond_t  full = PTHREAD_COND_INITIALIZER;    //缓冲区是否不为空
int buff_size = 0;

void *produce(void *arg){
  while(1){
    pthread_mutex_lock(&mutex);
    while(buff_size == MAXSIZE)    //若缓冲区满了，则等待
      pthread_cond_wait(&empty, &mutex);
    buff_size++;
    if(buff_size == 1)    //若写入之前缓冲区为空，则发送信号
      pthread_cond_signal(&full);
    pthread_mutex_unlock(&mutex);
    sleep(1);
  }
  pthread_exit(NULL);
}

void *consume(void *arg){
  while(1){
    pthread_mutex_lock(&mutex);
    while(buff_size == 0)    //缓冲区为空，等待
      pthread_cond_wait(&full, &mutex);
    buff_size--;
    if(buff_size == MAXSIZE - 1)  //读取前缓冲区是满的，发送信号
      pthread_cond_signal(&empty);
    pthread_mutex_unlock(&mutex);
    sleep(1);
  }
  pthread_exit(NULL);
}

int main(){
  pthread_t producer;
  pthread_t consumer;
  int tmp;
  tmp = pthread_create(&producer, NULL, produce, NULL);
  if(tmp)
    printf("error\n");
  tmp = pthread_create(&consumer, NULL, consume, NULL);
  if(tmp)
    printf("error\n");
  tmp = pthread_join(&producer, NULL);
  tmp = pthread_join(&consumer, NULL);
  pthread_exit(NULL);
  return 0;
}
```
