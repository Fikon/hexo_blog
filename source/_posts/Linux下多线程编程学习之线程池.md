---
title: Linux下多线程编程学习之线程池
date: 2016-08-24 03:03:46
tags: 多线程编程
categories: [notes, pthread]
---
继续多线程填坑之路，这次要填的坑是线程池技术。在说线程池之前不得不提一下池化资源的思想，池化资源的核心就是将可重复利用的资源统一管理，避免频繁的创建和销毁这些资源，以提高程序效率（Java开发中经常用到的ORM框架，Mybatis就遵循了这一思想，建立了数据链接池，避免每次操作都进行链接和断开操作）。在面向对象编程中，对象的创建和销毁是非常费时间的（为其申请内存，释放内存等），若能够减少对象创建和销毁的次数，则程序的效率将会有不错的提升。说回线程池，线程池预先申请了一定数量的线程，由线程池来统一调度管理，以一定的策略来执行即将到来的或者是任务队列中的任务，从而达到重复利用线程对象，减少线程创建和销毁的次数以提高程序效率。

以服务器程序为例来说明一下线程池如何提高效率的。对于一个服务器程序来说应用多线程的目的是为了充分利用CPU的闲置时间，提高其吞吐能力，但是使用不当的话却有可能产生我们并不想要的结果。一般来说，采用多线程的服务器程序完成某个任务的时间(T) = 线程创建时间(T1) + 执行任务时间(T2) + 销毁线程时间(T3)。理想情况是T1和T3远远小于T2或者要完成的任务数量不多。但现实不总是如此，当T2与T1加T3相差不多的情况下，服务器则会有很大一部分的时间不是用于执行任务；另外，当要完成的任务数量巨大时（服务器程序的日常，经常要处理大量的用户请求），即使T1和T3远小于T2,其累计的时间也是非常大的，仍然造成了很大的浪费。若是能够将T1与T3放在服务器程序启动，结束时段或者是服务器程序的空闲时段，则服务器大部分的时间可以地执行任务了。根据线程池的特性，其可以完美的实现这个需求。它首先估算一下要处理任务的量级预先申请了一定数量的线程，然后通过一定的调度策略重复使用他们来执行任务，最终程序结束的时候再将他们销毁，释放资源。

### 线程池的构成
一般来说，一个最简单的线程池也至少包括线程池管理（包括创建，销毁以及调度策略），线程，任务队列（存储未执行的任务）和任务接口（定义任务要执行的内容，最简单的就是一个函数指针加参数）。

<!-- more -->

### 一个简单的示例代码
直接上代码是有点粗暴，但是感觉结合代码说才更加容易理解，^_-（新偷学来的表情）。先说一下大体上的思路吧，首先申请了一定数量的线程，然后每个线程都重复如下动作：若任务队列中有任务，则取出队列中第一个任务，执行；若任务队列中没有任务，则等待（这里采用互斥锁和条件变量来实现互斥访问和同步）。

#### threadpool.h
```
/*
** Author: kiky.jiang@gmail.com
** desc:  My first attempt to complete threadpool by c
**
*/

#ifndef THREADPOOL_H
#define THREADPOOL_H

#include <pthread.h>
#include <semaphore.h>
#include <stdlib.h>
#include <stdio.h>

typedef void *(*fun_t)(void *);   //定义线程执行函数的类型,在pthread_create的时候传入的类型

/*----------------------------------------------------
  线程执行的任务，包括函数以及参数
----------------------------------------------------*/
typedef struct task{
  fun_t exec;              //完成任务要执行的函数
  void *arg;               //参数
  struct task *next;       //任务队列是用链表实现的，所以需要指向下一个节点
}task_t;

/*---------------------------------------------------
  线程池实例，包括线程链表，任务链表等
---------------------------------------------------*/
typedef struct threadpool{
  pthread_t *threads;       //线程数组起始位置
  task_t *task_head;        //任务链表表头
  task_t *task_tail;        //任务链表尾巴
  pthread_mutex_t mutex;    //访问任务数组的互斥锁
  pthread_cond_t cond;      //任务数组中有任务了
  int thread_num;          //线程池中线程的数量
  int task_count;           //当前任务的数量
}threadpool_t;


/*-----------------------------------------------
  判断并且提示出错信息
----------------------------------------------*/
static void error(char *info){
  printf("error happend when deal with %s\n", info);
}

/*--------------------------------------------------
  创建线程池，传入一个threads参数表示线程池中要创建的线程数
  ，创建成功则返回0，并将线程池实例赋予thread，否则返回-1
--------------------------------------------------*/
int threadpool_init(threadpool_t *pool, int thread_num);


/*------------------------------------------------
  销毁线程池，释放申请的内存，互斥锁，信号量等资源,成功返回0,
  否则返回-1
------------------------------------------------*/
int threadpool_destroy(threadpool_t *pool);


/*------------------------------------------------
  在任务队列中增加一个任务，成功返回0,否则返回-1
------------------------------------------------*/
int threadpool_add_task(threadpool_t *pool, task_t *task);


/*------------------------------------------------
  线程池中线程创建时传入的函数，若任务链表不为空，则取出任务
  队列中最前面的任务来执行，否则等待
------------------------------------------------*/
void * threadpool_fun(void *arg);

#endif

```

#### threadpool.c
```
/*
** Author: kiky.jiang@gmail.complete
** desc: My first attempt to complete threadpool by C
**
*/

#include "threadpool.h"

/* 初始化函数，创建线程，互斥锁，条件变量等    */
int threadpool_init(threadpool_t *pool, int thread_num){
  int res;
  int i;
  //给线程池指针分配内存
  pool = (threadpool_t *)malloc(sizeof(threadpool_t *));
  if(pool == NULL){
    error("malloc for pool");
    return -1;
  }


  //为线程数组分配内存
  pool -> threads = (pthread_t *)malloc(sizeof(pthread_t *) * thread_num);
  if(pool -> threads == NULL){
    error("malloc for threads");
    return -1;
  }

  pool -> task_head = NULL;
  pool -> task_tail = NULL;
  pool -> task_count = 0;

  //初始化互斥锁和条件变量
  res = pthread_mutex_init(&(pool -> mutex), NULL);
  if(res != 0){
    error("mutex init");
    return -1;
  }
  res = pthread_cond_init(&(pool -> cond), NULL);
  if(res != 0){
    error("cond init");
    return -1;
  }

  for(i = 0; i < thread_num; i++){
    res = pthread_create(&(pool -> threads[i]), NULL, threadpool_fun, (void *)pool);
    if(res != 0){
      error("create threads");
      return -1;
    }
  }
  return 0;
}
/*  线程池销毁函数，释放申请的资源   */
int threadpool_destroy(threadpool_t *pool){
  int res;
  int i;
  //唤醒所有的线程
  res = pthread_mutex_lock(&(pool -> mutex));
  if(res != 0){
    error("lock mutex for destroy pool");
    return -1;
  }
  res = pthread_cond_broadcast(&(pool -> cond));
  if(res != 0){
    error("broadcast");
    return -1;
  }
  res = pthread_mutex_unlock(&(pool -> mutex));
  if(res != 0){
    error("unlock mutex for destrory pool");
    return -1;
  }
  //回收线程资源
  for(i = 0; i < pool -> thread_num; i++){
    res = pthread_join(&(pool -> threads[i]), NULL);
    if(res != 0){
      error("thread join");
      return -1;
    }
  }
  //销毁互斥锁和条件变量
  res = pthread_mutex_destroy(&(pool -> mutex));
  if(res != 0){
    error("destroy mutex");
    return -1;
  }
  res = pthread_cond_destroy(&(pool -> cond));
  if(res != 0){
    error("destroy cond");
    return -1;
  }
  return 0;
}

/*  往任务队列中添加任务     */
int threadpool_add_task(threadpool_t *pool, task_t *task){

  int res;
  res = pthread_mutex_lock(&(pool -> mutex));
  if(res != 0){
    error("lock mutex for add task");
    return -1;
  }
  if(pool -> task_tail == NULL){
    pool -> task_head = task;
    pool -> task_tail = task;
  }
  else{
    pool -> task_tail -> next = task;
    pool -> task_tail = pool -> task_tail -> next;
  }
  pool -> task_count++;

  res = pthread_cond_signal(&(pool -> cond));
  if(res != 0){
    error("signal");
    return -1;
  }

  res = pthread_mutex_unlock(&(pool -> mutex));
  if(res != 0){
    error("unlock mutex for add task");
    return -1;
  }
  return 0;
}

/*  从任务队列中取出一个任务，执行   */
void *threadpool_fun(void *arg){
  int res;
  task_t *task;
  threadpool_t *pool = (threadpool_t *)arg;
  while(1){
    res = pthread_mutex_lock(&(pool -> mutex));
    if(res != 0){
      error("lock mutex for thread function");
      continue;
    }
    while(pool -> task_count == 0){
      res = pthread_cond_wait(&(pool -> cond), &(pool -> mutex));
      if(res != 0){
        error("cond wait");
        continue;
      }
    }
    task = pool -> task_head;
    pool -> task_head = pool -> task_head -> next;
    pool -> task_count--;
    pthread_mutex_unlock(&(pool -> mutex));
    *(task -> exec)(task -> arg);
  }
  pthread_exit(NULL);
}

```
### 优化的方向
线程池中的线程数量是有限的，却有可能需要处理大量的任务。此时，必然会有部分的任务需要等待，对于一些比较关注响应时间的服务器程序来说，这显然是不能接受的，所以前面介绍的简单线程池显然是不能满足需求了，必须进行优化。主要有三种思路的优化方案：
- *思路一：动态改变线程的数量。当要处理的任务数增加时，适当增加线程的数量；当任务数减少时，逐步减少线程数到一个合理的范围。当然，必须给线程的数量设置上限以及下限，不然线程池也需要频繁创建和销毁线程了，这就违背了我们使用线程池的初衷了*
- *思路二：对将要处理的任务数进行预估，从而申请合理数量的线程。这个代码实现是比较简单的，不过事先估计线程的数量难度比较大，需要平衡用户的耐心以及服务器的承受能力，通常采用经验值*
- *思路三：采用多个线程池，这个跟思路一有点相似，也是需要动态增加线程数量，个人觉得其实就是第一种思路中批采取批量增加线程数量的方法*
