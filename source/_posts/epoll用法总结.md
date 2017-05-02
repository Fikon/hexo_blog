---
title: C++网络编程之epoll
date: 2016-09-03 12:47:08
tags: 那些年的坑
categories: [学习心得, 网络编程]
---

epoll（）这是linux2.6才引进的一种复用IO的机制，算是对select()和poll的一个改进吧。相比select()和poll()，epoll()的高效是很明显的，
虽然poll解决了select（）打开文件数有限制的问题，但是依然存在两个明显的不足：
- 每次调用都会将FD集合从用户态拷贝到内核态，效率低，开销大
- 每次都会轮询集合中所有的FD以确定其是否就绪

epol的优点就是对前面两点的改进了，从名字就可以看出来了，efficient-poll嘛。针对以上两个问题，epoll是如是改进的：
- epoll采用共享内存的方式在内核态和用户态之间传递消息，避免了内存拷贝的开销
- epoll不会对集合中所有的FD都进行操作，他采取了了一种回调机制，只有那些就绪了的FD采用调用callback函数，因此节省了大量的轮询开销


<!-- more -->


### epoll函数介绍——三个函数与两种模式
linux男人（man）已经对epoll介绍的非常详细了, 还有一份参考代码(看了你会发现网上大部分的epoll教程代码都似曾相识)。epoll的使用非常简单，就只有三个函数，以及两种触发模式。
#### 三个函数
epoll的操作其实很简单就是三个函数：
- 创建：`int epoll_create(int size)` or `int epoll_create1(int flag)` size表示内核需要监听的FD个数，函数函数返回一个epoll FD。当epoll_create1(int flag)中flag为0时，两个函数是一样的，只是后者不用设置size了。
- 注册：`int epoll_ctl(int epfd, int op, int fd, epoll_event *event)`epfd就是在epoll_create中创建的epoll FD, op为要进行的操作，有三种：`epoll_ctl_ADD`添加fd到epfd中，并且将事件event关联到fd上；`epoll_ctl_MOD`修改已经注册到fd的event；`epoll_ctl_DEL`将fd从epfd中移除。event表示要监听的事件，
其结构如下：

```
typedef union epoll_data{
  void * ptr;
  int fd;
  uint32_t     u32;
  uint64_t     u64;
}epoll_data_t;

struct epoll_event {
uint32_t     events;      /* Epoll events */
epoll_data_t data;        /* User data variable */
};
/* event是一个枚举类型，可以使用"|"来增加类型，常用的类型包括：EPOLLIN表示读事件， EPOLLOUT表示写事件
，EPOLLET表示ET（边缘触发）工作模式，epoll默认是LT(水平触发) */
```
- 等待：`int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);`
  epfd跟前面注册函数的一样，events表示监听的事件集合，maxevents表示最大监听事件的数量，timeout表示超时时间

#### 两种模式
在前面介绍注册函数的时候有提到过在设置事件类型的时候可以选择工作模式，epoll有两种工作模式水平触发（LE，这是默认的方式）和边缘触发（ET）：
- ET：当有事件就绪时，epoll_wait就会得到通知，若未来得及一次性读/写完，则下一次调用epoll_wait时，其还会继续得到通知，若一直不处理完读写数据，则通知会一直持续
- LT： 当有事件就绪是，epoll_wait仅仅会通知一次，若有读写数据未处理完，则会等到下一次事件就绪才会得到通知

### 基于epoll的echo服务器程序
扯了那么多皮，还是得用个实际的例子才能更加理解各个函数是如何使用的，下面以一个简单的echo服务器为例。
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
#include <arpa/inet.h>
#include <sys/wait.h>
#include <sys/epoll.h>
#include <sys/fcntl.h>


#define PORT "4399"
#define BACKLOG 20 // 允许等待的最大连接数
#define EV_NUM 20   //最大监听事件数
#define EPOLL_SIZE 1024
#define MAXBUFFERSIZE 1024


// 出错打印
void error(const char * str){
  printf("Exception happened on %s\n", str);
  exit(1);
}

// 将FD设置为非阻塞模式
void set_nonblock(int fd){
  int flag;
  flag = fcntl(fd, F_GETFL, 0);
  if(flag == -1){
    error("fcntl get flags")
  }
  flag |= O_NONBLOCK;
  if(fcntl(fd, F_SETFL, flag) == -1){
    error("fcntl set flag");
  }
}

//初始化监听socket
int server_init(){

  struct addrinfo hints;
  struct addrinfo *ai;
  struct addrinfo *iter;
  int res;
  int reuse = 1;

  // 设置我们希望返回的地址的一些属性
  bzero(&hints, sizeof hints);
  hints.ai_family = AF_UNSPEC; //不指定，ipv4和ipv6都可以
  hints.ai_socktype = SOCK_STREAM;
  hints.ai_flags = AI_PASSIVE;

  if(getaddrinfo(NULL, PORT, &hints, &ai) != 0){
    error("get addrinfo");
  }
  for(iter = ai; iter != NULL; iter = iter -> ai_next){
    if((res = socket(iter -> ai_family, iter -> ai_socktype, iter -> ai_protocol)) == -1){
      continue;
    }
    // 将socket地址设置为可重用模式
    if(setsockopt(res, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(int)) == -1){
      continue;
    }
    if(bind(res, iter -> ai_addr, iter -> ai_addrlen) == -1){
      continue;
    }
    break;
  }
  freeaddrinfo(ai);
  if(iter == NULL){
    error("sock initilization");
  }
  return res;
}

int main(int argc, char *argv[]){

  int s_sock;
  int c_sock;
  int res;
  char buffer[MAXBUFFERSIZE];
  struct sockaddr_storage *client_addr;
  socklen_t caddr_len;

  struct epoll_event ev;
  struct epoll_event events[EV_NUM];
  int epfd;
  int nfds;

  s_sock = server_init();
  printf("Server listening on part %s\n", PORT);

  if(listen(s_sock, BACKLOG) == -1){
    error("listening");
  }

  epfd = epoll_create(EPOLL_SIZE);
  if(epfd == -1){
    error("create epoll");
  }
  set_nonblock(s_sock);
  ev.data.fd = s_sock;
  ev.events = EPOLLIN | EPOLLET;
  if(epoll_ctl(epfd, EPOLL_CTL_ADD, s_sock, &ev) == -1){
    error("register read event for server socket");
  }
  while(true){
    nfds = epoll_wait(epfd, events, EV_NUM, 500);
    if(nfds == -1){
      error("epoll wait");
    }
    for(int i = 0; i < nfds; i++){
      //若监听socket就绪，则接受新的连接
      if(events[i].data.fd == s_sock){
        c_sock = accept(s_sock, (struct sockaddr *)&client_addr, &caddr_len);
        if(c_sock == -1){
          continue;
        }
        ev.data.fd = c_sock;
        set_nonblock(c_sock);
        ev.events = EPOLLIN | EPOLLET;
        if(epoll_ctl(epfd, EPOLL_CTL_ADD, c_sock, &ev) == -1){
          continue;
        }
      }
      else if(events[i].events && EPOLLIN){ //处理读事件
        c_sock = events[i].data.fd;
        if(c_sock < 0){
          continue;
        }
        if((res = read(c_sock, buffer, MAXBUFFERSIZE)) < 0){
          continue;
        }
        else if(res == 0){ //客户端断开连接
          printf("the client close the connection\n");
          close(c_sock);
          epoll_ctl(epfd, EPOLL_CTL_DEL, c_sock, &events[i]);
          events[i].data.fd = -1;
        }
        else{
          printf("recieve string from client: %s\n", buffer);
          write(c_sock, buffer, sizeof buffer);
        }
      }
    }
  }
}
```
